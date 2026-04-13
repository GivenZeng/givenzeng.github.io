# 安装
docker pull qdrant/qdrant
docker run  -d -p 6333:6333 -p 6334:6334  qdrant/qdrant

# 基础概念
| 概念                   | 类型             | 作用                              | 常用参数 / 说明                                                |
| -------------------- | -------------- | ------------------------------- | -------------------------------------------------------- |
| **Collection**       | 数据库表           | 存储向量数据的最小单位                     | name: collection 名称；vectors: 向量配置；sparse_vectors: 稀疏向量配置 |
| **Vector**           | 数据字段           | Dense embedding，支持语义搜索          | size: 向量维度；distance: Cosine / Dot / Euclid               |
| **Sparse Vector**    | 数据字段           | 稀疏向量，用于关键词检索（BM25、SPLADE）       | BM25 或自定义稀疏向量；indices + values 存储                        |
| **Payload**          | Metadata       | 附加字段，可做 filter 或 formula rerank | 字段类型可为 keyword / integer / text；手动创建索引加速                 |
| **Point**            | 数据行            | Collection 内的一条向量 + payload     | id: 唯一标识；vector: 向量；payload: 附加字段                        |
| **HNSW Index**       | Vector Index   | Dense vector ANN 搜索索引           | m: 图连接数；ef_construct: 建索引搜索深度；ef: 查询精度                   |
| **Payload Index**    | Metadata Index | 加速 filter / match 查询            | keyword / integer / text；需手动创建                           |
| **Formula**          | Rerank         | 根据 payload 或其他规则对初步搜索结果加权       | sum / mult / key / match；可做复杂规则 rerank                   |
| **Prefetch**         | 查询参数           | 先做向量搜索取 topN，再用 formula rerank  | query: dense vector；limit: 取多少条候选                        |
| **Filter / Match**   | 查询条件           | 对 payload 做筛选                   | key: payload 字段名；match: 条件；any/all 支持                    |
| **Hybrid Search**    | 检索方式           | Dense + Sparse 组合搜索             | dense vector + sparse vector + merge + rerank            |
| **Distance Metrics** | 搜索配置           | 衡量向量相似度                         | Cosine / Dot / Euclid                                    |
| **Chunk / Document** | 语义层概念          | 文本分片，便于向量化和检索                   | chunk_id / doc_id / payload text                         |


- Dense Vector = 语义理解，捕捉文本整体意思
- Sparse Vector = 精确关键词匹配，保证搜索覆盖性
- Payload / Filter = 强大的条件查询和 formula rerank 支持
- Hybrid Search = Dense + Sparse + Formula Rerank 是主流 RAG 检索模式


# 和milvus的对比
| 特性                     | Qdrant                                                                | Milvus                                                  |
| ---------------------- | --------------------------------------------------------------------- | ------------------------------------------------------- |
| **向量类型**               | Dense vector + Sparse vector（BM25 / SPLADE）                           | Dense vector                                            |
| **关键词搜索**              | 支持 Payload index（keyword / integer / text），Sparse vector 可做精确或全文关键词搜索 | 仅支持 Payload filter（keyword / integer / float），全文搜索需外部系统 |
| **Hybrid Search**      | 原生支持 Dense + Sparse + Formula rerank                                  | 不支持原生 Hybrid，需自己实现 Dense + Payload 组合                   |
| **向量索引**               | HNSW（默认）、支持 IVF / DiskANN                                             | IVF, HNSW, ANNOY, etc.                                  |
| **索引配置**               | HNSW: m, ef_construct, ef 查询精度                                        | IVF: nlist, metric_type, HNSW: M, efConstruction, ef    |
| **数据类型灵活性**            | Dense + Sparse + Payload                                              | Dense + Payload                                         |
| **Payload / Metadata** | 强大，可做公式加权 rerank、过滤                                                   | 基本过滤                                                    |
| **查询方式**               | Vector search + Filter + Formula rerank                               | Vector search + Filter                                  |
| **RAG / LLM 检索适配**     | 很方便，适合 Hybrid 搜索和 Chunk rerank                                        | 需要额外方案，常配合 Elasticsearch 或 Milvus + Elastic 做关键词搜索      |
| **分布式**                | 支持（Qdrant Cloud / self-host）                                          | 支持（Milvus Cluster）                                      |
| **Python SDK**         | qdrant-client                                                         | pymilvus                                                |
| **Java / Go SDK**      | 有 SDK                                                                 | 有 SDK                                                   |
| **部署难度**               | 简单，官方 Docker / Cloud                                                  | 中等，Cluster 部署稍复杂                                        |
| **典型场景**               | RAG, Hybrid search, Knowledge Base, FAQ                               | 语义搜索, 向量数据库, AI 相似度检索                                   |

- Qdrant 更适合 RAG、Hybrid Search、结构化加权检索（公式 rerank + sparse vector）
- Milvus 更适合 大规模纯向量语义搜索，或者向量+payload简单过滤场景
- 使用上Qdrant相对简单