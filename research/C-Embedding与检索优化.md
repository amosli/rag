# C. Embedding 模型选择/优化/微调 + 相似性检索优化

> RAG 工程知识库研究笔记
> 调研日期: 2026-09-26 | 分工: 【Embedding 选型/优化/微调 + 相似性检索优化】
> 标注约定: ✅已验证 = 本次调研实际抓取并核对过原文; ⚠️未验证 = 仅搜索结果可见/记忆性知识,引用前需自行核对。
> 注: MTEB 官方榜单是动态 JS 应用,静态抓取拿不到排名表格(已尝试),榜单现状部分以模型卡内嵌数据 + 第三方文章佐证。

---

## 1. Embedding 模型选型

### 1.1 榜单现状: MTEB / MTEB-framing / C-MTEB

- **MTEB(Massive Text Embedding Benchmark)** 是事实标准; HuggingFace 榜单入口: https://huggingface.co/spaces/mteb/leaderboard (动态加载,需浏览器访问; 已尝试抓取,只返回 Space 外壳 ✅已验证[抓取行为], 排名数据 ⚠️未验证)。
- **C-MTEB**(中文版): 由 BGE 论文 C-Pack 提出, 收录 **35 个公开数据集、6 类任务**——Retrieval(主指标 nDCG@10)、Re-ranking(MAP)、STS(Spearman)、Classification(AP)、Pair-classification(AP)、Clustering(V-measure),总分为全部数据集平均。✅已验证 https://arxiv.org/html/2309.07597v2
- **榜单时间线(多语言 MTEB Multilingual, Mean(Task))**:
  - Gemini-embedding-exp-03-07: 68.37(2025-03 发布时曾居第一)
  - **Qwen3-Embedding-8B: 70.58(模型卡注明 2025-06-05 排名 #1)** ✅已验证 https://huggingface.co/Qwen/Qwen3-Embedding-8B
  - 第三方文章(2026-06)仍以 Qwen3-Embedding-8B 70.58 为 MTEB Multilingual v2 开源第一、超过 Gemini Embedding ⚠️未验证 https://airbyte.com (Best Embedding Models for RAG in 2026)
  - 结论: **2025-2026 开源第一梯队 = Qwen3-Embedding 系列;闭源 API = Gemini Embedding / OpenAI text-embedding-3-large / Voyage**。具体名次随时间变动,决策前应以榜单实时数据 + 自建评测集为准。

### 1.2 主流模型对比表(多语言/英文)

| 模型 | 厂商 | 参数量 | 输出维度 | 上下文 | 关键分数 | MRL 可变维度 | 开源/许可证 | 来源 |
|---|---|---|---|---|---|---|---|---|
| **Qwen3-Embedding-8B** | 阿里 | 7.57B | 4096 | 32K | MTEB-Multi 70.58 / Eng-v2 75.22 / C-MTEB 73.84 | 32~4096 任意 | Apache 2.0 | ✅ [HF 卡](https://huggingface.co/Qwen/Qwen3-Embedding-8B) |
| **Qwen3-Embedding-4B** | 阿里 | 4B | 2560 | 32K | MTEB-Multi 69.45 / C-MTEB 72.27 | 32~2560 | Apache 2.0 | ✅ 同上 |
| **Qwen3-Embedding-0.6B** | 阿里 | 0.6B | 1024 | 32K | MTEB-Multi 64.33 / C-MTEB 66.33 | 32~1024 | Apache 2.0 | ✅ 同上 |
| **text-embedding-3-large** | OpenAI | 闭源 | 3072 | 8192 | MTEB-Eng 64.6% | 支持 `dimensions` 参数(缩到 256 仍优于未裁剪 ada-002) | API, 约 $0.10~0.13/1M tokens | ✅ [官方文档](https://developers.openai.com/api/docs/guides/embeddings) |
| **text-embedding-3-small** | OpenAI | 闭源 | 1536 | 8192 | MTEB-Eng 62.3% | 同上 | API, 约 $0.016~0.02/1M tokens | ✅ 同上 |
| **gemini-embedding-exp-03-07** | Google | 闭源 | - | - | MTEB-Multi 68.37 | - | API | ✅ 数据引自 Qwen 模型卡对比表 |
| **jina-embeddings-v3** | Jina AI | 572M(含 LoRA) | 1024(默认) | 8192 | 多语最佳(发布时), Eng 接近 e5-mistral(仅差 ~1%) | 可截断至 32 维 | CC-BY-NC 4.0(商用走 API) | ✅ [官方博客](https://jina.ai/news/jina-embeddings-v3-a-frontier-multilingual-embedding-model) |
| **BGE-M3** | 智源 BAAI | ~568M ⚠️(卡面未标) | 1024 | 8192 | MIRACL nDCG@10: Dense 69.2 / Sparse 53.9 / ColBERT 70.5 / All 融合 71.5 | - | MIT | ✅ [HF 卡](https://huggingface.co/BAAI/bge-m3) + [论文](https://arxiv.org/html/2402.03216v5) |
| **e5-mistral-7b-instruct** | 微软 | 7B(LoRA r=16, 42M 可训练) | 4096 | 4K(RoPE 扩展演示至 32K) | MTEB-Eng 66.6(发布时 SOTA, 超 BGE-en-v1.5 的 64.2 达 2.4 分); BEIR nDCG@10 56.9 | 有 Matryoshka 截断(512/1024/2048/4096) ⚠️ | MIT | ✅ [论文](https://arxiv.org/html/2401.00368v3) + [HF 卡](https://huggingface.co/intfloat/e5-mistral-7b-instruct) |
| **gte-Qwen2-7B-instruct** | 阿里 | 7.6B ⚠️ | 3584 ⚠️ | 32K | C-MTEB 论文值 72.05; MSMARCO nDCG@10 45.98 / NQ 67.00 | - | Apache 2.0 | ✅ [HF 卡](https://huggingface.co/Alibaba-NLP/gte-Qwen2-7B-instruct)(任务级分数)/ 72.05 来自 Conan 论文表 ✅;参数/维度 ⚠️ |
| **GTE(gte-large 等初代)** | 阿里 | 110M~335M | 1024 | 512 | gte-base 以 110M 超过 OpenAI API(论文摘要); gte-large MTEB 63.13 ⚠️ | - | MIT 类 ⚠️ | ✅(摘要部分) [论文](https://arxiv.org/abs/2308.03281) |
| **voyage-3-large** | Voyage AI | 闭源 | 1024(可选 256/512/2048) | 32K | - | 是 | API $0.18/1M | ✅ [docs](https://docs.voyageai.com/docs/embeddings) + [pricing](https://docs.voyageai.com/docs/pricing) |
| **voyage-3 / 3.5** | Voyage AI | 闭源 | 1024 | 32K | - | - | API $0.06/1M | ✅ 同上 |
| **voyage-3-lite / 3.5-lite** | Voyage AI | 闭源 | 512 | 32K | - | - | API $0.02/1M | ✅ 同上 |
| **voyage-4 / 4-large / context-4 / code-4**(2026 在售) | Voyage AI | 闭源 | - | - | - | - | $0.06 / $0.12 / $0.12 / $0.12(4 家族前 200M tokens 免费) | ✅ pricing 页 |

### 1.3 中文模型对比表(C-MTEB)

| 模型 | 参数量 | 维度 | 上下文 | C-MTEB 平均 | 备注 | 来源 |
|---|---|---|---|---|---|---|
| **Conan-embedding** | 326M | 1792(线性层从 1024 扩展; MRL: 256/512/768/1024/1280/1536/1792) | 512 | **72.62**(论文自测, 曾居中文榜第一) | 动态难负例挖掘 + Cross-GPU Balance Loss | ✅ https://arxiv.org/html/2408.15710v2 |
| **xiaobu-embedding-v2** | - | - | - | 72.43 | Conan 表中对比项 | ✅ 同上 |
| **gte-Qwen2-7B-instruct** | 7.6B | 3584 ⚠️ | 32K | 72.05 | | ✅ 同上 |
| **piccolo-large-zh-v2** | ~1.2B(1.21GB 文件 ⚠️ 未标参数) | 1792(MRL: 256~1792) | 512 | **70.95**(发布时中文榜第一, 超 BERT 系 ~1.9 分) | 从 stella-v3.5 初始化; InfoNCE(in-batch neg)+CoSENT 混合损失 | ✅ https://huggingface.co/sensenova/piccolo-large-zh-v2 |
| **BGE-large-zh(C-Pack 论文值)** | 326M | 1024 | 512 | 63.96(Retrieval 71.53; 论文口径, 与榜单口径不同 ⚠️) | RetroMAE 预训练 + in-batch neg 最大 19,200 batch | ✅ https://arxiv.org/html/2309.07597v2 |
| **Qwen3-Embedding-8B** | 7.57B | 4096 | 32K | 73.84 | 当前中文最强开源之一 | ✅ https://huggingface.co/Qwen/Qwen3-Embedding-8B |

> 注意: C-MTEB 论文值(v1 口径)与 HF 榜单实时口径、模型自测口径互有差异,**横向比较必须同口径**。

### 1.4 选型决策因素清单

1. **语言匹配**: 中英混合/多语(100+ 语)→ Qwen3-Embedding、BGE-M3、jina-v3; 纯中文 → Conan、piccolo、BGE-zh 系; M3 训练语料覆盖 170+ 语但分布不均,小语种务必实测。✅ [FlagEmbedding FAQ](https://github.com/FlagOpen/FlagEmbedding/blob/master/docs/source/FAQ/index.rst)
2. **维度 vs 成本(MRL/Matryoshka)**: 支持可变维度的模型可按存储/延迟预算截断向量——OpenAI `dimensions` 参数(3-large 缩到 256 维仍胜未裁剪 ada-002 ✅)、Qwen3 32~4096 任意 ✅、jina-v3 截到 32 维 ✅、piccolo/Conan 按预设档位 ✅。
3. **上下文长度**: 长文档少切块可选 BGE-M3(8192)/Qwen3(32K)/Voyage(32K); 512 上下文模型(Conan/piccolo/BGE-zh)必须配合切块策略。
4. **推理成本与部署**: 本地开源(Apache 2.0 首选 Qwen3/BGE-M3; MIT: BGE-M3、E5 系; CC-BY-NC 需注意 jina-v3 许可) vs API(OpenAI ~$0.10/1M、Voyage $0.02~0.18/1M、Gemini)。0.6B 级(Qwen3-0.6B)适合高并发低成本在线服务。
5. **Instruction 前缀(指令感知)**:
   - Qwen3-Embedding: query 侧用 `"Instruct: {task}\nQuery: {text}"`,官方称加指令通常带来 **1~5% 提升**,建议指令用英文写; 文档侧不加。✅ HF 卡
   - BGE v1.5 中英文检索指令: `为这个句子生成表示以用于检索相关文章：` / `Represent this sentence for searching relevant passages: `,**只加在 query 侧,文档侧永远不加**。✅ [FlagEmbedding README](https://github.com/FlagOpen/FlagEmbedding) + [bge-large-en-v1.5 卡 FAQ](https://huggingface.co/BAAI/bge-large-en-v1.5)
   - BGE-M3: 官方明确 **不再需要** instruction 前缀。✅ HF 卡 FAQ
   - 是否加指令以"你自己的任务上实测更优"为准(FlagEmbedding FAQ 原话)。✅
6. **多任务适配**: jina-v3 通过 5 个 LoRA 任务适配器(retrieval.query / retrieval.passage / separation / classification / text-matching,合计 <3% 参数)切换任务模式 ✅; 非对称检索务必 query/passage 两侧分别处理。

---

## 2. Embedding 模型优化/微调

### 2.1 FlagEmbedding(BGE 官方)微调配方 —— step by step

来源: ✅ https://github.com/FlagOpen/FlagEmbedding/blob/master/examples/finetune/embedder/README.md

1. **造数据(JSONL)**: 每行 `{"query": str, "pos": List[str], "neg": List[str]}`; 可选 `pos_scores/neg_scores`(蒸馏用)、`prompt`。query 无负例时从全库随机采样兜底。
2. **难负例挖掘(hard negatives)**: 用 `hn_mine.py` 对每个 query 取 top-k 再从命中集(排除正例)随机采样:
   - 推荐参数: `--negative_number 15`、`--range_for_sampling 2-200`(即从第 2~200 名中采,"较难");
   - 想让负例更容易→放宽区间如 `60-300`。
   - 可用任意 embedder 做检索后端(`--embedder_name_or_path`,支持 faiss-gpu/fp16)。
3. **(可选)知识蒸馏**: `add_reranker_score.py` 用 `BAAI/bge-reranker-v2-m3` 打教师分,训练时 `--knowledge_distillation True` + `--kd_loss_type m3_kd_loss`(或 kl_div)。
4. **训练超参(官方 bash 脚本典型值)**:
   - 学习率: **encoder 模型(bge-large/bge-m3)= 1e-5;decoder 模型(gemma2/en-icl,LoRA r=32 α=64)= 1e-4**
   - batch: per-device 2 × `train_group_size 8`(1 query + 7 negatives/组)
   - **温度 temperature = 0.02**(InfoNCE/对比损失)
   - epochs: encoder 2 / decoder 1; `--negatives_cross_device` 跨卡共享负例; fp16 + gradient checkpointing + DeepSpeed
   - 归一化 `--normalize_embeddings True`; encoder 用 cls pooling,decoder 用 last-token pooling
5. **评估**: 在 C-MTEB/MTEB 相应子集或自建检索评测集上对比微调前后(见 §3.6)。

### 2.2 数据合成(LLM 合成训练对)

- **E5-mistral 范式**(✅ https://arxiv.org/html/2401.00368v3 + 摘要 ✅): 两阶段提示(先头脑风暴任务,再生成样本),~25% GPT-3.5-Turbo + 其余 GPT-4,共 ~180M tokens; **50 万样本、15 万条独立指令、93 种语言**;任务类型覆盖非对称(short-long/long-short/short-short/long-long)+ 对称(STS/bitext)。**仅用合成数据训到 MTEB 63.1,加 13 个标注数据集后 66.6**;总数据 ~180 万条、训练 <1k steps(摘要口径)、32×V100 约 18 小时。
- **LlamaIndex 轻量范式**(✅ https://www.llamaindex.ai/blog/fine-tuning-embeddings-for-rag-with-synthetic-data-e534409a3971): 让 LLM 对每个 chunk 出"测验题"生成 (query, chunk) 正对 + 随机负例,用 sentence-transformers `MultipleNegativesRankingLoss` 微调 `bge-small-en`:验证集 **hit-rate 78% → 84%**(ada-002 为 87%),整组指标提升 5~10%。
- 结论: 无标注时 LLM 合成对成本最低、收益明确; 大规模合成 + 少量标注混合是当前 SOTA 路径。

### 2.3 难负例挖掘进阶

- **BGE-M3 自知识蒸馏(Self-KD)消融**(✅ 论文 Table 5): 有 SKD 时 Sparse 53.9,无 SKD 掉到 **36.7**(Dense 69.2→68.7)——难例质量对稀疏分支尤其致命。
- **多阶段训练消融**(✅ BGE-M3 论文 Table 6): 仅微调 60.5 → +RetroMAE 预训练 66.1 → +无监督中间阶段 69.2。
- **Conan 动态难负例挖掘(DHNM)**(✅ https://arxiv.org/html/2408.15710v2): 每迭代 100 步检查一次,若负例得分 ×1.15 仍低于初始分且绝对值 <0.8(即"不够难")则替换为更难区段的样本; 配合 Cross-GPU Balance Loss。消融: Baseline 62.9 → Vanilla 68.8 → 仅 CBB 70.4 / 仅 DHNM 71.2 → 全量 72.62。

### 2.4 损失与温度

- 通用损失为对比学习 InfoNCE(带 in-batch negatives + 难负例),**temperature 常用 0.01~0.02**(BGE 卡面 0.01 ✅、FlagEmbedding 微调脚本 0.02 ✅、E5 论文 τ=0.02 ✅)。
- C-Pack 论文: in-batch negatives 最大 batch 做到 **19,200**(梯度检查点 + 跨设备 embedding 共享) ✅。
- 多任务中文模型 piccolo 用混合损失: 检索/重排用 InfoNCE(in-batch),STS/配对分类用 CoSENT,分类/聚类用无 in-batch 的三元组 InfoNCE ✅。

### 2.5 微调量化效果速查

| 干预 | 数据集/指标 | 前 → 后 | 来源 |
|---|---|---|---|
| 合成数据 + 难负例微调(E5-mistral 全流程) | MTEB-Eng avg | BGE-en-v1.5 64.2 → 66.6(合成-only 63.1) | ✅ arXiv 2401.00368 |
| 自建微调(bge-small-en, 合成 QA) | 验证集 hit-rate | 78% → 84%(+6pp) | ✅ LlamaIndex 博客 |
| Conan DHNM+CBB | C-MTEB | Baseline 62.9 → 72.62 | ✅ arXiv 2408.15710 |
| BGE-M3 多阶段(RetroMAE→无监督→微调) | MIRACL avg | 60.5 → 69.2 | ✅ arXiv 2402.03216 |
| BGE 加任务指令 | C-MTEB avg | 63.40 → 63.96(+0.56) | ✅ arXiv 2309.07597 |
| Qwen3 加 query 指令 | 各任务 | 官方称 +1~5% | ✅ Qwen HF 卡 |

### 2.6 常见坑

1. **微调后泛化变差 / 领域外退化**: E5 论文显示纯领域标注数据易过拟合,大规模多样合成数据反而提升 zero-shot; 难负例过难(只从 top1-10 采)会让模型把"相关但非最优"判为负,建议 `range_for_sampling 2-200` 并保留随机负例。✅(论文 + 官方脚本参数)
2. **分数绝对值无意义**: BGE FAQ 原话——对比学习温度 0.01 使相似度集中在 [0.6,1],"**>0.5 不代表相似**,关键是相对排序而非绝对值"; 不要拿余弦绝对值当阈值拍板,要按业务标注重校阈值。✅ https://huggingface.co/BAAI/bge-large-en-v1.5
3. **temperature 过大/过小**: 过小→分布过尖难训;过大→区分度不足;社区常用 0.01~0.02。(✅ 三处来源交叉)
4. **负例质量 > 数量**: 随机负例太易学不到东西;BGE-M3 消融显示无 SKD/难例时 Sparse 从 53.9 崩到 36.7。✅
5. **embedding 与 reranker 联动**: 社区反馈难负例微调 embedding 后叠加 reranker 有时反而变差(分配角色: embedding 召回宽、reranker 精排严),需在业务集上验证。⚠️未验证 https://github.com/FlagOpen/FlagEmbedding/issues/296
6. **非对比学习的"预训练 encoder"不能直接算相似度**,必须先经对比学习微调(BGE FAQ)。✅
7. 指令加错侧(加到文档侧)、未归一化就混用 dot/cosine、fp16 精度导致 CPU/GPU 结果不一致(FAQ 给出 `fp16=False` 方案)。✅ FlagEmbedding FAQ
8. 旧版 BGE FAQ 曾建议小学习率(1e-6~1e-5)、尽量大 batch、使用循环学习率(cyclic LR)调度。⚠️未验证(现行仓库文档已不含该段落)

---

## 3. 相似性检索优化

### 3.1 相似度度量: cosine vs dot vs euclidean

- **cosine(A,B) = dot(A_norm, B_norm)**——"Dot product on normalized embeddings is equivalent to cosine similarity"(sentence-transformers 官方 FAQ 原话)。因此:
  - 归一化后(L2-normalize): **cosine ≡ dot ≡ 与 euclidean 单调等价**(L2² = 2−2·cos),三者排序结果完全一致;
  - 未归一化: dot 会偏好长向量(norm 大者),此时只有 cosine/L2 语义正确。
- 工程建议: 统一 `normalize_embeddings=True` 后用 dot(省一次重复归一化,更快);SentenceTransformer 末尾带 Normalize 模块时官方明确建议选 `dot`。✅ https://sbert.net/docs/sentence_transformer/usage/semantic_textual_similarity.html
- FlagEmbedding Quick Start 亦以内积 `emb1 @ emb2.T` 计算相似度(即归一化 + dot)。✅ FlagEmbedding README

### 3.2 混合检索(BM25 + dense + 稀疏学习)

**为什么混**: BM25 精确词匹配补 dense 的语义泛化;BGE-M3 的 MIRACL 平均 nDCG@10: **BM25 31.9 / mE5-large 66.6 / M3 Dense 69.2 / M3 Sparse(学习型稀疏) 53.9 / M3 ColBERT 70.5 / Dense+Sparse 70.4 / All 融合 71.5**。✅ https://arxiv.org/html/2402.03216v5

**(1) RRF(Reciprocal Rank Fusion)**——只看排名、免调参、免分数归一化。Elasticsearch 官方公式(✅ https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html):

```
score(d) = Σ_q  1 / (k + rank_q(d))        # rank 从 1 开始
k: rank_constant,默认 60,要求 ≥1;k 越大,深排名文档影响越大
```

- ES 中 `rrf` retriever 至少挂 2 个子检索器(典型: `standard`(BM25) + `knn`(dense)),各子检索等权;`rank_window_size` 控制每路看多深(默认=size)。✅ 同上
- **加权变体**:
  - BGE-M3 论文加权线性融合: `s = w1·s_dense + w2·s_lex + w3·s_mul`;MIRACL 上 Dense+Sparse 用 **w=(1, 0.3, 0)**、All 用 **(1, 0.3, 1)**(对 top-200 dense 候选做 ColBERT 重排);长文档 MLDR 用 **(0.15, 0.5, 0.35)**(稀疏占主导)。✅ 论文
  - BGE-M3 代码示例权重 `[0.4, 0.2, 0.4]`(dense/sparse/colbert)。✅ HF 卡
  - Weaviate: `relativeScoreFusion`(v1.24 起默认,用相对分数而非排名融合)与 `rankedFusion`(RRF);`alpha` 控制权重——**alpha=1 纯向量、alpha=0 纯关键词**;BM25 腿支持按字段加权如 `question^2`,向量腿不支持字段权重。✅ https://docs.weaviate.io/weaviate/search/hybrid
  - Milvus RRFRanker/WeightedRanker(k=60 等) ⚠️未验证 https://milvus.io/docs/reranking.md (页面重定向失败)
- **实证**: Qdrant 对 5 个公开数据集做混合检索实验,"默认 RRF 在其中 4 个上击败单路更强检索器"(DBPedia-entity 除外);其同时建议用 nDCG@10 而非 MRR/Recall@100 打分。✅ https://qdrant.tech/articles/hybrid-search/
- **SPLADE/learned sparse**: 通过 MLM + max-pooling(log(1+exp)) 生成词表级稀疏权重,兼得倒排索引效率与语义扩展;论文报告 TREC-DL19 nDCG@10 提升超过 **9%**、BEIR SOTA(BEIR 平均 50.7 ⚠️)。✅摘要 https://arxiv.org/abs/2109.10086 / 细节数字 ⚠️。BGE-M3 的 sparse 输出即同思路(FAQ 原文把 BM25/unicoil/SPLADE 列为同类)。✅ FlagEmbedding FAQ

### 3.3 重排 Rerank

两阶段架构: 召回(top 50~200)→ cross-encoder/late-interaction 精排(top 5~10)。

| Reranker | 类型 | 规模/输入 | 说明 | 来源 |
|---|---|---|---|---|
| **bge-reranker-v2-m3** | cross-encoder | 567,755,777 参数;transformers 示例 max_length=512 | 多语言、轻量、易部署;输出 logit,sigmoid 后映射 [0,1];FlagEmbedding/TEI/VLLM 均可跑;Apache 2.0 | ✅ https://huggingface.co/BAAI/bge-reranker-v2-m3 |
| bge-reranker-v2-gemma / v2-layerwise | cross-encoder/层裁剪 | gemma-2b / MiniCPM-2B(8~40 层可选) | 官方选型建议: 多语选 m3 或 gemma;追效率选 m3 或 layerwise 低层;追上限选 layerwise/gemma | ✅ 同上 |
| **Cohere Rerank 3.5(rerank-v3.5)** | API | 单一多语模型 | 100+ 语言;结构化文档建议转 YAML 字符串再喂;2026 年在售新模型为 rerank-v4.0-pro / v4.0-fast;定价 rerank-3 $0.05/1M、lite $0.02/1M | ✅ https://docs.cohere.com/docs/rerank-overview + pricing ✅ |
| Cohere Rerank 3.5 vs 3 | - | - | "OOD 相关性提升 ~11.1%、疑难实体(SKU/缩写)更强、延迟更低"——正文未能抓取,**仅搜索可见** | ⚠️ https://cohere.com/blog/rerank-3pt5 (2024-12-02 发布,存档亦未含正文) |
| **ColBERT/ColBERTv2(late interaction)** | 多向量(token 级)late interaction | 每文档 N 个 token 向量 | MaxSim 后期交互;ColBERTv2 用残差压缩 + 去噪监督,把 late-interaction 存储降 **6~10×**;MRR@10 39.7(MS MARCO)⚠️(摘要未含) | ✅摘要 https://arxiv.org/abs/2112.01488 |
| Qwen3-Reranker 0.6B/4B/8B | 生成式 reranker | 与 Qwen3-Embedding 同期发布 | 配套使用 | ⚠️(Qwen 卡提及配套 reranker 家族) |

**Rerank 带来的指标提升(已验证数字)**:

- **BEIR 17 数据集 zero-shot 平均 nDCG@10: BM25 41.9 → BM25+CrossEncoder 44.7(+2.8),ColBERT 43.7**;论文原话"BM25 仅被神经重排方法击败"。✅ https://arxiv.org/html/2104.08663v2
- **BGE-M3 All 融合(含对 top-200 dense 候选的 ColBERT 重排) 71.5 vs 纯 Dense 69.2(+2.3)**(MIRACL)。✅ BGE-M3 论文
- jina late-chunking 属"改 embedding 环节"而非 rerank,顺带记录: BEIR nDCG@10,SciFact 64.20→66.10、NFCorpus 23.46→**29.98**、TRECCOVID 63.36→64.70(朴素切块 vs late chunking,~256 token 块);增益随文档变长而放大。✅ https://jina.ai/news/late-chunking-in-long-context-embedding-models

### 3.4 Query 处理

| 技术 | 做法 | 已验证效果 | 来源 |
|---|---|---|---|
| **HyDE(Hypothetical Document Embeddings)** | LLM 先生成"假设答案文档",embedding(假设文档)代替 embedding(查询)去检索;对比编码器的瓶颈会滤掉幻觉细节 | nDCG@10: TREC-COVID **Contriever 27.3 → HyDE 59.3**(≈微调版 ContrieverFT 59.6);DL19 44.5→61.3;FiQA 24.5→27.3;但仍全面低于微调检索器 | ✅ https://arxiv.org/abs/2212.10496 + https://arxiv.org/html/2212.10496v1 |
| **Multi-Query** | LLM 从不同视角生成多个 query 变体,分别检索后取并集(unique union),缓解单表述相似度召回盲区 | LangChain `MultiQueryRetriever.from_llm(...)`(官方 how-to 页已迁移,示例 query "approaches to Task Decomposition" 生成 3~4 变体) | ⚠️未验证 https://python.langchain.com/docs/how_to/MultiQueryRetriever/ (301 迁移) |
| **Step-Back Prompting** | 先抽象(高层概念/原理)再作答,可配合检索抽象层文档 | PaLM-2L: MMLU Physics +7%、Chemistry +11%、TimeQA +27%、MuSiQue +7% | ✅ https://arxiv.org/abs/2310.06117 |
| Query rewriting/expansion | 对口语化/含指代 query 先改写消歧(如"它"→实体)再检索 | 常规工程手段,公开统一数字少;Multi-Query/HyDE 为其特例 | ⚠️(综合) |

### 3.5 ANN 索引参数调优

**HNSW**(✅ https://qdrant.tech/documentation/concepts/indexing/ + faiss wiki ✅)

| 参数 | 阶段 | 作用 | 调大影响 | 推荐默认 |
|---|---|---|---|---|
| `m` | 建索引 | 每节点边数 | 召回↑、内存↑(faiss 公式: 每向量内存 = d×4 + M×2×4 字节; faiss 建议范围 **4 ≤ M ≤ 64**) | Qdrant 16; faiss 常用 32 |
| `ef_construct` | 建索引 | 建图时候选队列长度 | 召回↑、建索引耗时↑ | Qdrant 100 |
| `ef`(= hnsw_ef, 查询时) | 查询 | 搜索候选队列长度 | 召回↑、查询延迟↑;Qdrant 默认= ef_construct | 按延迟预算扫描 100→500 |
| `full_scan_threshold` | 查询 | 命中过滤条件的结果集小于阈值(KB)则放弃 HNSW 走全量扫描 | 精确召回;Qdrant 换算 1KB≈1 个 256 维向量 | 10,000(即 ~10MB) |
| `efSearch`(faiss 命名) | 查询 | 同 ef | 速度-精度旋钮 | - |

来源注: Qdrant 文档原文 "Larger the value(m) - more accurate the search, more space required";"Larger ef_construct - more accurate, more time to build"。✅

**IVF**(✅ faiss wiki "Guidelines to choose an index": https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index)

| 参数/场景 | 建议 |
|---|---|
| `nlist`(聚类数 K) | **4×√N ~ 16×√N**(N=库内向量数,<1M 场景);训练样本需 30×K ~ 256×K 条 |
| 1M~10M | `IVF65536_HNSW32`(粗聚类用 HNSW 分配);10M~100M 用 262144;100M~1B 用 1048576 |
| `nprobe`(查询探查簇数) | 速度-精度旋钮: 越大召回↑延迟↑;nprobe=nlist 退化为精确搜索 |
| 训练 | IVF 需在代表性样本上 k-means 预训练(flat/HNSW 免训练) |

**量化(压缩)对召回的影响**(✅ https://qdrant.tech/documentation/guides/quantization/)

| 方法 | 压缩 | 召回影响 | 补偿手段 |
|---|---|---|---|
| Scalar int8 | 4× | 误差通常 <1%(Qdrant 实验),还吃 SIMD 加速 | `quantile: 0.99` 裁极值 |
| Binary(1-bit) | 32×(1.5/2-bit: 24×/16×) | 需中心化高维分布(ada-002 1536d、Cohere 4096d 实测配合 oversampling 达 **0.98 recall**);最高 40× 提速 | **必须开 rescoring**(用原始向量重排 top-k)+ oversampling(如 limit 100、oversampling 2.4 → 先取 240 个再重排) |
| Product Quantization | 最高 64×(x16 压缩档常见) | 距离计算不 SIMD 友好,比 scalar 慢 | 仅在内存极限场景用 |
| TurboQuant(4/2/1.5/1-bit) | 8×/16×/24×/32× | 4-bit 召回≈scalar、压缩翻倍 | 低 bit 自动开 rescoring |
| faiss OPQ+PQ(M 个 4-bit 码) | M/2 字节/向量 | 降维+重排后压缩 | `k_factor` rerank |
| faiss RaBitQ | d/8+8 字节/向量 | 1 bit/维 + 随机旋转;2~9 bit 变体换精度 | 配 IVF/Refine |

### 3.6 检索质量评估

**指标**(与 C-MTEB/BEIR 口径一致 ✅):

- **Recall@k / Hit rate@k**: top-k 中是否含相关文档(0/1 平均)——RAG 召回环节第一指标;
- **MRR**: 首个相关文档排名倒数的平均——衡量"第一条命中的位置";
- **nDCG@10**: 折损分级相关性的排名质量,BEIR/C-MTEB 检索主指标;Qdrant 混合检索实验也建议用 nDCG@10 而非 MRR/Recall@100 ✅;
- C-MTEB 六类任务各自主指标: Retrieval=nDCG@10,Re-rank=MAP,STS=Spearman,Classification/Pair=AP,Clustering=V-measure。✅ C-Pack 论文

**评估集构建**:

1. **公开基准**: C-MTEB(中文 35 集)、BEIR(17~18 集 zero-shot)、MIRACL(多语)、MS MARCO/MTEB 任务集——用于横评模型;
2. **人工标注**: 每条 query 标相关文档,最可靠但贵;适合上线前的最终校准;
3. **LLM 合成**(当前主流做法): LlamaIndex 范式——对每个 chunk 生成 quiz 式问题作 query,chunk 即正例 ✅(hit-rate 78%→84% 那套);E5-mistral 证明 50 万级合成对 + 指令可撑起 SOTA ✅。要点: query 要模拟真实用户问法(非对称 short-long 为主),保留 holdout 防止"在合成集上过拟合、在真实分布上失效"。

---

## 4. 实践推荐(基于以上证据)

1. **中文 RAG 默认起点**: Qwen3-Embedding-0.6B/4B(32K 上下文 + MRL + Apache 2.0,指令可再 +1~5%)或 BGE-M3(8K 上下文 + dense/sparse/colbert 三路混合,MIT);预算充足要上限 → Qwen3-Embedding-8B。
2. **管线组合**: 混合检索(BM25 或 BGE-M3 sparse + dense,RRF k=60 或加权 0.4/0.2/0.4)召回 50~200 → bge-reranker-v2-m3(或 API reranker)精排 top-10;评估盯 nDCG@10 + Recall@k。
3. **先评估再优化**: 用 LLM 合成 + 人工抽检建 200~1000 条自建评测集,得到 hit rate/MRR 基线后,再决定"换模型 / 微调 / 调索引参数"哪个性价比高(经验序: rerank > hybrid > 微调 > 换更大模型,由 §2.5/§3.3 的 +2~6pp 量级支撑)。
4. **存储/延迟紧张**: MRL 截维度(Qwen3 32~4096 任意)→ scalar int8(4×,<1% 误差)→ 仍不够再 binary+rescoring(32×,oversampling 2.4 可回 0.98 recall)。

## 5. 参考来源汇总

✅已验证(本次抓取核对):
1. Qwen/Qwen3-Embedding-8B 模型卡 — https://huggingface.co/Qwen/Qwen3-Embedding-8B
2. BAAI/bge-m3 模型卡 — https://huggingface.co/BAAI/bge-m3
3. BGE-M3 论文(M3-Embedding) — https://arxiv.org/html/2402.03216v5 (abs: https://arxiv.org/abs/2402.03216)
4. FlagEmbedding 微调文档 — https://github.com/FlagOpen/FlagEmbedding/blob/master/examples/finetune/embedder/README.md
5. FlagEmbedding README(指令前缀) — https://github.com/FlagOpen/FlagEmbedding
6. FlagEmbedding FAQ — https://github.com/FlagOpen/FlagEmbedding/blob/master/docs/source/FAQ/index.rst
7. BAAI/bge-large-en-v1.5 卡 FAQ — https://huggingface.co/BAAI/bge-large-en-v1.5
8. jina-embeddings-v3 博客 — https://jina.ai/news/jina-embeddings-v3-a-frontier-multilingual-embedding-model
9. OpenAI embeddings 指南 — https://developers.openai.com/api/docs/guides/embeddings
10. Voyage embeddings/pricing — https://docs.voyageai.com/docs/embeddings , https://docs.voyageai.com/docs/pricing
11. Conan-embedding 论文 — https://arxiv.org/html/2408.15710v2 (abs: https://arxiv.org/abs/2408.15710)
12. piccolo-large-zh-v2 卡 — https://huggingface.co/sensenova/piccolo-large-zh-v2
13. E5-mistral 论文 — https://arxiv.org/html/2401.00368v3 (abs: https://arxiv.org/abs/2401.00368);HF 卡 — https://huggingface.co/intfloat/e5-mistral-7b-instruct
14. gte-Qwen2-7B-instruct 卡 — https://huggingface.co/Alibaba-NLP/gte-Qwen2-7B-instruct;GTE 论文摘要 — https://arxiv.org/abs/2308.03281
15. C-Pack(BGE/C-MTEB)论文 — https://arxiv.org/html/2309.07597v2
16. Qdrant HNSW 索引文档 — https://qdrant.tech/documentation/concepts/indexing/
17. Qdrant 量化文档 — https://qdrant.tech/documentation/guides/quantization/
18. Qdrant 混合检索文章 — https://qdrant.tech/articles/hybrid-search/
19. Elasticsearch RRF — https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html
20. Weaviate hybrid search — https://docs.weaviate.io/weaviate/search/hybrid
21. faiss wiki 选型指南 — https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index
22. sentence-transformers 相似度 FAQ — https://sbert.net/docs/sentence_transformer/usage/semantic_textual_similarity.html
23. Jina late chunking — https://jina.ai/news/late-chunking-in-long-context-embedding-models
24. HyDE 论文 — https://arxiv.org/abs/2212.10496 + https://arxiv.org/html/2212.10496v1
25. Step-Back Prompting — https://arxiv.org/abs/2310.06117
26. bge-reranker-v2-m3 卡 — https://huggingface.co/BAAI/bge-reranker-v2-m3
27. Cohere rerank 文档 — https://docs.cohere.com/docs/rerank-overview
28. ColBERTv2 摘要 — https://arxiv.org/abs/2112.01488
29. BEIR 论文 — https://arxiv.org/html/2104.08663v2 (abs: https://arxiv.org/abs/2104.08663)
30. SPLADE 摘要 — https://arxiv.org/abs/2109.10086
31. LlamaIndex 微调博客 — https://www.llamaindex.ai/blog/fine-tuning-embeddings-for-rag-with-synthetic-data-e534409a3971

⚠️未验证(搜索可见/未抓到正文,引用前需核对):
32. MTEB 榜单实时排名(JS 应用) — https://huggingface.co/spaces/mteb/leaderboard
33. Airbyte "Best Embedding Models for RAG in 2026"(Qwen3-8B 仍居首) — https://airbyte.com
34. Cohere Rerank 3.5 博客正文(11.1% OOD 等数字) — https://cohere.com/blog/rerank-3pt5
35. Milvus RRFRanker/WeightedRanker 文档 — https://milvus.io/docs/reranking.md
36. LangChain MultiQueryRetriever how-to — https://python.langchain.com/docs/how_to/MultiQueryRetriever/
37. GitHub issue #296(embedding/reranker 微调联动坑) — https://github.com/FlagOpen/FlagEmbedding/issues/296
38. 腾讯云 BGE 技术解读(120M/232M 样本对、温度 0.01 分布 [0.6,1]) — https://cloud.tencent.com/developer/article (中文搜索可见)
39. SPLADE-v2 BEIR 50.7 / ColBERTv2 MS MARCO MRR@10 39.7(正文表格未抓到,仅常见引用值)
40. e5-mistral Matryoshka 截断档位、gte-Qwen2 参数/维度 3584、gte-large MTEB 63.13(卡面/论文正文未抓到)
