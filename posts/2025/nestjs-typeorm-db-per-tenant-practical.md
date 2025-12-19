---
title: NestJS + TypeORM 多库多租户（DB per tenant）代码落地：DataSource 管理、并发单飞与回收
tags: [Node.js, NestJS, TypeORM, MySQL, PostgreSQL, 多租户, 连接池, 架构, 实战]
categories: [后端]
date: 2025-12-19
description: 多库多租户（每个租户一个数据库）在 NestJS + TypeORM v0.3+ 下的可落地实现：tenant 解析、TenantDataSourceManager（并发单飞锁）、REQUEST scope provider 注入、Repository/EntityManager 使用方式、连接池容量规划与回收策略。
articleGPT: 这篇文章给出一个可直接照抄的 NestJS 多库多租户实现骨架：通过中间件解析 tenantId，使用 TenantRegistry 存储租户数据库连接信息，用 TenantDataSourceManager 缓存/复用 DataSource 并用 in-flight promise 防止并发重复初始化；再用 REQUEST scope provider 将当前租户的 DataSource 或 EntityManager 注入业务层。最后补充连接数容量公式、LRU/TTL 回收、迁移策略与常见坑。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/nestjs.png
---

## 目标与前提

目标：在 NestJS + TypeORM（v0.3+）项目中实现 **DB per tenant**：

- 每个租户一个独立数据库（或独立 schema）
- 同一套服务处理所有租户请求
- 请求进来后按 `tenantId` 选择正确的数据库连接池（DataSource）
- DataSource **缓存复用**（不能每次请求都 new）
- 并发场景下避免同一租户重复 `initialize()`（否则连接池会爆炸）
- 可选：对“长时间不活跃租户”的 DataSource 做回收

本文假设你已经能跑起来一个普通 NestJS + TypeORM 项目。

---

## 总体架构（建议照这个分层）

```txt
请求 -> TenantMiddleware（解析 tenantId）
     -> TenantRegistryService（从“租户元数据表”拿 DB 配置）
     -> TenantDataSourceManager（按 tenantId 缓存 DataSource，单飞锁初始化）
     -> REQUEST scope Provider（把本次请求的 tenant DataSource / EntityManager 注入）
     -> 业务 Service（像正常 TypeORM 一样用 Repository/EntityManager）
```

关键原则：

- **REQUEST scope 只负责“取哪个租户”**，不要在请求级创建连接池
- **DataSource 是跨请求复用的**，应该由单例 manager 缓存管理

---

## Step 1：租户从哪里来？（建议：JWT / 网关注入 / 子域名）

为了聚焦实现，这里用 `x-tenant-id` 演示。

注意：

- 对公网服务来说，`x-tenant-id` **不可信**，必须由网关校验注入，或从 JWT 解析。
- 你真正落地时，只需要把本文的 `TenantMiddleware` 里解析 tenant 的逻辑替换掉即可。

### TenantMiddleware

```ts
import { BadRequestException, Injectable, NestMiddleware } from "@nestjs/common";
import type { NextFunction, Request, Response } from "express";

declare module "express-serve-static-core" {
  interface Request {
    tenantId?: string;
  }
}

@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request, _res: Response, next: NextFunction) {
    const tenantId = req.header("x-tenant-id");
    if (!tenantId) throw new BadRequestException("Missing x-tenant-id");

    req.tenantId = tenantId;
    next();
  }
}
```

在某个模块（比如 `AppModule`）里应用：

```ts
import { MiddlewareConsumer, Module, NestModule } from "@nestjs/common";

@Module({
  // imports/providers ...
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(TenantMiddleware).forRoutes("*");
  }
}
```

---

## Step 2：租户注册表（Tenant Registry）

你需要一个“主库”（或配置中心）存放租户的 DB 信息。

一种常见做法：

- 固定连接一个 `registry_db`
- 里面有一张 `tenants` 表：`tenant_id, db_host, db_port, db_user, db_pass, db_name, status...`

### 定义返回结构

```ts
export type TenantDbConfig = {
  tenantId: string;
  type: "mysql" | "postgres";
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
};
```

### TenantRegistryService（示意）

这里不强行绑定你一定要用 TypeORM 查 registry（你也可以用 Prisma、SQL、甚至配置文件）。核心是：

- 输入 tenantId
- 输出该租户的 DB 连接配置

```ts
import { Injectable, NotFoundException } from "@nestjs/common";

@Injectable()
export class TenantRegistryService {
  async getTenantDbConfig(tenantId: string): Promise<TenantDbConfig> {
    // 示例：你这里可以查 registry_db 的 tenants 表
    // 这里用伪数据占位（真实项目别写死）
    if (tenantId === "demo") {
      return {
        tenantId,
        type: "mysql",
        host: process.env.DEMO_DB_HOST ?? "127.0.0.1",
        port: Number(process.env.DEMO_DB_PORT ?? 3306),
        username: process.env.DEMO_DB_USER ?? "root",
        password: process.env.DEMO_DB_PASS ?? "root",
        database: process.env.DEMO_DB_NAME ?? "tenant_demo",
      };
    }

    throw new NotFoundException(`Unknown tenant: ${tenantId}`);
  }
}
```

建议：Registry 查询结果也可以加缓存（比如 Redis），避免每次请求都查 registry。

---

## Step 3：TenantDataSourceManager（核心：缓存 + 并发单飞锁）

这里是 DB per tenant 的关键。

### 你必须解决的两个问题

- **重复初始化**：同一租户并发请求进来，可能同时走到 `initialize()`。
- **无限增长**：租户非常多时，DataSource 缓存会越堆越多。

下面给出一个可落地骨架：

- `dataSources: Map<string, DataSource>` 缓存已初始化的数据源
- `inFlight: Map<string, Promise<DataSource>>` 作为“单飞锁”
- `lastUsedAt: Map<string, number>` 记录使用时间，便于回收（可选）

> 这里的回收策略写成“可选能力”，你也可以先不做，等租户数大了再加。

```ts
import { Injectable, Logger } from "@nestjs/common";
import { DataSource, type DataSourceOptions } from "typeorm";

@Injectable()
export class TenantDataSourceManager {
  private readonly logger = new Logger(TenantDataSourceManager.name);

  private readonly dataSources = new Map<string, DataSource>();
  private readonly inFlight = new Map<string, Promise<DataSource>>();
  private readonly lastUsedAt = new Map<string, number>();

  constructor(private readonly registry: TenantRegistryService) {}

  async getDataSource(tenantId: string): Promise<DataSource> {
    const existed = this.dataSources.get(tenantId);
    if (existed?.isInitialized) {
      this.lastUsedAt.set(tenantId, Date.now());
      return existed;
    }

    const inflight = this.inFlight.get(tenantId);
    if (inflight) return inflight;

    const creating = this.createAndInitDataSource(tenantId)
      .then((ds) => {
        this.dataSources.set(tenantId, ds);
        this.lastUsedAt.set(tenantId, Date.now());
        return ds;
      })
      .catch((err) => {
        // 初始化失败要清理 inFlight，避免后续永远卡死
        this.logger.error(`Init datasource failed for tenant=${tenantId}`, err);
        this.dataSources.delete(tenantId);
        throw err;
      })
      .finally(() => {
        this.inFlight.delete(tenantId);
      });

    this.inFlight.set(tenantId, creating);
    return creating;
  }

  private async createAndInitDataSource(tenantId: string): Promise<DataSource> {
    const cfg = await this.registry.getTenantDbConfig(tenantId);

    const base: DataSourceOptions = {
      type: cfg.type,
      host: cfg.host,
      port: cfg.port,
      username: cfg.username,
      password: cfg.password,
      database: cfg.database,

      // 建议：实体统一一套（所有租户结构一致）
      // 你可以用 autoLoadEntities + forFeature 的方式，或手动 entities 列表
      // 这里写成占位：
      entities: [__dirname + "/../**/*.entity{.ts,.js}"],

      synchronize: false,
      logging: false,

      // 连接池：透传给驱动
      extra:
        cfg.type === "mysql"
          ? { connectionLimit: 10 }
          : { max: 10, idleTimeoutMillis: 30_000, connectionTimeoutMillis: 2_000 },
    };

    const ds = new DataSource(base);
    await ds.initialize();
    return ds;
  }

  // 可选：回收不活跃租户 DataSource（需要你定时触发）
  async cleanupIdle(maxIdleMs: number) {
    const now = Date.now();

    for (const [tenantId, ds] of this.dataSources.entries()) {
      const last = this.lastUsedAt.get(tenantId) ?? 0;
      if (now - last < maxIdleMs) continue;
      if (!ds.isInitialized) continue;

      try {
        await ds.destroy();
      } catch (e) {
        this.logger.warn(`Destroy datasource failed tenant=${tenantId}: ${String(e)}`);
      } finally {
        this.dataSources.delete(tenantId);
        this.lastUsedAt.delete(tenantId);
      }
    }
  }
}
```

### 连接数容量规划（务必算一遍）

DB per tenant 最容易出事故的是“连接数爆炸”。

粗略公式：

```txt
总连接数 ≈ 服务实例数 × 同时活跃租户数 × 每租户 pool 上限
```

例子：

- 5 个服务实例
- 同时活跃 50 个租户
- 每租户 `connectionLimit = 10`

那就是 `5 * 50 * 10 = 2500` 个连接，很容易超过 DB 端上限。

因此你通常需要：

- 控制“活跃租户 DataSource 缓存数”（LRU/TTL 回收）
- 降低每租户 pool 上限（并优化 SQL）
- 大租户独立服务/独立集群

---

## Step 4：REQUEST scope Provider（把 tenant DataSource / EntityManager 注入到业务层）

推荐注入 **EntityManager**，比到处 `getRepository()` 更好收口。

### Provider：TENANT_ENTITY_MANAGER

```ts
import { Scope } from "@nestjs/common";
import { REQUEST } from "@nestjs/core";
import type { Request } from "express";

export const TENANT_ENTITY_MANAGER = "TENANT_ENTITY_MANAGER";

export const TenantEntityManagerProvider = {
  provide: TENANT_ENTITY_MANAGER,
  scope: Scope.REQUEST,
  inject: [REQUEST, TenantDataSourceManager],
  useFactory: async (req: Request, mgr: TenantDataSourceManager) => {
    const tenantId = req.tenantId;
    if (!tenantId) throw new Error("tenantId missing on request");

    const ds = await mgr.getDataSource(tenantId);
    return ds.manager;
  },
};
```

在你的 `TenantModule` 中注册：

```ts
import { Module } from "@nestjs/common";

@Module({
  providers: [TenantRegistryService, TenantDataSourceManager, TenantEntityManagerProvider],
  exports: [TenantDataSourceManager, TenantEntityManagerProvider],
})
export class TenantModule {}
```

---

## Step 5：业务层怎么用（像正常 TypeORM 一样写）

### 示例：用 EntityManager 拿 Repository

```ts
import { Inject, Injectable } from "@nestjs/common";
import type { EntityManager } from "typeorm";

@Injectable()
export class OrdersService {
  constructor(@Inject(TENANT_ENTITY_MANAGER) private readonly em: EntityManager) {}

  async list() {
    const repo = this.em.getRepository(OrderEntity);
    return repo.find({ order: { id: "DESC" } });
  }
}
```

### 事务怎么做？

使用 `this.em.transaction()`（事务自然发生在当前租户 DB 上）：

```ts
async createOrder(dto: any) {
  return this.em.transaction(async (tx) => {
    const repo = tx.getRepository(OrderEntity);
    const entity = repo.create(dto);
    return repo.save(entity);
  });
}
```

---

## Step 6：迁移（migrations）在 DB per tenant 的策略

常见策略：

- **新租户接入时跑迁移**（推荐）
  - 创建 DB -> `runMigrations()` -> 标记租户 ready
- **发版时批处理跑迁移**
  - 遍历租户逐个跑（要做可重试、可观测、失败告警）

TypeORM v0.3+ 可以用：

- `ds.runMigrations()`

但你需要考虑：

- 并发迁移会不会锁表影响业务（一般建议串行或限流）
- 租户非常多时，迁移时间会很长（需要任务队列）

---

## 常见坑与排障建议

### 1）同租户并发初始化导致连接池爆炸

表现：

- 连接数突然飙升
- DB 报 `Too many connections`

解决：

- 文章里的 `inFlight: Map<string, Promise<DataSource>>` 单飞锁是必需品

### 2）DataSource 缓存无限增长

租户多且长尾时，Map 会越来越大。

解决：

- 引入 LRU/TTL 回收（文中 `cleanupIdle`）
- 限制最大缓存数量（超出就回收最久未使用）

### 3）实体加载方式不当

如果你用 `autoLoadEntities` + 动态 DataSource，可能会出现“某些实体没被加载”。

建议：

- 在 `createAndInitDataSource` 中显式指定 `entities`（如本文示意）
- 或者统一导出实体数组（`entities: [...entities]`）

### 4）tenantId 来源不可信

如果 tenantId 由客户端随便传，等于“让用户选择数据库”。

建议：

- tenantId 从 JWT 解析（payload 里带 tenant）
- 或由网关鉴权后注入 header

---

## 小结

DB per tenant 在 NestJS + TypeORM 里能做，但必须把这几件事做对：

- **DataSource 缓存复用**：不能每请求创建连接池
- **并发单飞锁**：避免重复 initialize
- **连接数容量规划**：实例数 × 活跃租户 × pool 上限
- **回收策略（可选但强烈建议）**：LRU/TTL 回收长尾租户 DataSource
- **迁移策略**：新租户创建时迁移 or 批处理迁移

如果你补充一下你的实际环境：

- 你用的是 **MySQL 还是 Postgres**？
- 预计租户数大概多少？同时活跃租户多少？
- 你现在是从 **JWT** 还是 **网关 header** 获取 tenantId？

我可以把本文的“示意代码”进一步收敛成更贴近你项目的目录结构（module/service/provider），并给出更具体的 pool 参数建议。
