---
title: "Docker 常见使用技巧"
date: 2026-11-17T21:00:00+08:00
tags: ["Docker", "运维"]
---

把日常用得最多的一些 Docker 操作整理一下，都是踩过坑之后留下来的。

<!--more-->

## 一、先看空间去哪了

磁盘被镜像和容器塞满是最高频的问题。按顺序排查：

```bash
# 总览：镜像 / 容器 / 卷 / 构建缓存 各占多少
docker system df -v

# 只清理停止的容器、悬空镜像、无用网络和构建缓存
docker system prune

# 连没在用的镜像和卷一起清（注意：卷删了数据就没了）
docker system prune -a --volumes
```

清理前一定要先跑 `docker system df -v` 看清楚，`--volumes` 这一刀很容易把数据库的数据卷一起送走。

## 二、看日志的正确姿势

```bash
# 持续跟踪，只要最后 200 行
docker logs -f --tail 200 容器名

# 只看最近 10 分钟
docker logs --since 10m 容器名

# 加时间戳（很多镜像默认不带时间，排查时间线时很关键）
docker logs -ft 容器名
```

容器的日志如果不限制大小，能一直写到把磁盘撑爆。启动时就要设上限：

```bash
docker run -d \
  --log-opt max-size=50m \
  --log-opt max-file=3 \
  镜像名
```

## 三、进容器里看

```bash
docker exec -it 容器名 sh     # alpine 类镜像没有 bash
docker exec -it 容器名 bash
```

如果镜像是 distroless 或 scratch，里面根本没有 shell。这时候用调试容器挂进去：

```bash
# 起一个带工具的容器，共享目标容器的网络和进程命名空间
docker run -it --rm --network container:目标容器名 --pid container:目标容器名 \
  nicolaka/netshoot
```

## 四、容器和宿主机互传文件

```bash
docker cp 容器名:/app/config.yml ./config.yml
docker cp ./config.yml 容器名:/app/config.yml
```

`docker cp` 不受挂载限制，容器里的任何路径都能拷出来，救急很好用。

## 五、查看容器内部信息

```bash
# 只看 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 容器名

# 只看退出码（排查容器反复重启）
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' 容器名
```

`OOMKilled: true` 说明是被内存限制杀掉的，不是程序自己崩的。

## 六、资源限制

```bash
docker run -d --memory=512m --memory-swap=512m --cpus=1.5 镜像名
```

`--memory-swap` 和 `--memory` 设成相同值，等于禁用 swap，行为更可预测。

## 七、网络排查

容器里访问 `127.0.0.1` 指的是容器自己，不是宿主机——这是新手最容易卡住的地方。

```bash
docker network ls
docker network inspect bridge

# 容器里要看宿主机，用 host.docker.internal（Docker Desktop）
# 或在 Linux 上用 --add-host=host.docker.internal:host-gateway
```

## 八、docker compose 常用命令

```bash
docker compose up -d              # 后台启动
docker compose up -d --build      # 改了 Dockerfile 要加 --build
docker compose logs -f 服务名
docker compose exec 服务名 sh
docker compose ps
docker compose down               # 停容器+删网络，保留卷
docker compose down -v            # 连卷一起删
docker compose config             # 校验并展开变量，排查配置错误神器
```

改了 `docker-compose.yml` 里的镜像版本，`docker compose up -d` 会自动重建；只改了挂载文件则不用重启。

## 九、镜像瘦身

1. **多阶段构建**：编译环境和运行环境分开，产物只带二进制
2. **`.dockerignore`**：把 `.git`、`node_modules`、日志排除掉，否则构建上下文巨大
3. 基础镜像优先选 `alpine` 或 `distroless`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

## 十、健康检查

容器在跑不代表服务可用。加 healthcheck 让编排系统能判断真实状态：

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 40s
```

`start_period` 很重要，给应用留出启动时间，否则会被误判为不健康反复重启。

## 十一、几个反复踩的坑

| 现象 | 原因 |
|---|---|
| 容器起来几秒就退出 | 主进程是后台运行的，加 `-it` 或让主进程前台运行 |
| 挂载的目录权限报错 | 容器内 UID 和宿主机不一致，注意 `user:` 或目录属主 |
| 改了代码不生效 | 镜像里的代码是构建时拷进去的，得重新 build |
| 端口占用 | `docker ps` 看不到的话，用 `ss -tlnp` 查宿主机 |
| 时区不对 | 加 `-e TZ=Asia/Shanghai` 或在镜像里装 tzdata |

---

这套东西基本能覆盖日常 90% 的操作场景。真正的经验都在踩坑里，遇到问题先 `inspect` 和 `logs`，比猜快得多。
