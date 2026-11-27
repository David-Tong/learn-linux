# 容器运行时如何使用 namespace 和 cgroup

[Namespace](namespace.md) 提供隔离后的资源视图，[cgroup](cgroup.md) 管理一组进程的资源用量。容器运行时把它们与根文件系统、网络配置和权限控制组合起来，启动容器进程。本文以 Linux 和 cgroup v2 为例，说明配置如何落到内核机制上。

## 从容器配置到进程

以 Docker Engine 为例，用户通过 `docker run` 指定镜像、命令、网络和资源限制。Docker 等上层组件准备镜像文件系统及网络等配置，并通过 containerd 等组件调用符合 OCI Runtime Specification 的低层运行时（例如 runc）。具体调用链会随产品和配置变化；OCI 规范描述运行时配置和行为，并不规定所有上层组件的实现。

一个典型的启动过程可以理解为：

1. 上层组件准备容器根文件系统和 OCI 配置，其中包含要运行的程序、挂载、namespace 和资源限制。
2. 低层运行时创建或加入配置指定的 namespace，准备根文件系统和挂载，并通过 cgroup 文件系统或 systemd 管理容器的 cgroup。
3. 运行时设置资源限制，将容器进程放入目标 cgroup，完成权限等设置后启动应用。网络设备、地址和路由通常还需要运行时或网络插件单独配置。
4. 容器中的子进程通常继承父进程的 namespace 和 cgroup；进程退出后，相关组件清理网络、挂载和 cgroup 等资源。

这只是便于理解的概括，实际操作顺序和分工因实现而异。容器中的应用仍是宿主机上的进程，共享宿主机内核。

## 运行时如何设置 namespace

OCI 配置的 `linux.namespaces` 指定 namespace 类型。指定类型但不提供 `path` 时，运行时创建新的 namespace；提供已有 namespace 的路径时，运行时加入它。未列出的类型沿用运行时所处的相应 namespace。运行时通常借助 `clone`、`unshare` 和 `setns` 等内核接口完成这些操作。

| 类型 | 容器中的典型用途 | 还需要做什么 |
| --- | --- | --- |
| PID | 让容器看到独立的进程编号和进程树 | 挂载对应的 `/proc`；容器内 PID 1 需要处理信号和孤儿进程 |
| Mount | 让容器拥有独立的挂载视图 | 准备 rootfs、切换根目录，并挂载 `/proc`、卷等 |
| Network | 提供独立的网络设备、路由和端口空间 | 配置接口、地址和路由；namespace 本身不会提供网络连接 |
| UTS、IPC | 分别隔离主机名和 IPC 资源 | 按需设置主机名等 |
| User | 映射容器内外的用户和组 ID | 配置 UID/GID 映射；是否启用取决于运行时和配置 |
| Cgroup | 改变进程看到的 cgroup 路径视图 | 仍需另外创建 cgroup 和设置资源限制 |

这些隔离可以按配置共享。例如 Docker 的 `--network host` 使用宿主机网络 namespace；容器不再有独立的网络设备、路由和端口空间。并不是所有容器都会启用独立的 user namespace。

## 运行时如何设置 cgroup

OCI 的 `linux.resources` 描述 CPU、内存、进程数等限制，`linux.cgroupsPath` 可指定 cgroup 路径。运行时或其 cgroup 管理器将配置转换为内核接口，在可用且已启用的控制器上写入限制，并让容器进程加入目标组。使用 systemd 时，cgroup 的创建和管理也可能经由 systemd；实际路径取决于 cgroup 驱动和系统配置。

| 用户指定的资源 | cgroup v2 中常见的对应文件 | 含义 |
| --- | --- | --- |
| CPU 配额 | `cpu.max` | 一段周期内整个组可使用的 CPU 时间上限 |
| 内存上限 | `memory.max` | 组的内存硬上限，无法回收足够内存时可能触发 OOM |
| 进程数上限 | `pids.max` | 组及后代可创建的任务数量上限，包括线程 |

父 cgroup 的限制同样约束容器；容器自己的限制不能突破祖先的上限。控制器的可用性、systemd 委托和 rootless 权限等会影响哪些设置能够生效。cgroup namespace 只改变路径的可见方式，本身不施加资源限制。

## 实践：观察 Docker 容器

以下示例需要一台使用 cgroup v2 的 Linux 测试机、可用的 Docker Engine，以及允许使用 Docker 的账户；首次运行可能需要拉取 `alpine` 镜像。命令在宿主机的 Bash 中执行。Docker Desktop 的 Linux 容器运行在虚拟机中，下面的宿主机 `/proc` 路径不一定适用。

创建一个后台容器，并设置 CPU、内存和任务数上限：

```bash
docker run -d --name ns-cgroup-demo --cpus=0.5 --memory=256m --pids-limit=64 alpine sleep 600
```

如果名称已存在，先确认该容器的用途，再选择其他名称或清理自己的旧实验。`--cpus=0.5` 设置约半个逻辑 CPU 的总配额；`--memory=256m` 设置内存上限；`--pids-limit=64` 限制任务数。这些是上限，不是资源预留。

查看容器主进程在宿主机上的 PID，并比较 namespace：

```bash
pid=$(docker inspect --format '{{.State.Pid}}' ns-cgroup-demo)
echo "$pid"
sudo ls -l /proc/"$pid"/ns
sudo ls -l /proc/1/ns
docker exec ns-cgroup-demo sh -c 'echo "容器内 PID: $$"; cat /proc/self/cgroup'
```

`/proc/<pid>/ns` 中同类型链接的编号相同表示共享该 namespace，不同表示属于不同的 namespace。`docker exec` 启动的是额外进程，输出的 PID 通常不是 1。容器内的 cgroup 路径可能因 cgroup namespace 而与宿主机看到的路径不同。

在宿主机上查找容器主进程的 cgroup，并读取限制：

```bash
cat /proc/"$pid"/cgroup
cg=$(awk -F: '$1 == "0" {print $3}' /proc/"$pid"/cgroup)
echo "$cg"
sudo cat "/sys/fs/cgroup${cg}/cpu.max"
sudo cat "/sys/fs/cgroup${cg}/memory.max"
sudo cat "/sys/fs/cgroup${cg}/pids.max"
```

这里假设宿主机的统一 cgroup v2 层级挂载在 `/sys/fs/cgroup`，并从宿主机的 cgroup namespace 观察。`cpu.max` 可能显示 `50000 100000`，`memory.max` 通常为 `268435456`，`pids.max` 为 `64`；具体 CPU 周期及路径取决于运行时配置。文件不存在或数值不符时，先检查 cgroup 版本、挂载位置、控制器及 Docker 的配置。容器退出后其 PID 和 cgroup 可能不再存在。

完成后删除实验容器：

```bash
docker rm -f ns-cgroup-demo
```

## 参考资料

- [OCI Runtime Specification：Linux 配置](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md)
- [Docker：资源限制](https://docs.docker.com/engine/containers/resource_constraints/)
- [Docker：运行时指标和 cgroup](https://docs.docker.com/engine/containers/runmetrics/)
- [Linux 内核：Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
