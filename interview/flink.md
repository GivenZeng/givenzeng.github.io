
---

# 一、基础 & 架构

## 1. Flink 是什么？核心特点？
**答案：**
Flink 是一个**分布式、高性能、高可用、精准一次语义**的**流式计算引擎**，同时支持**流批一体**。

核心特点：
1. **真正的流处理**
   批是流的特例（有界流）。
2. **低延迟 & 高吞吐**
   毫秒级延迟，百万 TPS。
3. **精准一次语义 Exactly-Once**
   基于 Checkpoint + 两阶段提交。
4. **事件时间 EventTime 为主**
   支持乱序、延迟数据处理。
5. **有状态计算**
   状态后端、状态快照、状态扩缩容。
6. **丰富的窗口、Join、维表关联**
7. **高可用**
   无单点，JobManager 主备。

---

## 2. Flink 核心架构组件
**答案：**
1. **Client**
   提交作业、生成 JobGraph。
2. **JobManager（JM）**
   - 协调作业执行
   - 调度 Task、管理 Checkpoint、恢复作业
   包含：
   - Dispatcher：提交作业
   - JobMaster：管理单个作业
   - ResourceManager：资源管理
3. **TaskManager（TM）**
   - 真正执行任务的节点
   - 运行 Slot、执行 Task、管理状态
4. **Slot**
   - TM 内资源最小单位（内存+CPU）
   - 一个 Slot 运行一个**并行子任务链**。

---

## 3. Parallelism、OperatorChain、Slot 关系
**答案：**
- **Parallelism（并行度）**：算子同时运行的子任务数量。
- **Slot**：TM 提供的资源槽。
- **OperatorChain**：Flink 将**无 shuffle、上下游并行度相同**的算子串成一个链，在一个线程执行，减少线程切换、网络开销。

原则：
1. 一个 Slot 可以执行**一整条算子链**。
2. 作业最大并行度 = 总 Slot 数。
3. 一个 Slot 可以放多个算子，但同一时间只跑一个并行任务。

---

# 二、Time & Watermark（必考）

## 4. Flink 三种时间
**答案：**
1. **ProcessingTime**
   系统处理时间，最简单，不精确。
2. **EventTime**
   数据自带的事件时间，**生产最常用**，乱序问题必须用它。
3. **IngestionTime**
   进入 Flink 的时间，很少用。

---

## 5. Watermark 是什么？作用？原理？
**答案：**
**Watermark = 时间戳 + 延迟容忍**，是一个**全局进度标记**，表示：
> 小于该 Watermark 的数据都到齐了。

作用：
1. 解决**乱序、延迟**数据问题。
2. 触发 Window 关闭、计算。

原理：
1. Watermark 由数据源或算子生成，向下游广播。
2. 多并行度下，取**最小 Watermark**作为全局进度。
3. Watermark 一超过窗口结束时间，窗口触发。

公式：
```
Watermark = 最大观察到的事件时间 - 允许的延迟时间
```

---

## 6. 迟到数据怎么处理？
**答案：**
三种策略：
1. **丢弃（默认）**
2. **侧输出流 SideOutput**
   把迟到数据打到旁路，后期补算。
3. **窗口允许延迟 AllowedLateness**
   在延迟时间内，数据来一次触发一次。

生产常用：
`AllowedLateness + SideOutput`，保证数据不丢、可修正。

---

# 三、Window（必问）

## 7. Flink 有哪些 Window？
**答案：**
1. **滚动窗口 Tumpling**
   无重叠、固定长度。
2. **滑动窗口 Sliding**
   有重叠。
3. **会话窗口 Session**
   按间隔切分，无固定长度。
4. **全局窗口 GlobalWindow**
   一直不关闭，需自定义触发器。

按驱动分：
- 时间窗口 TimeWindow
- 计数窗口 CountWindow

---

## 8. Window 执行原理简要流程
**答案：**
1. 数据按 Key 分组。
2. 按 Window 分配器，归属到对应窗口。
3. 窗口内数据存入状态。
4. Watermark 越过窗口结束时间 → 触发计算。
5. 计算后，窗口数据根据策略清理/保留。

---

# 四、状态 & Checkpoint（超级高频）

## 9. Flink 状态分类
**答案：**
1. **KeyedState**（按 Key 隔离，最常用）
   - ValueState
   - ListState
   - MapState
   - ReducingState/AggregatingState
2. **OperatorState**（算子级，不按 Key）
   常用于 Source（如 Kafka 消费位点）。

---

## 10. 状态后端 StateBackend 有哪些？
**答案：**
1. **HashMapStateBackend**
   - 状态存 TM 内存
   - 速度极快
   - 适合中小状态
2. **EmbeddedRocksDBStateBackend**
   - 基于 RocksDB，磁盘+内存
   - 支持超大状态 TB 级
   - 生产**默认首选**

---

## 11. Checkpoint 原理（详细）
**答案：**
Checkpoint 是 Flink 实现**Exactly-Once**的核心机制。

流程（基于 Chandy-Lamport 算法）：
1. JM 向所有 Source 发送 **Checkpoint Barrier**。
2. Source 把 barrier 向下游发送，并快照自身状态（如 Kafka offset）。
3. 算子遇到 barrier：
   - 暂停数据处理
   - 把当前状态异步快照到持久化存储（HDFS/S3）
   - 确认完成
4. Sink 完成后，JM 标记本次 CK 完成。

特点：
- 异步、增量，不阻塞正常处理。
- 故障时从最近 CK 恢复。

---

## 12. Savepoint 和 Checkpoint 区别
**答案：**
1. **Checkpoint**
   - 自动、周期性
   - 增量、内部格式
   - 用于故障自动恢复
2. **Savepoint**
   - 手动触发
   - 标准格式、兼容版本
   - 用于**发布、升级、扩缩容、迁移**

---

## 13. Flink 如何保证 Exactly-Once？
**答案：**
两部分：
1. **内部状态 Exactly-Once**
   Checkpoint + 状态快照 + Barrier 对齐。
2. **端到端 Exactly-Once**
   需要三端配合：
   - Source 可重播（Kafka）
   - Flink 状态持久化
   - Sink 满足：
     - 幂等写入（Idempotent）
     - 或 **两阶段提交 2PC（Transactional Kafka Sink）**

---

# 五、反压（背压）Backpressure（字节必问）

## 14. 什么是反压？Flink 如何实现？
**答案：**
反压：下游处理不过来 → 自动降低上游发送速度，保证不丢数据、不崩溃。

Flink 反压机制：
- 基于**信用模型 Credit-Based**（新版）。
- 每个 Task 之间有**网络缓冲池**。
- 下游消费慢 → 缓冲区占满 → 上游无法写入 → 自动限速。
- 纯分布式、自动传导，不需要 JM 参与。

---

# 六、双流 Join（高频）

## 15. Flink 流 join 方式
**答案：**
1. **Window Join**
   同一个窗口内的数据才能 join。
2. **Interval Join**
   按时间区间 join（A 在 B 的前后一段时间内）。
3. **CoGroup**
   同窗口，更灵活的联合处理。
4. **DimJoin（维表关联）**
   流 join 外部库（MySQL/HBase/Redis）。
   优化：
   - 缓存
   - 异步 IO
   - 布隆过滤器

---

# 七、核心高级问题

## 16. Flink 为什么比 SparkStreaming 快？
**答案：**
1. **真正流式**
   SparkStreaming 是小批 MicroBatch，Flink 是逐条处理。
2. **低延迟**
   Flink 毫秒级，SparkStreaming 至少百毫秒~秒级。
3. **EventTime + Watermark 更完善**
4. **状态设计更专业**
   支持超大状态、增量快照。
5. **反压机制更优雅**
6. **JVM 优化更好**
   堆外内存、序列化更少GC。

---

## 17. Flink 重启策略
**答案：**
1. **固定延迟重启**
2. **故障率重启**
3. **不重启**

配合 Checkpoint 使用，故障自动恢复。

---

## 18. Kafka + Flink 如何做到端到端精准一次？
**答案：**
1. Source：
   `setStartFromGroupOffsets` + 开启 Checkpoint
2. Flink：
   开启 CK，状态后端用 RocksDB
3. Sink：
   - 使用 **FlinkKafkaProducer 事务模式**
     `Semantic.EXACTLY_ONCE`
   - 配置事务超时、事务前缀
4. Kafka 主题：
   `transaction.max.timeout.ms` 要大于 Flink 事务超时

---

# 八、生产调优（字节必问）

## 19. Flink 作业常见调优手段
**答案：**
1. **合理设置并行度**
   Source 并行度 = Kafka 分区数
2. **OperatorChain 开启**
   减少线程切换、网络开销。
3. **使用 RocksDBStateBackend**
   支持大状态。
4. **开启增量 Checkpoint**
   减小 CK 体积、提升速度。
5. **网络缓冲调优**
   反压、吞吐平衡。
6. **异步 IO 做维表关联**
7. **状态 TTL 清理**
   防止状态无限膨胀。
8. **TM 内存配置**
   框架堆内存 / 网络缓冲 / 托管内存（RocksDB）合理配比。

---

## 20. 数据倾斜在 Flink 里怎么解决？
**答案：**
1. **Key 加盐**
   加随机前缀，打散热点 Key。
2. **局部聚合 + 全局聚合**
   先局部汇总，再全局汇总。
3. **SQL 用 GROUPING SETS / 分拆**
4. **热点 Key 单独处理**
   分流、旁路、缓存。

---

# 九、你面试可以直接背的 10 条 Flink 金句
1. Flink 是**真正流处理**，批是有界流。
2. **EventTime + Watermark** 解决乱序。
3. **Checkpoint** 实现状态精准一次。
4. **Barrier** 对齐是分布式快照核心。
5. 状态分 **KeyedState / OperatorState**。
6. 生产首选 **RocksDB** 状态后端。
7. **反压** 自动传导，不丢数据不崩溃。
8. 端到端精准一次 = 可重放Source + CK + 幂等/2PC Sink。
9. **Slot 是资源**，并行度是计算并发。
10. 调优顺序：并行度 → 链化 → 状态 → CK → 反压 → 倾斜。
