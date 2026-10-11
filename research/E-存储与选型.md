# E - 向量数据库存储容量计算 + 技术选型依据

> 调研日期:2026-09-26。用途:RAG 工程知识库培训文档。
> 标注说明:✅已验证 = 本轮实际抓取并核对过原文;⚠️未验证 = 仅搜索可见/常识,未打开原文核对。
> 正文中文 + 英文术语;所有关键结论附来源 URL。

---

## 1. 向量存储容量计算(核心公式)

### 1.1 原始向量内存/磁盘公式(所有系统通用)

```text
raw_bytes = num_vectors × dimensions × bytes_per_dim

float32: 4 bytes/维(绝大多数 embedding 默认)
fp16/bf16: 2 bytes/维
int8 标量量化: 1 byte/维
binary(1-bit): 1/8 byte/维 = dims/8 bytes
```

- pgvector 官方:`vector` 类型每条向量占 `4 × dimensions + 8` bytes,最多 16,000 维;`halfvec`(半精度)可索引到 4,000 维;`bit`(binary)可到 64,000 维。✅已验证
  来源:https://github.com/pgvector/pgvector (README)
- Qdrant 官方公式:`dense_size = base × dimensions × bytes_per_dim`,其中 `base = number_of_points × replication_factor`;bytes_per_dim: float32=4, float16=2, uint8=1, turbo4=0.5。✅已验证
  来源:https://qdrant.tech/documentation/capacity-planning/
- Elastic 官方(dense_vector 各 index_type 的每维内存):float=4B,int8="reduces the memory footprint by 75% (or 4x)",int4=8x,bbq(1-bit)="96% (or 32x)"。✅已验证
  来源:https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector

**经验法则**:1M 条 × 768 维 float32 ≈ 2.87 GB(裸向量,不含索引);粗估口径 `GB ≈ 维度数 × 4 × 条数 / 10^9`。

### 1.2 HNSW 图索引额外开销

三个官方口径(相互印证):

**(a) Qdrant 新版官方公式**(✅已验证,https://qdrant.tech/documentation/capacity-planning/ ):

```text
hnsw_size = base × m × 2 × 4 bytes × 1.2

m: 每节点边数,默认 16;顶层每点 2×m 条连接,每条连接为 4-byte 引用;
   1.2 因子覆盖下层与簿记。
示例(官方原文):1M 点 × replication 2 × m16
   = 2,000,000 × 16 × 2 × 4 × 1.2 ≈ 0.29 GB
```

即默认 m=16 时每条向量图开销 ≈ 154 bytes,仅为向量本身的零头(1024 维 float32 为 4KB/条)。

**(b) OpenSearch 官方公式**(✅已验证,https://docs.opensearch.org/latest/mappings/supported-field-types/knn-methods-engines/ ):

```text
HNSW 内存 = 1.1 × (4 × dimension + 8 × m) bytes/vector
官方示例:1M 条 × 256 维 × m=16 → 1.1 × (1024 + 128) × 1e6 ≈ 1.267 GB
IVF  内存 = 1.1 × ((4 × dimension × num_vectors) + 4 × nlist × d)
另:replica 会使向量总数翻倍;circuit_breaker_limit 默认占 50% RAM
```

`(8 × m)` 就是每点 ~128 bytes 的图连接开销(8 bytes/邻接 × 16),与 Qdrant 口径一致。

**(c) Weaviate 官方口径**(✅已验证,https://docs.weaviate.io/weaviate/concepts/vector-index ):
"HNSW is an in-memory index, where each node in the graph as well as each edge between nodes are stored in memory";节点 ≈ `4B(float) × N dimensions`,边 ≈ `10B × 20 connections`(~200B/节点)。Weaviate 的图边开销估计偏高(因为对象 ID/metadata 也占内存),适合作为上界。

**旧版概念(供对照,⚠️未验证原文,搜索可见)**:HNSW level-0 每点 `m0 = 2m` 条链接、上层 m 条;Qdrant 新公式用 `2m × 4B × 1.2` 把它简化掉了。

> 结论(按公式推算):**HNSW 图开销 ≈ 每向量 128~250 bytes(m=16)**,相对 768~1536 维向量(3~6KB/条)占比 3~8%;但注意 pgvector HNSW 索引还会复存储向量本身(索引内含向量用于距离计算),Postgres 实际内存远超理论值(见 2.4)。

### 1.3 量化压缩(Quantization):压缩比与召回影响

全部取自 Qdrant 官方 quantization 指南 ✅已验证(https://qdrant.tech/documentation/guides/quantization/ ),除注明外:

| 方法 | 原理 | 压缩比 | 速度 | 质量 |
|---|---|---|---|---|
| Scalar (int8) | float32→uint8,per-dimension 线性映射 | **4x** | SIMD 加速,最快 | "error usually less than 1%" |
| PQ (Product) | 向量切 m 段,每段 k-means 256 质心 → 1 byte/段 | **最高 64x**(常用 x16;`dim/m × 1 byte`) | 慢(非 SIMD 友好) | 高维向量才推荐 |
| Binary (1-bit) | 每维 1 bit | **32x** | "up to a 40x speedup" | 仅适合高维中心化分布 |
| TurboQuant 1.5/2 bit (v1.15) | 1.5-bit=24x、2-bit=16x,2-bit 显式编码 0 | 24x / 16x | — | <1000 维改善精度 |

**召回率补偿三件套**(同一来源):
- `rescore`:用量化后 top 候选回原向量精排;binary 量化默认开启 rescoring。
- `oversampling`:先取更多候选再精排,"if oversampling is 2.4 and limit is 100, then 240 vectors will be pre-selected"。
- `quantile`(scalar):截断离群值(如 0.99),影响精度不影响内存。

**官方 recall 实测数**(同一来源):binary quantization 下 OpenAI text-embedding-ada-002 (1536d) "0.98 recall@100 with 4x oversampling";Cohere embed-english-v2.0 (4096d) "0.98 recall@50 with 2x oversampling"。

**其他系统对照**:
- Elastic int8/int4/bbq 也要"keep the raw float vector values on disk for reranking"(磁盘开销 +25%/+12.5%/+3.1%),bbq 靠 `rescore_vector.oversample` 补偿(bbq_disk bits=1 默认 3.0x oversampling)。✅已验证(dense_vector mapping reference)
- OpenSearch Faiss 支持 scalar/PQ/binary 量化 + disk-based vector search(⚠️未验证细节,导航页可见)。
- Weaviate:默认 "HNSW with RQ8 quantization by default"(8-bit rotational quantization,4x);新 HFresh 索引把 posting list 用 1-bit RQ 放磁盘。✅已验证(https://docs.weaviate.io/weaviate/concepts/vector-index + https://weaviate.io/llms.txt )

### 1.4 磁盘 vs 内存:mmap 与 on-disk 索引

**Qdrant(✅已验证)**:
- 三档 memory tier:`pinned`(堆内存常驻)/ `cached`(mmap 启动预热)/ `cold`(mmap 按需加载)。来源:https://qdrant.tech/documentation/capacity-planning/
- 存储优先配置:向量放 `cold`,"if you store half as many vectors in RAM, search latency will roughly double";HNSW 图不要放 cold("graph traversal does many small random reads that suffer badly from disk latency")。来源同上
- 旧版 memmap 机制(concepts/storage)✅已验证:https://qdrant.tech/documentation/concepts/storage/
- RAM/Disk 经验终值都要 **+20% headroom**(OS page cache、WAL、快照、optimizer 临时段)。✅已验证

**Milvus(✅已验证)**:
- mmap 文档:启用 mmap 后单节点可容纳约 **4 倍**数据量(2CPU/8GB 对比测试),HNSW 是最适合 mmap 的索引,推荐 NVMe SSD。来源:https://milvus.io/docs/mmap.md
- DiskANN:Vamana 图在磁盘 + PQ 压缩码在内存;`PQCodeBudgetGBRatio` 默认 0.125(内存中 PQ 码 = 原始数据的 1/8,即 8x 压缩,可调至 0.0625=16x / 0.25=4x),`SearchCacheBudgetGBRatio` 0.10、`MaxDegree` 56、`BeamWidthRatio` 4;面向 billion-scale。来源:https://milvus.io/docs/diskann.md
- pgvectorscale(Timescale):`StreamingDiskANN` 索引,`storage_layout=memory_optimized` 用 SBQ 压缩(每维 2 bit(维度<900)或 1 bit)。✅已验证 https://github.com/timescale/pgvectorscale

**Disk vs Memory 的极端对照( billion-scale,✅已验证)**:AWS 官方博客用 OpenSearch 测 10 亿条 × 128 维(BIGANN):
- HNSW 全内存 ≈ **1,408 GB**,集群 ~$75/hour
- IVFPQ 内存 ≈ **70 GB**(约 1/20),集群 ~$11/hour(约 7x 省钱)
- 代价:IVFPQ recall@10 仅 0.17~0.61 vs HNSW 0.99
来源:https://aws.amazon.com/blogs/big-data/choose-the-k-nn-algorithm-for-your-billion-scale-use-case-with-opensearch/

### 1.5 payload / 元数据 / id 开销(Qdrant 官方,✅已验证)

```text
payload 磁盘 = base × avg_payload_size × 1.5     (1.5 = 后端存储开销)
payload 内存 = base × avg_payload_size × 1.5 × 3  (cached 时)
payload 索引 ≈ 被索引字段大小 × 2                  (pinned,常驻 RAM)
id_tracker  = base × 52 bytes                     (外部 id → 内部位置 + 版本,始终驻留 RAM)
官方示例:1M 点 × replication 2 × 1KB payload(cold)→ disk ≈ 2.86 GB
```

稀疏向量(BM25/splade 类):
```text
sparse_size  = base × nnz × bytes_per_dim
sparse_index = base × nnz × bytes_per_dim × 1.5    (默认 pinned,吃 RAM)
```
来源:https://qdrant.tech/documentation/capacity-planning/

### 1.6 embedding 原文存储(chunk text + metadata)

不在向量库公式内,但必须一起预算(PG 表 / 对象存储):
- 中文 chunk 512 字 ≈ 1.5 KB(UTF-8 3 bytes/汉字)+ metadata(title/url/日期等 0.2~1 KB)→ **按 2~3 KB/条估**。
- 10M chunks ≈ 20~30 GB 纯文本;若放 Postgres 再加 PK/索引/TOAST 膨胀 1.5~2x(⚠️按经验推算)。
- 向量库 payload 只放过滤需要的字段(如 doc_id、tenant_id、chunk_index),全文放 PG/对象存储——这也是 Qdrant 文档"只索引要过滤的字段"建议的配套设计。✅已验证(capacity planning payload 章节)

---

## 2. 端到端容量估算示例

> 约定:Qdrant 公式系(✅官方);未加说明的为**按公式推算**。GB 按 10^9 bytes,GiB 按 2^30。m=16。

### 2.1 算例一:100 万条 × 1024 维(中型 RAG,单机)

| 项 | 公式 | 结果 |
|---|---|---|
| 原始向量 float32 | 1e6 × 1024 × 4 | **4.10 GB**(3.81 GiB) |
| fp16 存储 | × 2 | 2.05 GB |
| HNSW 图 | 1e6 × 16×2×4×1.2 | **0.15 GB**(官方口径) |
| id_tracker | 1e6 × 52 | 0.05 GB |
| RAM(Qdrant 默认 cached,无副本) | (4.10+0.15+0.05) × 1.2 | **≈ 5.2 GB** |
| Disk(含 1KB payload ×1.5) | (4.10+0.15+1.5+0.05) × 1.2 | **≈ 7.0 GB** |
| int8 量化后向量内存 | 1e6 × 1024 × 1 | **1.02 GB**(4x↓) |
| PQ(32 bytes/条) | 1e6 × 32 | **32 MB**(32x↓) |
| binary | 1e6 × 128 | **128 MB**(32x↓) |

参考:OpenSearch 官方同型算例(1M × 256d × m16 → 1.267 GB 总索引内存)验证公式量级 ✅。

### 2.2 算例二:1000 万条 × 1536 维(OpenAI 级 embedding,生产)

| 项 | 公式 | 结果 |
|---|---|---|
| 原始向量 float32 | 1e7 × 1536 × 4 | **61.4 GB**(57.2 GiB) |
| HNSW 图 | 1e7 × 16×2×4×1.2 | **1.54 GB** |
| id_tracker | 1e7 × 52 | 0.52 GB |
| RAM(Qdrant cached,replication=1) | (61.4+1.54+0.52) × 1.2 | **≈ 76 GB** → 建议 96~128 GB 机器 |
| RAM(replication=2) | base×2 | ≈ 152 GB → 3 节点集群起步 |
| Disk(float32 + 1KB payload) | (61.4+1.54+15+0.52) × 1.2 | **≈ 94 GB** |
| int8 量化 | 1e7 × 1536 × 1 | **15.4 GB**(内存降 4x,原向量落盘) |
| binary + 4x oversampling | 1e7 × 192 | **1.9 GB** |
| 原文存储(PG/对象存储) | 1e7 × 2KB | ≈ 20 GB |

**实测对照(Supabase,✅已验证,https://supabase.com/blog/increase-performance-pgvector-hnsw )**:1M × 1536 维 OpenAI 向量在 Supabase 上用 HNSW(m=24/ef_construction=56),*"Postgres uses approximately 30-35GB of RAM to achieve optimal performance"*——即 PG 实际内存 ≈ 理论向量体积(6.1GB)的 5 倍(索引膨胀+TOAST+work_mem)。推算 10M 条在 PG+pgvector 需 300GB+ RAM,基本不可行 → **千条万级以上优先专用向量库或量化**。

### 2.3 算例三:1 亿条 × 1024 维(亿级)

| 方案 | 内存 | 磁盘 | 说明 |
|---|---|---|---|
| float32 + HNSW 全内存(按公式推算) | 381 + 15.4 ≈ **400 GB+** | 同量级 | Qdrant 官方:"a single node typically tops out around 100 million vectors"(768d cached 场景) |
| int8 + HNSW(按公式推算) | **95 GB** + 图 15 GB | ~100 GB | 单台大内存机或 2~3 节点 |
| binary/PQ + rescoring(按公式推算) | PQ(32B)≈ **3.2 GB**;binary ≈ **11.9 GB** | 原向量落盘 | Qdrant cold tier + quantized pinned 模式 |
| Milvus DiskANN(✅官方默认参数) | PQ 码 = raw/8 ≈ **48 GB** | Vamana 图 + 原向量在 NVMe SSD | 面向 billion-scale |
| 对照:OpenSearch 1B×128d(✅AWS 官方) | HNSW 1408 GB vs IVFPQ **70 GB** | — | 20x 内存差,recall 代价见 1.4 |

### 2.4 RAG 全链路成本框架(加分项)

```text
总成本 ≈ embedding 生成 + 向量库存储/计算 + 原文存储 + LLM token

1) embedding 生成:chunks × avg_tokens × 单价
   (例:10M chunks × 512 tokens × $0.02/1M tokens ≈ $100,按 OpenAI text-embedding-3-small 牌价推算 ⚠️未验证牌价)
2) 存储:见 2.2/2.3;云托管价以 VectorDBBench cost leaderboard 口径横向比
3) LLM token:QPS × 上下文长度(top-k chunks tokens + prompt)× 单价
```

---

## 3. 技术选型对比

### 3.1 候选系统事实卡(逐条附来源)

**Milvus / Zilliz Cloud**
- 许可证 Apache 2.0(✅LICENSE 文件);形态:Lite(嵌入式)/Standalone/Distributed(K8s)✅(https://milvus.io/llms.txt )
- 混合检索:BM25 full-text function 建集合时声明;`hybrid_search()` 多路 AnnSearchRequest + ranker;partition 支持多租户分片 ✅(llms.txt)
- 索引:HNSW/IVF/DiskANN/AISAQ;mmap 装载 4x 数据 ✅
- VectorDBBench 榜单:Milvus-16c64g-sq8 在 $1000/月档 5,973 QPS;ZillizCloud 13,316 QPS ✅(https://zilliz.com/benchmark )

**Qdrant**
- Apache 2.0(✅LICENSE);Rust 单二进制,Standalone + 分布式分片/副本
- 官方 capacity planning 全套公式、三档 memory tier、sizing calculator(sizing.qdrant.tech)✅
- 稀疏向量 + 文本检索做混合检索(capacity planning sparse 章节)✅;榜单上 QdrantCloud-16c64g recall 0.9474(榜单最高)✅

**Weaviate**
- BSD-3-Clause + Weaviate BSL 双许可(✅LICENSE);"primary database" 定位:对象 + 向量 + 倒排(LSM store),BM25 BlockMaxWAND,filters 用 Roaring bitmaps ✅(https://weaviate.io/llms.txt )
- 一等公民 multi-tenancy(官方宣称 first-class)✅;dynamic index(flat→HNSW,阈值默认 10,000 对象)适合"多数租户小(~100k)、少数大"的 SaaS ✅(vector-index 文档)

**pgvector(+ pgvectorscale)**
- PostgreSQL License(✅pgvectorscale LICENSE;pgvector 同族 ⚠️)
- `vector` 2000 维(HNSW)/`halfvec` 4000 维/`bit` 64000 维 ✅(README)
- 过滤是 **post-filter**:"With approximate indexes, filtering is applied *after* the index is scanned";ef_search 默认 40,过滤选择率 10% 时平均只剩 4 行命中 → 0.8.0+ 用 iterative index scans(strict/relaxed order)补偿 ✅(README)
- 多租户警告(官方):"sharing an approximate index between tenants means vectors from one tenant can affect recall (and speed) for other tenants" ✅(README)
- 性能:Supabase 基准 HNSW 比 IVFFlat 高 3~6x(同 recall)✅;pgvectorscale vs Pinecone s1:50M×768d @99% recall,"28x lower p95 latency, 16x higher query throughput" ✅(pgvectorscale README)
- 定位:已有 PG、数据 <~1M、事务一致性需求、团队无新运维栈时的"就近"选择

**Elasticsearch / OpenSearch**
- Elastic 官方 dense_vector:int8 4x / int4 8x / bbq 32x,原向量保留磁盘供 rescore;oversample 默认 3x ✅
- OpenSearch HNSW 内存公式官方化(见 1.2b),支持 disk-based(⚠️细节未验证)
- 许可证:Elastic License 2.0/SSPL/AGPLv3 多重(⚠️未验证);OpenSearch Apache 2.0(AWS 分叉,⚠️未验证 LICENSE 原文)
- 定位:已有 ES 栈时"就近"选择,BM25+向量混合检索天然成熟

**Chroma**
- Apache 2.0(✅官方 intro 页);"open-source data infrastructure for AI",dense/sparse/hybrid + full-text/regex,本地/自托管/Chroma Cloud ✅;单机为主、原型阶段友好(⚠️架构细节未验证)

**FAISS**
- MIT(✅README);"a library for efficient similarity search",非数据库:无持久化/CRUD/过滤/多租户;支持 IVF/HNSW/PQ/binary,GPU,"scale to billions of vectors in main memory on a single server" ✅(README)
- 定位:离线批量构建索引、研究/嵌入库底层;生产 RAG 直连 FAISS 的常见踩坑 = 持久化与更新(⚠️搜索可见的社区共识)

**LanceDB**
- Lance 格式 Apache 2.0(✅LICENSE);"multimodal lakehouse for AI",OSS 嵌入式库 + Cloud;向量/全文/混合检索 ✅(docs.lancedb.com);磁盘列存、serverless 按存储计费定位(⚠️索引细节未验证)

**Vespa / Vald(可选)**:Vespa Apache 2.0、Vald Apache 2.0(⚠️均未验证,搜索可见)。

### 3.2 对比表骨架(可直接用于培训)

| 维度 | Milvus | Qdrant | Weaviate | pgvector | ES/OpenSearch | Chroma | FAISS | LanceDB |
|---|---|---|---|---|---|---|---|---|
| 类型 | 分布式 DB | DB | DB | PG 扩展 | 搜索引擎 | 嵌入式→DB | 库 | 嵌入式/湖仓 |
| 许可证 | Apache 2.0 ✅ | Apache 2.0 ✅ | BSD-3+BSL ✅ | PostgreSQL ✅ | Elastic L2/SSPL ⚠️ | Apache 2.0 ✅ | MIT ✅ | Apache 2.0 ✅ |
| 索引 | HNSW/IVF/DiskANN | HNSW(+量化) | HNSW+RQ8/HFresh | HNSW/IVFFlat/SPANN(pvs) | HNSW/IVFPQ/DiskANN | HNSW | 全系(IVF/HNSW/PQ/GPU) | IVF_PQ 等 ⚠️ |
| 过滤 | pre-filter(标量/分区) ✅ | pre-filter,payload index ✅ | Roaring bitmap pre-filter ✅ | post-filter + iterative(0.8.0+)✅ | pre/post 可配 ⚠️ | metadata 过滤 ⚠️ | 无(ID 选择) ⚠️ | SQL 过滤 ⚠️ |
| 混合检索 | BM25+hybrid_search ✅ | sparse vector+fusion ✅ | BM25(BlockMaxWAND)✅ | 需 tsvector 自拼 ⚠️ | 原生 BM25+kNN ✅ | dense/sparse/hybrid ✅ | 无 | hybrid ✅ |
| 多租户 | partition key ✅ | 多 collection/payload ✅ | first-class MT ✅ | 官方提醒共享索引有干扰 ✅ | 索引别名 ⚠️ | 多 DB ⚠️ | 无 | 多表 ⚠️ |
| 分布式/副本 | K8s,强 ✅ | 分片+replication ✅ | 集群+复制 ✅ | PG 原生(读副本) | 原生 | 弱 | 无 | serverless ⚠️ |
| 内存策略 | mmap(4x)/DiskANN | pinned/cached/cold tiers | cache+HFresh 磁盘化 | shared_buffers 限制 | circuit breaker 50% | — | 手动 | 磁盘优先 |
| 基准(VDBBench $1k/月档) | 5,973 QPS ✅ | 榜单最高 recall 0.9474 ✅ | 未上榜(此页) | 未上榜(此页) | OpenSearch 3,055 / ES 2,354 QPS ✅ | 未上榜 | 不适用 | TurboPuffer 类 650 QPS ✅ |
| 云托管 | Zilliz Cloud | Qdrant Cloud | Weaviate Cloud | Supabase/RDS/自管 | Elastic Cloud/AOS | Chroma Cloud | 无 | LanceDB Cloud |
| 典型量级 | 亿~百亿 | 千万~亿 | 千万~亿 | <1M(舒适区) | 千万~亿 | <1M | 离线任意 | 亿(冷数据)⚠️ |

(✅=该格事实已验证来源见上文;⚠️=待补/常识)

### 3.3 选型决策树(综合官方数据)

```text
数据 <10 万,原型/个人项目
  └─ Chroma / LanceDB / FAISS(最快出活,别当生产 DB)
数据 10 万~100 万
  ├─ 已有 PostgreSQL? ──→ pgvector(团队零新增运维;注意 ef_search 与过滤,0.8.0+ iterative scan)
  ├─ 需要 BM25 混合 + 强过滤? ──→ Qdrant(sparse+fusion)/ Weaviate(BM25+MT)/ ES
  └─ 纯 ANN 极致性能? ──→ Qdrant / Milvus Standalone
数据 100 万~1000 万
  ├─ PG 重度用户且 QPS 低(<50)─→ pgvector + int8 量化意识(实测 1M×1536 需 30-35GB RAM)
  ├─ 高 QPS + 过滤 + 多租户 SaaS ─→ Qdrant / Weaviate / Milvus
  └─ 已有 ES 集群 ─→ ES/OpenSearch kNN(bbq 32x 省内存)
数据 1000 万~1 亿+
  ├─ 有 K8s/运维团队,高 QPS ─→ Milvus Distributed(榜单 QPS 领先)或 Qdrant 集群(3 节点+replication 2 官方建议)
  ├─ 内存预算受限 ─→ 量化(cold tier + quantized pinned)或 DiskANN(Milvus/pgvectorscale/OpenSearch)
  └─ 冷数据/成本敏感 ──→ LanceDB / on-disk 方案
万用原则:先测自己数据(recall/QPS/成本),VectorDBBench 可复现,sizing 工具:Qdrant sizing.qdrant.tech、Milvus /tools/sizing
```

**企业实践选型理由(真实文章,搜索可见 ⚠️,共性结论)**:
- 从 FAISS 迁移的理由:FAISS 无持久化、无 CRUD、无认证/过滤,重启丢数据,横向扩展要自研(https://dev.to 、towardsai 多篇 2026 年文章);与 FAISS README 自述"library"一致 ✅。
- 选 pgvector 的理由:复用 PG 运维/备份/权限,数据量不大时性能足够;pgvectorscale 后已能对标专用引擎(28x p95 vs Pinecone)✅。
- 选 Milvus 的理由:十亿级规模 + 高 QPS + 云原生(K8s)生态;VectorDBBench 同价位 QPS 领先 ✅。

---

## 4. 来源清单(共 23 项,21 项 ✅已验证)

| # | 来源 | 状态 | 用途 |
|---|---|---|---|
| 1 | https://qdrant.tech/documentation/capacity-planning/ | ✅(markdown 抓取) | 全部容量公式、tier、+20% headroom、100M/节点 |
| 2 | https://qdrant.tech/documentation/guides/quantization/ | ✅ | scalar/PQ/binary 压缩比、rescore/oversample、recall 数 |
| 3 | https://qdrant.tech/documentation/manage-data/indexing/ | ✅ | m=16、ef_construct=100、full_scan_threshold(1KB=256 维向量) |
| 4 | https://qdrant.tech/documentation/concepts/storage/ | ✅ | memmap、segment 结构 |
| 5 | https://sizing.qdrant.tech/ | ⚠️(官方 capacity planning 页引用) | Qdrant Sizing Calculator |
| 6 | https://milvus.io/docs/diskann.md | ✅(web_reader) | DiskANN 参数与 PQCodeBudgetGBRatio=0.125 |
| 7 | https://milvus.io/docs/mmap.md | ✅(web_reader) | mmap 4x 数据、HNSW 最适合 mmap |
| 8 | https://milvus.io/llms.txt | ✅ | BM25/hybrid_search 规则、partition、部署形态、sizing tool 链接 |
| 9 | https://milvus.io/tools/sizing | ⚠️(llms.txt 列出,页面 403 未抓) | Milvus Sizing Tool |
| 10 | https://github.com/pgvector/pgvector | ✅ | 4d+8 bytes、维度上限、post-filter、iterative scan、MT 警告 |
| 11 | https://github.com/timescale/pgvectorscale | ✅ | 28x/16x vs Pinecone、StreamingDiskANN、SBQ 2/1 bit |
| 12 | https://supabase.com/blog/increase-performance-pgvector-hnsw | ✅ | 1M×1536 需 30-35GB RAM、HNSW vs IVFFlat 3~6x |
| 13 | https://docs.weaviate.io/weaviate/concepts/vector-index | ✅ | HNSW 内存构成、dynamic index 阈值 1 万、RQ8/HFresh |
| 14 | https://weaviate.io/llms.txt | ✅ | LSM 存储、Roaring bitmap、BM25、first-class MT |
| 15 | https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector | ✅ | int8/int4/bbq 4x/8x/32x、磁盘开销、oversample 3x |
| 16 | https://github.com/zilliztech/VectorDBBench | ✅ | 基准工具、数据集、FTS 场景 |
| 17 | https://zilliz.com/benchmark | ✅ | $1000/月档 QPS/recall 横向数据 |
| 18 | https://github.com/facebookresearch/faiss | ✅ | library 定位、MIT、billion-scale in-memory |
| 19 | https://docs.trychroma.com/docs/overview/introduction | ✅ | 定位、Apache 2.0 |
| 20 | https://docs.lancedb.com/ | ✅ | lakehouse 定位、OSS 嵌入式 |
| 21 | https://docs.opensearch.org/latest/mappings/supported-field-types/knn-methods-engines/ | ✅ | HNSW/IVF 内存公式与官方算例、circuit breaker 50% |
| 22 | https://aws.amazon.com/blogs/big-data/choose-the-k-nn-algorithm-for-your-billion-scale-use-case-with-opensearch/ | ✅ | 1B×128d:HNSW 1408GB vs IVFPQ 70GB、$75 vs $11/h |
| 23 | http://ann-benchmarks.com/ | ✅ | recall vs QPS 图、38 个算法(hnswlib/faiss-ivf/vamana/pgvector/qdrant/Milvus) |

**未决/待补**:Elastic 许可证细节、OpenSearch disk-based ANN 细节、LanceDB 索引参数、Chroma 生产规模上限、Vald/Vespa——均标 ⚠️,建议后续单独一轮验证。
