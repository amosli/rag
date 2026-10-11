# RAG 选型与容量规划

> 版本 v0.1 · 2026-09-26 联网调研 · 适用范围：向量数据库存储容量计算、量化压缩选型、向量库技术选型依据、RAG 全链路成本框架。
>
> 依据来源：Qdrant capacity planning / Milvus（DiskANN、mmap）/ pgvector & pgvectorscale / OpenSearch 官方公式 / AWS 亿级实测 / VectorDBBench / Supabase 基准等 21 项实抓核验来源（2026-09-26），调研底稿 `research/E-存储与选型.md`；检索调参部分交叉引用 `research/C`。
>
> 条款编号承接 01/02（【RG1~13】），本篇为 **【RG14~18】**。标注"按公式推算"的数字未经实测，作为规划量级参考。

---

## 1. 存储容量计算 【RG14.x】

### 1.1 基础公式（所有系统通用）

```text
raw_bytes = num_vectors × dimensions × bytes_per_dim

float32: 4 bytes/维（绝大多数 embedding 默认）      fp16/bf16: 2 bytes/维
int8 标量量化: 1 byte/维                            binary(1-bit): dims/8 bytes
```

- pgvector 官方：`vector` 类型每条 `4 × dimensions + 8` bytes，上限 16,000 维；`halfvec` 4,000 维；`bit` 64,000 维。来源：https://github.com/pgvector/pgvector ✅
- Qdrant 官方：`dense_size = base × dimensions × bytes_per_dim`（base = 点数 × 副本数）。来源：https://qdrant.tech/documentation/capacity-planning/ ✅
- Elastic 官方：float=4B/维；int8 省 75%（4x）；int4 省 8x；bbq（1-bit）省 96%（32x）。来源：https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector ✅

**经验法则**：`GB ≈ 维度 × 4 × 条数 / 10⁹`（裸 float32 向量，不含索引）。1M × 768 维 ≈ 2.87 GB。

### 1.2 HNSW 图索引额外开销（三家官方口径互证）

| 来源 | 公式 | 默认 m=16 时每向量 |
|---|---|---|
| Qdrant ✅ | `hnsw_size = base × m × 2 × 4 × 1.2`（官方算例：1M 点×副本2×m16 ≈ 0.29 GB） | ~154 bytes |
| OpenSearch ✅ | `1.1 × (4 × dim + 8 × m)` bytes/向量（官方算例：1M×256d×m16 → 1.267 GB） | ~128 bytes |
| Weaviate ✅ | 节点 ≈ 4B×dim，边 ≈ 10B×20 连接（偏上界，含对象 ID/metadata） | ~200 bytes+ |

来源：https://qdrant.tech/documentation/capacity-planning/ ✅、https://docs.opensearch.org/latest/mappings/supported-field-types/knn-methods-engines/ ✅、https://docs.weaviate.io/weaviate/concepts/vector-index ✅

【RG14.1】**HNSW 图开销 ≈ 每向量 128~250 bytes（m=16），相对 768~1536 维 float32 向量（3~6KB/条）只占 3~8%——容量规划的主项永远是向量本身，不是图**；但 pgvector 的 HNSW 索引内还会复存储向量本身用于距离计算，Postgres 实际内存远超理论值（见 §2.2 Supabase 实测）。

### 1.3 量化压缩：压缩比与召回补偿

来源：https://qdrant.tech/documentation/guides/quantization/ ✅、https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector ✅

| 方法 | 压缩比 | 速度 | 质量 | 补偿手段 |
|---|---|---|---|---|
| Scalar int8 | **4x** | 最快（SIMD） | 误差通常 <1% | `quantile 0.99` 裁极值 |
| PQ（乘积量化） | **最高 64x**（常用 16x；dim/m × 1 byte） | 慢（非 SIMD 友好） | 高维才推荐 | rescoring |
| Binary（1-bit） | **32x**（1.5/2-bit：24x/16x） | 最高 40x 提速 | 仅适合高维中心化分布 | **必须 rescoring + oversampling** |

- 官方 recall 实测：binary 下 ada-002（1536d）**0.98 recall@100 @ 4x oversampling**；Cohere 4096d 0.98@50 @ 2x。
- oversampling 定义（Qdrant 原文）："if oversampling is 2.4 and limit is 100, then 240 vectors will be pre-selected"。
- Elastic 侧：int8/int4/bbq 均需"keep the raw float vector values on disk for reranking"（磁盘开销 +25%/+12.5%/+3.1%）；bbq 默认 3.0x oversampling。

【RG14.2】**量化选型序：int8（4x，几乎无代价）→ binary（32x，需 oversampling 2~4x 补偿）→ PQ（64x，仅内存极限）**；量化省的是内存与距离计算，**原向量仍要留在磁盘上供 rescoring**——磁盘预算不省。

### 1.4 内存分层与磁盘化

- **Qdrant 三档** ✅：`pinned`（堆内存常驻）/ `cached`（mmap 预热）/ `cold`（mmap 按需）。官方口径："if you store half as many vectors in RAM, search latency will roughly double"；**HNSW 图不要放 cold**（图遍历是小随机读，磁盘延迟致命）。RAM/Disk 终值都要 **+20% headroom**（page cache、WAL、快照、临时段）。来源：https://qdrant.tech/documentation/capacity-planning/ ✅
- **Milvus** ✅：mmap 后单节点可容纳约 **4 倍**数据量（2C8G 对比实测），HNSW 最适合 mmap，需 NVMe SSD（https://milvus.io/docs/mmap.md ✅）；DiskANN = Vamana 图在磁盘 + PQ 码在内存（`PQCodeBudgetGBRatio` 默认 0.125 即 8x，可调 16x/4x），面向 billion-scale（https://milvus.io/docs/diskann.md ✅）。
- **pgvectorscale** ✅：`StreamingDiskANN` + SBQ 压缩（每维 2 bit/1 bit）。来源：https://github.com/timescale/pgvectorscale ✅

**极端对照（AWS 官方实测，10 亿条 × 128 维，OpenSearch）** ✅ https://aws.amazon.com/blogs/big-data/choose-the-k-nn-algorithm-for-your-billion-scale-use-case-with-opensearch/ ：HNSW 全内存 ≈ **1,408 GB**（~$75/h）vs IVFPQ ≈ **70 GB**（~$11/h，1/20 内存）；代价 recall@10 从 0.99 掉到 0.17~0.61。

### 1.5 元数据/payload 开销（Qdrant 官方）✅

```text
payload 磁盘 = base × avg_payload_size × 1.5        payload 内存(cached) = 上式 × 3
payload 索引 ≈ 被索引字段大小 × 2 (pinned)           id_tracker = base × 52 bytes (常驻 RAM)
官方算例: 1M 点 × 副本2 × 1KB payload(cold) → disk ≈ 2.86 GB
稀疏向量: sparse_size = base × nnz × bytes_per_dim;  sparse_index = 上式 × 1.5 (pinned, 吃 RAM)
```

【RG14.3】**向量库 payload 只放过滤需要的字段**（doc_id / tenant_id / chunk_index / date / authority），chunk 全文放 Postgres/对象存储——payload 既便宜又该省，全文进 payload 会三倍膨胀内存。

---

## 2. 三个端到端算例（GB 按 10⁹，m=16，Qdrant 公式系）

### 2.1 算例一：100 万条 × 1024 维（中型 RAG，单机）

| 项 | 公式 | 结果 |
|---|---|---|
| 原始向量 float32 | 1e6 × 1024 × 4 | **4.10 GB** |
| HNSW 图 | 1e6 × 16×2×4×1.2 | 0.15 GB |
| id_tracker | 1e6 × 52 | 0.05 GB |
| **RAM**（默认 cached，无副本，×1.2） | (4.10+0.15+0.05) × 1.2 | **≈ 5.2 GB** |
| **Disk**（+1KB payload×1.5，×1.2） | (4.10+0.15+1.5+0.05) × 1.2 | **≈ 7.0 GB** |
| int8 量化后 | 1e6 × 1024 × 1 | 1.02 GB（4x↓） |
| PQ（32B/条） / binary | — | 32 MB / 128 MB |

### 2.2 算例二：1000 万条 × 1536 维（OpenAI 级 embedding，生产）

| 项 | 公式 | 结果 |
|---|---|---|
| 原始向量 float32 | 1e7 × 1536 × 4 | **61.4 GB** |
| HNSW 图 + id_tracker | — | 1.54 + 0.52 GB |
| **RAM**（cached，副本 1，×1.2） | — | **≈ 76 GB** → 建议 96~128 GB 机型 |
| RAM（副本 2） | ×2 | ≈ 152 GB → 3 节点集群起步 |
| **Disk**（+1KB payload） | — | ≈ 94 GB |
| int8 量化 | 1e7 × 1536 × 1 | 15.4 GB（原向量落盘） |
| binary + 4x oversampling | 1e7 × 192 | 1.9 GB |
| 原文存储（PG/对象存储） | 1e7 × 2KB | ≈ 20 GB |

⚠️ **pgvector 实测对照（Supabase 官方基准）** ✅ https://supabase.com/blog/increase-performance-pgvector-hnsw ：1M × 1536 维用 HNSW（m=24/ef_construction=56）时 *"Postgres uses approximately 30-35GB of RAM to achieve optimal performance"*——**实际内存 ≈ 理论向量体积（6.1GB）的 5 倍**（索引膨胀 + TOAST + work_mem）。按公式推算 10M 条在 pgvector 需 300GB+ RAM，基本不可行 → **千万级以上优先专用向量库或量化**。

### 2.3 算例三：1 亿条 × 1024 维（亿级）

| 方案 | 内存 | 说明 |
|---|---|---|
| float32 + HNSW 全内存（按公式推算） | ≈ 400 GB+ | Qdrant 官方："a single node typically tops out around 100 million vectors"（768d cached） |
| int8 + HNSW（按公式推算） | ≈ 95 GB + 图 15 GB | 单台大内存机或 2~3 节点 |
| PQ（32B/条）+ cold tier（按公式推算） | ≈ 3.2 GB + 原向量落盘 | 延迟上升（cold 半量在 RAM → 延迟翻倍） |
| Milvus DiskANN（官方默认参数）✅ | PQ 码 = raw/8 ≈ 48 GB | Vamana 图 + 原向量在 NVMe SSD，billion-scale 定位 |
| 对照：OpenSearch 1B×128d ✅ | HNSW 1408 GB vs IVFPQ 70 GB | 20x 内存差，recall 代价见 §1.4 |

### 2.4 原文存储（配套预算）

中文 chunk 512 字 ≈ 1.5 KB（UTF-8 3 bytes/汉字）+ 元数据 0.2~1 KB → **按 2~3 KB/条估**；10M chunks ≈ 20~30 GB 纯文本；放 Postgres 另加 PK/索引/TOAST 膨胀 1.5~2x（按经验推算）。

---

## 3. 向量数据库技术选型 【RG15~16】

### 3.1 候选系统事实卡（许可证与关键事实均已核验，详见 `research/E` §3.1）

| 系统 | 类型/许可 | 索引 | 过滤 | 混合检索 | 多租户 | 舒适量级 |
|---|---|---|---|---|---|---|
| **Milvus/Zilliz** | 分布式 DB，Apache 2.0 ✅ | HNSW/IVF/DiskANN | pre-filter（分区/标量）✅ | BM25 + hybrid_search ✅ | partition key ✅ | 亿~百亿 |
| **Qdrant** | DB（Rust 单二进制），Apache 2.0 ✅ | HNSW+全套量化 | pre-filter，payload index ✅ | sparse vector + fusion ✅ | 多 collection/payload ✅ | 千万~亿 |
| **Weaviate** | DB，BSD-3+BSL ✅ | HNSW+RQ8 默认/HFresh | Roaring bitmap pre-filter ✅ | BM25（BlockMaxWAND）✅ | first-class ✅ | 千万~亿 |
| **pgvector(+pvs)** | PG 扩展，PostgreSQL 许可 ✅ | HNSW/IVFFlat/StreamingDiskANN | **post-filter** + 0.8.0+ iterative scan ✅ | 需 tsvector 自拼 ⚠️ | 官方警告共享索引互相干扰 ✅ | <1M 舒适区 |
| **ES/OpenSearch** | 搜索引擎（Elastic L2/SSPL ⚠️ / Apache 2.0 ⚠️） | HNSW/IVFPQ/DiskANN | 可配 ✅ | 原生 BM25+kNN ✅ | 索引别名 ⚠️ | 千万~亿 |
| **Chroma** | 嵌入式→DB，Apache 2.0 ✅ | HNSW | metadata ⚠️ | dense/sparse/hybrid ✅ | 多 DB ⚠️ | <1M |
| **FAISS** | 库（非 DB），MIT ✅ | 全系+GPU | 无 | 无 | 无 | 离线任意 |
| **LanceDB** | 嵌入式/湖仓，Apache 2.0 ✅ | IVF_PQ 等 ⚠️ | SQL ⚠️ | hybrid ✅ | 多表 ⚠️ | 亿（冷数据）⚠️ |

基准参照（VectorDBBench，$1000/月档 ✅ https://zilliz.com/benchmark ）：Milvus-16c64g-sq8 5,973 QPS；ZillizCloud 13,316 QPS；QdrantCloud recall 0.9474（该页最高）；OpenSearch 3,055 / ES 2,354 QPS。ann-benchmarks（38 算法 recall-QPS 曲线）✅ http://ann-benchmarks.com/ 。**基准只回答量级匹配，最终以自己数据集复测为准（VectorDBBench 可复现）。**

### 3.2 pgvector 的三个官方警告（选它之前必读）

来源：https://github.com/pgvector/pgvector ✅

1. **过滤是 post-filter**："With approximate indexes, filtering is applied *after* the index is scanned"——过滤选择率 10% 时平均只剩 4 行命中；0.8.0+ 用 iterative index scans 补偿；
2. **多租户共享索引互相污染 recall**："sharing an approximate index between tenants means vectors from one tenant can affect recall (and speed) for other tenants"；
3. **内存实际开销 ≈ 理论值 5 倍**（§2.2 Supabase 实测）。

【RG15.1】pgvector 的正确定位：**已有 PG、数据 <~1M、需要事务一致性、团队不想新增运维栈时的"就近"选择**；pgvectorscale 之后性能可对标专用引擎（官方口径：50M×768d @99% recall 下 vs Pinecone s1，28x lower p95 / 16x throughput ✅），但量级与过滤语义的天花板要清楚。

【RG15.2】FAISS 的正确定位：**离线批量构建索引、研究、底层库**；直连生产 RAG 的常见事故 = 无持久化/无 upsert（1 万→50 万篇后 200ms→3s、加文档重建索引）、无过滤、无多租户。来源：https://github.com/facebookresearch/faiss ✅、https://blog.csdn.net/weixin_39042870/article/details/162609188 ✅

### 3.3 选型决策树

```text
数据 <10 万，原型/个人项目
  └─ Chroma / LanceDB / FAISS（最快出活，别当生产 DB）

数据 10 万~100 万
  ├─ 已有 PostgreSQL? ──────→ pgvector（零新增运维；注意 post-filter 与内存 5x 定律）
  ├─ 需要 BM25 混合 + 强过滤? → Qdrant（sparse+fusion）/ Weaviate（BM25+多租户）/ ES
  └─ 纯 ANN 极致性能? ──────→ Qdrant / Milvus Standalone

数据 100 万~1000 万
  ├─ PG 重度用户且 QPS 低(<50) → pgvector + int8 量化意识（1M×1536 实测需 30-35GB RAM）
  ├─ 高 QPS + 过滤 + 多租户 SaaS → Qdrant / Weaviate / Milvus
  └─ 已有 ES 集群 ──────────→ ES/OpenSearch kNN（bbq 32x 省内存）

数据 1000 万~1 亿+
  ├─ 有 K8s/运维团队，高 QPS ─→ Milvus Distributed（榜单 QPS 领先）或 Qdrant 集群
  ├─ 内存预算受限 ──────────→ 量化（cold tier + quantized pinned）或 DiskANN
  └─ 冷数据/成本敏感 ───────→ LanceDB / on-disk 方案

万用原则: 先测自己数据（recall/QPS/成本）; sizing 工具:
  Qdrant sizing.qdrant.tech ⚠️ / Milvus milvus.io/tools/sizing ⚠️
```

【RG16.1】**"就近"原则优先于"最优"原则**：已有 ES → 用 ES kNN（混合检索天然成熟）；已有 PG 且量级小 → pgvector；只为引入 Milvus 而新增一整套 K8s 运维栈，在千万级以下是负 ROI。

---

## 4. 全链路成本框架 【RG17~18】

来源：`research/E` §2.4；牌价核对见《04-参考资料.md》。

```text
总成本 ≈ embedding 生成 + 向量库存储/计算 + 原文存储 + LLM token

1) embedding 生成 = chunks × avg_tokens × 单价
   例（按牌价推算）: 10M chunks × 512 tokens × $0.02/1M tokens ≈ $100（text-embedding-3-small）⚠️牌价以官网为准
   本地自部署: BGE-M3 + bge-reranker ≈ 4GB FP16 显存（社区实践口径）
2) 向量库: 按 §2 算例选档; 云托管横向比价用 VectorDBBench cost 口径
3) LLM token = QPS × (top-k chunks tokens + prompt) × 单价 —— 通常是每月大头（社区口径占 60%~80%）
   降本三件套: 分级路由（80% 简单查询走小模型）+ 语义缓存 + 上下文压缩
```

【RG17.1】**容量规划交付物清单**：向量条数与维度、精度档（float32/int8/binary）、副本数、内存/磁盘终值（×1.2 headroom）、原文存储、QPS 与延迟预算（rerank 占多少毫秒）、增长半年预估——七项齐了再选型，缺一项就是拍脑袋。

【RG18.1】**换 embedding 模型 = 全量重建索引**（向量空间不可迁移）：规划时把"重嵌入全部语料"的成本与停机窗口纳入预案；MRL 模型（Qwen3/jina/ada）可先裁维度试水再定终版维度，减少反复重建。

---

*本篇所有 ✅ 来源于 2026-09-26 实抓核验（`research/E-存储与选型.md` §4 清单，23 项中 21 项 ✅）；⚠️ 两项为官方 sizing 计算器页面（403/JS 渲染），公式与算例已从 capacity planning 文档交叉验证。*
