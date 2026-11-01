# Docker 安装与镜像管理

> 适用：Ubuntu。脚本方式安装（官方 `get.docker.com`），适合开发机快速起环境。

## 1. 安装

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

注意：官方脚本在国内直连较慢，可用镜像站替代脚本地址；脚本会装 `docker-ce` + `containerd` + `docker-buildx`。

## 2. 免 sudo

```bash
sudo groupadd docker 2>/dev/null
sudo usermod -aG docker $USER
newgrp docker                 # 当前会话立即生效，不用重启
docker run hello-world        # 验证
```

`docker` 组等价于 root 权限，只在本机开发环境这么配。

## 3. 开机自启

```bash
sudo systemctl enable --now docker
systemctl status docker
```

## 4. 镜像的导入导出（离线传输）

```bash
# 导出：镜像 → tar
docker save -o image_v1.tar repo/name:v1

# 导入：tar → 镜像
docker load -i image_v1.tar
docker load -i image_v1.tar.gz      # 支持 gzip，无需先解压

# 确认
docker images | grep <关键字>
```

`save` 保留完整镜像元数据（标签、层），`export` 是容器快照会丢标签——传镜像一律用 `save`。

## 5. 清理空间

```bash
docker system df -v         # 看空间被什么占了（镜像/容器/卷/构建缓存）
docker image prune          # 清悬空镜像（<none> 标签）
docker image prune -a       # 清所有没被容器引用的镜像
docker builder prune -a     # 清构建缓存（最占地方，常被忽略）
docker rmi <镜像ID或name:tag>
docker system prune -a      # 一把梭：镜像+容器+网络+缓存（卷默认不动）
```

排查「磁盘莫名满了」的顺序：`docker system df -v` → 看 `RECLAIMABLE` 列 → 针对性 `prune`。

## 6. 构建时走宿主机代理

容器内构建常常拉不到外网，需要把宿主机的代理地址注入 build：

```bash
# 宿主机监听 0.0.0.0:7897 时，容器里用 172.17.0.1（docker0 网关）
docker build \
  --build-arg http_proxy=http://172.17.0.1:7897 \
  --build-arg https_proxy=http://172.17.0.1:7897 \
  -t <镜像名>:<tag> .
```

脚本版（自动探测网关，兼容网关不固定的机器）：

```bash
PROXY_IP=$(ip route show | grep docker0 | awk '{print $9}')
[ -z "$PROXY_IP" ] && PROXY_IP="172.17.0.1"

sudo docker build \
  --add-host=host.docker.internal:${PROXY_IP} \
  --build-arg http_proxy=http://${PROXY_IP}:7897 \
  --build-arg https_proxy=http://${PROXY_IP}:7897 \
  -t <镜像名>:<tag> .
```

前提：宿主机代理必须允许来自 docker 网段的连接（Clash 需开 `Allow LAN`），否则容器报 `Connection refused`。
