# Dify 私有化部署 SOP(受限网络环境实战版)

> 适用:Docker Hub / GitHub 直连受限的内网或国内环境;目标机器为 Linux 服务器(虚拟机/云主机均可)。
> 实测环境:Ubuntu 22.04,4C/6GB 内存/58GB 磁盘,Dify 1.17.1(14 容器)。

## 1. 前置检查

```bash
free -h && df -h / && nproc          # 内存≥6G,磁盘≥40G
docker --version && docker compose version
docker ps                             # 确认当前用户在 docker 组,免 sudo
```

## 2. 获取编排文件(GitHub 不可达时走代理)

```bash
P="https://ghproxy.icu/https://raw.githubusercontent.com/langgenius/dify/<版本>/docker"
# 需下载:docker-compose.yaml / .env.example
#   nginx/:nginx.conf.template proxy.conf.template https.conf.template conf.d/default.conf.template docker-entrypoint.sh
#   ssrf_proxy/:squid.conf.template squid-common.conf.template squid-agent.conf.template docker-entrypoint.sh docker-agent-entrypoint.sh
#   envs/databases/:db-postgres.env.example redis.env.example → 复制为 .env 同名去掉 .example
#   volumes/sandbox/conf/config.yaml
curl -sSL "$P/<文件路径>" -o <本地路径>
```

> ⚠️ **最大的坑**:compose 引用的宿主文件路径若不存在,Docker 会把它自动建成**目录**,容器启动即崩溃循环
> (`mount a directory onto a file`)。**必须先备齐全部文件再 `up`**。
> 修复方法:`rmdir` 假目录 → 补文件 → `docker compose up -d --force-recreate <服务>`。
> root 属主目录写不进去时,借容器写入:`docker run --rm -v /tmp/文件:/src:ro -v <目标目录>:/target busybox cp /src /target/<文件名>`

## 3. 镜像获取(Docker Hub 被封时:镜像站预拉 + retag)

```bash
docker pull docker.1panel.live/langgenius/dify-api:1.17.1
docker tag  docker.1panel.live/langgenius/dify-api:1.17.1 langgenius/dify-api:1.17.1
docker rmi  docker.1panel.live/langgenius/dify-api:1.17.1
```

需处理 12 个镜像:dify-api / dify-web / dify-sandbox:0.2.15 / dify-plugin-daemon / dify-agent-backend / dify-agent-local-sandbox / postgres:15-alpine / redis:6-alpine / nginx / busybox / ubuntu:squid / weaviate(官方仓库在 cr.weaviate.io,镜像站走 `semitechnologies/weaviate`)。
备用镜像站:docker.m.daocloud.io、docker.1ms.run。单个站卡住(吞吐骤降)就 kill 当前 pull 换站——已下载的层会保留复用。

## 4. .env 调优(小内存机器)

```bash
cp .env.example .env
sed -i 's/,collaboration//' .env                       # 去掉独立 websocket 服务,省约 700MB 内存
sed -i "s|^SECRET_KEY=.*|SECRET_KEY=$(openssl rand -base64 42)|" .env
```

## 5. 启动与验收

```bash
docker compose up -d
docker compose ps                       # 全部 Up,api/db/sandbox/redis 显示 (healthy)
curl -o /dev/null -w "%{http_code}" http://localhost/apps   # 307/200 即通
```

浏览器访问 `http://<服务器IP>` 初始化管理员账号(纯本地注册)。

## 6. 日常运维

- 重启后不会自启:`cd <目录> && docker compose up -d`
- 排障三板斧:`docker compose ps` 看状态 → `docker logs <容器名>` 看日志 → `--force-recreate` 重建
- 配置模型:设置 → 模型供应商 → SiliconFlow(API 方式)或 Ollama(本地方式);启用对话/向量/重排三类模型并在"系统模型设置"里指定默认值

## 7. 常见故障速查

| 现象 | 定位 | 处置 |
|---|---|---|
| 容器循环 Restarting | `docker logs` 看 mount 报错 | 补齐缺失宿主文件,force-recreate |
| envsubst: Is a directory | nginx 挂载的模板文件缺失被建成目录 | 同上 |
| sandbox panic: config.yaml not found | volumes/sandbox/conf/ 缺配置 | 从仓库下载,busybox 容器写入 |
| 镜像拉取极慢/超时 | 镜像站单链路劣化 | 换镜像站重拉,层缓存自动续传 |
