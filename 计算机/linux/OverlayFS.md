# UnionFS / OverlayFS
讲一下 **UnionFS / OverlayFS** 到底是什么、怎么工作、为什么 Docker 离不开它。
OverlayFS 本身不是一个真正的磁盘文件系统，而是“堆叠在别的文件系统之上”的虚拟文件系统。**

我给你用最清晰、最本质的话讲透：

---

**OverlayFS 是一个“堆叠文件系统”（stacked filesystem）。**
它**不管理磁盘、不格式化分区、不自己存数据**。

它真正做的只有一件事：
### **拿别人已经存在的目录（比如 ext4/xfs 上的目录），叠起来给你看。**

所以：
- lowerdir、upperdir、workdir **必须在某个真实文件系统上**
  - 比如 ext4、xfs、btrfs 都行
- OverlayFS 只是**把它们拼成 merged**
- 所有真实 I/O 最终都会落到下层真实文件系统（ext4/xfs 等）

---

# 2. 结构关系（非常关键）

```
你的应用
   ↑
OverlayFS（虚拟层，只做合并、CoW、whiteout）
   ↑
ext4 / xfs / btrfs（真正管理磁盘、存储数据）
   ↑
磁盘硬件
```

**OverlayFS 不直接读写磁盘。
它只是“转发 + 转换”操作到底层文件系统。**
- **UnionFS = 联合文件系统**
  把**多个目录**“拼成”**一个目录**给用户看，目录之间有优先级。

- **OverlayFS = 现代 Linux 内核官方版 UnionFS**
  现在 Docker / K8s 底层用的**全是 OverlayFS**，UnionFS 是老一代。

它们的核心能力只有 3 个：
1. **多层合并**（多层目录 → 一层视图）
2. **写时复制（Copy-On-Write）**
3. **上层覆盖下层**

这就是**容器镜像分层**的底层原理。

---

# 二、OverlayFS 核心模型（必懂）
OverlayFS 固定只有 **4 个角色**：

```
lowerdir1  只读层（基础镜像，如 Ubuntu）
lowerdir2  只读层（如 Java）
upperdir   读写层（容器运行时修改的内容）
merged     合并后给进程看到的视图（挂载点）
```

规则：
- **上层文件 覆盖 下层同名文件**
- **下层只读，不能修改**
- **修改文件时，自动从下层拷贝到上层（CoW）**
- **删除文件 = 在上层创建 whiteout 文件**

---

# 三、最经典的工作流程（Docker 就是这样）
假设镜像有 3 层：
1. Ubuntu
2. JDK
3. 你的应用

启动容器时：
- 所有镜像层 = **lowerdir（只读）**
- 新建一层空目录 = **upperdir（读写）**
- 合并展示 = **merged**

### 1. 读文件
- 从上往下找
- 找到就返回，不继续往下

### 2. 修改文件
- 文件在只读层 → **不能直接改**
- 自动执行：
  **copy_up = 复制到 upperdir，再修改副本**
- 下层原文件不动，**上层覆盖它**

### 3. 删除文件
不是真删，而是在上层创建一个
**whiteout（遮挡文件）**
让 merged 视图里看不见它。

### 4. 新建文件
直接写在 upperdir。

---

# 四、OverlayFS 为什么强？（Docker 为什么用它）
### 1. 镜像分层存储
- 基础镜像共享
- 只存差异，不存重复
→ **极省空间**

### 2. 写时复制（CoW）
- 启动极快（不用拷贝整个镜像）
- 修改只拷贝文件，不拷贝整个层
→ **极快、极轻**

### 3. 内核原生
Linux 3.18+ 内置，稳定、高性能。

---

# 五、UnionFS vs OverlayFS 区别（简单记）
| 特性 | UnionFS（旧） | OverlayFS（新） |
|------|--------------|------------------|
| 内核支持 | 早期、补丁 | **官方原生** |
| 层数 | 支持很多层 | 多 lowerdir，结构简单 |
| 性能 | 一般 | **更快** |
| Docker 现在用 | 不用 | **默认就是它** |

一句话：
**UnionFS 是爷爷，OverlayFS 是现在的爹。**

---

# 六、OverlayFS 实际挂载命令（一看就懂）
```bash
mount -t overlay overlay \
  -o lowerdir=layer1:layer2:layer3,upperdir=rw,workdir=work,merged=merged
```

- `workdir`：内部临时目录，必须空、同一个文件系统。
- `merged`：最终看到的目录。

---

# 七、最关键的局限（一定要知道）
### 1. 不支持“递归合并”
不能把 overlay 再叠加 overlay（会不稳定）。

### 2. 写大文件性能差
Copy-Up 大文件会卡顿。
→ 解决：**数据卷 volume 绕过 overlay**。

### 3. 删除不是真删
只是 whiteout 遮挡。

### 4. 多 lowerdir 是“堆叠顺序”
越往后优先级越低。

---

# 八、和容器的关系（终极总结）
Docker / K8s 实现：
- 镜像每层 = lowerdir
- 容器运行层 = upperdir
- 容器看到的根目录 / = merged

**容器的读写隔离、镜像共享、快速启动、分层存储，全靠 OverlayFS。**

---

如果你愿意，我可以给你画一张
**OverlayFS 多层合并 + Copy-Up 原理流程图**
一看就彻底懂。
