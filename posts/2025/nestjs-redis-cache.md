---
title: NestJS 使用 Redis 缓存：从接入到失效策略与最佳实践
tags: [Node.js, NestJS, Redis, 缓存, 性能优化, 学习笔记]
categories: [后端]
date: 2025-12-19
description: 以 NestJS 为例介绍如何接入 Redis 做缓存，包括 CacheModule 接入、接口级缓存、手动缓存、缓存失效、Key 设计以及常见坑与最佳实践。
articleGPT: 这篇文章从“为什么要缓存”讲起，给出 NestJS 中两种常见 Redis 缓存方案（cache-manager 与直接使用 Redis 客户端），并重点演示 CacheModule 的落地方式：全局配置、接口级缓存、手动 get/set、缓存失效（删除/更新）、Key 规范、以及缓存穿透/击穿/雪崩的应对策略。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/redis.png
---

## 为什么要在 NestJS 里用 Redis 做缓存？

缓存的核心目标很简单：**用空间换时间**。

典型收益：

- **降低数据库压力**：热点接口不再每次都查 DB。
- **降低接口延迟**：命中缓存时直接从内存/Redis 返回。
- **提升吞吐**：同样的机器能支撑更多请求。

典型适用场景：

- 读多写少的列表/详情：文章详情、配置、字典表。
- 昂贵计算：聚合统计、复杂 SQL、跨服务拼装数据。
- 三方接口：调用成本高、限流严格，缓存结果避免重复请求。

## 两种做法：Nest 缓存体系 vs 直接用 Redis 客户端

在 NestJS 里使用 Redis 缓存，大体有两条路：

### 方案 A：`@nestjs/cache-manager` + Redis store（推荐先用这个）

优点：

- 和 Nest 的 **依赖注入**、**拦截器**天然契合。
- 接口级缓存可以直接用 `CacheInterceptor`。
- 可以统一管理默认 TTL、Key 前缀等。

缺点：

- 更复杂的策略（tag 失效、分布式锁、预热等）需要自己扩展。
- 不同 Nest / cache-manager / store 版本间有兼容性差异，升级要留意。

### 方案 B：直接用 `redis` / `ioredis`（更灵活）

优点：

- 可控性最强：pipeline、lua、分布式锁、复杂 Key 结构都能做。
- 适合复杂业务缓存（多 Key 同时失效、缓存预热、延迟双删等）。

缺点：

- 需要自己封装一层缓存工具与规范。

本文以 **方案 A 为主**，最后补一个 **方案 B 的封装思路**。

---

## 准备：启动 Redis

本地开发可以用 Docker：

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

生产建议：

- 设定最大内存与淘汰策略（例如 `allkeys-lru`）。
- 开启密码与网络隔离（不要裸奔在公网）。
- 监控：QPS、内存、命中率、慢查询、key 数量。

---

## 方案 A：用 CacheModule 接入 Redis（全局配置）

### 1）安装依赖

```bash
npm i @nestjs/cache-manager cache-manager
npm i cache-manager-redis-store
```

> 如果你项目是 ESM 或 Nest 版本更新较快，建议固定依赖版本并在 CI 里跑一次启动验证，避免 store 兼容性问题。

### 2）在 `AppModule` 注册缓存（建议全局）

```ts
import { Module } from "@nestjs/common";
import { CacheModule } from "@nestjs/cache-manager";
import { redisStore } from "cache-manager-redis-store";

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: async () => ({
        store: await redisStore({
          socket: {
            host: process.env.REDIS_HOST ?? "127.0.0.1",
            port: Number(process.env.REDIS_PORT ?? 6379),
          },
          // password: process.env.REDIS_PASSWORD,
        }),
        ttl: 60,
      }),
    }),
  ],
})
export class AppModule {}
```

- `ttl`：默认缓存时间（秒）。实际建议按接口设置，不要“一刀切”。
- `isGlobal: true`：全局可用，不用每个模块重复 imports。

---

## 接口级缓存：`CacheInterceptor`

### 1）快速给 GET 接口加缓存

```ts
import { Controller, Get, UseInterceptors } from "@nestjs/common";
import { CacheInterceptor, CacheTTL } from "@nestjs/cache-manager";

@Controller("posts")
@UseInterceptors(CacheInterceptor)
export class PostsController {
  @Get("hot")
  @CacheTTL(30)
  async hotPosts() {
    // 这里假设是昂贵查询
    return [{ id: 1, title: "hot post" }];
  }
}
```

说明：

- `CacheInterceptor` 会对返回值进行缓存。
- `@CacheTTL(30)` 覆盖默认 TTL。

### 2）Key 如何生成？（重要）

默认情况下拦截器会基于请求信息生成 key（通常包含 URL）。

但在真实项目里你通常需要：

- 给 key 加业务前缀（避免跨系统冲突）
- 把 query 参数、用户身份、分页信息纳入 key

可以用 `@CacheKey()` 人工指定 key（适合简单且 key 固定的接口）：

```ts
import { CacheInterceptor, CacheKey, CacheTTL } from '@nestjs/cache-manager';

@CacheKey('posts:hot')
@CacheTTL(30)
@Get('hot')
hotPosts() {
  return this.postsService.getHotPosts();
}
```

> 如果 key 需要动态拼（比如分页/筛选），建议走“手动缓存”的方式（下一节）。

---

## 手动缓存：在 Service 中 get/set（更通用）

很多时候你需要更细粒度控制：

- Key 包含分页、筛选、用户
- 命中缓存直接返回，未命中才查询 DB
- 写操作后主动删除相关缓存

### 1）注入 `CACHE_MANAGER`

```ts
import { Injectable, Inject } from "@nestjs/common";
import { CACHE_MANAGER } from "@nestjs/cache-manager";
import type { Cache } from "cache-manager";

@Injectable()
export class PostsService {
  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}
}
```

### 2）实现“缓存优先”的读取

```ts
async getPostDetail(postId: number) {
  const key = `posts:detail:${postId}`;

  const cached = await this.cache.get<any>(key);
  if (cached) return cached;

  const data = await this.queryFromDb(postId);

  await this.cache.set(key, data, 120);
  return data;
}
```

建议：

- `key` 明确包含领域前缀与业务含义。
- `ttl` 不要太长，避免数据长期不一致。
- 序列化：cache-manager 默认会把对象存起来（底层 store 可能会 JSON 化），但你依旧要确保返回值是可序列化的纯数据。

---

## 缓存失效（删除 / 更新）怎么做？

缓存最大的问题不是“写不进去”，而是“数据变了怎么办”。

### 1）写操作后删除缓存（常用）

比如更新文章：

```ts
async updatePost(postId: number, payload: any) {
  const updated = await this.updateDb(postId, payload);

  await this.cache.del(`posts:detail:${postId}`);
  await this.cache.del('posts:list:hot');

  return updated;
}
```

这叫 **Cache Aside（旁路缓存）模式**：

- 读：先读缓存，没命中读 DB，再写缓存
- 写：先写 DB，再删缓存（让下一次读回源 DB 再重建缓存）

### 2）延迟双删（写多读多、并发较高时考虑）

在极端并发下可能出现：

- A 更新 DB -> 删缓存
- B 读到缓存未命中 -> 回源 DB（但 DB 还没提交/或读到旧数据）-> 写回旧缓存

一种工程化做法是 **延迟双删**：

- 写 DB
- 删除缓存
- 等待一个短延迟（比如 200~500ms，视数据库事务时间）
- 再删除一次缓存

是否需要用到它取决于业务一致性要求与并发情况。

---

## Key 设计规范（强烈建议统一）

推荐 key 结构：

```txt
{app}:{domain}:{resource}:{identifier}:{version?}:{params_hash?}
```

例如：

- `blog:posts:detail:123`
- `blog:posts:list:hot:v1`
- `blog:posts:list:search:v1:kw=nestjs&page=1&pageSize=20`（更建议把 params 做 hash，避免 key 过长）

要点：

- **前缀隔离**：防止不同项目/不同环境 key 冲突。
- **可读**：排障时能一眼看懂是什么。
- **可控长度**：key 太长会影响内存与网络。
- **版本号**：当你改了缓存结构，可以直接升级 `v1 -> v2`，避免旧缓存污染。

---

## 常见缓存问题与应对

### 1）缓存穿透（大量请求不存在的数据）

现象：请求的 id 根本不存在，导致每次都回源 DB。

应对：

- 缓存“空值”（短 TTL，例如 10~30 秒）
- 或使用布隆过滤器（复杂一点，但很有效）

### 2）缓存击穿（热点 key 过期瞬间大量回源）

现象：某个超热点 key 刚好过期，大量请求同时回源 DB。

应对：

- 给热点 key 做“互斥锁重建”（单飞）
- 或做“提前刷新/后台预热”
- TTL 加随机抖动（避免同一批 key 同时过期）

### 3）缓存雪崩（大量 key 同时过期）

现象：一大片 key 同时失效，引发 DB 被打爆。

应对：

- TTL 加随机抖动（例如 `ttl = base + random(0..30)`）
- 分批预热
- 限流/熔断/降级

---

## 方案 B：直接用 Redis 客户端的封装思路（简版）

如果你打算做更复杂策略（分布式锁、pipeline、lua、tag 失效），建议直接用 `ioredis` 或官方 `redis` 客户端，然后在 Nest 里封装一个 `RedisService`。

你一般会封装这些能力：

- `getJson(key)` / `setJson(key, value, ttl)`
- `del(keys)` 支持批量删除
- `withLock(lockKey, ttl, fn)` 支持互斥重建
- key 规范：统一在一个地方生成，避免到处拼字符串

如果你希望我把这套封装按你仓库的 Nest 项目结构（module/service）写成可直接复制的代码，我可以再补一篇“代码落地版”。

---

## 小结

本文用 NestJS 介绍了 Redis 缓存的常见落地方式：

- **CacheModule 接入 Redis**：统一管理缓存能力
- **接口级缓存**：`CacheInterceptor` + `@CacheTTL` 快速提升性能
- **手动缓存**：在 Service 中 `get/set/del`，实现可控 key 与失效策略
- **最佳实践**：Key 设计、TTL 策略，以及穿透/击穿/雪崩的应对

下一步你可以在自己的业务里选一个“慢接口/热点接口”，优先把它改造成 cache-aside，并配上合理的失效策略与监控（命中率、回源率），通常会很快看到收益。
