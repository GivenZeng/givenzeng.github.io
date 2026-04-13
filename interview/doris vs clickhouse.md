Apache Doris 与 ClickHouse 同属主流 MPP 架构的 OLAP 引擎，但在**架构设计**与**存储引擎**上有显著差异。Doris 是**纯分布式、强一致、易用优先**的架构，而 ClickHouse 是**单节点性能极致、灵活、运维复杂度高**的架构。

---

## 一、架构区别（核心差异）

### 1. 整体架构模型

**Apache Doris**
- **纯 MPP + 存算一体（原生）+ 存算分离（可选）**
- **两层架构**：FE (Frontend) + BE (Backend)


  - **FE (Java)**：
    - 负责 SQL 解析、查询优化（CBO）、元数据管理、集群调度
    - 主从高可用（Master/Follower/Observer），内置 BDBJE 一致性协议
    - 无外部依赖（不依赖 ZooKeeper）
  - **BE (C++)**：
    - 存储 + 计算一体化，MPP 并行执行
    - 数据分片（Tablet）多副本自动均衡
- **核心优势**：**完全分布式、强一致、高可用、易运维、JOIN 强**

**ClickHouse**
- **Share-Nothing + 本地优先 + 分布式逻辑层**
- **单层节点 + 分布式引擎**架构


  - 所有节点对等，无中心协调节点
  - 分布式依赖 **ZooKeeper** 管理副本与元数据
  - **Distributed** 引擎为逻辑表，不存数据，仅路由
- **核心优势**：**单节点性能极致、写入极快、压缩极高**

### 2. 分布式与查询执行

| 维度 | Apache Doris | ClickHouse |
|:--- |:--- |:--- |
| **JOIN 能力** | **原生分布式 JOIN**（Shuffle Join、Broadcast Join、Colocate Join） | **弱 JOIN**，主要靠**大表小表广播、子查询、反规范化** |
| **查询优化器** | **CBO 代价优化器**，自动选最优计划 | **无 CBO**，规则优化，需大量手动调优 |
| **执行模式** | **MPP 全分布式**，中间结果跨节点 Shuffle | **Scatter-Gather**，单节点算完再汇总，Shuffle 弱 |
| **事务** | **支持 ACID 事务、两阶段提交、读写一致性** | **无事务**，最终一致性，写入可能不一致 |
| **高可用** | FE 自动选主；BE 副本自动修复、均衡 | 依赖 ZooKeeper 选主；副本同步、修复需手动/半手动 |
| **外部依赖** | **无**（内置一致性） | **强依赖 ZooKeeper**（集群必备） |

### 3. 扩展性与运维

- **Doris**：
  - 弹性扩缩容，自动数据均衡、副本修复
  - 在线 Schema 变更、在线加节点
  - 运维简单，监控告警完善
- **ClickHouse**：
  - 扩节点需手动迁移分片
  - 副本、分区、合并策略复杂
  - 运维成本高，文件多、易失控

---

## 二、存储引擎区别（核心差异）

### 1. 数据组织模型

**Apache Doris**
- **Table → Partition（Range）→ Bucket（Hash）→ Tablet → Rowset → Segment → Page**


  - **Partition**：纵向切割（通常时间）
  - **Bucket**：横向哈希分片（分布式单元）
  - **Tablet**：最小物理副本单元
  - **Segment**：存储文件（含列数据、索引）
  - **Page**：压缩/编码/缓存单位

**ClickHouse**
- **Database → Local Table → Partition → Part → Column → Granule**

  - **Partition**：同 Doris（时间/表达式）
  - **Part**：写入生成的小文件，后台合并
  - **每列独立文件**：`.bin`（数据）+ `.mrk`（标记）+ 索引
  - **Granule**：稀疏索引粒度（默认 8192 行）

### 2. 核心存储引擎

**Apache Doris（自研列式引擎）**
- **LSM-Tree 思想 + MVCC + 列存**
- **三大数据模型**（内置，自动处理）
  1. **Duplicate Key（明细）**：原始日志，无序
  2. **Aggregate Key（聚合）**：写入预聚合（SUM/MAX/MIN）
  3. **Unique Key（更新）**：主键唯一，**Merge-on-Write / Merge-on-Read**
- **写入**：MemTable → 刷盘成 Rowset → 后台 Compaction
- **更新**：**强一致行级更新/删除**，支持高频实时更新

**ClickHouse（MergeTree 家族）**
- **LSM-Tree 变种，追加式，列存**
- **引擎多、需手动选**
  - **MergeTree**：基础排序
  - **ReplacingMergeTree**：合并去重（**非实时、最终一致**）
  - **SummingMergeTree**：合并求和
  - **CollapsingMergeTree**：标记折叠（需业务配合）
- **写入**：内存缓冲区 → 落盘 Part → 后台异步合并
- **更新**：**无原生行更新**，靠标记删除+追加，合并后才生效

### 3. 索引与查询加速

**Apache Doris**
- **Sorted Compound Key Index**：排序键（1-3 列），稀疏索引
- **ZoneMap/Min-Max**：块级过滤
- **Bloom Filter**：高基列等值查询
- **Inverted Index（可选）**：全文检索
- **物化视图**：自动匹配、自动刷新

**ClickHouse**
- **Primary Key / Order By**：数据全局排序，稀疏索引（Granule 级）
- **Data Skipping Index**：二级跳数索引（手动建）
- **每列独立索引**：查询精准度高
- **Count 索引**：`count()` 极快

### 4. 写入、更新、合并

| 特性 | Apache Doris | ClickHouse |
|:--- |:--- |:--- |
| **写入模型** | **实时可见、强一致** | **追加、异步合并、最终一致** |
| **更新/删除** | **行级、实时、强一致**（Unique Key） | **无原生更新**，靠引擎合并（延迟大） |
| **合并策略** | 自动 Compaction，版本管理 | 后台线程合并，需调优（易 Part 过多） |
| **事务** | ✅ 支持 ACID、原子写入 | ❌ 无事务，并发写入可能冲突 |

### 5. 压缩与存储效率
- **Doris**：LZ4/ZSTD，列编码（RLE、Dict、Delta），压缩比 **5:1~10:1**
- **ClickHouse**：LZ4/ZSTD，多种高级编码，压缩比 **10:1~20:1**（更高）

---

## 三、核心差异总结

### 架构层面
- **Doris**：**分布式友好、强一致、易运维、JOIN 强、SQL 标准、MySQL 兼容**
- **ClickHouse**：**单节点性能天花板、写入最快、压缩最高、JOIN 弱、运维复杂**

### 存储层面
- **Doris**：**强一致、实时更新、内置多模型、自动合并、事务支持**
- **ClickHouse**：**追加式、最终一致、多引擎手动选、合并延迟、无事务**

---

## 四、选型建议
- **选 Doris**：
  - 需要**复杂 JOIN、多表关联、实时更新、强一致**
  - 企业级数仓、高并发报表、用户行为分析
  - 看重**运维简单、高可用、弹性扩展**
- **选 ClickHouse**：
  - **单表超大、宽表、极少 JOIN、极高写入吞吐**
  - 日志检索、时序监控、简单大宽表聚合
  - 追求**极致压缩、极致单节点查询速度**
