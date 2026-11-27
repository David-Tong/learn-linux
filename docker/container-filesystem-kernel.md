# 容器的文件系统、镜像与宿主机内核兼容性

容器中的 Ubuntu 用户空间可以运行在另一种发行版的 Linux 主机上，但容器通常不带自己的运行内核。要理解这种组合，需要分别看**文件从哪里来**、**修改存在哪里**，以及**用户程序怎样请求内核提供功能**。本文以 Linux 上的 Docker Engine 为例，接续 [Ubuntu Docker 教程](ubuntu-docker.md)。

## 先回答几个问题

- 在运行中的容器内使用 `apt install`，通常修改的是该容器的可写层，**不会改变原始镜像**。
- **停止或重启同一个容器不会删除可写层**；删除容器后，其可写层才会随之删除。使用 `docker run --rm` 时，容器退出后会被自动删除，因此可写层也会消失。
- 容器看到的文件并不全部来自镜像：可写层、卷、绑定挂载、`/proc`、`/sys` 等也会参与组成它的文件系统视图。
- 镜像提供的 glibc 等用户空间需要与实际运行的 Linux 内核 ABI 兼容。Linux 努力保持已有用户空间接口的向后兼容，但较新的用户空间可能依赖旧内核没有提供的功能；容器并不会自动解决这一问题。

## 镜像、容器和可写层

镜像包含文件系统层和启动配置等元数据。Dockerfile 通常从基础镜像开始，用 `RUN`、`COPY` 等指令构建新的镜像；基础镜像本身也提供文件。镜像可以由 Dockerfile 构建，但并不是所有镜像都必须由 Dockerfile 生成。

Docker 根据镜像创建容器时，会为这个容器提供独立的可写层。以常见的分层存储为例，可以这样理解：

```text
容器看到的根文件系统
┌───────────────────────────────┐
│ 容器自己的可写层：运行时修改  │
├───────────────────────────────┤
│ 镜像的只读文件系统层          │
└───────────────────────────────┘
```

容器修改已有镜像文件时，存储驱动通常通过写时复制等机制在可写层记录变化；新增文件也写入可写层。具体实现因存储后端而异。删除镜像中的文件只会在该容器视图中隐藏它，底层镜像不会被改写。同一镜像创建的不同容器拥有各自的可写层。

以下表格假设文件写在普通的容器可写层中，而不是挂载目录：

| 操作 | 文件是否保留 | 说明 |
| --- | --- | --- |
| `docker stop` 然后 `docker start` 同一容器 | 保留 | 容器和其可写层仍存在；内存中的进程状态不会保留 |
| `docker restart` | 保留 | 同一个容器停止后重新启动 |
| `docker rm` 删除容器 | 不保留 | 容器可写层被删除，原镜像不变 |
| `docker run --rm` 创建的容器退出 | 不保留 | Docker 自动删除该容器及其可写层 |
| 从同一镜像新建另一个容器 | 不会看到第一个容器的修改 | 新容器有自己的可写层 |

因此，上一篇中在 `learn-ubuntu` 里安装的 `procps` 和 `iproute2`，停止并重新启动**同一个**容器后通常仍在；执行 `docker rm -f learn-ubuntu` 删除容器后就不在了。重新从原 `ubuntu:24.04` 镜像创建容器，也不会自动获得这些安装。

### 实验：停止与删除的区别

在 Ubuntu 宿主机上执行。需要 Docker 和可用的 `ubuntu:24.04` 镜像；名称已存在时请先检查，或改用其他名称。

```bash
sudo docker run -d --name layer-demo ubuntu:24.04 sleep infinity
sudo docker exec layer-demo sh -c 'echo hello > /tmp/layer-demo.txt'
sudo docker stop layer-demo
sudo docker start layer-demo
sudo docker exec layer-demo cat /tmp/layer-demo.txt
```

最后一条命令应输出 `hello`：停止没有删除容器的可写层。下面在原镜像创建的**另一个**容器中检查同一路径：

```bash
sudo docker run --rm ubuntu:24.04 sh -c 'if test -e /tmp/layer-demo.txt; then echo 存在; else echo 不存在; fi'
```

通常输出 `不存在`。完成后删除第一个实验容器及其可写层：

```bash
sudo docker rm -f layer-demo
```

## 如果想保留修改

根据需要选择保存方式：

| 目标 | 常见方式 | 生命周期和用途 |
| --- | --- | --- |
| 让以后创建的容器都自带工具或程序 | 编写 Dockerfile 并构建新镜像 | 安装步骤可复现；新容器从新镜像开始 |
| 保存数据库或应用产生的数据 | 命名卷 | 数据独立于容器可写层；删除容器一般不会删除命名卷，但删除卷会删除其数据 |
| 与宿主机共享源码或配置 | bind mount（绑定挂载） | 容器使用宿主机指定路径；更改会影响该路径上的数据 |
| 临时查看容器状态 | 可写层 | 适合不需要在删除容器后保留的修改 |

例如，在一个空目录创建 `Dockerfile`：

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    procps iproute2 util-linux iputils-ping \
    && rm -rf /var/lib/apt/lists/*
CMD ["bash"]
```

在该目录构建并运行：

```bash
sudo docker build -t ubuntu-tools:demo .
sudo docker run --rm -it ubuntu-tools:demo
```

这里安装工具发生在**构建镜像时**，新镜像包含安装结果；以后从该镜像创建的容器也能使用这些工具。`--rm` 只在退出时删除本次容器，不删除 `ubuntu-tools:demo` 镜像。实际项目可固定基础镜像版本或摘要，并结合软件源和包版本管理可复现性。

`docker commit` 也可以将现有容器的文件系统变化保存为新镜像，但挂载卷中的数据不会被包含；它不如 Dockerfile 容易审查和复现，通常适合临时实验。

## 容器看到的文件系统从哪里来

容器的路径视图由镜像和运行时配置共同决定，不能只看 Dockerfile：

| 来源 | 例子 | 特点 |
| --- | --- | --- |
| 镜像层 | `/bin`、应用程序、glibc、镜像内的配置 | 提供初始用户空间文件 |
| 容器可写层 | 在普通路径下新建或修改的文件 | 每个容器独立，删除容器时删除 |
| 命名卷 | `--mount type=volume,source=data,target=/data` | Docker 管理的外部存储；可跨容器生命周期保留 |
| 绑定挂载 | `--mount type=bind,source=/host/path,target=/data` | 将 Docker 守护进程所在主机的路径挂入容器 |
| 内核虚拟文件系统 | `/proc`、`/sys`，以及常见的 `/dev` | 由内核和运行时提供，内容受挂载、namespace 和权限影响 |
| 运行时提供的配置 | 常见的 `/etc/hosts`、`/etc/resolv.conf` | Docker 可能为容器提供或挂载这些文件，具体依配置而异 |
| tmpfs | 使用 `--mount type=tmpfs` 的路径 | 内存支持的临时挂载；容器停止或重启后数据不保留 |

挂载会遮住目标路径上原先可见的内容；写入挂载点通常写到挂载的存储，而不是容器可写层。例如在 `/data` 挂载卷后，往 `/data` 写的数据保存在卷中。卷是否删除还取决于卷类型和删除选项；不要把所有挂载都视为永久保存。

## 用户空间和内核的分工

Linux 主机上的普通 Linux 容器共享运行 Docker Engine 的 Linux 内核。镜像中的 `/etc/os-release`、shell、应用、动态链接器和 glibc 属于**用户空间**，不是正在运行的内核。即使镜像里有名为 `vmlinuz` 的文件，它也不会因为容器启动而自动成为该容器的内核。

```text
容器进程：应用 → glibc 等用户空间库 → Linux 系统调用 ABI
                                              ↓
                              实际运行的 Linux 内核
                                              ↓
                                     硬件、文件系统等
```

例如 Ubuntu 用户空间可能运行在 Debian 或其他 Linux 发行版的宿主机内核上。`cat /etc/os-release` 反映镜像中的发行版信息，`uname -r` 通常显示实际运行内核的版本信息；容器运行时也可以通过配置改变 UTS 相关显示，因此不能仅凭显示字符串证明某项内核功能可用。在 Docker Desktop 或某些 WSL 环境中，实际内核位于 Linux 虚拟机或 WSL 环境中，而非直接是 Windows/macOS 内核。

在上一篇创建的 `learn-ubuntu` 容器运行时，可以在宿主机执行：

```bash
uname -r
sudo docker exec learn-ubuntu uname -r
sudo docker exec learn-ubuntu cat /etc/os-release
sudo docker exec learn-ubuntu getconf GNU_LIBC_VERSION
```

通常两次 `uname -r` 一致，而 `/etc/os-release` 和 glibc 版本来自容器镜像。`getconf GNU_LIBC_VERSION` 适用于这里的 glibc 环境；Alpine 等使用其他 C 库的镜像不一定支持该查询。

## glibc 与系统调用如何配合

应用调用的是用户空间 API，glibc 等库可能将它转换为一个或多个 Linux 系统调用；也可能使用用户空间实现或 vDSO 等机制，无需每次都陷入内核。应用还可能绕过 glibc 封装，直接调用 `syscall()` 或使用其他语言运行时。因此，实际使用哪些系统调用由程序、库、架构和执行路径共同决定。

系统调用 ABI 规定系统调用号、参数传递方式、返回值和数据布局等。glibc 根据目标架构遵循相应约定，内核再实现这些接口。Linux 对已有内核到用户空间接口提供很强的向后兼容性：**旧用户空间在较新内核上通常可以工作**。这不是保证任意组合都能运行；而且反过来，旧内核不可能实现发布之后才出现的所有功能。

一种可能的过程是：

1. 新版本应用或 glibc 尝试使用某个较新的系统调用。
2. 运行中的旧内核没有实现该调用，通常返回 `ENOSYS`（功能未实现）。
3. 如果程序或库实现了兼容回退，它可能改用旧接口；如果没有，操作就会失败，甚至程序无法启动。回退是否存在取决于具体版本和调用，并不保证所有功能都有替代方案。

此外，发行版可能对内核功能做回移植，因此不能仅根据 `uname -r` 中的版本号判断；内核编译选项、架构和配置也影响功能。容器的 seccomp 策略、capability、LSM 或权限可能拒绝某个调用或操作，即使内核本身支持，也可能得到 `EPERM`、`EACCES` 等错误。过滤器甚至可以配置为返回 `ENOSYS`，因此看到一个错误码也不能单独证明内核缺少该调用。

## 不兼容时会出现什么，怎么处理

| 现象或原因 | 排查与处理方向 |
| --- | --- |
| 用户空间依赖的系统调用或功能在内核中不存在 | 查看镜像、glibc 和应用的最低内核要求；升级宿主机内核，或选择明确支持当前内核的镜像和应用版本 |
| 系统调用存在，但被 seccomp、权限或安全策略拒绝 | 查看 Docker 配置、审计日志和具体错误；针对需求调整策略，不要仅为排错就长期关闭所有安全限制 |
| 应用依赖特定的 `/proc`、`/sys`、设备或内核模块 | 检查内核配置、设备挂载和权限；镜像不能自行提供宿主机正在使用的驱动或内核模块 |
| 程序与镜像内 glibc 或动态链接器不兼容，例如缺少 `GLIBC_...` 版本 | 检查程序的构建环境和镜像中的库；更换匹配的基础镜像或重新构建程序。这是用户空间库之间的兼容性问题，未必是内核问题 |
| CPU 架构不匹配 | 选择对应架构的镜像；有适当配置时可使用 QEMU 用户态仿真，但它仍依赖实际 Linux 内核，不等于启动了另一个内核 |
| 程序必须使用特定内核或内核配置 | 使用具有所需内核的虚拟机或合适的宿主机；普通容器不能替换运行中的宿主机内核 |

检查时应同时记录宿主机内核、CPU 架构、镜像的发行版和应用依赖，以及容器的安全配置。`strace` 可以帮助观察实际系统调用和返回错误，但可能受 ptrace 权限及容器安全策略限制；输出也要结合上下文分析。不要只因为升级镜像后程序失败，就认定一定是 glibc 或内核版本问题。

## 参考资料

- [Docker：容器存储层](https://docs.docker.com/engine/storage/drivers/)
- [Docker：存储、卷和挂载](https://docs.docker.com/engine/storage/)
- [Docker：多平台镜像](https://docs.docker.com/build/building/multi-platform/)
- [Linux 内核：用户空间 ABI](https://docs.kernel.org/admin-guide/abi.html)
- [syscall(2)：系统调用接口](https://man7.org/linux/man-pages/man2/syscall.2.html)
- [Docker：seccomp 安全配置](https://docs.docker.com/engine/security/seccomp/)
