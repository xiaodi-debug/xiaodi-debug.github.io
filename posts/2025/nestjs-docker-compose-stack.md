---
title: 一套可直接跑的 Docker Compose：NestJS + MySQL + Redis + Nginx（含 .env/初始化 SQL/反代）
tags: [Docker, Docker Compose, NestJS, MySQL, Redis, Nginx, 部署, 实战]
categories: [运维]
date: 2025-12-19
description: 给出一套可直接复制运行的 Docker Compose 栈：Nginx 作为入口反代到 NestJS，MySQL 持久化并支持初始化 SQL，Redis 提供缓存能力，包含目录结构、.env、nginx 配置、init.sql 与 Dockerfile。
articleGPT: 这篇文章提供一个开箱即用的 compose 示例，覆盖典型后端开发/测试环境：MySQL（持久化+初始化脚本）、Redis、NestJS（Dockerfile 构建）、Nginx 反向代理。并解释容器内网络访问、depends_on 的限制、初始化 SQL 只在首次初始化数据卷执行等常见坑。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/docker.png
---

## 1. 你会得到什么？

一个适合本地开发/测试的组合：

- **Nginx**：对外暴露 `80`，作为统一入口反向代理到 NestJS
- **NestJS**：容器内监听 `3000`
- **MySQL**：数据卷持久化 + 支持初始化 SQL
- **Redis**：缓存/队列等

启动只需要一条命令：

```bash
docker compose up -d --build
```

---

## 2. 推荐目录结构

```txt
your-project/
  compose.yml
  .env
  nginx/
    default.conf
  mysql/
    init/
      001-init.sql
  server/
    Dockerfile
    package.json
    src/
      main.ts
      ...
```

说明：

- `server/` 是你的 NestJS 项目目录。
- 如果你的 NestJS 项目就在仓库根目录：
  - 把 `compose.yml` 里的 `build.context` 从 `./server` 改成 `.`
  - 并把 Dockerfile 路径对应调整。

---

## 3. `.env` 示例

在 `compose.yml` 同级创建 `.env`：

```env
# MySQL
MYSQL_ROOT_PASSWORD=123456
MYSQL_DATABASE=app_db
MYSQL_USER=app
MYSQL_PASSWORD=app123456
MYSQL_PORT=3306

# Redis
REDIS_PORT=6379

# NestJS
APP_PORT=3000

# Nginx
NGINX_PORT=80
```

---

## 4. MySQL 初始化 SQL（可选）

在 `mysql/init/001-init.sql` 写入（示例）：

```sql
CREATE TABLE IF NOT EXISTS users (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  username VARCHAR(64) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_username (username)
);

INSERT INTO users (username)
VALUES ('admin')
ON DUPLICATE KEY UPDATE username = VALUES(username);
```

重要说明：

- MySQL 镜像会自动执行 `/docker-entrypoint-initdb.d/` 下的 `.sql`。
- **仅在首次初始化数据目录时执行**（也就是首次创建数据卷时）。
- 如果你已经有 `mysql_data` 卷了，想让脚本重新执行：
  - 需要删除卷后再 `up`（会丢数据，谨慎）。

---

## 5. Nginx 反向代理配置

在 `nginx/default.conf` 写入：

```nginx
server {
  listen 80;
  server_name _;

  location / {
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_pass http://app:3000;
  }
}
```

说明：

- `proxy_pass http://app:3000;` 里的 `app` 是 compose 里 NestJS 的服务名。
- Docker 内部网络会给每个服务名提供 DNS 解析。

---

## 6. NestJS 的 Dockerfile（示例）

在 `server/Dockerfile` 写入一个常用版本：

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "run", "start:prod"]
```

备注：

- 如果你是 pnpm/yarn，把 `npm ci` / `npm run build` / `npm run start:prod` 改成对应命令。
- 生产建议用多阶段构建进一步瘦身，这里先保证“复制即可跑通”。

---

## 7. `compose.yml` 完整示例

在仓库根目录创建 `compose.yml`：

```yaml
name: nestjs-stack

services:
  mysql:
    image: mysql:8
    container_name: nest-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "${MYSQL_PORT}:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci

  redis:
    image: redis:7
    container_name: nest-redis
    restart: unless-stopped
    ports:
      - "${REDIS_PORT}:6379"

  app:
    build:
      context: ./server
      dockerfile: Dockerfile
    container_name: nest-app
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: ${APP_PORT}
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: ${MYSQL_USER}
      DB_PASS: ${MYSQL_PASSWORD}
      DB_NAME: ${MYSQL_DATABASE}
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      - mysql
      - redis
    expose:
      - "${APP_PORT}"

  nginx:
    image: nginx:1.27-alpine
    container_name: nest-nginx
    restart: unless-stopped
    ports:
      - "${NGINX_PORT}:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app

volumes:
  mysql_data:
```

---

## 8. 启动与验证

启动：

```bash
docker compose up -d --build
```

查看状态：

```bash
docker compose ps
```

看日志：

```bash
docker compose logs -f app
docker compose logs -f nginx
```

验证：

- 浏览器打开 `http://localhost`（或 `http://localhost:80`），应该能访问到 NestJS 服务。
- 进入 MySQL 容器验证初始化表：

```bash
docker exec -it nest-mysql mysql -u${MYSQL_USER} -p${MYSQL_PASSWORD} ${MYSQL_DATABASE}
```

---

## 9. 常见坑（这个组合最容易踩）

- **容器内访问 DB/Redis 不要用 localhost**
  - 容器内的 `localhost` 指的是“容器自己”。
  - 应该用服务名：`mysql`、`redis`。
- **MySQL 初始化 SQL 没执行**
  - 只会在“数据卷首次初始化”时执行。
  - 已有 `mysql_data` 卷时不会再次执行。
- **depends_on 不等于 ready**
  - 只保证启动顺序，不保证 MySQL 已可连接。
  - 建议应用侧实现 DB 重试，或为 MySQL/Redis 添加 `healthcheck`。

---

## 10. 可选升级方向

- **开发模式热更新**：把 `app` 改成 `start:dev` 并挂载源码目录
- **健康检查**：为 MySQL/Redis 配 `healthcheck` + `condition: service_healthy`
- **生产增强**：Nginx gzip、静态资源缓存头、HTTPS 等
