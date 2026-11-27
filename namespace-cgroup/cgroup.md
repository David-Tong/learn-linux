# Linux cgroup：管理进程的资源使用

cgroup（Control Group）是 Linux 内核提供的进程分组与资源管理机制。它可以统计一组进程的资源使用，并通过控制器限制或分配 CPU、内存、进程数等资源。容器通常用 [namespace](namespace.md) 隔离资源视图，用 cgroup 管理资源用量。

## 核心概念

- **cgroup**：进程所属的资源管理组；子进程通常继承父进程所属的组。
- **层级**：组构成树形结构，子组受到祖先组的限制，不能突破祖先的资源上限。
- **控制器**：针对某类资源的机制，例如 `cpu`、`memory`、`io` 和 `pids`。
- **控制文件**：通过 cgroup 文件系统读取统计或设置限制，例如 `cpu.max` 和 `memory.current`。

cgroup v1 为不同控制器提供独立层级；v2 使用统一层级和一致的接口。本文仅实践 **cgroup v2**。限制不等于预留：例如 CPU 上限为半核，并不保证繁忙时一定能获得半核。

## 查看当前环境

在 Linux 上执行：

```bash
stat -fc %T /sys/fs/cgroup
cat /proc/self/cgroup
cat /sys/fs/cgroup/cgroup.controllers
```

第一条输出 `cgroup2fs` 表示该挂载点使用 v2；`/proc/self/cgroup` 中形如 `0::/some/path` 的行给出当前进程在 v2 层级中的路径。`cgroup.controllers` 列出当前节点可供其子组启用的控制器，具体内容因环境而异。

## 实践：限制 CPU 并观察统计

以下示例需要 root、可写的 cgroup v2 挂载点以及可用的 `cpu` 控制器。请在专用测试虚拟机运行：它会在根 cgroup 启用 CPU 控制器并创建临时组；某些 systemd 管理的系统、容器或只读挂载环境不允许直接修改。不要在生产环境中直接修改 systemd 管理的层级。

先进入 root shell，检查前提：

```bash
sudo -i
stat -fc %T /sys/fs/cgroup
cat /sys/fs/cgroup/cgroup.controllers
cat /sys/fs/cgroup/cgroup.subtree_control
```

确认第一条输出为 `cgroup2fs`，且 `cgroup.controllers` 包含 `cpu`。下面启用根节点对子组的 CPU 控制，并建立实验组：

```bash
cd /sys/fs/cgroup
echo +cpu > cgroup.subtree_control
mkdir learn-cgroup
cat learn-cgroup/cpu.max
```

若 `mkdir` 提示目录已存在，请先检查它是否属于其他任务，不要继续复用。默认 `cpu.max` 常为 `max 100000`，表示没有 CPU 配额上限。设置限制：

```bash
echo '50000 100000' > learn-cgroup/cpu.max
cat learn-cgroup/cpu.max
```

这表示每 100000 微秒的周期内，该组所有进程合计最多使用 50000 微秒的 CPU 时间，长期平均约为一个逻辑 CPU 的 50%。这是总配额，不是每个进程各 50%。

启动一个持续占用 CPU 的任务，并在其启动实际工作前将自身加入实验组：

```bash
bash -c 'echo "$BASHPID" > /sys/fs/cgroup/learn-cgroup/cgroup.procs; exec yes > /dev/null' &
job=$!
cat learn-cgroup/cgroup.procs
cat learn-cgroup/cpu.stat
```

等待几秒后再次读取 `cpu.stat`；`usage_usec` 会增长，`nr_throttled` 和 `throttled_usec` 在触发配额限制时通常也会增长。调度和系统负载会影响具体数值。

清理实验任务与目录：

```bash
kill "$job"
wait "$job" 2>/dev/null
rmdir /sys/fs/cgroup/learn-cgroup
exit
```

如果删除目录失败，先检查 `learn-cgroup/cgroup.procs` 中是否还有实验进程并停止它们。实验不会自动撤销根节点启用的 CPU 控制器；如果它在实验前未启用且确认没有其他使用者，管理员可另行评估是否执行 `echo -cpu > /sys/fs/cgroup/cgroup.subtree_control`。

## 其他常用控制文件

以下文件是否存在取决于父组是否启用了对应控制器。它们作用于所在 cgroup 及其后代，具体语义请参考内核文档。

| 文件 | 用途 | 示例 |
| --- | --- | --- |
| `cpu.max` | CPU 带宽上限 | `50000 100000`，约半个逻辑 CPU |
| `cpu.weight` | 同级组竞争 CPU 时的相对权重 | `100` 为默认值，不是百分比 |
| `memory.current` | 当前内存用量 | 只读，单位为字节 |
| `memory.max` | 内存硬上限 | `268435456` 表示 256 MiB；无法回收足够内存时可能触发组内 OOM |
| `memory.events` | 内存限制和 OOM 等事件计数 | 查看 `max`、`oom`、`oom_kill` |
| `pids.max` | 进程和线程数量上限 | `100`；达到上限时创建新任务会失败 |
| `io.max` | 指定设备的 I/O 速率上限 | 按设备主次设备号设置读写 BPS 或 IOPS |

控制器先由父组在 `cgroup.subtree_control` 中启用，子组才会出现相应控制文件。非根组启用域控制器时还需遵守不能同时容纳进程并向子组分配资源等规则；在 systemd 系统中，应优先通过 systemd 的资源控制和委托机制管理服务或子树。

## 参考资料

- [Linux 内核：Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html)
