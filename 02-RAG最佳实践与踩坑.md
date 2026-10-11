# RAG 最佳实践与踩坑

> 版本 v0.1 · 2026-09-26 联网调研 · 适用范围：RAG 系统从 Demo 到生产的全链路调优——召回率提升、Embedding 选型与微调、相似性检索优化、Query 处理、幻觉抑制、评估闭环、性能成本。
>
> 依据来源：英文一手（Anthropic / LlamaIndex / Qdrant / Elastic / Wang et al. / Lost in the Middle 等，2026-09-26 实抓核验）+ 中文社区实践复盘（掘金 / CSDN / 知乎 / 博客园 / 有道，20 篇实抓）；完整链接与核验状态见《04-参考资料.md》，调研底稿 `research/A|B|C`。原理与复杂逻辑见《01-RAG技术原理与复杂逻辑.md》，容量计算与选型见《03-选型与容量规划.md》。
>
> 条款编号承接 01（【RG1~5】），本篇为 **【RG6~13】**；引用社区实验数字（作者自述、未独立复现）已标注"社区实践数据"。

---

## 0. 优化总路线图（先看这张表再动手）

### 0.1 一个完整的演进曲线（投研知识库 12 万份 PDF，社区实践数据）

来源：https://blog.csdn.net/qq_43407325/article/details/163386366 ✅（Recall@10，评估集 300~500 条）

| 步骤 | 召回率 | 增量 | 说明 |
|---|---|---|---|
| 基线：RecursiveCharacterTextSplitter(1000/200) + ada-002 + 单路向量 + k=5 | 61.2% | — | 典型"Demo 翻车配置" |
| → 语义分块 | 68.4% | +7.2pp | 数据侧 |
| → 上下文检索（chunk 前置 50 字上下文摘要） | 73.8% | +5.4pp | 对应 01 §2.2 |
| → BGE-M3 + 领域微调 | 78.5% | +4.7pp | 见 §3 |
| → 混合检索（RRF k=60） | 82.1% | +3.6pp（+8.3pp 相对上一格的口径差见原文） | 见 §4.2 |
| → Reranker（bge-reranker-v2-m3） | 86.4% | +4.3pp | 见 §4.3 |
| → 查询重写 | 89.0% | +2.6pp（模糊查询 45%→68%，+23pp） | 见 §5 |
| → HyDE | 91.2% | +2.2pp | 见 §5 |
| → 自适应 top-K + 评估闭环 | **94.1%** | +2.9pp | NDCG 0.89 / Faithfulness 0.84 |

### 0.2 ROI 排序（多篇社区复盘 + 英文实验交叉）

**Reranker > 混合检索 > 语义分块 > 查询重写 > 领域微调**；先做前三件，2~3 天可提 10~15pp（社区实践数据）。英文侧印证：BEIR 上 BM25 41.9 → +CrossEncoder 44.7（rerank +2.8）；混合检索在 Qdrant 实验中 4/5 数据集击败单路更强检索器。来源：https://arxiv.org/html/2104.08663v2 ✅、https://qdrant.tech/articles/hybrid-search/ ✅

### 0.3 六条社区共识（多来源独立交叉）

1. 上限由"数据治理 + 检索"决定，不是模型（【RG1.1】）；
2. 分块是第一个也是最隐蔽的坑（"约七成答不准的问题源于分块不当"——社区实践数据）；
3. 混合检索 + rerank 是性价比最高的两个优化，单点各提 4~21pp；
4. "召回越多越好"是幻觉：top-k 调大后 Lost in the Middle 反降（【RG1.4】），收敛到 rerank 后 top 3~6；
5. 必须有量化评估闭环（RAGAS/DeepEval + 回归集），否则调优是开盲盒（【RG1.2】）；
6. 换 Embedding 模型会让相似度阈值全部失效——模型名/阈值/top-k 必须做成配置项并回归测试。

---

## 1. 数据侧：切块与入库 【RG6.x】

### 1.1 基线参数（起点，不是终点）

| 参数 | 英文语料 | 中文语料 | 来源 |
|---|---|---|---|
| chunk 大小 | 512 token 起步（Wang et al. 实测 faithfulness 97.59 最优；LlamaIndex Uber 10-K 实测 1024 双峰） | **300~500 汉字**（512 token ≈ 450+ 汉字）；问答库 ~400 字最稳，纯条款类可 200 字/块 | https://ar5iv.labs.arxiv.org/html/2407.01219 ✅、https://www.llamaindex.ai/blog/evaluating-the-ideal-chunk-size-for-a-rag-system-using-llamaindex-6207e5d3fec5 ✅、https://blog.csdn.net/2601_96315689/article/details/162117759 ✅ |
| overlap | chunk 的 10%~20%（Weaviate）；20 token（Wang et al.） | 50~100 汉字（索引体积约 +20%，"断头召回"基本消除——社区实践数据） | https://weaviate.io/blog/chunking-strategies-for-rag ✅ |
| 切分方法 | Recursive（分隔符 `\n\n`→`\n`→` `→``）为默认可靠选择 | **结构优先**：标题/空行/列表项天然边界，超长段落再字数兜底 | https://weaviate.io/blog/chunking-strategies-for-rag ✅ |
| 决策判据 | "一块文本单独读起来语义自洽，就适合做 chunk"；上限对齐 embedding 窗口 | 同左 | https://www.pinecone.io/learn/chunking-strategies/ ✅ |

【RG6.1】**没有"最优 chunk size"，只有"在自己的语料上重跑 eval"**——LlamaIndex 1024 结论强依赖 Uber 10-K 数据集，不可移植；新接知识库先抽 20 条真实问题做召回测试再调参（社区实践）。

【RG6.2】**英文 token 分块器默认参数对中文标点不敏感**：FAQ 在顿号处被切开（"售前、售后、退换"→"售前、售"+"后、退换"）——中文必须用中文友好分隔符或结构切分。来源：https://blog.csdn.net/2601_96315689/article/details/162117759 ✅

### 1.2 原子语义块与表格（复杂文档）

来源：https://blog.csdn.net/u013970991/article/details/161995517 ✅、https://docs.unstructured.io/open-source/core-functionality/chunking ✅

【RG6.3】固定长度切块对结构化内容是"毁灭性破坏"（表头与数据行分离、代码断裂、公式与上下文割裂 → 检索命中但证据不完整 → 幻觉）。**八类不可切割的原子语义块**：Markdown 表格、代码块、图片/流程图、层级化 YAML、带逻辑约束的 SQL、合同法条、专业公式、系统配置项。unstructured 的铁律一致：**Table 永远独立成块**。

【RG6.4】**表格三级策略**：小型 → 标题+表头+全数据行整体存；中型 → 双轨存储（原文用于生成，LLM 生成的自然语言摘要用于检索）；超长 → 按行分组拆分、每子表强制重复表头；**复杂表格（筛选/计算/多表关联）脱离向量检索，结构化入库为 SQL/DataFrame 走工具调用**。代码块同理："原文用于输出、摘要用于检索"。

【RG6.5】多表示检索：同一块生成摘要向量 + 关键词向量 + 模拟问题向量，任一命中即按块 ID 回填原始内容（HyPE 思路："预计算每个 chunk 的假想问题"，把"问题-文档"匹配变成"问题-问题"匹配，零查询时 LLM 成本）。来源：https://blog.csdn.net/u013970991/article/details/161995517 ✅、https://github.com/NirDiamant/RAG_Techniques ✅

### 1.3 PDF 解析（切块质量的上游）

- 三大失真源：**多栏阅读序、表格结构、公式**。生产管线通常 "marker/unstructured/RAGFlow DeepDoc 解析 → 按元素/标题切块 → 表格独立块保留 Markdown/HTML 结构 → 必要时 VLM 补描述"。
- 实测参照（olmocr-bench，1,403 份 PDF）：Marker GPU 76.0 分、数字原生件 83.5，对比 MinerU 72.7、docling 50.3；吞吐 fast 无 OCR 23.7 页/秒（B200）。来源：https://github.com/VikParuchuri/marker ✅
- 多模态方案（图表/公式密集型文档）：MinerU 提取 → 公式转 LaTeX、图表转语义描述再 embedding；纯文本页跳过 VLM 控成本（自评检索准确率 8/10 vs LangChain+PyPDF 2/10——社区实践数据）。来源：https://juejin.cn/post/7631595204001480731 ✅
- 中文投研实践：pymupdf4llm 解析为 Markdown、按 `##` 标题分块；表格 chunk 用 3000 token、超 3500 按逻辑分组拆行，文本 chunk 用 500（社区实践数据）。来源：https://blog.csdn.net/qq_43407325/article/details/163386366 ✅

### 1.4 元数据（入库时打 Tag）

【RG6.6】每个 chunk 入库时必须携带 Tag 元数据：**source（来源/权威度）/ doc_date / version / business_domain / effective_date / sensitivity**。用途：① 混合检索后的业务加权（新版压旧版、官方压社区帖，见 01 §4.4）；② pre-filter 缩小检索域；③ 答案引用回链。来源：https://blog.csdn.net/qq_62915969/article/details/161899536 ✅

【RG6.7】**不要过度清洗文本**——保留标题/表格/列表结构信息，它们是切块边界与元数据的来源。来源：https://blog.csdn.net/weixin_43107715/article/details/158152876 ✅

---

## 2. Embedding 模型选型 【RG7.x】

### 2.1 选型决策清单（六问）

来源：模型卡与论文均实抓，见《04-参考资料.md》C 组。

1. **语言匹配**：中英混合/多语 → Qwen3-Embedding、BGE-M3、jina-v3；纯中文 → Conan、piccolo、BGE-zh 系。
2. **维度与成本（MRL 可变维度）**：支持 Matryoshka 截断的模型可按预算裁维度——Qwen3 32~4096 任意、OpenAI `dimensions` 参数（3-large 缩到 256 维仍胜未裁剪 ada-002）、jina-v3 截到 32 维。
3. **上下文长度**：长文档少切块选 BGE-M3（8192）/ Qwen3（32K）/ Voyage（32K）；512 上下文模型（Conan/piccolo/BGE-zh）必须配合切块策略。
4. **部署方式与许可**：本地 Apache 2.0 首选 Qwen3-Embedding；MIT 有 BGE-M3、E5 系；jina-v3 是 CC-BY-NC（商用走 API）。API 牌价：OpenAI 3-small ~$0.02/1M tokens、3-large ~$0.10~0.13/1M；Voyage $0.02~0.18/1M。
5. **指令前缀（instruction-aware）**：见 §2.3。
6. **配套 reranker**：同家族优先（Qwen3-Reranker / bge-reranker）。

### 2.2 主流模型速查表（2026-09，同口径横向比较必须核对榜单实时值）

| 模型 | 参数 | 维度 | 上下文 | 关键分数 | 许可 |
|---|---|---|---|---|---|
| Qwen3-Embedding-8B | 7.57B | 4096（MRL 32~4096） | 32K | MTEB-Multi 70.58（2025-06 #1）/ C-MTEB 73.84 | Apache 2.0 |
| Qwen3-Embedding-4B | 4B | 2560 | 32K | MTEB-Multi 69.45 / C-MTEB 72.27 | Apache 2.0 |
| Qwen3-Embedding-0.6B | 0.6B | 1024 | 32K | MTEB-Multi 64.33 / C-MTEB 66.33 | Apache 2.0 |
| BGE-M3 | ~568M | 1024 | 8192 | MIRACL：Dense 69.2 / Sparse 53.9 / ColBERT 70.5 / 融合 71.5 | MIT |
| Conan-embedding | 326M | 1792（MRL 档位） | 512 | C-MTEB 72.62（论文自测） | 学术/检查许可 |
| piccolo-large-zh-v2 | ~1.2B | 1792（MRL 256~1792） | 512 | C-MTEB 70.95 | 检查许可 |
| text-embedding-3-large | 闭源 | 3072（可缩） | 8192 | MTEB-Eng 64.6% | API |
| voyage-3/4 系 | 闭源 | 512~2048 可选 | 32K | —（Anthropic 测试中与 Gemini 表现最佳） | API |

来源：https://huggingface.co/Qwen/Qwen3-Embedding-8B ✅、https://huggingface.co/BAAI/bge-m3 ✅、https://arxiv.org/html/2408.15710v2 ✅、https://huggingface.co/sensenova/piccolo-large-zh-v2 ✅、https://developers.openai.com/api/docs/guides/embeddings ✅、https://www.anthropic.com/engineering/contextual-retrieval ✅

【RG7.1】**C-MTEB/MTEB 论文值、榜单实时值、模型卡自测值口径互有差异，横向比较必须同口径；最终以自建评测集为准**（LlamaIndex / Qdrant / Wang et al. 三方一致结论："embedding 排名因数据集而异"）。MTEB 榜单为动态页面，决策前用浏览器查看 https://huggingface.co/spaces/mteb/leaderboard ⚠️。

【RG7.2】中文中小场景不要盲目上最大模型：社区实践口径 bge-m3 或 bge-large-zh-v1.5 + bge-reranker-v2-m3 即可覆盖大多数场景（BGE-M3 + reranker 约 4GB FP16 显存）；0.6B 级适合高并发在线服务。维度越高精度越高但存储/延迟同步上涨（见 03 §1）。来源：https://juejin.cn/post/7438079312542777398 ✅、https://blog.csdn.net/qq_43407325/article/details/163386366 ✅

### 2.3 指令前缀（最容易被忽略的免费提升）

- **Qwen3-Embedding**：query 侧 `"Instruct: {task}\nQuery: {text}"`，官方称 +1~5%，指令建议用英文写；**文档侧不加**。✅ HF 卡
- **BGE v1.5 系**：query 侧 `为这个句子生成表示以用于检索相关文章：`（中文）/ `Represent this sentence for searching relevant passages: `（英文）；**文档侧永远不加**。✅ FlagEmbedding README
- **BGE-M3**：官方明确**不再需要**指令前缀。✅ HF 卡 FAQ
- jina-v3 按 5 个 LoRA 任务适配器切换模式（retrieval.query / passage 等）✅。

【RG7.3】**指令加错侧（加到文档侧）是常见事故**；是否加指令以"你自己的任务上实测更优"为准（FlagEmbedding FAQ 原话）。

### 2.4 换模型的坑（社区高频事故）

来源：https://blog.csdn.net/weixin_29034963/article/details/164578816 ✅、https://huggingface.co/BAAI/bge-large-en-v1.5 ✅

【RG7.4】**新旧 embedding 模型的向量空间分布不同，相似度分数区间大变——换模型后所有相似度阈值全部失效**。对策：模型名 / 阈值 / top-k 全部进配置中心；换模型预留至少一天回归测试。注意 BGE FAQ 的提醒：对比学习温度 0.01 使相似度集中在 [0.6, 1]，"**>0.5 不代表相似**"——阈值必须按业务标注重校，不要拍脑袋。

---

## 3. Embedding 优化与微调 【RG8.x】

### 3.1 什么时候微调

判断顺序（经验序，量级由 §3.3 表支撑）：**先 rerank（+2.8~21pp）→ 混合检索（+3~8pp）→ 微调（+2~10pp，领域缩写/术语混淆场景收益最大）→ 换更大模型（+1~3pp）**。微调的典型信号：领域缩写检索错（PB/ROE 混淆案例：缩写检索准确率 41%→82%——社区实践数据）；通用模型对行话/产品名的召回盲区。来源：https://blog.csdn.net/qq_43407325/article/details/163386366 ✅

### 3.2 FlagEmbedding 官方微调配方（step-by-step）

来源：https://github.com/FlagOpen/FlagEmbedding/blob/master/examples/finetune/embedder/README.md ✅

1. **造数据**：JSONL 每行 `{"query", "pos": [...], "neg": [...]}`；
2. **难负例挖掘**（`hn_mine.py`）：每 query 取 top-k，从命中集（排除正例）采样，`--negative_number 15 --range_for_sampling 2-200`（从第 2~200 名中采"较难"负例；想更易 → 放宽为 60-300）；
3. **（可选）知识蒸馏**：用 bge-reranker-v2-m3 打教师分做 KD；
4. **训练超参**：LR **1e-5**（encoder）/ 1e-4（decoder+LoRA）；temperature **0.02**；`train_group_size 8`（1 正 + 7 负）；epochs 2（encoder）；`--negatives_cross_device` 跨卡共享负例；`--normalize_embeddings True`；
5. **评估**：微调前后在同一评测集对比（hit rate / nDCG@10）。

### 3.3 数据合成与效果速查

| 干预 | 前 → 后 | 来源 |
|---|---|---|
| LLM 合成 QA 微调 bge-small-en（LlamaIndex 范式） | 验证集 hit-rate 78% → 84%（+6pp） | ✅ https://www.llamaindex.ai/blog/fine-tuning-embeddings-for-rag-with-synthetic-data-e534409a3971 |
| E5-mistral：50 万合成对 + 15 万指令（93 语） | MTEB 63.1（纯合成）→ 66.6（+13 标注集） | ✅ https://arxiv.org/html/2401.00368v3 |
| Conan 动态难负例挖掘 DHNM + CBB Loss | C-MTEB 62.9 → 72.62 | ✅ https://arxiv.org/html/2408.15710v2 |
| BGE-M3 多阶段（RetroMAE→无监督→微调） | MIRACL 60.5 → 69.2 | ✅ https://arxiv.org/html/2402.03216v5 |
| 难负例自知识蒸馏消融（Sparse 分支） | 无 SKD 时 53.9 → **36.7**（崩） | ✅ 同上 |

### 3.4 微调常见坑

【RG8.1】**难负例不是越难越好**：只从 top1-10 采会让模型把"相关但非最优"判为负例——用 `range_for_sampling 2-200` 并保留随机负例；负例质量 > 数量（BGE-M3 消融：无难例/蒸馏时 Sparse 从 53.9 崩到 36.7）。

【RG8.2】**纯领域标注数据易过拟合**：E5 论文显示大规模多样合成数据反而提升 zero-shot；"微调后泛化变差"多源于数据单一。温度 0.01~0.02（三处来源交叉）；归一化后再算相似度；fp16 精度会导致 CPU/GPU 结果不一致（FAQ 给出 `fp16=False` 方案）。

【RG8.3】**embedding 与 reranker 要分配角色**（embedding 召回宽、reranker 精排严）：社区反馈难负例把 embedding 微调得"过严"后叠加 reranker 有时反而变差（⚠️https://github.com/FlagOpen/FlagEmbedding/issues/296 ），需在业务集上验证。

---

## 4. 相似性检索优化 【RG9.x】

### 4.1 相似度度量：先归一化，再谈度量

来源：https://sbert.net/docs/sentence_transformer/usage/semantic_textual_similarity.html ✅、https://github.com/FlagOpen/FlagEmbedding ✅

- **L2 归一化后 cosine ≡ dot ≡ euclidean（单调等价）**，排序结果完全一致；
- 未归一化时 dot 偏好长向量，语义错误。
- 【RG9.1】统一 `normalize_embeddings=True` 后用 dot（快）；向量库建 collection 时显式声明 Cosine/IP 并与写入侧归一化保持一致——**两侧不一致是静默检索退化的常见根因**。

### 4.2 混合检索（dense + sparse）

**为什么必须混**（BGE-M3 MIRACL 实测）：BM25 31.9 / mE5-large 66.6 / M3 Dense 69.2 / M3 Sparse 53.9 / ColBERT 70.5 / **Dense+Sparse 70.4 / All 融合 71.5**。纯向量对**人名、缩写、ID、错误码**无能为力（"合同编号 A-1007"检索案例：Dense NDCG 0.17 vs BM25 0.91 vs **RRF 混合 0.93**）；关键词搞不定同义词。来源：https://arxiv.org/html/2402.03216v5 ✅、https://eugeneyan.com/writing/llm-patterns/ ✅、https://blog.csdn.net/qq_43407325/article/details/163386366 ✅

**RRF 公式**（Elasticsearch 官方，https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html ✅）：

```text
score(d) = Σ_q  1 / (k + rank_q(d))     # rank 从 1 开始；k=rank_constant 默认 60
```

- 【RG9.2】**优先 RRF 而非加权分数求和**：BM25 分与余弦相似度量纲不同，归一化易翻车；RRF 只看排名、免调参。每路 prefetch top-20、k=60 是大多数语料的免调起点（社区实践口径与 ES 默认一致）。Qdrant 实证：默认 RRF 在 5 个数据集中的 4 个上击败单路更强检索器。✅
- 【RG9.3】加权变体在需要"语义主导"时使用：BGE-M3 论文 MIRACL 用 w=(1, 0.3, 1)（dense/lex/colbert）、长文档 MLDR 用 (0.15, 0.5, 0.3)（稀疏主导）；社区加权融合口径 0.6 向量 / 0.4 关键词。**权重必须用 nDCG@10 在自建集上扫出来，不抄别人**。
- BM25 中文注意：需要分词（jieba 等），默认分词器对中文无效。

### 4.3 重排 Rerank（ROI 最高的单点）

两阶段架构：**召回 top 20~200 → cross-encoder 精排 → top 3~10 送 LLM**。

| 证据 | 数字 | 来源 |
|---|---|---|
| BEIR 17 数据集 | BM25 41.9 → BM25+CrossEncoder **44.7**（+2.8）nDCG@10 | https://arxiv.org/html/2104.08663v2 ✅ |
| Anthropic 全链路 | 加 rerank 把失败率 -49% 推到 **-67%**（top-150→top-20） | https://www.anthropic.com/engineering/contextual-retrieval ✅ |
| LlamaIndex 横评 | 几乎所有 embedding 加 reranker 都显著变强（最强组合 Jina-base + bge-reranker-large Hit Rate 0.938/MRR 0.869）；**但"基础检索太差时再好的 reranker 也救不回来"** | https://www.llamaindex.ai/blog/boosting-rag-picking-the-best-embedding-reranker-models-42d079022e83 ✅ |
| 社区实践 | 向量 top-10 准确率 68% → 重排后 89%（+21pp），延迟 150ms→280ms；bge-reranker-v2-m3 GPU 50~100ms；粗排+精排两级计算量 -80% | https://blog.csdn.net/weixin_39042870/article/details/162609188 ✅、https://juejin.cn/post/7340920192996360202 ✅（有道 QAnything：粗召回 top-25 → 精排 top 3~5） |

**Qdrant"reranker 值不值得上"决策程序**（https://qdrant.tech/documentation/search-tuning/when-a-reranker-is-worth-it ✅）：

【RG9.4】上 reranker 前先测 **oracle gap**（完美排序 vs 当前输出的 nDCG@10 差距）——没 gap 就别加；**先调好 RRF 融合再上 reranker**，否则只是追平调好的融合；从 10 个候选测起（10 个输就不会在 200 个赢；DBPedia-entity 在 50 候选时已拿到 96% 收益）；选型看窗口长度/领域匹配度不是模型大小。

【RG9.5】rerank 延迟预算：CPU 上 bge-reranker-base 16~45 docs/s、MiniLM 64~212 docs/s；100 候选约 0.5~5s/查询——生产候选量收敛到 20~50，或上 GPU。Wang et al. 横向对比：monoT5-base 是精度/延迟平衡点（MRR 31.78，4.5s），RankLLaMA 精度最高（32.35）但 ~82s/查询，TILDEv2 最快（~0.02s）。来源：https://ar5iv.labs.arxiv.org/html/2407.01219 ✅

### 4.4 ANN 索引参数调优

**HNSW**（来源：https://qdrant.tech/documentation/concepts/indexing/ ✅、https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index ✅）：

| 参数 | 阶段 | 调大影响 | 推荐默认 |
|---|---|---|---|
| `m`（每节点边数） | 建索引 | 召回↑、内存↑（faiss：每向量内存 ≈ d×4 + **M×2×4 字节**；范围 4~64） | Qdrant 16 / faiss 常用 32 |
| `ef_construct`（建图候选队列） | 建索引 | 召回↑、建索引耗时↑ | Qdrant 100；中文实践 200~400 |
| `ef` / `hnsw_ef`（查询候选队列） | 查询 | 召回↑、延迟↑（主速度-精度旋钮） | 按延迟预算 100→500 扫 |
| `full_scan_threshold` | 查询 | 小结果集放弃 HNSW 走精确全扫 | Qdrant 10,000（~10MB） |

中文实践口径（社区实践数据，https://blog.csdn.net/weixin_43107715/article/details/158152876 ✅）：K 阶段数据 <10 万用 Flat（100% 召回）、10 万~1000 万 IVF（nlist=4×√N，nprobe 10-20）、>1000 万 HNSW（M=16-32，ef_search 100-200）。

**IVF**（faiss wiki ✅）：`nlist` 取 **4×√N ~ 16×√N**（N<1M）；训练样本 30~256×K 条；1M~10M 用 `IVF65536_HNSW32`；`nprobe` 越大召回↑延迟↑，nprobe=nlist 退化为精确搜索；IVF 需在代表性样本上预训练。

### 4.5 量化降本链（存储/延迟紧张时按序启用）

来源：https://qdrant.tech/documentation/guides/quantization/ ✅、https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector ✅

```text
MRL 裁维度(Qwen3 32~4096 任意) → int8 标量量化(4x, 误差<1%, SIMD 提速)
→ 仍不够: binary(32x)/PQ(≤64x) + 必须 rescoring + oversampling
```

- 官方 recall 实测：binary 量化下 ada-002（1536d）**0.98 recall@100 @ 4x oversampling**；Cohere 4096d 0.98@50 @ 2x。
- 【RG9.6】**binary/PQ 必须配 rescoring（原向量精排 top 候选）+ oversampling（先取 2~4 倍候选）**，否则召回崩；Elastic bbq 同理（磁盘保留原向量 +25%，oversample 默认 3x）。PQ 距离计算非 SIMD 友好、比 scalar 慢——仅在内存极限场景用。

### 4.6 top-k 与上下文组装

【RG9.7】top-k 实测（社区实践数据）：K=3 召回 62%、K=5 78%、K=10 91%、K=20 96% 但噪声多——**配合 rerank 用"宽召回（20~50）+ 窄输出（3~6）"**；最相关片段放开头（【RG1.4】reverse repacking），提示词注明"优先参考排前片段"；超预算时按句压缩（拆句→与问题算相似度→保留高阈值句→按原文顺序拼回）。来源：https://blog.csdn.net/weixin_39042870/article/details/162609188 ✅、https://blog.csdn.net/jbgt2026/article/details/163565148 ✅、https://ar5iv.labs.arxiv.org/html/2407.01219 ✅（Recomp 摘要式压缩平均分最高且几乎不损分）

---

## 5. Query 侧优化 【RG10.x】

| 手段 | 做法 | 实测 | 触发条件 |
|---|---|---|---|
| **Rewrite 改写** | 小模型消解指代、补全模糊词（temperature=0） | 模糊查询 Recall 45%→68%（+23pp，社区实践数据） | 口语化/含指代/效果差时触发，不做一刀切 |
| **HyDE** | LLM 生成 1 篇假设性答案，embed 它去检索（对比编码器的瓶颈会滤掉幻觉细节） | TREC-COVID：Contriever 27.3 → **59.3**（≈微调版 59.6）；社区长尾语义查询 +9pp、代价 +300ms | 仅 >15 字、非精确匹配查询；**1 篇伪文档就够**（8 篇延迟翻倍收益甚微） |
| **Multi-Query** | 多角度变体并行检索取并集 | 比单查询召回更全（社区实践数据） | 复杂/多面问题 |
| **Step-Back** | 先抽象高层概念再检索/作答 | MMLU Physics +7%、TimeQA +27% | 推理型问题 |
| **Query 分类路由** | 先分类（点查/综述/闲聊/查数据）再走不同管线 | 平均分 0.428→0.443 且延迟 16.4s→11.58s（双赢） | **默认推荐** |

来源：https://arxiv.org/abs/2212.10496 ✅、https://arxiv.org/abs/2310.06117 ✅、https://ar5iv.labs.arxiv.org/html/2407.01219 ✅、https://blog.csdn.net/qq_43407325/article/details/163386366 ✅、https://blog.csdn.net/qq_62915969/article/details/161899536 ✅

【RG10.1】⚠️ **反直觉警告（Wang et al. 系统实验）**：query rewriting 与 query decomposition 在其基准上**反而伤害性能**（低于裸查询+好 embedding）——查询侧增强不是免费的，每项都要过评估。有效且稳的是：分类路由（双赢）、HyDE（强基线场景）、精确编号类直接走 BM25。

【RG10.2】多实体查询自适应 top-K：实体数 +5/实体、趋势 +8、因果 +10，上限 50（社区实践数据 +2.1pp）——与【RG9.7】"宽召回窄输出"配合。

---

## 6. 生成侧：上下文组装、幻觉抑制、兜底 【RG11.x】

### 6.1 幻觉抑制多层防线（按社区实测有效性排序）

来源：https://blog.csdn.net/jbgt2026/article/details/163565148 ✅（幻觉率约 30%→5% 以下，社区实践数据）

1. **temperature=0**（事实型问答基线）；
2. **答案强制引用标注**（"[文档3]"式，最有效——既约束模型又便于核对）；
3. Prompt 硬约束："只依据以下材料回答，材料没有就说不知道"——**约束句放开头和结尾各一次**；
4. 自检：生成后让模型对照上下文复核 claims；
5. 置信度拒绝：logprobs 平均对数概率的指数 <0.7 → 拒答转人工。

【RG11.1】三级兜底必须显式实现：① 库内无内容 → 直接说"未找到"（对应 FP1）；② 检索到但置信度不高 → "根据现有资料，可能……"（弱化措辞）；③ 明确超范围 → 引导渠道/换问法。阈值定法：50 个有答案 + 50 个库外问题跑分数分布，取两分布交点再手动调（社区实践）。

### 6.2 上下文打包

【RG11.2】上下文预算装填而非塞满：OpenAI Cookbook 即便 128K 模型也只装 ~3.6K token；中文 token 校准注意 tokenizer 差异（社区案例：Qwen tokenizer 下中文字符×1.2、英文单词×1.3 估算）；最相关放开头结尾（【RG1.4】）；给每个片段带元数据标签（文件名+chunk 号，Barnett FP2/FP4 缓解措施）。

---

## 7. 评估闭环 【RG12.x】

### 7.1 指标体系（RAGAS 全家桶 + Triad 诊断）

来源：https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/ ✅、https://www.trulens.org/getting_started/core_concepts/rag_triad/ ✅、https://deepeval.com/docs/metrics-faithfulness ✅、https://www.cnblogs.com/pass-ion/p/19113902 ✅

| 指标 | 测什么 | 需要金标? | 低分诊断 |
|---|---|---|---|
| Faithfulness | 答案 claims 被上下文支持的比例 | 否 | **生成问题**；高 relevancy+低 faithful = 模型在用参数记忆（"自信地错"） |
| Answer Relevancy | 答案切题度 | 否 | 调 prompt / 降 temperature |
| Context Precision | 相关 chunk 是否排在前列 | 是 | 减 top-K 或加 reranker |
| Context Recall | 参考答案要点被检索覆盖比例 | 是 | 调 chunk_size/overlap/混合检索 |
| Noise Sensitivity | 噪声上下文下的退化 | 是 | 上下文压缩/清洗 |

### 7.2 工程化评估流程（把评估当产品做）

来源：https://hamel.dev/blog/posts/evals/ ✅（两次抓取交叉验证）、https://www.smallyoung.cn/docs/031-RAG质量评估RAGAS与生产级监控实践 ✅（生产级 Java 实现）

【RG12.1】**三级评估**：L1 单测（pytest 式断言，阈值是产品决策，不要求 100% 过）→ L2 人评 + LLM-as-judge（前提先记录 traces；judge 用能负担的最强模型，**每次人工标 25~50 条对齐**，类别不平衡时看 precision/recall 而非 raw agreement）→ L3 A/B（产品成熟后再做）。

【RG12.2】**评估模型与生产模型分开**（同模型自评有 self-enhancement 偏差；实践口径：生产 deepseek-chat、评估 qwen-max）；LLM-judge 指标分数当相对值不当绝对值（Seven Failure Points 案例中自动评估比人工更悲观）。

【RG12.3】运营参数（生产实践口径）：黄金集 50~200 条起步；CI/CD 每次变更跑回归；生产定期抽样 1%~5% 持续评估；集成测试断言 faithfulness>0.7、relevancy>0.7；综合分=多维调和平均（对短板敏感）。

【RG12.4】**RAGAS 版本兼容是第一大坑**（ragas 0.4.3 + langchain-openai 锁 1.1.10，≥1.2.0 ImportError；contexts 必须是字符串列表，单字符串返回 NaN；RunConfig 限流防过载）。来源：https://blog.csdn.net/WMXJY/article/details/158770319 ✅

【RG12.5】评估集构建三板斧：公开基准横评模型（C-MTEB/BEIR/MIRACL）→ LLM 从 chunk 合成问题（保留 holdout 防合成集过拟合）→ 人工标注上线前校准。离线指标与实际体验可能不符（GraphRAG 实验教训），要结合业务场景解读。来源：https://juejin.cn/post/7395849559370858508 ✅

---

## 8. 性能、成本与数据更新 【RG13.x】

来源：https://blog.csdn.net/weixin_43107715/article/details/158152876 ✅、https://blog.csdn.net/jbgt2026/article/details/163565148 ✅（社区实践数据）

【RG13.1】**瓶颈定位**：全链路埋点看 P95/P99（非平均）；P99>2s 且推理占比<50% → 查检索/重排/预处理；P99>3s 且推理>70% → 推理瓶颈。

【RG13.2】缓存三层：embedding 结果缓存（文本 MD5 键，省 50~90% 重复计算）→ 检索语义缓存（精确 hash + 相似度阈值 0.9，30% 重复问题 <100ms）→ FAQ 答案缓存（800ms→20ms）。注意语义缓存的时效性失效策略要跟数据更新联动。

【RG13.3】分级模型路由：80% 简单场景走 7B 本地（省 ~90%），<10% 复杂才用高价 API，整体 -70%~90% 成本、效果损失 <5%；月 API 费 >5000 元考虑本地部署（社区实践口径）。推理侧 vLLM（PagedAttention+连续批处理）吞吐 10~20 倍。

【RG13.4】**数据更新**：90% 商用场景用增量更新；文件 MD5+内容哈希变更检测；增量版本化+软删除（is_latest 字段）秒级回滚；至少留 3 个历史版本、禁止物理删除；上线"禁无校验、禁无回滚预案"。向量库选型时**增量 upsert 能力是硬指标**（FAISS 静态索引的教训：1 万→50 万篇后检索 200ms→3s，加文档要重建索引）。来源：https://blog.csdn.net/weixin_39042870/article/details/162609188 ✅

---

## 9. 上线检查单（全链路 22 条）

**数据侧**
- [ ] 结构化切块（中文 300~500 字 + overlap 50~100 字起步），20 条真实问题召回测试先行
- [ ] 表格/代码/公式按原子语义块处理；复杂表格走 SQL/工具调用
- [ ] chunk 元数据（source/date/version/domain/authority）齐全；跨版本去重豁免
- [ ] PDF 解析用结构感知工具（marker/unstructured/DeepDoc），多栏/表格/公式专项验证

**模型侧**
- [ ] Embedding 选型过自建评测集（不是只看榜单）；指令前缀加对侧
- [ ] 模型名/阈值/top-k 进配置中心；换模型有回归测试预案
- [ ] reranker 上线前测过 oracle gap；候选量与延迟预算匹配

**检索侧**
- [ ] 混合检索（RRF k=60 起步）；BM25 中文分词正确
- [ ] 归一化与度量口径一致（normalize + dot/cosine 同侧声明）
- [ ] ANN 参数按数据量级选择（Flat/IVF/HNSW）；量化开启时 rescoring+oversampling 生效
- [ ] top-k"宽召回窄输出"（20~50 → 3~6），最相关放开头

**生成侧**
- [ ] 只依据材料 + 强制引用 + 约束句首尾各一次 + temperature=0
- [ ] 三级兜底（未找到 → 不确定措辞 → 转人工）与置信度阈值
- [ ] 矛盾处理机制（01 §4：版本化/权威度/显式对比）按业务域配置

**评估与运营**
- [ ] 黄金集 ≥50 条 + RAGAS/DeepEval 四指标基线；每次变更跑回归
- [ ] 评估模型 ≠ 生产模型；judge 用人工 25~50 条对齐
- [ ] 全链路 P95/P99 埋点；三层缓存；分级模型路由
- [ ] 增量更新 + 软删除 + ≥3 历史版本 + 回滚预案
- [ ] 未命中问题清单定期回流补库（反馈飞轮）
- [ ] FP1~FP7 失败分类的运行时监控面板
- [ ] 灰度 5%~10% 用户 1~2 周达标再扩量（80% 失败源于跳过需求评估——社区实践口径）
