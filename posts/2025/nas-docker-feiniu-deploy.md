---
title: 在飞牛 NAS 上用 Docker 折腾 CloudSave、quark-auto-save、v2raya
tags: [NAS, Docker, 自建服务, 踩坑记录]
categories: [NAS]
date: 2025-12-10
description: 记录我在飞牛 NAS 上用 Docker 部署 CloudSave、quark-auto-save、v2raya、飞牛影视和飞牛图库的过程，以及中间踩过的一些小坑。
articleGPT: 这是一篇 NAS 折腾记录，主要围绕在飞牛 NAS 上通过 Docker 部署 CloudSave、quark-auto-save、v2raya、飞牛影视和飞牛图库，顺带整理一下网络、端口、持久化卷以及自动化的注意事项。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/docker.png
---

## 背景：为什么要在 NAS 上折腾这么多服务？

有了 NAS 之后，很自然地会想：

- 把各种下载 / 同步的东西**集中管理**；
- 一台机器搞定**云盘备份、自动离线下载、科学上网、家庭影音、相册管理**；
- 能在外网也比较方便地访问自己的服务。

于是就有了这一套组合：

- CloudSave：做多云盘之间的同步、备份；
- quark-auto-save：自动把资源存到夸克网盘；
- v2raya：图形化管理代理客户端；
- 飞牛影视：家庭流媒体中心；
- 飞牛图库：家庭相册 / 图片管理。

这篇文章主要记录我在飞牛 NAS 上用 Docker 把这些服务跑起来的过程，以及中间踩过的坑，方便以后自己查，也希望能帮到同样在折腾 NAS 的你。

---

## 一、准备工作：Docker 环境和基础网络

### 1. NAS 上的 Docker 环境

我使用的是飞牛 NAS 自带的 Docker 环境，大致确认了几件事情：

- Docker 能正常拉取镜像（国内网络建议配置镜像加速）；
- 有一个**专门放配置和数据的共享文件夹**，比如：`/volume1/docker-data`；
- 宿主机的时区、时间已经校准好，不然后面日志和计划任务会很迷惑。

目录大致规划如下：

```text
/volume1/docker-data
  ├─ cloudsave
  ├─ quark-auto-save
  ├─ v2raya
  ├─ feiniu-movie
  └─ feiniu-gallery
```

> 踩坑小记：一开始我把配置随便放，结果迁移 / 备份的时候特别痛苦。后来统一约定所有 Docker 容器的数据都进 `docker-data`，心情愉快很多。

### 2. 网络和端口规划

在一台 NAS 上跑这么多服务，端口规划非常重要，否则：

- 要么冲突启动不了；
- 要么记不住哪个服务对应哪个端口。

我最后的规划类似这样（仅示例）：

- CloudSave：`5801`
- quark-auto-save：`5802`
- v2raya：`5803`
- 飞牛影视：`5804`
- 飞牛图库：`5805`

原则：

- 统一用一段连续的端口，方便记忆；
- 如果有反向代理（比如 nginx / caddy），把这些端口都代理到对应的域名路径下。

---

## 二、CloudSave：统一管理多云盘同步

### 1. 容器部署思路

CloudSave 的部署比较常规：

- 一个 Web 管理界面；
- 挂载配置目录，用来保存账号信息、任务配置；
- 挂载一个数据目录，作为临时缓存或中转区。

在飞牛 NAS 的 Docker 界面里创建容器时，注意：

- 映射端口：宿主 `5801` → 容器内部 Web 端口；
- 映射卷：
  - 配置：`/volume1/docker-data/cloudsave/config` → `/app/config`
  - 数据：`/volume1/docker-data/cloudsave/data` → `/app/data`

### 2. 登录云盘和任务配置

部署好之后，访问：

- `http://NAS-IP:5801`

主要做几件事情：

- 添加各个云盘账号（阿里云盘 / 夸克 / OneDrive 等）；
- 为常用的目录建**同步任务**，比如：
  - 本地某个备份目录 → 阿里云盘；
  - 本地某个媒体目录 → 另一块盘。

> 踩坑：有的云盘登录需要扫码或 Cookies，有有效期的，建议定期检查一下，防止任务长期失败却没发现。

---

## 三、quark-auto-save：自动把资源存到夸克

有了 CloudSave 之后，我还想解决一个痛点：

> 某些资源分享链接想第一时间保存到夸克网盘，但又不想每次手动点“保存”。

于是就用上了 quark-auto-save。

### 1. 部署要点

这里同样是在 Docker 里拉镜像，注意：

- 端口映射：宿主 `5802` → 容器 Web 端口；
- 配置目录：`/volume1/docker-data/quark-auto-save/config` → `/app/config`；
- 如果有需要，也可以挂载一个日志目录方便排查问题。

### 2. 夸克登录与自动任务

启动后访问：`http://NAS-IP:5802`，大致流程：

- 登录夸克账号（通常是扫码 / Cookie）；
- 配置一个或多个**监控任务**：
  - 比如定期去某个 RSS / 订阅源抓取链接；
  - 或者提供一个接口，外部脚本调用，让它自动保存。

> 踩坑：如果你把 quark-auto-save 暴露到公网，一定记得加上鉴权（反代加 BasicAuth 或者 IP 限制），否则别人也能帮你“自动存东西”。

---

## 四、v2raya：代理客户端的图形管理

v2raya 的好处是：

- 有 Web 界面，方便在 NAS 上集中管理多个节点；
- 可以配合 NAS 的路由 / 容器网络，实现“某些服务走代理、某些直连”。

### 1. 容器部署

仍然是在 Docker 中创建容器：

- 端口映射：宿主 `5803` → v2raya Web 管理端口；
- 配置目录：`/volume1/docker-data/v2raya` → `/etc/v2raya`；
- 视乎你的网络环境，是否需要：
  - `NET_ADMIN` 权限；
  - 加入特定的 Docker 网络（比如与需要代理的容器放在同一个自定义网络里）。

### 2. 使用体验与注意点

- 在 Web UI 中导入节点（订阅 / 手动）；
- 可以单独为某些容器配置“走 v2raya 网络”；
- 注意不要把 NAS 自己的管理端口全部代理出去了，否则排错会很痛苦。

> 小建议：刚开始可以只让某个测试容器走 v2raya，看一切正常再大范围使用。

---

## 五、飞牛影视 & 飞牛图库：家庭娱乐中心

这两个算是飞牛生态里的“重头戏”：

- 飞牛影视：类似家庭版流媒体中心；
- 飞牛图库：家庭照片 / 图片管理系统。

### 1. 目录结构规划

为了让它们都能读到统一的数据，我在 NAS 上做了这样一个结构：

```text
/volume1/media
  ├─ movies      # 电影
  ├─ tv          # 电视剧 / 综艺
  └─ photos      # 照片 / 图库
```

在 Docker 里：

- 飞牛影视挂载 `/volume1/media/movies`、`/volume1/media/tv` 到容器内部对应目录；
- 飞牛图库挂载 `/volume1/media/photos`。

### 2. Docker 容器配置要点

飞牛影视：

- 端口：宿主 `5804` → 容器 Web 端口；
- 配置：`/volume1/docker-data/feiniu-movie/config` → `/app/config`；
- 媒体库：`/volume1/media` → `/media`。

飞牛图库：

- 端口：宿主 `5805` → 容器 Web 端口；
- 配置：`/volume1/docker-data/feiniu-gallery/config` → `/app/config`；
- 照片：`/volume1/media/photos` → `/photos`。

### 3. 元数据与扫描

部署完成后：

- 先在 Web 界面配置媒体库路径；
- 再触发一次全量扫描，之后可以改成定时增量扫描。

> 踩坑：文件名、目录结构尽量规范化，不然自动识别影片 / 剧集信息会经常错；
> 照片建议按 `年份/月份` 这样分类，图库加载速度也会更稳定一些。

---

## 六、统一访问：反向代理与 HTTPS

当服务越来越多时，端口访问会变得很难记：

- `5801` 是 CloudSave 还是 quark-auto-save？
- 家人想看飞牛影视，还得记 `5804`？

所以我最后加了一层反代（比如 Nginx / Caddy）：

- `https://nas.example.com/cloudsave` → 内部 `http://NAS-IP:5801`
- `https://nas.example.com/quark` → `http://NAS-IP:5802`
- `https://nas.example.com/v2` → `http://NAS-IP:5803`
- `https://nas.example.com/movie` → `http://NAS-IP:5804`
- `https://nas.example.com/gallery` → `http://NAS-IP:5805`

并且：

- 整体走一个 HTTPS 证书（例如用 Cloudflare / Let7s Encrypt）；
- 内网访问时也保持同一套域名，方便配置书签和快捷方式。

> 建议：对暴露到公网的服务，务必加上登录保护 / IP 白名单，否则风险太大。

---

## 七、这一轮 NAS 折腾我学到了什么？

最后简单总结一下这次在飞牛 NAS 上用 Docker 折腾这一堆服务的收获：

- **统一的目录和端口规划很重要**  
  一开始乱七八糟，后面迁移 / 备份、排错都很痛苦。统一用 `docker-data` + 连续端口之后清爽很多。

- **Docker 容器尽量“无状态”，数据都挂到卷里**  
  以后换 NAS / 换宿主，只要把 `docker-compose` / 配置和数据卷搬过去，大部分服务可以无痛恢复。

- **针对 NAS 的性能做一点取舍**  
  不要一次性开太多重型服务，尤其是转码、缩略图之类，注意 CPU / 内存 / IO 使用情况。

- **公开服务一定要过一遍安全清单**  
  哪些服务必须外网能访问？哪些只在内网？是否有登录保护？这些问题最好在搭建前就想清楚。

接下来我还打算在 NAS 上继续尝试：

- 加一个轻量级监控 / 可视化（比如 Grafana + Prometheus）；
- 进一步把 CloudSave、quark-auto-save 和媒体库打通，做一条“从订阅 → 下载 → 整理 → 入库”的自动化流水线。

如果你也在折腾 NAS，或者对上面的某个服务感兴趣，欢迎在「畅所欲言」页面留言交流，一起把家里的小盒子用出点花来。
