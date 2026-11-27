# Linux Namespace：进程眼中的独立世界

Namespace（命名空间）是 Linux 内核提供的隔离机制。它让不同进程对某类系统资源看到不同的视图：例如，两个进程可以看到不同的主机名、进程列表或网络设备。容器通常利用多个 namespace 组合出隔离的运行环境。

Namespace 隔离的是资源的**视图**，并不限制资源用量；CPU、内存等资源的限制由 [cgroup](cgroup.md) 等机制负责。容器仍然共享宿主机内核，namespace 也不是完整的安全边界。

## 常见类型

| 类型 | 隔离的内容 | 例子 |
| --- | --- | --- |
| Mount | 挂载点视图 | 容器有自己的 `/proc`、文件系统挂载 |
| PID | 进程编号和进程树视图 | 容器内的首个进程看到自己是 PID 1 |
| Network | 网络设备、地址、路由、端口等 | 不同容器可分别监听同一端口 |
| UTS | 主机名和 NIS 域名 | 容器有自己的 hostname |
| IPC | System V IPC、POSIX 消息队列等 | 隔离共享内存、信号量等 |
| User | 用户和组 ID、相关权限 | namespace 内的 root 映射为宿主机普通用户 |
| Cgroup | cgroup 路径视图 | 容器看到相对自己的 cgroup 路径 |
| Time | 部分时钟的偏移 | 调整单调时钟等的视图 |

一个进程同时属于多种 namespace；不同进程可以共享其中一些，也可以各自拥有独立的 namespace。PID namespace 可以嵌套：同一进程在不同层级可能有不同 PID，父级通常能看到子级的进程。

## unshare 命令

`unshare` 用于创建新的 namespace，并在其中运行指定程序。它的基本格式是：

```bash
unshare [选项] [程序及参数]
```

例如，`sudo unshare --uts /bin/bash` 创建一个新的 UTS namespace，然后在其中启动 Bash。新 shell 修改主机名时，修改作用于这个 UTS namespace，不会改变原来 UTS namespace 中的主机名。创建 namespace 通常需要相应权限；user namespace 在系统允许的情况下可以由普通用户创建。

常用选项如下，可以组合使用：

| 选项 | 作用 |
| --- | --- |
| `--uts` | 创建 UTS namespace，隔离主机名等 |
| `--pid` | 创建 PID namespace，隔离进程编号和进程视图 |
| `--mount` | 创建 mount namespace，隔离挂载点视图 |
| `--net` | 创建 network namespace，隔离网络设备、地址、路由等 |
| `--user` | 创建 user namespace，隔离用户和组 ID 等 |
| `--ipc` | 创建 IPC namespace，隔离共享内存、信号量等 |
| `--cgroup` | 创建 cgroup namespace，隔离 cgroup 路径视图 |
| `--time` | 创建 time namespace，隔离部分时钟的偏移 |
| `--fork` | 先创建子进程，再在子进程中运行指定程序 |
| `--mount-proc` | 挂载新的 `/proc`，同时创建 mount namespace |
| `--map-root-user` | 在新 user namespace 中将当前用户映射为 UID 0 |

例如，`unshare --pid --fork --mount-proc /bin/bash` 中，`--fork` 让 Bash 成为新 PID namespace 的首个进程，`--mount-proc` 则让 `ps` 等工具通过对应的 `/proc` 查看进程。下面的实践会分别演示这些选项。

`unshare` 不会自动配置容器网络、准备完整文件系统，也不会限制 CPU 或内存用量；资源限制通常由 cgroup 等机制配合实现。

## 实践准备

以下命令在 Linux 的 Bash 中运行，使用 `util-linux` 提供的 `unshare`、`lsns`、`nsenter` 等工具。建议在测试虚拟机中操作。不同发行版可能限制普通用户创建 user namespace；若出现 `Operation not permitted`，请检查系统策略，或在可信的测试环境中使用 root。容器内也可能受到权限和安全策略限制。

### 查看当前 namespace

```bash
ls -l /proc/$$/ns
lsns
```

`$$` 是当前 shell 的 PID。`/proc/<pid>/ns` 中的链接如 `uts:[4026531838]` 标识 namespace；相同类型、相同编号表示进程共享该 namespace。`lsns` 的可见结果受权限和当前环境影响。

### 创建 UTS namespace

创建一个新的 UTS namespace 并打开 shell：

```bash
sudo unshare --uts /bin/bash
```

- `sudo`：以 root 权限运行，以便创建 UTS namespace。
- `unshare`：创建指定的 namespace，并运行后面的程序。
- `--uts`：创建新的 UTS namespace，提供独立的主机名视图。
- `/bin/bash`：在新的 namespace 中启动 Bash。

在新 shell 中修改并查看主机名：

```bash
hostname demo-ns
hostname
```

输出应为 `demo-ns`。保持这个 shell 打开，在宿主机的另一个终端执行 `hostname`，结果仍是原来的主机名。输入 `exit` 退出新 shell，namespace 在不再被进程或其他引用使用时会被销毁。

### 创建 PID namespace

```bash
sudo unshare --pid --fork --mount-proc /bin/bash
```

这条命令各部分的作用是：

- `sudo`：以 root 权限运行，以便创建 namespace 和挂载 `/proc`。
- `unshare`：创建指定的 namespace，并在其中运行程序。
- `--pid`：创建新的 PID namespace，提供独立的进程编号和进程视图。
- `--fork`：先创建子进程，再运行 Bash，使 Bash 成为新 PID namespace 中的首个进程。
- `--mount-proc`：挂载与新 PID namespace 对应的 `/proc`，供 `ps` 等工具读取；该选项也会创建 mount namespace，避免影响原来的 `/proc` 挂载。
- `/bin/bash`：在新环境中启动 Bash。

在新 shell 中运行：

```bash
echo $$
ps -ef
```

这个 shell 通常显示 PID 1，`ps` 只显示当前 PID namespace 可见的进程。从宿主机的 PID namespace 看，同一个 Bash 仍有一个宿主机中的 PID。PID 1 有特殊的信号处理和孤儿进程回收职责，因此真实容器需要妥善处理进程退出。

#### 进入已有的 PID namespace

保持上面的 shell 运行，在宿主机的另一个终端查找 `unshare` 进程：

```bash
ps -ef | grep '[u]nshare --pid'
```

根据输出找到本次实验的 `unshare` 进程 PID，再查看它的子进程：

```bash
ps --ppid <unshare的宿主机PID> -o pid,ppid,cmd
```

将尖括号及其中的文字替换为实际数字，不要输入尖括号。子进程列表中的 `/bin/bash` 就是实验 shell；记下其宿主机 PID。如果有多个实验，请根据父进程和命令确认目标。

```bash
sudo nsenter --target <Bash的宿主机PID> --pid --mount /bin/bash
```

同样将占位符替换为实际数字。`sudo` 提供进入 namespace 所需的权限；`nsenter` 进入已有 namespace；`--target` 指定用于定位 namespace 的宿主机进程；`--pid` 加入其 PID namespace；`--mount` 加入其 mount namespace，从而使用前面为该 PID namespace 挂载的 `/proc`；`/bin/bash` 是进入后运行的程序。

在新 shell 中运行 `echo $$` 和 `ps -ef`，可以查看自己的 PID 和进程列表。这个 Bash 通常不是 PID 1，原先创建的 Bash 才是该 namespace 的 PID 1。输入 `exit` 退出新 shell；再回到原实验 shell 输入 `exit`，结束实验。

### 观察 user namespace 的 ID 映射

```bash
id
unshare --user --map-root-user /bin/bash
```

第一条 `id` 显示进入前的身份。第二条命令创建 user namespace 并启动新 shell：

- `unshare`：创建指定的 namespace，并运行后面的程序。
- `--user`：创建新的 user namespace，提供独立的用户和组 ID 映射。
- `--map-root-user`：把调用者的有效用户和组 ID 映射为新 namespace 内的 UID 0 和 GID 0；它也隐含 `--user`。
- `/bin/bash`：在新 user namespace 中启动 Bash。

在新 shell 中查看身份和映射，完成后退出：

```bash
id
cat /proc/self/uid_map
cat /proc/self/gid_map
exit
```

第二次 `id` 通常显示 UID 0；映射文件的三列分别表示 namespace 内起始 ID、父 user namespace 中起始 ID、映射数量。例如 `0 1000 1` 表示内部 UID 0 对应外部 UID 1000。这里的 root 只在相应 user namespace 内拥有相应权限，不会因此成为宿主机 root。

### 进入已有 namespace

在一个终端启动实验 shell 并记下其 PID：

```bash
sudo unshare --uts /bin/bash
```

该命令与前面的 UTS 实践相同：`sudo` 提供权限，`unshare --uts` 创建新的 UTS namespace，`/bin/bash` 在其中启动 shell。在这个新 shell 中设置主机名并记录 PID：

```bash
hostname demo-ns
echo $$
```

在另一个终端，将下面的 `<PID>` 换成上一步输出的 PID：

```bash
sudo nsenter --target <PID> --uts hostname
```

应看到 `demo-ns`。这里仅加入 UTS namespace；加入其他类型需要相应选项及权限。完成后退出第一个终端里的实验 shell。

## 参考资料

- [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [unshare(1)](https://man7.org/linux/man-pages/man1/unshare.1.html)
- [nsenter(1)](https://man7.org/linux/man-pages/man1/nsenter.1.html)
