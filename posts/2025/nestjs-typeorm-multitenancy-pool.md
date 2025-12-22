---
title: NestJS + TypeORM：连接池配置与多租户实践（可落地方案）
tags: [Node.js, NestJS, TypeORM, MySQL, PostgreSQL, 连接池, 多租户, 架构]
categories: [后端]
comment: true
date: 2025-12-19
description: 介绍在 NestJS 中使用 TypeORM 时如何正确配置连接池，并给出多租户（单库多租户 / 多库多租户）两种常见实现方案：租户解析、DataSource 管理、Provider 注入、迁移策略与常见坑。
articleGPT: 这篇文章从 TypeORM 的连接池参数与并发模型讲起，先说明“多连接 vs 连接池”的区别，然后重点讲多租户的两条主线：共享数据库（tenant_id 隔离）与按租户独立数据库（DataSource per tenant）。文章给出 NestJS 中解析租户（中间件/守卫/请求上下文）、按租户复用 DataSource、连接池与回收、迁移与索引、以及穿透与安全边界等最佳实践。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/typeorm.png
---

## 先说清楚：你要的“多线程池”是什么？

很多同学说的“多线程池”，在 Node.js 服务里通常指的是：

- **数据库连接池（connection pool）**：同一时间复用多条 DB 连接，提升并发。
- **多 DataSource / 多连接池**：比如读写分离、分库分表、或多租户（每个租户一个库）。

TypeORM 在 Node.js 中跑在单线程事件循环上，但数据库 I/O 是异步的；真正决定并发能力的，往往是：

- DB 端能扛多少连接
- 连接池 `max`（或类似参数）配置是否合理
- 查询是否有索引、是否有 N+1、是否有慢 SQL

所以本文会把重点放在：

- **如何在 NestJS + TypeORM 正确配置连接池**
- **如何实现多租户**（并保证隔离与可维护性）

---

## Part 1：TypeORM 连接池怎么配？

### 1）TypeORM 的池来自哪里？

TypeORM 本身不实现连接池，它依赖驱动：

- MySQL/MariaDB：通常由 `mysql2` 提供 pool
- PostgreSQL：由 `pg` 提供 pool

在 TypeORM v0.3+ 里你一般通过 `DataSourceOptions.extra`（或对应驱动参数）把 pool 配置透传进去。

### 2）示例：MySQL（mysql2）连接池参数

```ts
import { TypeOrmModule } from "@nestjs/typeorm";

TypeOrmModule.forRoot({
  type: "mysql",
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT ?? 3306),
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  autoLoadEntities: true,
  synchronize: false,

  // 关键：透传给 mysql2 的 pool 配置
  extra: {
    connectionLimit: 20, // 池内最大连接数（核心参数）
    // queueLimit: 0,     // 请求排队上限（0=无限）
    // waitForConnections: true,
  },
});
```

建议：

- `connectionLimit` 不要乱拉高：DB 连接不是越多越好。
- 先用压测/监控观察：QPS、平均耗时、99 线、DB CPU、慢查询。

### 3）示例：PostgreSQL（pg）连接池参数

```ts
TypeOrmModule.forRoot({
  type: "postgres",
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT ?? 5432),
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  autoLoadEntities: true,
  synchronize: false,

  extra: {
    max: 20, // pg pool 最大连接数
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 2_000,
  },
});
```

### 4）连接池应该配多大？

一个简单的经验：

- **从小开始**（例如 10~30），观察瓶颈再调
- 结合 DB 的 `max_connections` / 实例规格
- 对于高并发读：优先优化索引和 SQL，其次才是加大连接池

过大的连接池常见副作用：

- DB CPU 飙升，锁竞争更严重
- 平均响应时间反而变差（抖动更大）
- 在多实例部署时，连接数会按实例倍增（容易打满 DB）

---

## Part 2：多租户怎么做？两条路线先选清楚

多租户（Multi-tenancy）的本质是：**同一套服务，服务多个“租户”**，并确保数据隔离。

最常见两种模式：

### 模式 A：单库多租户（Shared Database, Shared Schema）

- 所有租户共享一个数据库/表结构
- 每张业务表都有 `tenant_id`

优点：

- 成本低、运维简单
- 新租户接入快（无需建库/迁移）

缺点：

- 隔离弱：需要非常严格的代码约束，避免越权
- 单库膨胀快，热点租户会影响其他租户

适合：租户数量多、小租户为主、隔离要求中等。

### 模式 B：多库多租户（Database per Tenant）

- 每个租户一个独立数据库（或一个独立 schema）
- 服务按请求中的租户信息选择对应 DB 连接

优点：

- 隔离强（天然隔离）
- 大租户可独立扩容与迁移

缺点：

- 运维成本高：建库、迁移、备份、监控都要按租户维度做
- 连接数更容易爆：多个 DataSource + 每个都有 pool

适合：租户数量不多、付费大客户、强隔离要求。

> 下面我会分别给出两套实现思路。你可以先选一个模式落地。

---

## Part 3：单库多租户（tenant_id 隔离）落地方案

### 1）关键点

- **租户识别**：从 `Header`/子域名/JWT 中解析 `tenantId`
- **把 tenantId 注入请求上下文**：后续 service 能拿到
- **所有查询强制带 tenantId 条件**：这是安全边界

### 2）租户从哪里来？（推荐顺序）

- 从 JWT `payload`（登录时确定 tenant，后续可信）
- 从 Header（例如 `x-tenant-id`，适合内部系统，但要防伪造）
- 从子域名（例如 `tenantA.api.xxx.com`）

### 3）在 Nest 中保存 tenantId（请求级上下文）

你可以用中间件把 `tenantId` 挂到 `req` 上：

```ts
import { Injectable, NestMiddleware, BadRequestException } from "@nestjs/common";
import type { Request, Response, NextFunction } from "express";

@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request & { tenantId?: string }, _res: Response, next: NextFunction) {
    const tenantId = req.header("x-tenant-id");
    if (!tenantId) throw new BadRequestException("Missing x-tenant-id");

    req.tenantId = tenantId;
    next();
  }
}
```

### 4）查询时强制带 tenantId

最朴素的做法：在 service 里每个 repository 查询都带上 `tenantId`。

更推荐：

- 封装一个 `TenantRepository`
- 或者在 QueryBuilder 的基础上统一追加 `tenant_id`

并且要做到：

- **写入时自动填 tenant_id**
- **更新/删除必须带 tenant_id 条件**（否则就是越权漏洞）

### 5）表结构与索引

单库多租户一定要注意索引：

- 绝大多数查询都需要 `tenant_id` 参与过滤
- 常见组合索引：`(tenant_id, id)`、`(tenant_id, created_at)`、`(tenant_id, status, created_at)`

否则你会看到：

- 明明加了缓存/连接池，接口还是慢（根本原因是慢 SQL）

---

## Part 4：多库多租户（DataSource per tenant）落地方案

这里是大家最关心、也最容易踩坑的部分：

- 每个租户一个库 -> 需要为每个租户维护一个 `DataSource`
- 但你不能“每个请求 new 一个 DataSource”，那会把 DB 打爆

核心思想：

- **按 tenantId 缓存 DataSource（Map）**
- **DataSource 复用连接池**
- **设定最大租户连接数上限与回收策略**

### 1）租户配置从哪里来？

你需要一个“租户元数据库”（也叫 `tenant_registry`）：

- 存每个租户的 DB host/user/pass/dbName
- 或者存 DSN、schema 名、只读/读写配置等

通常做法：

- 先用一个固定的主库连接（`registryDataSource`）
- 根据请求 tenantId 查询 registry 拿连接信息

### 2）实现一个 `TenantDataSourceManager`

伪代码结构（示意用途）：

```ts
class TenantDataSourceManager {
  private readonly cache = new Map<string, DataSource>();

  async getDataSource(tenantId: string): Promise<DataSource> {
    const existed = this.cache.get(tenantId);
    if (existed?.isInitialized) return existed;

    const config = await this.loadTenantDbConfig(tenantId);
    const ds = new DataSource({
      type: "mysql",
      host: config.host,
      port: config.port,
      username: config.username,
      password: config.password,
      database: config.database,
      entities: [
        /* ... */
      ],
      synchronize: false,
      extra: {
        connectionLimit: 10,
      },
    });

    await ds.initialize();
    this.cache.set(tenantId, ds);
    return ds;
  }
}
```

需要你特别注意的点：

- **并发初始化**：同一个 tenantId 的并发请求可能导致重复 initialize。
  - 需要加“单飞锁”（Promise in-flight）避免重复创建。
- **缓存增长**：租户很多时 Map 会无限长。
  - 需要 LRU / TTL 回收，或限制最大租户连接数量。
- **连接数爆炸**：`租户数 * 每租户 pool max * 服务实例数` 很快超过 DB 上限。
  - 必须做容量规划。

### 3）在 NestJS 里把“按请求选择 DataSource”做成 Provider

思路：用 `REQUEST` 注入拿到 `tenantId`，然后返回该租户的 DataSource / EntityManager / Repository。

关键点：

- 这个 provider 通常需要 `scope: Scope.REQUEST`（请求级）
- 但 DataSource 本身是缓存复用的（单例管理器里）

示意结构（仅表达思路）：

```ts
import { Scope } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';

{
  provide: 'TENANT_DATASOURCE',
  scope: Scope.REQUEST,
  inject: [REQUEST, TenantDataSourceManager],
  useFactory: async (req: any, mgr: TenantDataSourceManager) => {
    return mgr.getDataSource(req.tenantId);
  },
}
```

然后你的 service 里可以注入 `'TENANT_DATASOURCE'`，再通过 `getRepository()` 获取 repository。

> 实战里我更建议注入 `EntityManager` 或封装 `TenantDbService`，避免 service 层到处写 `getRepository`。

---

## Part 5：迁移（migrations）怎么处理？

多租户项目最容易被忽略的就是迁移策略。

### 单库多租户

- 迁移跟普通项目一样跑一次即可
- 重点是确保所有表都带 `tenant_id`，并补齐索引

### 多库多租户

你有两种常见策略：

- **策略 1：新租户创建时跑迁移**
  - 创建数据库 -> 执行 migrations -> 标记 tenant ready
- **策略 2：统一批处理迁移**
  - 发版时遍历租户逐个跑迁移（需要任务队列/可重试）

建议：

- 一定要有 `schema_version` 记录（每个租户一份）
- 迁移要可重试、可观测（日志/告警/失败回滚策略）

---

## Part 6：多租户常见坑（踩过一次就会记一辈子）

### 1）忘了加 tenant 条件 = 越权漏洞

单库多租户里最常见事故：

- 代码里有一条 `findOne({ where: { id } })`
- 少了 `tenant_id`，直接把其他租户的数据查出来

应对：

- 代码审计与约束：封装统一 repository/manager
- DB 层隔离：能多库就别单库；或用 RLS（Postgres）

### 2）DataSource per tenant 没有并发保护

并发进来时，多次 initialize 同一个租户 DataSource：

- 会创建多组连接池
- 连接数飙升，DB 直接被打挂

应对：

- “in-flight Promise” 单飞锁
- 失败时清理缓存，避免脏状态

### 3）连接池配置一把梭，导致 DB max_connections 爆掉

容量规划公式必须牢记：

- **总连接数 ≈ 实例数 × 租户同时活跃数 × 每租户 pool 上限**

应对：

- 限制活跃租户 DataSource 数量（LRU 回收）
- 热门租户单独实例/单独 DB
- 控制 pool 上限 + 降低慢查询

### 4）租户解析来源不可信

如果你从 `x-tenant-id` 取租户：

- 外部请求很容易伪造 header

应对：

- tenantId 应该来自 JWT/Session 等可信身份体系
- 或者 header 只对内网/网关可信，并在网关做校验与注入

---

## 小结

你在 NestJS + TypeORM 里做“连接池 + 多租户”，建议按这个顺序落地：

1. 先把**连接池参数**配合理（别盲目拉高）
2. 明确选择多租户模式：
   - 单库多租户：重点是 **tenant_id 强制约束 + 索引**
   - 多库多租户：重点是 **DataSource 缓存复用 + 并发单飞 + 回收 + 容量规划**
3. 把迁移/观测/安全边界补齐，否则后期会非常痛苦

如果你告诉我：

- 你用的是 MySQL 还是 Postgres
- 租户数大概多少、是否有大客户
- 你希望租户从 JWT、子域名还是 header 来

我可以把文中“示意结构”进一步写成一套更贴近你项目的可复制代码（包括：`TenantDataSourceManager` 的单飞锁、LRU 回收、以及 Nest Provider 的注入方式）。
