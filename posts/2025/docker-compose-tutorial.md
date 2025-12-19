---
title: Docker 安装 + 常用命令 + Docker Compose 使用教程（从0到能用）
tags: [Docker, Docker Compose, DevOps, 部署, 学习笔记]
categories: [运维]
date: 2025-12-19
description: 一篇面向新手的 Docker 实战教程：从安装 Docker（Windows/Linux）开始，整理最常用的 Docker 命令，并用 Docker Compose 演示如何一键启动多服务（MySQL/Redis/Nginx/应用）。
articleGPT: 这篇文章覆盖 Docker 的安装、镜像/容器/网络/卷的基本概念与常用命令速查，并重点讲 Docker Compose：compose.yml 的核心语法、启动/停止/日志/重建/清理命令、环境变量与数据持久化、以及一个可直接复制的多服务示例。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/docker.png
---

## 1. Docker 是什么？先把几个概念捋清楚

很多人刚接触 Docker 最痛苦的不是命令，而是名词：

- **镜像（Image）**：相当于“软件安装包 + 运行环境”，是静态模板。
- **容器（Container）**：镜像运行起来的实例，是进程级隔离的运行环境。
- **仓库（Registry）**：镜像存放的地方，例如 Docker Hub。
- **数据卷（Volume）**：把容器内的数据持久化到宿主机（否则删容器数据就没了）。
- **网络（Network）**：容器之间通信通常通过 Docker 网络进行。

你可以用一句话理解：

- **镜像 = 模板，容器 = 运行中的实例**。

---

## 2. 安装 Docker（Windows / Linux）

### 2.1 Windows：安装 Docker Desktop（推荐）

1. 到 Docker 官网下载 Docker Desktop 并安装。
2. 安装时一般会让你启用：
   - WSL2（推荐）
   - Hyper-V（部分机器/系统版本才会用到）
3. 安装完成后打开 Docker Desktop，等待 Docker Engine 启动。

验证：打开 PowerShell：

```bash
docker version
docker info
```

如果你看到 server/client 信息，就说明安装成功。

常见坑：

- Docker Desktop 起不来：先检查 WSL2 是否安装并启用。
- 端口被占用：容器映射端口时，宿主机端口必须空闲。

### 2.2 Linux（以 Ubuntu 为例）

Linux 更推荐安装 Docker Engine（服务端），不一定需要 Desktop。

思路：

- 安装 Docker Engine
- 把当前用户加入 `docker` 组（可选，避免每次 sudo）

安装完成后同样用：

```bash
docker version
docker info
```

来验证。

> 不同发行版安装命令略有差异，生产环境建议直接按 Docker 官方文档的步骤来，避免装到旧版本。

---

## 3. Docker 常用命令速查（最常用的 20 个）

下面这些基本覆盖日常 80% 的需求。

### 3.1 镜像相关

- **拉取镜像**

```bash
docker pull redis:7
```

- **查看本地镜像**

```bash
docker images
```

- **删除镜像**

```bash
docker rmi redis:7
```

- **构建镜像（有 Dockerfile 时）**

```bash
docker build -t myapp:latest .
```

### 3.2 容器相关

- **运行容器（最常用）**

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

参数说明：

- `-d`：后台运行
- `--name`：容器名
- `-p 宿主机端口:容器端口`：端口映射

- **查看正在运行的容器**

```bash
docker ps
```

- **查看所有容器（包含停止的）**

```bash
docker ps -a
```

- **停止/启动/重启容器**

```bash
docker stop redis
docker start redis
docker restart redis
```

- **删除容器**

```bash
docker rm redis
```

- **进入容器（排障很常用）**

```bash
docker exec -it redis sh
```

> 镜像里不一定有 `bash`，很多轻量镜像只有 `sh`。

- **查看容器日志**

```bash
docker logs redis

# 实时追踪
docker logs -f redis
```

- **查看容器资源使用**

```bash
docker stats
```

### 3.3 网络相关

- **查看网络**

```bash
docker network ls
```

- **创建网络**

```bash
docker network create mynet
```

- **把容器接入网络**

```bash
docker run -d --name redis --network mynet redis:7
```

### 3.4 数据卷相关（持久化）

- **查看卷**

```bash
docker volume ls
```

- **创建卷**

```bash
docker volume create mysql_data
```

- **运行容器时挂载卷**

```bash
docker run -d --name mysql -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -v mysql_data:/var/lib/mysql \
  mysql:8
```

### 3.5 清理相关（释放空间）

谨慎使用：会删东西。

- **清理未使用的容器/网络/镜像（保留卷）**

```bash
docker system prune
```

- **连卷一起清理（更危险）**

```bash
docker system prune -a --volumes
```

---

## 4. Docker Compose 是什么？

如果你要启动的不止一个服务（例如：MySQL + Redis + 后端 + Nginx），用 `docker run` 会很痛苦：

- 参数长
- 网络、卷、环境变量难维护
- 需要顺序启动、需要统一停止

**Docker Compose** 的目标就是：

- 用一个 `compose.yml` 描述多容器应用
- 一条命令启动/停止/重建整个环境

### 4.1 `docker-compose` vs `docker compose`

- **`docker compose`**：Docker 新版本内置的 Compose（推荐）
- **`docker-compose`**：旧版独立二进制（也能用，但逐步被替代）

本文命令以 **`docker compose`** 为主（旧版只需要把空格换成横杠即可）。

---

## 5. Docker Compose 最常用命令

在 `compose.yml` 所在目录执行：

- **启动（后台）**

```bash
docker compose up -d
```

- **查看服务状态**

```bash
docker compose ps
```

- **查看日志（实时）**

```bash
docker compose logs -f

# 只看某个服务
docker compose logs -f mysql
```

- **停止（不删除容器）**

```bash
docker compose stop
```

- **停止并删除容器/网络（默认保留卷）**

```bash
docker compose down
```

- **连卷也删掉（危险：数据没了）**

```bash
docker compose down -v
```

- **修改 compose 后重建**

```bash
docker compose up -d --build
```

---

## 6. `compose.yml` 核心语法（看完就能写）

一个 compose 文件里最常用的几个字段：

- `services`：定义服务
- `image` / `build`：镜像来源（拉镜像 or 用 Dockerfile 构建）
- `ports`：端口映射
- `environment`：环境变量
- `volumes`：数据持久化
- `depends_on`：启动顺序（注意：不保证“服务已就绪”）
- `networks`：网络

---

## 7. 实战示例：MySQL + Redis + Adminer 一键启动

把下面内容保存为 `compose.yml`：

```yaml
name: dev-stack

services:
  mysql:
    image: mysql:8
    container_name: dev-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: app_db
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7
    container_name: dev-redis
    restart: unless-stopped
    ports:
      - "6379:6379"

  adminer:
    image: adminer:latest
    container_name: dev-adminer
    restart: unless-stopped
    ports:
      - "8080:8080"
    depends_on:
      - mysql

volumes:
  mysql_data:
```

启动：

```bash
docker compose up -d
```

验证：

- MySQL：`localhost:3306`
- Redis：`localhost:6379`
- Adminer：浏览器打开 `http://localhost:8080`

停止并删除容器：

```bash
docker compose down
```

如果你想把 MySQL 数据也删掉：

```bash
docker compose down -v
```

---

## 8. 常见问题（新手高频）

### 8.1 端口映射是什么意思？

`"8080:8080"` 表示：

- 宿主机 `8080` 端口 -> 转发到容器内 `8080` 端口

你访问 `http://localhost:8080`，实际上访问的是容器服务。

### 8.2 容器删了数据怎么还在？

因为你挂载了 `volumes`。比如：

- `mysql_data:/var/lib/mysql`

容器里 MySQL 的数据目录被映射到宿主机的 volume。

### 8.3 `depends_on` 不等于“服务已就绪”

`depends_on` 只能保证“启动顺序”，并不保证 MySQL 已经能接连接。

如果你有强依赖（比如应用启动前必须等 DB ready），建议：

- 应用自身做重试（推荐）
- 或使用健康检查 `healthcheck` 配合（进阶）

### 8.4 Compose 文件要不要写 version？

现在推荐使用新的 Compose 规范（不写 `version:`），Docker 会自动识别。

---

## 9. 一句话总结

- 单个服务：用 `docker run` 够用
- 多服务开发环境：**优先用 Docker Compose**
- 数据要保留：记得用 `volumes`
- 清理空间：`docker system prune` 先看清楚再执行

如果你需要一个更完整的多服务模板（带 `.env`、MySQL 初始化 SQL、Nginx 反代、NestJS Dockerfile），我把它单独放到这篇文章里：

- **/posts/2025/nestjs-docker-compose-stack.md**
