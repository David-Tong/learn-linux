# Linux 文件目录结构详解

Linux 将文件、设备和内核提供的信息组织在一棵以 `/` 为根的目录树中。磁盘分区、网络文件系统和虚拟文件系统都可以挂载到这棵树的某个目录下，因此目录与磁盘分区不是一一对应的。目录结构主要遵循文件系统层次标准（FHS）和发行版约定；下面介绍常见布局，实际内容会因发行版、安装的软件和系统配置而不同。

## 先看全貌

| 路径 | 主要用途 |
| --- | --- |
| `/` | 整棵目录树的根；也是根文件系统的挂载点 |
| `/etc` | 本机系统和服务的配置 |
| `/usr` | 大多数系统程序、库和共享数据 |
| `/var` | 运行中经常变化的持久数据，如日志、数据库和缓存 |
| `/run` | 本次启动期间的运行时状态 |
| `/home`、`/root` | 普通用户和 root 用户的个人目录 |
| `/boot` | 启动内核所需的文件 |
| `/dev` | 设备节点和部分特殊设备接口 |
| `/proc`、`/sys` | 内核提供的虚拟文件系统接口 |
| `/tmp`、`/var/tmp` | 临时文件 |
| `/opt` | 附加软件包及其静态文件 |
| `/mnt`、`/media`、`/srv` | 临时挂载、可移动介质、服务数据的常见位置 |

`/root` 是 root 用户的家目录，`/` 才是根目录。`/usr` 也不是用户家目录，普通用户的个人文件通常在 `/home/<用户名>`。

## `/etc`：系统配置

`/etc` 主要存放针对本机的系统配置。软件包提供的默认配置可能在 `/usr`，管理员的本机配置通常在 `/etc`；具体优先级由各程序决定。目录中的某些文件可能是符号链接或由其他服务生成。

| 路径 | 用途 |
| --- | --- |
| `/etc/passwd` | 用户账户信息，如用户名、UID、家目录和登录 shell；通常不存放密码哈希 |
| `/etc/shadow` | 本地账户密码哈希及密码有效期等敏感信息，通常只允许特权用户读取 |
| `/etc/group`、`/etc/gshadow` | 用户组信息及受保护的组信息 |
| `/etc/sudoers`、`/etc/sudoers.d/` | sudo 授权规则；使用 `visudo` 检查和编辑 |
| `/etc/hostname` | 常见系统上的静态主机名配置 |
| `/etc/hosts` | 本机静态主机名与 IP 地址映射 |
| `/etc/resolv.conf` | DNS 解析器配置；可能由 NetworkManager、systemd-resolved 等管理 |
| `/etc/nsswitch.conf` | 用户、主机名等信息的查询来源及顺序 |
| `/etc/fstab` | 文件系统及其挂载选项的静态配置 |
| `/etc/crypttab` | 部分系统中加密设备的解锁配置 |
| `/etc/os-release` | 操作系统标识和版本信息；可能链接到 `/usr/lib/os-release` |
| `/etc/environment`、`/etc/profile` | 部分全局环境配置；是否生效取决于登录方式和 shell |
| `/etc/ssh/` | OpenSSH 服务端及客户端配置和主机密钥，如 `sshd_config` |
| `/etc/systemd/system/` | systemd 系统单元及管理员覆盖配置 |
| `/etc/sysctl.conf`、`/etc/sysctl.d/` | 内核参数的持久配置；实际加载顺序取决于系统工具 |
| `/etc/security/`、`/etc/pam.d/` | 安全策略和 PAM 认证配置 |
| `/etc/cron.*`、`/etc/crontab` | cron 定时任务配置，具体目录由发行版和 cron 实现决定 |
| `/etc/apt/`、`/etc/yum.repos.d/` | 分别常见于 APT 和 DNF/YUM 系统的软件源配置 |
| `/etc/ssl/`、`/etc/pki/` | 证书和信任库相关配置，布局因发行版而异 |
| `/etc/opt/<软件名>/` | 按 FHS 约定存放 `/opt` 附加软件的本机配置 |

更改认证、网络或挂载配置前应备份并了解对应程序的加载方式；错误配置可能影响登录或启动。

## `/usr`：程序、库和共享数据

`/usr` 是系统软件的主要目录，通常由软件包管理器维护；它包含较少随日常运行变化的程序和数据，运行时产生的可变数据通常放在 `/var`。`/usr` 并不意味着在所有系统上都实际以只读方式挂载。

| 路径 | 用途 |
| --- | --- |
| `/usr/bin/` | 大多数用户命令和程序，例如 `ls`、`python3`（具体命令因系统而异） |
| `/usr/sbin/` | 传统上供系统管理使用的命令；有的发行版与 `/usr/bin` 合并 |
| `/usr/lib/` | 程序、共享库、内部辅助程序和架构相关数据；具体布局因发行版和架构而异 |
| `/usr/lib64/` | 一些发行版存放 64 位库的位置，也可能是链接 |
| `/usr/include/` | C/C++ 等开发使用的头文件 |
| `/usr/share/` | 与 CPU 架构无关的共享数据，如文档、手册和区域设置 |
| `/usr/local/` | 管理员在本机安装的软件，通常独立于发行版软件包管理 |
| `/usr/src/` | 源码或内核头文件等，内容由安装的软件决定 |

### `/usr/lib` 常见内容

- `/usr/lib/*.so*`：共享库的一种常见位置；多架构发行版也可能使用 `/usr/lib/x86_64-linux-gnu/` 等架构目录。动态链接器根据配置和缓存查找库。
- `/usr/lib/modules/<内核版本>/`：对应内核版本的可加载模块及其依赖信息。
- `/usr/lib/systemd/system/`：systemd 软件包提供的系统单元常见位置；某些发行版使用 `/lib/systemd/system/` 等路径。
- `/usr/lib/udev/`：udev 辅助程序和规则等，具体子目录因发行版而异。
- `/usr/lib/firmware/`：部分系统存放设备固件的位置，也常见 `/lib/firmware/`。
- `/usr/lib/os-release`：发行版提供的系统标识文件；`/etc/os-release` 存在时通常优先使用它。
- `/usr/lib/<程序名>/`、`/usr/libexec/`：程序内部辅助文件或可执行程序的常见位置，并非每个发行版都有相同布局。

### `/usr/share` 和 `/usr/local`

`/usr/share/man/` 存放手册页，`/usr/share/doc/` 存放软件文档，`/usr/share/locale/` 存放本地化资源，`/usr/share/zoneinfo/` 存放时区数据。`/usr/local` 通常再分为 `bin/`、`sbin/`、`lib/`、`include/` 和 `share/`，适合本机管理员安装的软件。

许多现代发行版采用合并的 `/usr` 布局：`/bin`、`/sbin`、`/lib` 和可能的 `/lib64` 分别链接到 `/usr` 下的对应目录；其他系统仍可能保留独立目录。可以用 `ls -ld /bin /sbin /lib /lib64` 查看实际情况。

## `/var`：经常变化的数据

| 路径 | 用途 |
| --- | --- |
| `/var/log/` | 系统和服务日志；可能包含 `journal/`、`auth.log`、`syslog` 或 `messages`，具体由日志服务和发行版决定 |
| `/var/lib/` | 应用持久状态，如数据库、包管理器数据库、容器数据；具体路径由服务配置决定 |
| `/var/cache/` | 可重新生成的缓存，例如软件包管理器缓存；清理前确认对应程序要求 |
| `/var/spool/` | 排队等待处理的数据，例如邮件或打印任务 |
| `/var/tmp/` | 临时文件，按惯例比 `/tmp` 保留更久，但仍可能被清理 |
| `/var/backups/` | 部分发行版或工具生成的备份；不能替代独立备份方案 |
| `/var/mail/` | 本地用户邮箱，具体邮件系统可能使用其他位置 |
| `/var/www/` | 某些 Web 服务使用的站点数据位置，取决于配置 |
| `/var/opt/<软件名>/` | `/opt` 附加软件的可变数据 |
| `/var/run`、`/var/lock` | 现代系统中通常分别链接到 `/run` 和 `/run/lock` |

例如，systemd journal 是否持久保存到 `/var/log/journal` 取决于配置；Docker 的数据目录常见为 `/var/lib/docker`，但可以修改，也取决于具体后端。不要仅凭路径假设目录可安全删除。

## `/proc`：进程和内核信息

`/proc` 通常挂载 procfs，是内核动态提供的虚拟文件系统；其中许多文件不是磁盘上保存的普通文件。内容会随着进程和内核状态变化，访问权限也可能受限制。

| 路径 | 用途 |
| --- | --- |
| `/proc/<PID>/` | 指定进程的信息；进程退出后目录消失 |
| `/proc/<PID>/status` | 进程状态、UID、内存和线程数等摘要 |
| `/proc/<PID>/cmdline`、`environ` | 启动参数和环境信息，内容以 NUL 分隔；可能包含敏感信息 |
| `/proc/<PID>/exe`、`cwd`、`root` | 指向可执行文件、当前工作目录、进程根目录的链接 |
| `/proc/<PID>/fd/` | 进程打开的文件描述符 |
| `/proc/<PID>/maps`、`smaps` | 内存映射及更详细的内存统计 |
| `/proc/<PID>/mountinfo` | 该进程所见的挂载信息 |
| `/proc/<PID>/ns/` | 进程所属 namespace 的引用；详见 [namespace](../namespace-cgroup/namespace.md) |
| `/proc/<PID>/cgroup` | 进程的 cgroup 成员信息；详见 [cgroup](../namespace-cgroup/cgroup.md) |
| `/proc/self`、`/proc/thread-self` | 分别指向访问者自身的进程和线程信息 |
| `/proc/cpuinfo`、`meminfo` | CPU 信息和内存统计 |
| `/proc/loadavg`、`uptime` | 系统负载和运行时间 |
| `/proc/stat`、`diskstats` | CPU 等系统统计及块设备 I/O 统计 |
| `/proc/partitions`、`mounts` | 内核识别的分区及当前进程的挂载视图；`mounts` 通常关联 `/proc/self/mounts` |
| `/proc/net/` | 当前网络 namespace 的网络信息 |
| `/proc/sys/` | 部分可读取或调整的内核运行参数，对应 `sysctl` 接口 |

例如 `cat /proc/meminfo` 查看内存信息，`sysctl kernel.hostname` 查看内核参数。修改 `/proc/sys` 会影响运行中的系统；需要持久生效时应使用发行版的 sysctl 配置机制。

## `/sys`：设备和内核对象

`/sys` 通常挂载 sysfs，以目录、链接和属性文件暴露内核设备模型及其他内核对象。许多条目由内核动态生成，符号链接从不同角度指向同一对象；具体内容取决于硬件、驱动和内核配置。

| 路径 | 用途 |
| --- | --- |
| `/sys/devices/` | 内核设备层级和设备属性，许多其他目录链接到这里 |
| `/sys/class/` | 按功能分类的设备视图，如 `net/`、`block/`、`tty/` |
| `/sys/block/` | 块设备视图，如磁盘和虚拟块设备 |
| `/sys/bus/` | 总线、设备和驱动信息，如 `pci/`、`usb/` |
| `/sys/dev/` | 按字符设备和块设备的主、次设备号索引 |
| `/sys/module/` | 内核模块的状态及可见参数 |
| `/sys/kernel/` | 内核子系统接口；实际子目录随系统而异 |
| `/sys/firmware/` | 固件接口，例如支持时的 EFI 信息 |
| `/sys/fs/cgroup/` | 常见的 cgroup 文件系统挂载点，提供资源控制接口 |
| `/sys/fs/bpf/` | 某些系统上 BPF 文件系统的挂载点，并非所有系统都有 |

例如，`ls /sys/class/net` 列出当前可见的网络接口，`cat /sys/class/net/<接口名>/operstate` 查看接口状态。部分属性可写，写入可能立即影响设备或内核行为。

## `/dev`：设备与特殊文件

`/dev` 中主要是设备节点，程序通过它们与内核驱动交互；设备节点不是设备中的数据本身。现代 Linux 常用 devtmpfs 和 udev 动态管理设备节点和链接。

| 路径 | 用途 |
| --- | --- |
| `/dev/sda`、`/dev/nvme0n1` | 常见的整个块设备命名示例；具体设备名因驱动而异 |
| `/dev/sda1`、`/dev/nvme0n1p1` | 对应分区的命名示例 |
| `/dev/disk/by-uuid/`、`by-id/` | 指向块设备的较稳定标识链接；UUID 可能因复制或重建文件系统改变 |
| `/dev/mapper/` | device-mapper 管理的设备，如 LVM 逻辑卷或加密映射 |
| `/dev/null` | 丢弃写入内容，读取时立即返回文件结束 |
| `/dev/zero` | 读取时提供零字节流 |
| `/dev/random`、`/dev/urandom` | 内核随机数接口 |
| `/dev/tty`、`/dev/pts/` | 控制终端和伪终端设备 |
| `/dev/shm/` | 常见的共享内存文件系统挂载点，通常是 tmpfs |
| `/dev/loop*` | 将文件等后备存储作为块设备访问的 loop 设备 |

使用 `ls -l /dev/null /dev/zero` 可以看到字符设备标记 `c` 和主、次设备号。直接向磁盘设备写入可能破坏数据，不要把设备节点当作普通文件随意写入。

## `/boot`：启动文件

`/boot` 通常包含内核映像（例如 `vmlinuz-*`）、初始内存文件系统（例如 `initramfs-*` 或 `initrd.img-*`）、内核配置及符号表，以及启动管理器的文件（如 `grub/`）。初始内存文件系统用于在真正的根文件系统可用前加载驱动和执行早期启动工作。

UEFI 系统还会使用 EFI System Partition，常见挂载点有 `/boot/efi` 或 `/efi`；具体布局、启动器和文件名因发行版而异。`/boot` 或 EFI 分区也可能单独挂载。内核升级和清理应使用发行版提供的工具，以免移除启动所需文件。

## `/opt`：附加软件

`/opt/<软件名>/` 常用于安装自带目录结构的第三方或附加软件，例如其中可能有 `bin/`、`lib/`、`share/`。按 FHS 约定，这类软件的本机配置和可变数据可分别放在 `/etc/opt/<软件名>/` 和 `/var/opt/<软件名>/`；实际软件不一定遵守。`/opt` 与 `/usr/local` 都可用于管理员安装的软件，具体位置取决于软件打包方式和本机约定。

## 其他重要目录

- `/run`：本次启动期间的状态，如服务 PID 文件、套接字和锁文件；通常位于内存文件系统，重启后重新创建。`/run/user/<UID>/` 常保存用户会话的运行时数据。
- `/tmp`：临时文件，系统可能定期或在重启时清理；不要依赖长期保存。它是否使用内存文件系统取决于系统配置。
- `/home/<用户名>`：普通用户的个人文件和配置，如 `~/.config/`、`~/.local/share/` 和 `~/.ssh/`；具体家目录由账户配置决定。
- `/root`：root 用户的家目录，不等于 `/`。
- `/mnt`：管理员临时挂载文件系统的常见位置；`/media` 常用于可移动介质的挂载点。
- `/srv`：本机服务提供的数据可以放在这里，服务也可能使用 `/var/lib` 或其他配置位置。
- `/lost+found`：某些文件系统根目录下由修复工具使用的目录，用于存放恢复出的无名文件；不是每种文件系统都有。

## 在自己的系统上查看

下面的命令以只读观察为主，不要求所有路径都存在：

```bash
ls -la /                         # 顶层目录和符号链接
findmnt /proc                    # 查看指定挂载点
findmnt                          # 查看完整挂载树
df -hT                           # 查看已挂载文件系统的类型和空间
ls -ld /bin /sbin /lib /lib64    # 查看传统目录是否为链接
ls /proc/$$/ns                   # 查看当前 shell 的 namespace
cat /proc/self/mountinfo         # 查看当前进程的挂载视图
ls /sys/class/net                # 查看网络接口
lsblk -f                         # 查看块设备、文件系统和挂载点
man hier                         # 查看本机目录层次说明
```

容器中的 `/proc`、`/sys`、`/dev` 和挂载视图可能受到 namespace、权限及运行时配置影响；同一路径在容器内外看到的内容不一定相同。

## 参考资料

- [Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
- [Linux 内核：proc 文件系统](https://docs.kernel.org/filesystems/proc.html)
- [Linux 内核：sysfs](https://docs.kernel.org/filesystems/sysfs.html)
- [systemd：文件系统层次](https://www.freedesktop.org/software/systemd/man/latest/file-hierarchy.html)
- [hier(7)](https://man7.org/linux/man-pages/man7/hier.7.html)
