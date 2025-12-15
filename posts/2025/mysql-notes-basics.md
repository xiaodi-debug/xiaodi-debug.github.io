---
title: MySQL 学习笔记：从表设计到常见查询（附踩坑记录）
tags: [MySQL, 数据库设计, 学习笔记, 踩坑记录]
categories: [数据库]
date: 2025-11-20
description: 记录我在用 MySQL 做博客/业务系统时的一些基础表设计思路、常见查询写法，以及编码、索引、N+1 查询等踩过的坑。
articleGPT: 这是一篇面向真实业务的 MySQL 学习笔记，围绕“用户 + 文章”这类常见业务表，记录表结构设计、基础查询、联表查询的写法，以及编码、索引、N+1 查询、误删数据等过程中踩到的坑和解决办法。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/mysql.png
---

## 为什么重新系统学 MySQL？

以前对 MySQL 的印象就是：**能查出数据就行**，表怎么设计、索引怎么建、字段类型选啥，都是「以后再说」。

但当我开始写后端（比如 Node.js + NestJS）时，发现数据库这块如果设计一开始就乱：

- 后面查询会越来越痛苦，SQL 又长又慢
- 想加新功能经常要「大改表结构」
- 一不小心就出现「线上误删数据」「查询超级慢」这种灾难

于是我决定从一个简单但真实的业务场景入手：**用户 + 文章**，重新梳理下 MySQL 的使用。

---

## 场景：用 MySQL 支撑一个简单博客系统

假设我要做的系统只是一个简单版博客，有：

- 用户（User）
- 文章（Post）
- 分类 / 标签（先简单一点，可以只把分类做成字段）

### 我想要实现的功能

1. 用户可以登录（后端可能用 NestJS 做 User 表的增删改查）
2. 每篇文章属于一个用户
3. 支持按：
   - 分类
   - 标题关键字
   - 发布时间
     来筛选文章列表

这个小目标其实就够用上 MySQL 的几个常用知识点了。

---

## 第一步：表设计 —— 用户表和文章表

### 1. 用户表 `users`

需求很常见：

- id
- 用户名（唯一）
- 昵称
- 密码（后续要加密存储）
- 创建时间 / 更新时间

一个简单的表结构示意：

```sql
CREATE TABLE `users` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键 ID',
  `username` VARCHAR(50) NOT NULL COMMENT '登录用户名',
  `nickname` VARCHAR(50) NOT NULL COMMENT '昵称',
  `password` VARCHAR(255) NOT NULL COMMENT '密码（加密后）',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

**踩坑 1：字符集选错导致表情 / 中文乱码**

以前随手就 `utf8`，后来才知道 MySQL 里的 `utf8` 是「阉割版」，不支持 4 字节字符，比如表情、某些特殊符号。

- 如果要支持 Emoji 或更完整的 Unicode，应该用：`utf8mb4`
- 建议直接：数据库、表、连接都统一用 `utf8mb4`，少很多坑

---

### 2. 文章表 `posts`

每篇文章属于一个用户，还要有分类、标题、内容、状态等：

```sql
CREATE TABLE `posts` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键 ID',
  `user_id` BIGINT UNSIGNED NOT NULL COMMENT '作者用户 ID',
  `title` VARCHAR(200) NOT NULL COMMENT '标题',
  `category` VARCHAR(50) NOT NULL DEFAULT '随便说说' COMMENT '分类',
  `content` TEXT NOT NULL COMMENT '正文内容',
  `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1=草稿, 2=已发布, 3=已删除',
  `published_at` DATETIME DEFAULT NULL COMMENT '发布时间',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_category_published_at` (`category`, `published_at`),
  CONSTRAINT `fk_posts_user` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='文章表';
```

这里故意加了一个联合索引：`(category, published_at)`，用来支撑「按分类 + 发布时间排序」的查询。

**踩坑 2：索引不是随便加，加错了等于没加**

一开始我只随手给 `category` 加了一个索引，后来查询写的是：

```sql
SELECT * FROM posts
WHERE category = '前端'
ORDER BY published_at DESC
LIMIT 10 OFFSET 0;
```

结果执行计划显示：

- 只用到了 `category` 索引，排序还要额外做 filesort

后来改成联合索引：

```sql
KEY `idx_category_published_at` (`category`, `published_at`)
```

这个查询就可以一边走索引一边排序，明显快很多。

---

## 第二步：常见查询写法

### 1. 查某个用户自己写的已发布文章（带分页）

```sql
SELECT
  p.id,
  p.title,
  p.category,
  p.published_at
FROM
  posts p
WHERE
  p.user_id = ?
  AND p.status = 2
ORDER BY
  p.published_at DESC
LIMIT ? OFFSET ?;
```

这里有几个点：

- `status = 2` 表示只查已发布的文章
- `ORDER BY published_at DESC` 配合前面提到的索引会更高效
- `LIMIT` + `OFFSET` 做简单分页

**踩坑 3：大 OFFSET 性能问题**

一开始我用 `LIMIT 10 OFFSET 100000` 这种玩法，结果数据一多，后面翻页非常慢。

更好的思路：

- 用「基于游标/ID」的分页，比如记录最后一条记录的 `published_at` 和 `id`，下次从这里继续查；
- 或者前端限制「最多翻到多少页」。

不过在博客这类场景，数据量通常不会大到离谱，前期用 OFFSET 问题不大，但**要知道这会是个潜在瓶颈**。

---

### 2. 按标题关键字 + 分类搜索文章

```sql
SELECT
  p.id,
  p.title,
  p.category,
  p.published_at
FROM
  posts p
WHERE
  p.status = 2
  AND ( :category = '全部' OR p.category = :category )
  AND ( :keyword = '' OR p.title LIKE CONCAT('%', :keyword, '%') )
ORDER BY
  p.published_at DESC
LIMIT :limit OFFSET :offset;
```

这里我用的是**参数占位符**（便于和后端配合），按条件动态加筛选。

**踩坑 4：LIKE 前后都有 `%` 时，普通索引没用**

如果你写的是：

```sql
p.title LIKE '%abc%'
```

普通的 `INDEX(title)` 基本用不上（因为无法从左边开始匹配），MySQL 会倾向于全表扫描。

简单几种思路：

- 把搜索范围控制在**比较少的数据集**，比如先按分类 / 作者过滤，再做 LIKE；
- 如果标题不是特别长，也可以忍受一定的扫描；
- 真正要做搜索，可以考虑引入全文索引 / 搜索引擎（比如 ElasticSearch / Meilisearch 等）。

---

### 3. 把用户信息一并查出来（JOIN）

有时你想要在列表里直接看到作者昵称，可以这样查：

```sql
SELECT
  p.id,
  p.title,
  p.category,
  p.published_at,
  u.nickname AS author_nickname
FROM
  posts p
JOIN
  users u ON p.user_id = u.id
WHERE
  p.status = 2
ORDER BY
  p.published_at DESC
LIMIT :limit OFFSET :offset;
```

**踩坑 5：N+1 查询**

如果你不用联表，而是先查文章列表，再循环里一个个去查用户：

1. `SELECT * FROM posts LIMIT 10`
2. 循环 10 篇文章，每篇再 `SELECT * FROM users WHERE id = ?`

这就是典型的 **N+1 查询**。表一多，延迟会非常明显。

- 解决：该 `JOIN` 的还是要 `JOIN`
- 如果业务场景复杂，可以考虑在服务层做好「批量查」的封装，例如一次性查出所有相关用户，再用 Map 关联，而不是一条一条查。

---

## 第三步：和 NestJS / Node.js 的结合点

在实际项目里，我会这样配合使用：

1. 用 NestJS / TypeORM / Prisma 等 ORM 与 MySQL 建立连接；
2. 表结构设计好后，让 ORM 的实体跟表字段一一对应；
3. 常见查询（分页、过滤、联表）可以用 ORM 的 QueryBuilder 实现；
4. 真的遇到很复杂、性能要求高的 SQL，可以手写原生 SQL，通过 ORM 的「原生查询接口」执行。

### 踩坑 6：字段名不统一导致的「半天查不出数据」

比如：

- 数据库字段是 `published_at`
- TypeScript 接口里写的是 `publishedAt`
- ORM 映射忘了加 `@Column({ name: 'published_at' })`

结果就是：

- 写入时字段对不上
- 查出来的结果为空 / 不包含预期字段

**解决办法：**

- 统一命名规范：数据库用 `snake_case`，代码用 `camelCase`，中间靠 ORM 的配置桥接；
- 不要偷懒，认真写实体/模型的映射。

---

## 第四步：几个「血的教训」级别的注意点

这些坑我虽然没全踩完，但已经看到不少案例，所以提前记一下。

### 1. 永远不要在生产上随便跑没有 WHERE 的 UPDATE / DELETE

```sql
DELETE FROM posts;
UPDATE users SET password = '123456';
```

这种操作在本地怎么玩都行，但一旦在生产执行就完蛋了。

**防御手段：**

- 在客户端层面（比如数据库管理工具）限制没有 WHERE 的语句；
- 在 CI / 管理规范上强制：对生产库只能执行审查过的脚本；
- 养成习惯：先写 `SELECT` 版确认影响行数，再改成 `UPDATE` / `DELETE`。

---

### 2. 对重要数据定期做备份（哪怕是本地测试）

哪怕只是「自己的博客」，你也绝对不希望哪天一个失误把所有文章删了。

简单几种方式：

- 定期 `mysqldump` 整库导出
- 对重要表额外导出一份
- 在 Docker 环境中，把 MySQL 的数据目录挂到宿主机上，避免容器删除就全没了

示例备份命令：

```bash
mysqldump -u root -p my_blog_db > backup_2025_01_20.sql
```

---

## 小结：这轮 MySQL 学习我最重视的几个点

这篇主要围绕「用户 + 文章」的博客业务，梳理了一下 MySQL 的几个关键点：

- **表设计**：字段要能支撑未来的查询场景（比如分类 + 时间排序）
- **字符集统一用 utf8mb4**，避免中文/Emoji 乱码
- **索引尽量贴合查询条件**，特别是联合索引的字段顺序
- **避免 N+1 查询**，适当使用 JOIN 或批量查询
- **和后端实体映射要清晰统一**，蛇形命名和驼峰命名通过 ORM 桥接
- **生产环境的安全操作习惯**（WHERE、备份等）

后面打算继续往下做：

- 用 Docker 起一个 MySQL 实例，让开发环境更好复用
- 在 NestJS 项目里接上 TypeORM / Prisma，写出完整的「用户 + 文章」后台
- 针对慢查询开一篇专门的「性能优化」笔记

如果你也在一点点补数据库基础，希望这篇「学习笔记 + 踩坑记录」能让你对 MySQL 不再只是「能查出数据就行」，而是想着如何「查得对、查得快、查得安全」。
