
## 一、Spark 基础原理（必问）
### 1. Spark 是什么？为什么比 MapReduce 快？（超详细答案）
**答案：**
Spark 是一个**基于内存的分布式计算引擎**，用于大数据批处理、流处理、交互式分析、机器学习。
比 MapReduce 快的核心原因：
1. **基于内存计算**
   - MapReduce 每轮计算必须落磁盘，Spark 中间结果存在 **Executor 内存/堆外内存**。
   - 迭代计算（机器学习、图计算）性能提升 10~100 倍。
2. **DAG 有向无环图调度**
   - Spark 会把多个任务串联成一个 Pipeline，**减少多次 I/O**。
   - MapReduce 只能两阶段（Map → Reduce）。
3. **高效的 Shuffle 机制**
   - Spark 优化了 Shuffle 读写、排序、缓存策略。
4. **惰性求值（Lazy Evaluation）**
   - 只在 Action 算子触发时才真正执行，**可全局优化**。
5. **线程级并行**
   - Executor 内多线程运行 Task，**启动快、资源复用**。
   - MapReduce 是进程级，启动开销巨大。

---

### 2. RDD / DataFrame / Dataset 区别？（字节必问）
**答案：**
1. **RDD**
   - 弹性分布式数据集，Spark 最底层抽象
   - 类型安全，面向对象 API
   - 缺点：无优化，序列化开销大，性能差
2. **DataFrame**
   - RDD + Schema（结构）
   - 以**列**存储，类似数据库表
   - 有 Catalyst 优化器，速度远快于 RDD
3. **Dataset**
   - DataFrame 的**类型安全版**
   - 编译时检查类型
   - 同时拥有 DataFrame 优化 + RDD 类型安全
   - 统一了强类型 + 高效执行

**官方推荐优先级：**
`Dataset > DataFrame > RDD`

---

### 3. Spark 核心运行架构（详细）
**答案：**
1. **Driver**
   - 提交应用的主程序
   - 构建 DAG、划分 Stage、分发 Task
   - 收集结果、监控作业
2. **Executor**
   - 工作节点进程
   - 执行 Task
   - 存储缓存数据（RDD Cache）
3. **Cluster Manager**
   - 资源管理器：Yarn / Standalone / K8s
   - 负责分配 Executor 资源
4. **Job / Stage / Task**
   - Job：Action 触发的作业
   - Stage：根据 Shuffle 切分的任务阶段
   - Task：**最小执行单元**，发送给 Executor 线程

---

## 二、Spark 核心机制（高频）
### 4. 宽窄依赖是什么？Stage 如何划分？（必考）
**答案：**
#### （1）窄依赖
- 父RDD的一个分区 → 子RDD的**一个分区**
- 无 Shuffle，可流水线执行
- 例：map / filter / flatMap

#### （2）宽依赖
- 父RDD的一个分区 → 子RDD的**多个分区**
- 发生 **Shuffle**，数据重新分区
- 例：groupByKey / reduceByKey / join / distinct

#### （3）Stage 划分规则
- DAGScheduler 从后往前回溯
- **遇到宽依赖（Shuffle）就切分 Stage**
- Stage 数量 = Shuffle 次数 + 1

---

### 5. Spark Shuffle 原理详解（字节最爱问）
**答案：**
Shuffle 是**不同节点之间重新分区、分发数据**的过程。

#### Spark Shuffle 发展：
1. **Hash Shuffle**
   - 直接写文件，产生大量小文件
   - 高IO、高内存、性能差
   - 已废弃
2. **Sort Shuffle（默认）**
   - 每个 Task 输出**一个数据文件 + 一个索引文件**
   - 先分区，再排序
   - 减少文件数，降低随机IO
3. **Tungsten-Sort Shuffle**
   - 内存高效二进制格式
   - 堆外内存，减少GC
   - 速度最快

#### Shuffle 流程：
1. Map 端写分区数据（磁盘）
2. Reduce 端通过索引拉取对应分区
3. 数据聚合/join/排序

---

### 6. Spark 容错机制：Lineage + Checkpoint
**答案：**
#### （1）血统（Lineage）
- 记录 RDD 依赖关系
- 失败时**只重算丢失分区**，不需要全局备份
- 轻量级高效

#### （2）Checkpoint
- 将 RDD 持久化到 HDFS
- 切断血统，适合**长链路、迭代任务**
- 优点：容错更快
- 缺点：I/O 开销大

#### （3）Cache / Persist
- 缓存到内存/磁盘
- 加速重复计算
- 不是容错方案，丢失仍需重算

---

## 三、Spark SQL 优化（字节必考）
### 7. Catalyst 优化器原理（详细）
**答案：**
Catalyst 是 Spark SQL 的核心优化器，分 4 步：
1. **解析（Parser）**
   - 把 SQL 转成未解析的逻辑计划
2. **分析（Analyzer）**
   - 绑定元数据、字段、表
3. **逻辑优化（Optimizer）**
   - 谓词下推
   - 列裁剪
   - 常量折叠
   - 消除子查询
4. **物理计划（Planner）**
   - 生成可执行计划
   - 选择最优 Join 策略

**一句话总结：Catalyst 让 SQL 自动变快。**

---

### 8. Spark 四种 Join 机制（超级高频）
**答案：**
#### 1. Broadcast Join（广播join）**最优**
- 小表广播到所有 Executor
- 无 Shuffle，速度极快
- 适用：一张表很小
- 自动触发：`spark.sql.autoBroadcastJoinThreshold` 默认 10MB

#### 2. Shuffle Hash Join
- 两张表都按 Join Key 分区
- 分区内构建哈希表匹配
- 适用：表中等，内存充足

#### 3. Sort Merge Join（默认）
- 大数据量默认 Join
- 先 Shuffle 分区 → 各分区排序 → 归并合并
- 适用：TB 级大表 Join

#### 4. Cartesian Join（笛卡尔积）
- 无 Join Key
- 性能极差，禁止使用

---

## 四、Spark 调优（字节面试 90% 会问）
### 9. Spark 资源参数调优（生产标准）
**答案：**
```
--num-executors 100         Executor数量
--executor-memory 12G       每个Executor内存
--executor-cores 4           每个Executor并行Task数
--driver-memory 8G
```
**最优配置原则：**
- **executor-cores = 4~5**（避免GC飙升）
- **并行度 = Executors * cores * 2~3**
- 并行度设置不足是性能慢第一原因

---

### 10. Spark 数据倾斜怎么解决？（详细标准答案）
**答案：**
数据倾斜 = 某个 Key 数据量巨大 → 单个 Task 卡死

#### 解决方案（6种，字节标准）：
1. **过滤无效Key**（空值、异常值）
2. **广播Join**（把小表广播）
3. **加盐（Key 加随机后缀）**
   - 把倾斜Key打散成多份
   - 先局部聚合，再全局聚合
4. **单独处理倾斜Key**
   - 倾斜Key走广播
   - 非倾斜Key正常Join
5. **提高并行度**
6. **使用Map-Side预聚合**（reduceByKey 替代 groupByKey）

---

### 11. Cache / Persist / Checkpoint 区别
**答案：**
1. **Cache**
   - 默认存储级别：`MEMORY_ONLY`
2. **Persist**
   - 可指定存储级别：内存、磁盘、堆外、2副本
3. **Checkpoint**
   - 写入HDFS
   - 切断依赖，用于容错
   - 会阻塞 Job

**使用场景：**
- 重复用 → Cache
- 长链路、迭代 → Checkpoint

---

### 12. repartition 和 coalesce 区别
**答案：**
- **repartition**
  - 重分区，**会发生 Shuffle**
  - 可增多可减少
- **coalesce**
  - 合并分区，**无 Shuffle**
  - 只能减少分区数

---

## 五、Spark Streaming & Structured Streaming
### 13. Spark Streaming 背压机制（BackPressure）
**答案：**
自动根据处理能力调节消费速率，防止速率突增压垮Executor。
- 自动计算最大处理速率
- 动态限制 Receiver 接收速度
- 保证系统稳定不崩溃

---

### 14. Structured Streaming 精确一次语义（Exactly-Once）
**答案：**
实现 3 点：
1. **Source 可重播**（Kafka）
2. **Sink 幂等写入**
3. **Checkpoint + WAL 记录偏移量**

---

## 六、字节高频压轴题
### 15. Spark 为什么比 Flink 批处理快？
**答案：**
- Spark 批处理基于**内存 + 高效执行**
- 代码生成、Tungsten 优化强
- 批处理场景调度更轻
- Flink 面向流设计，批是模拟流

---

### 16. Spark 堆外内存作用？
**答案：**
- 减少 GC 停顿
- 提升 Shuffle/序列化/网络传输性能
- Tungsten 引擎直接使用

---

### 17. groupByKey 和 reduceByKey 区别？
**答案：**
- **groupByKey**
  - 先Shuffle全量数据
  - 内存压力大
- **reduceByKey**
  - **Map端预聚合**
  - 减少Shuffle数据量
  - 性能高10倍以上

---

# 七、必须背会的 12 条 Spark 金句
1. 宽依赖 = Shuffle = 划分 Stage
2. 并行度不足是 Spark 作业慢 #1 原因
3. 小表 Join 一定要用 Broadcast
4. 数据倾斜 90% 可以用加盐 + 广播解决
5. reduceByKey 永远优于 groupByKey
6. Catalyst 优化器 = Spark SQL 速度核心
7. 缓存用 Cache，容错用 Checkpoint
8. Shuffle 是 Spark 性能最大瓶颈
9. 谓词下推 + 列裁剪 = 基础优化
10. Executor 内存不宜过大，GC 会卡死
11. 堆外内存大幅降低 GC
12. 调优顺序：并行度 → 广播 → 倾斜 → Shuffle

---
