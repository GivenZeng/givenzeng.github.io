# Linux的cgroup（Control Group）
它是Linux内核提供的核心资源管理机制，能对进程组的CPU、内存、IO等资源进行精细化限制、隔离和监控，是容器（Docker/K8s）、轻量级虚拟化的底层基础。

我会从cgroup的核心概念、架构、核心功能、常用子系统、实操示例四个维度讲解，让你理解它的工作原理和实际应用。

### 一、cgroup的核心定义与目标
cgroup（控制组）的本质是：**将Linux系统中的进程组织成若干“组”，为每个组分配/限制系统资源（CPU、内存、磁盘IO等），并提供资源使用的监控能力**。

核心目标：
1. **资源限制**：比如限制某个进程组最多使用2核CPU、4GB内存；
2. **资源隔离**：不同进程组的资源使用相互隔离（如A组的内存耗尽不会影响B组）；
3. **资源监控**：统计进程组的资源使用情况（如CPU使用率、内存占用）；
4. **资源优先级**：为不同进程组设置资源使用优先级（如核心业务组优先使用CPU）；
5. **任务控制**：对进程组执行统一操作（如冻结、恢复、终止）。

### 二、cgroup的核心架构与关键概念
cgroup的架构分为两层（v1版本，v2是统一版，后文会提）：

#### 1. 核心概念
| 概念 | 含义 | 类比 |
|------|------|------|
| 子系统（Subsystem） | 内核提供的资源管控模块，每个子系统对应一种资源（如cpu子系统管CPU，memory子系统管内存） | 不同的“资源控制器” |
| 控制组（Cgroup） | 进程的集合，每个控制组可以关联一个或多个子系统，设置资源限制 | 一个“资源隔离的进程分组” |
| 层级（Hierarchy） | 控制组的树形组织结构（父子关系），子组会继承父组的资源限制 | 文件夹的目录树（/parent/child） |
| 进程（Process） | 可以加入任意层级的控制组，一个进程可属于多个层级的不同控制组（但同一层级只能属于一个） | 被管控的“员工” |

#### 2. 核心规则
- 每个层级可以挂载一个或多个子系统（如层级1挂载cpu+cpuset，层级2挂载memory）；
- 一个子系统只能挂载到一个层级（v1限制，v2已解决）；
- 进程加入层级后，会自动继承该层级控制组的资源限制；
- 控制组的树形结构支持资源限制的“继承+覆盖”（子组可在父组限制内调整）。

### 三、cgroup的核心子系统（v1）
Linux内核提供了十多个cgroup子系统，常用的如下：

| 子系统 | 作用 | 核心配置示例 |
|--------|------|--------------|
| cpu | 限制进程组的CPU使用率，设置CPU调度优先级 | `cpu.cfs_quota_us`（CPU时间配额）、`cpu.shares`（CPU权重） |
| cpuset | 绑定进程组到指定CPU核心/NUMA节点 | `cpuset.cpus`（允许使用的CPU核，如0-1）、`cpuset.mems`（NUMA节点） |
| memory | 限制进程组的内存使用（物理内存+交换分区） | `memory.limit_in_bytes`（最大内存）、`memory.swappiness`（交换分区使用倾向） |
| blkio | 限制进程组的块设备IO（磁盘读写） | `blkio.weight`（IO权重）、`blkio.throttle.read_bps_device`（读速度限制） |
| pids | 限制进程组的最大进程数 | `pids.max`（最大进程数，如100） |
| net_cls/net_prio | 标记网络数据包，配合tc/iptables做网络带宽限制/优先级 | `net_cls.classid`（数据包标记） |

### 四、cgroup的使用方式（实操示例）
cgroup的配置通过文件系统实现（挂载到`/sys/fs/cgroup`），以下是基础实操步骤（以cpu子系统为例）：

#### 1. 查看已挂载的cgroup层级
```bash
# 查看所有cgroup子系统的挂载点
mount | grep cgroup

# 典型输出（不同系统路径可能略有差异）
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
cgroup on /sys/fs/cgroup/cpu type cgroup (rw,nosuid,nodev,noexec,relatime,cpu)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
```

#### 2. 创建控制组并设置CPU限制
```bash
# 1. 进入cpu子系统的挂载目录
cd /sys/fs/cgroup/cpu

# 2. 创建名为test_group的控制组（自动生成配置文件）
mkdir test_group

# 3. 限制该组的CPU使用率为50%（cfs_quota_us=50000，cfs_period_us=100000 → 50000/100000=50%）
echo 50000 > test_group/cpu.cfs_quota_us
echo 100000 > test_group/cpu.cfs_period_us

# 4. 将进程（PID=1234）加入该控制组
echo 1234 > test_group/cgroup.procs

# 5. 查看该组的CPU使用情况
cat test_group/cpuacct.usage  # 累计CPU使用时间（纳秒）
cat test_group/cpu.stat       # CPU调度统计
```

#### 3. 限制内存使用示例
```bash
# 1. 进入memory子系统目录
cd /sys/fs/cgroup/memory

# 2. 创建控制组
mkdir mem_group

# 3. 限制最大内存为2GB
echo 2147483648 > mem_group/memory.limit_in_bytes

# 4. 禁止使用交换分区
echo 0 > mem_group/memory.swappiness

# 5. 加入进程
echo 1234 > mem_group/cgroup.procs

# 6. 查看内存使用
cat mem_group/memory.usage_in_bytes  # 当前内存占用
cat mem_group/memory.stat            # 内存统计
```

### 五、cgroup v1 vs v2（核心改进）
cgroup v1是早期版本，存在子系统分散、层级限制多等问题；Linux 4.5+推出cgroup v2（统一控制组），核心改进：
| 特性 | cgroup v1 | cgroup v2 |
|------|-----------|-----------|
| 子系统管理 | 子系统分散，一个子系统只能挂载到一个层级 | 所有资源统一到单个层级，支持多资源联合管控 |
| 进程归属 | 同一层级只能属于一个控制组 | 支持进程动态迁移，更灵活 |
| 资源管控 | 部分子系统（如memory）限制不精准 | 支持“弹性资源限制”“资源回收”，管控更精准 |
| 兼容性 | 兼容所有老版本工具（Docker早期） | 逐步成为主流（K8s 1.25+默认推荐） |
| 复杂度 | 配置分散，学习成本高 | 配置统一，简化管理 |

### 六、cgroup的典型应用场景
1. **容器技术（Docker/K8s）**：
   - Docker为每个容器创建独立的cgroup控制组，限制容器的CPU、内存、IO等资源；
   - K8s的`resources.limits`/`resources.requests`底层就是通过cgroup实现的。
2. **服务器资源隔离**：
   - 将不同业务的进程放入不同cgroup，避免单个业务占用全部资源（如数据库进程限制CPU核数，防止拖垮整机）。
3. **资源监控与计费**：
   - 通过cgroup统计进程组的资源使用，实现按资源使用量计费（如云服务器）。
4. **进程组管理**：
   - 冻结/恢复一组进程（如`echo freeze > test_group/cgroup.freeze`），用于故障排查或资源腾挪。

### 七、cgroup的核心优势与局限性
#### 优势
- **内核原生支持**：无需额外安装软件，性能损耗极低（<1%）；
- **精细化管控**：支持CPU核心绑定、内存精准限制、IO带宽控制等；
- **隔离性强**：不同控制组的资源使用相互独立，避免“资源抢占”；
- **可监控性**：实时统计资源使用，便于问题排查。

#### 局限性
- **v1配置复杂**：子系统分散，层级规则严格，易出错；
- **部分资源限制不精准**：如v1的CPU限制是“按时间片”，高并发下可能有波动；
- **无法限制内核资源**：如内核态内存、网络连接数（需配合其他工具）；
- **依赖内核版本**：v2的新特性需要高版本内核支持（Linux 4.5+）。

### 总结
1. cgroup是Linux内核的**进程组资源管控机制**，核心是“分组+资源限制/监控/隔离”；
2. v1通过多个子系统管控不同资源（cpu/memory/blkio），v2统一为单个层级，更易用；
3. 核心应用是容器技术（Docker/K8s），也是服务器资源隔离的核心工具；
4. 关键优势：内核原生、低损耗、精细化管控；局限性：v1配置复杂，部分资源限制不精准。

简单来说，cgroup就是Linux系统的“资源管家”——它能把进程分成不同的“小组”，给每个小组分配固定的CPU、内存等资源，防止某个小组“霸占”所有资源，是实现容器隔离和资源管控的底层基石。