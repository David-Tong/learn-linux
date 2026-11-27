# 在 Ubuntu 上安装和运行 Docker

本文介绍在 Ubuntu 上通过 Docker 官方 APT 软件源安装 Docker Engine，并运行、管理容器。示例在 Ubuntu 的 Bash 终端中执行，使用 `sudo` 访问默认的 Docker 守护进程。建议先在测试机上练习。

## Docker 的几个概念

- **镜像（image）**：创建容器所需的文件和配置的只读模板，例如 `nginx`。
- **容器（container）**：由镜像创建的运行实例。一个镜像可以创建多个容器。
- **Docker Engine**：包括负责管理容器的守护进程 `dockerd`、与之通信的 `docker` 命令行工具等组件。
- **镜像仓库（registry）**：存放和分发镜像的服务，例如 Docker Hub。
- **卷（volume）**：由 Docker 管理的持久化存储，可供容器使用。

容器通常共享宿主机内核，通过 namespace 和 cgroup 等机制实现隔离和资源控制。相关原理见 [容器运行时如何使用 namespace 和 cgroup](../namespace-cgroup/container-runtime.md)。

## 安装前检查

本文安装的是 Linux 上的 Docker Engine，而不是 Docker Desktop。需要受 Docker 官方支持的 Ubuntu 版本和 CPU 架构、具有 sudo 权限的账户，以及能够访问 Docker 软件源和镜像仓库的网络。支持列表会变化，请以 [Docker 官方 Ubuntu 安装文档](https://docs.docker.com/engine/install/ubuntu/) 为准。

```bash
cat /etc/os-release
dpkg --print-architecture
```

Ubuntu 的衍生发行版和 WSL 环境可能需要额外配置；本文假设是使用 systemd 的 Ubuntu 主机。若系统已有 Docker、containerd 或 Kubernetes 等工作负载，先确认其安装方式和依赖，再处理冲突，避免中断现有服务。

Docker 官方列出的潜在冲突包包括 `docker.io`、`docker-compose`、`docker-compose-v2`、`docker-doc`、`docker-buildx`、`podman-docker`、`containerd` 和 `runc`。例如，使用以下命令检查已安装的相关包：

```bash
dpkg -l docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc 2>/dev/null
```

列表中以 `ii` 开头的行表示包已安装。确认不再需要且了解其他服务的依赖后，再按官方文档移除冲突包；不要在已有工作负载的机器上直接批量卸载。卸载软件包通常不会自动删除已有镜像、容器和卷。

## 配置官方软件源并安装

添加 Docker 的签名密钥和 APT 软件源：

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

这里使用系统版本代号和架构选择对应的软件包，`Signed-By` 指定验证软件源签名的密钥。如果 `apt update` 提示仓库不支持当前版本，先检查版本和官方支持列表，不要随意替换为其他版本代号。

安装 Docker Engine、命令行工具、containerd、Buildx 和 Compose 插件：

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Ubuntu 上安装后服务通常会自动启动。检查版本和服务状态：

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager
```

如果服务没有运行，可以启动它：

```bash
sudo systemctl start docker
```

## 运行第一个容器

```bash
sudo docker run --rm hello-world
```

Docker 在本地没有镜像时会尝试拉取它，创建并运行容器。看到欢迎信息后，测试程序退出；`--rm` 会在退出时删除这个容器，不会删除镜像。这个例子同时验证了客户端、守护进程和拉取镜像的基本流程。

## 运行一个 Web 服务

启动一个后台 Nginx 容器：

```bash
sudo docker run -d --name learn-nginx -p 127.0.0.1:8080:80 nginx
```

- `-d`：后台运行。
- `--name learn-nginx`：为容器指定便于管理的名称。
- `-p 127.0.0.1:8080:80`：将宿主机回环地址的 TCP 8080 端口映射到容器的 80 端口。
- `nginx`：使用的镜像；未指定标签时使用默认标签 `latest`。

在 Ubuntu 主机上访问：

```bash
curl http://127.0.0.1:8080/
sudo docker ps
sudo docker logs learn-nginx
```

正常时 `curl` 返回 Nginx 欢迎页面的 HTML。若名称已被占用，先确认旧容器用途，再换名称或删除自己的旧实验；若 8080 端口被占用，可以将映射左侧改为其他端口。

这里显式绑定 `127.0.0.1` 供本机访问。如果使用 `-p 8080:80`，默认会发布到宿主机所有地址，可能被外部访问。Docker 的端口发布规则可能绕过 Ubuntu 的 UFW 规则；需要对外提供服务时，应结合 Docker 防火墙文档和实际网络检查访问控制。Docker 28.0.0 之前的版本还可能让同一二层网络上的主机访问发布到 localhost 的端口。

## 进入容器并检查 namespace

下面单独创建一个 Ubuntu 实验容器，并安装 `ps`、`ip` 等工具。标准 Ubuntu 镜像不一定预装这些命令；安装需要容器能访问 Ubuntu 软件源。这个实验与前面的 Nginx 容器相互独立。

在宿主机启动容器：

```bash
sudo docker run -d --name learn-ubuntu --hostname learn-container ubuntu:24.04 sleep infinity
sudo docker ps
sudo docker exec -it learn-ubuntu bash
```

`-d` 让容器在后台运行；`sleep infinity` 作为容器主进程，使其保持运行。`docker exec` 在运行中的容器内启动额外进程，`-i` 保持标准输入打开，`-t` 分配终端。执行最后一条命令后，以下命令均在**容器内**运行。

先安装工具：

```bash
apt update
apt install -y procps iproute2 util-linux iputils-ping
```

其中 `procps` 提供 `ps`，`iproute2` 提供 `ip`，`util-linux` 提供 `lsns`、`findmnt` 等工具，`iputils-ping` 提供 `ping`。这些安装发生在当前容器的可写层，删除容器后不会保留，也不会修改原始镜像。

### 查看进程、网络和文件系统

```bash
id                               # 容器内当前用户和组
cat /etc/os-release              # 镜像中的发行版信息
ps -ef                           # 当前 PID namespace 可见的进程
ip addr                          # 网络接口和地址
ip route                         # 路由表
hostname                         # UTS namespace 中的主机名
findmnt                          # 当前进程看到的挂载树
cat /proc/1/cgroup               # PID 1 的 cgroup 成员信息
```

`ps` 中的 PID 1 通常是此例的 `sleep infinity`；`docker exec` 启动的 Bash 是额外的进程。容器里的 `ip addr` 和 `ip route` 展示其网络 namespace 的视图，实际地址由 Docker 网络配置决定。`cat /etc/resolv.conf` 可以查看容器使用的 DNS 配置。

### 查看 namespace

```bash
ls -l /proc/1/ns                 # 容器主进程的 namespace
ls -l /proc/self/ns              # 执行 ls 的进程的 namespace
lsns                             # 列出当前可见的 namespace
cat /proc/self/uid_map           # user namespace 的 UID 映射
cat /proc/self/gid_map           # user namespace 的 GID 映射
cat /proc/self/mountinfo         # mount namespace 中的挂载信息
```

链接形式如 `net:[402653xxxx]`：对同一种 namespace 类型，编号相同表示共享，编号不同表示不共享。`lsns` 的结果受当前 PID namespace、权限和 `/proc` 挂载方式影响，未列出并不意味着宿主机上不存在其他 namespace。UID/GID 映射的三列分别是当前 user namespace 起始 ID、父 namespace 起始 ID 和映射数量；是否启用独立 user namespace 取决于 Docker 配置。

输入 `exit` 返回**宿主机**；这只结束 `docker exec` 启动的 Bash，`sleep` 仍在运行。接下来从宿主机比较容器主进程和宿主机 PID 1 的 namespace：

```bash
pid=$(sudo docker inspect --format '{{.State.Pid}}' learn-ubuntu)
echo "容器主进程的宿主机 PID: $pid"
sudo ls -l /proc/"$pid"/ns
sudo ls -l /proc/1/ns
```

逐项比较 `pid`、`net`、`mnt`、`uts`、`ipc`、`user` 和 `cgroup` 等同名链接的编号。默认配置下，一些类型会隔离，另一些可能共享；结果取决于 Docker、内核和启动选项，不能假设所有类型都不同。容器进程在宿主机上有自己的 PID，而在容器的 PID namespace 中看到的是 PID 1。容器退出后，其宿主机 PID 和 `/proc` 目录可能不再有效。

也可以在宿主机直接执行单条容器内命令，或检查 Docker 记录的挂载配置：

```bash
sudo docker exec learn-ubuntu ip addr
sudo docker inspect --format '{{json .Mounts}}' learn-ubuntu
```

`docker exec` 要求容器主进程正在运行；若容器退出，可先使用 `sudo docker ps -a` 和 `sudo docker logs learn-ubuntu` 排查。实验结束后，在宿主机删除实验容器：

```bash
sudo docker rm -f learn-ubuntu
```

更深入的机制和手动实验见 [Linux Namespace](../namespace-cgroup/namespace.md)。

## 管理容器和镜像

```bash
sudo docker ps                    # 运行中的容器
sudo docker ps -a                 # 包括已停止的容器
sudo docker logs learn-nginx      # 查看输出日志
sudo docker inspect learn-nginx   # 查看容器详细配置和状态
sudo docker exec -it learn-nginx sh  # 在运行中的容器内启动 shell
sudo docker stop learn-nginx      # 停止容器
sudo docker start learn-nginx     # 再次启动容器
sudo docker images                # 查看本地镜像
```

在容器 shell 中执行 `exit` 只退出该 shell，通常不会停止容器主进程。停止容器也不会删除它；若要结束这个实验：

```bash
sudo docker stop learn-nginx
sudo docker rm learn-nginx
```

容器可写层适合临时数据：删除容器后，其中没有放在卷或绑定挂载中的数据会丢失。镜像仍保留在本地，可使用 `sudo docker image rm nginx` 删除不再使用的镜像；如果镜像仍被容器引用，需先处理这些容器。

## 使用卷保存数据

以下例子创建一个命名卷，写入数据，然后用另一个容器读取，说明容器删除后卷仍然存在：

```bash
sudo docker volume create learn-data
sudo docker run --rm -v learn-data:/data alpine sh -c 'echo hello > /data/message.txt'
sudo docker run --rm -v learn-data:/data alpine cat /data/message.txt
```

最后一条命令输出 `hello`。`-v learn-data:/data` 将卷挂载到容器的 `/data`，两个容器可以先后使用同一个卷。完成后，确认不再需要其中的数据，再删除实验卷：

```bash
sudo docker volume rm learn-data
```

卷不等于备份；重要数据仍需按照业务需求备份和验证恢复。

## 可选：不使用 sudo 运行 Docker

默认情况下，Docker 守护进程以 root 身份运行，其 Unix socket 由 root 和 `docker` 组控制。可以将当前用户加入该组：

```bash
sudo groupadd --force docker
sudo usermod -aG docker "$USER"
```

退出登录并重新登录后运行 `id -nG`，确认组列表包含 `docker`，再执行 `docker run --rm hello-world`。也可以在当前终端运行 `newgrp docker` 启动具有新组身份的子 shell，输入 `exit` 返回原 shell。

**`docker` 组提供相当于 root 的权限**，只应授予可信用户。加入该组不等于 rootless 模式；如果希望守护进程以非 root 用户运行，请参考 [Docker rootless 模式文档](https://docs.docker.com/engine/security/rootless/)。本文其他示例使用 `sudo`，已配置组权限后可省略它。

## 常见问题

- **无法连接 Docker daemon**：先用 `sudo systemctl status docker --no-pager` 检查服务；查看 `sudo journalctl -u docker -n 100 --no-pager` 了解启动错误。若只有不带 sudo 时失败，检查 socket 权限和组成员身份。
- **拉取镜像失败**：检查网络、DNS、代理、镜像名称和仓库权限。避免未经评估地使用不可信镜像源。
- **端口被占用**：使用 `sudo ss -ltnp` 检查监听端口，或者选择其他宿主机端口。
- **容器启动后立即退出**：运行 `sudo docker ps -a` 和 `sudo docker logs <容器名>`；容器通常在主进程退出后停止。
- **磁盘空间不足**：使用 `docker system df` 查看用量，先确认哪些镜像、容器和卷不再需要，再有选择地清理。不要在不了解影响时运行会批量删除数据的清理命令。

## 更新和参考资料

配置好官方源后，可通过 `sudo apt update` 和 `sudo apt upgrade` 按系统的软件更新流程升级；生产环境应先评估版本兼容性和服务重启影响。

- [Docker：在 Ubuntu 安装 Docker Engine](https://docs.docker.com/engine/install/ubuntu/)
- [Docker：Linux 安装后的配置](https://docs.docker.com/engine/install/linux-postinstall/)
- [Docker：容器端口发布](https://docs.docker.com/engine/network/port-publishing/)
- [Docker：防火墙和数据包过滤](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
- [Docker：存储卷](https://docs.docker.com/engine/storage/volumes/)
