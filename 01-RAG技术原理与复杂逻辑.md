# RAG 技术原理与复杂逻辑

> 版本 v0.1 · 2026-09-26 联网调研 · 适用范围：企业知识库/文档问答类 RAG 系统的原理理解、复杂逻辑设计（跨章节引用、矛盾文档处理）与边界判断。
>
> 依据来源：Anthropic / Microsoft Research / LlamaIndex / Weaviate / Pinecone / Qdrant 官方一手资料与 arXiv 论文为主，中文社区（掘金/CSDN/知乎/博客园）实践复盘为辅；全部链接的实抓核验状态见同目录《04-参考资料.md》，调研底稿在 `research/` 子目录（A 中文社区 / B 英文一手 / C Embedding / D 复杂 RAG / E 存储选型）。
>
> 条款编号前缀 **【RG#.#】**（RAG）。"规范"级条款 = 必须遵守；"建议"级 = 默认遵守、偏离需说明理由。选型与容量计算见《03-选型与容量规划.md》，落地清单见《02-最佳实践与踩坑.md》。

---

## 0. 术语与定位

### 0.1 RAG 解决什么问题

RAG（Retrieval-Augmented Generation，检索增强生成）= 先从外部知识库检索相关片段，再让 LLM 基于片段生成答案。它买四样东西，任何一样不成立就要重新考虑是否用 RAG：

1. **私有知识**：LLM 训练语料里没有的企业内部文档；
2. **时效性**：训练截止后的事实（政策更新、价格、版本）；
3. **可溯源**：答案能引用到具体文档段落（合规审计刚需）；
4. **成本**：检索 + 短上下文远比"整库塞 prompt"便宜（见 §5）。

> 分界线（Anthropic 实测建议）：知识库总量 ≤ ~20 万 token（约 500 页）时，直接整库放进 prompt 即可，不必建 RAG。—— https://www.anthropic.com/engineering/contextual-retrieval ✅

### 0.2 两级管线（本目录所有讨论的公共骨架）

```text
离线（数据侧）                          在线（查询侧）
parse 解析（PDF/表格/OCR）              query 理解（改写/分类/时间意图）
  → chunk 切块（策略见 02 §1）            → retrieve 检索（混合多路召回）
  → enrich 增强（上下文/摘要/元数据）  →   → rerank 重排（cross-encoder 精排）
  → embed 向量化                        → assemble 组装（排序/压缩/预算）
  → index 入库（向量 + 元数据 + 图）      → generate 生成（引用/拒答/兜底）
  → 冲突扫描 / 版本链接（§4）             → evaluate 评估（RAGAS/Triad，闭环）
```

【RG1.1】**RAG 效果的上限由数据治理与检索决定，生成端（换模型/改 prompt）只能逼近这个上限，不能突破它。** 中文社区多个独立复盘交叉印证："检索没做好之前调生成端是白费力气"（61%→94% 投研案例全程未换 LLM），与英文一手结论"检索上游决定下游"（DPR：检索精度↑ → 端到端 QA↑）一致。来源：https://blog.csdn.net/qq_43407325/article/details/163386366 ✅、https://eugeneyan.com/writing/llm-patterns/ ✅

【RG1.2】**任何检索组件上线前必须先建评估集**（50~200 条起步），否则调参是开盲盒；且"最优参数"强依赖数据集，别人的实验数字（包括本文所有数字）只能当先验，不能当结论。—— Barnett et al.："RAG validation is only feasible during operation"，健壮性是演化出来的而非设计出来的。来源：https://arxiv.org/abs/2401.05856 ✅、https://hamel.dev/blog/posts/evals/ ✅

### 0.3 与邻近方向的关系

| 概念 | 关注点 | 与 RAG 的关系 |
|---|---|---|
| Long-context LLM | 上下文"放得下" | 竞争又互补：RAG 解决"找得准、可溯源、可更新、成本可控"（§5） |
| Agent / Graph 编排 | 多步决策与路由 | 复杂 RAG 的多跳检索、意图路由本质是 agent 循环（见 `../loop-engineering/`、`../graph-engineering/`） |
| 知识图谱 | 实体关系结构化 | GraphRAG 的索引底座；独立 KG 工程不在本目录范围 |
| 决策模型（jev） | 高频封闭判断 | RAG 管线中的意图分类/路由环节可用判别模型承载（见 `../jev/`） |

---

## 1. RAG 的失败模式（先讲透"为什么会翻车"）

### 1.1 七个失败点（Barnett et al. 2024，三个真实案例归纳）

来源：https://arxiv.org/abs/2401.05856 ✅（注意正确 ID 是 2401.058**56**）

| # | 失败点 | 一句话 | 主要修复手段 |
|---|---|---|---|
| FP1 | Missing Content | 答案根本不在库里，系统却"硬答" | 库外问题识别 + 拒答兜底；语义缓存补 FAQ |
| FP2 | Missed Top Ranked | 答案在库里但没进 top-K | 混合检索、rerank、调 chunk 粒度 |
| FP3 | Not in Context | 检索到了，组装 prompt 时被丢掉 | 上下文预算装填、压缩、排序策略 |
| FP4 | Not Extracted | 答案在上下文里但 LLM 没提取对 | 更大窗口（8K vs 4K 有实测改善）、降噪、引用约束 |
| FP5 | Wrong Format | 忽略格式指令 | 显式格式指令 + few-shot |
| FP6 | Incorrect Specificity | 答得过笼统/过细 | 查询澄清、粒度对齐 |
| FP7 | Incomplete | 答一半（典型：一次问多个文档） | 查询分解、逐文档回答再合并 |

【RG1.3】**把 FP1–FP7 做成运行时监控的分类标签**，每个坏例归入一类再统计分布——失败模式分布决定优化优先级，而不是凭感觉调组件。

### 1.2 RAG Triad（TruLens：按架构三条边定位失败）

来源：https://www.trulens.org/getting_started/core_concepts/rag_triad/ ✅

1. **Context Relevance**：检回的 chunk 与问题相关吗？挂 → 检索问题；
2. **Groundedness**：答案逐 claim 有上下文支撑吗？挂 → 生成幻觉；
3. **Answer Relevance**：回答切题吗？挂 → 前两者对但没答到点上。

与 RAGAS 指标的映射（RAGAS：https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/ ✅）：Triad 的诊断三角形对应 RAGAS 的 faithfulness（groundedness）、answer relevancy、context precision/recall。**DeepEval 给出关键排障规则：低 faithfulness = 生成问题；高 relevancy + 低 faithfulness = 模型在用参数知识而非检索上下文（"自信地错"）**。来源：https://deepeval.com/docs/metrics-faithfulness ✅

### 1.3 两个被反复证实的"物理定律"

**（a）位置偏差 Lost in the Middle（Liu et al., TACL 2023）** https://arxiv.org/abs/2307.03172 ✅

- GPT-3.5-Turbo 多文档 QA，正确文档位置与准确率呈 **U 形**：20 文档时开头 **75.8%** → 中间 **53.8%** → 末尾 63.2%；30 文档时中间跌到 50.5%，**低于闭书基线 56.1%**（oracle 上限 88.3%）；
- 检索文档数超过 20 后收益仅 ~1.5%（GPT-3.5）/ ~1%（Claude-1.3）。

【RG1.4】**top-k 不是越大越好；组装上下文时把最相关内容放在开头和结尾**（reverse repacking），长 prompt 中间藏关键信息属高危操作。来源：同上 ✅、https://ar5iv.labs.arxiv.org/html/2407.01219 ✅

**（b）上下文退化 Context Rot（Chroma Research 2025，18 个模型）** https://www.trychroma.com/research/context-rot ✅

- 输入 token 越多性能越低，**即使是最简单的"找一句话"任务**；
- 1 个干扰文档就显著伤害性能，4 个干扰叠加；
- 关键在 context engineering：**信息如何呈现比是否存在更重要**。

【RG1.5】**"塞得下"≠"用得上"**。即使 128K 窗口，也按"检索 + 裁剪 + 排序"组装（OpenAI 官方 Cookbook 即便在 128K 模型上也只装填 ~3.6K token 预算）。来源：https://developers.openai.com/cookbook/examples/question_answering_using_embeddings ✅

---

## 2. Chunk 的上下文丢失与三种修复路线（复杂逻辑核心一）

### 2.1 问题本质

切块把文档变成"孤立句子"：`"company revenue grew 3%"` 不知道是哪家公司哪个季度；维基百科"Berlin"条目后文用 "its / the city" 指代，与人口数字不在同一 chunk。**naive chunk-then-encode 是 i.i.d. 独立编码，跨块指代与背景必然断裂**——这是检索失败（FP2/FP3）的结构性根源，不是调参能解决的。

三条修复路线（可组合）：**改索引**（给 chunk 补上下文）、**改 embedding 流程**（让向量携带全文上下文）、**改检索结构**（检索小块、返回大块）。

### 2.2 路线 A：改索引 —— Anthropic Contextual Retrieval（2024-09）

来源：https://www.anthropic.com/engineering/contextual-retrieval ✅

**机制（三件套）**：
1. **Contextual Embeddings**：对每个 chunk，用 LLM（示例 Claude 3 Haiku）以"整文档 + 该 chunk"为输入，生成 **50–100 token 的位置/作用说明**，拼在 chunk 前再 embedding；
2. **Contextual BM25**：同样的上下文前置后建 BM25 倒排，双路检索后 rank fusion 去重合并；
3. **Reranking**：先取 top-150，Cohere rerank 打分，只留 top-20 给生成器。

**实测（top-20 检索失败率，基线 5.7%）**：

| 方案 | 失败率 | 相对降幅 |
|---|---|---|
| 仅 Contextual Embeddings | 5.7% → 3.7% | **-35%** |
| + Contextual BM25 | 5.7% → 2.9% | **-49%** |
| + Reranking | 5.7% → **1.9%** | **-67%** |

**成本控制（关键工程点）**：用 prompt caching 缓存文档前缀，逐 chunk 生成上下文仅 **$1.02 / 百万文档 token**（假设 800-token chunk、8K 文档）；caching 同时降 >2x 延迟、最高省 90% 成本。中文社区低成本变体：用小模型给 chunk 前置 50 字中文摘要，单点 +5pp（61%→94% 案例中"上下文检索"一步 68.4%→73.8%）。来源：https://blog.csdn.net/qq_43407325/article/details/163386366 ✅

【RG2.1】上下文增强的产物**必须与 chunk 原文区分**（说明前置，原文保留），生成端引用仍指向原文段落；contextualizer prompt 可注入领域术语表。

### 2.3 路线 B：改 embedding 流程 —— Late Chunking（Jina AI, 2024）

来源：论文 https://arxiv.org/abs/2409.04701 ✅、博客 https://jina.ai/news/late-chunking-in-long-context-embedding-models ✅

**机制**：把"先切块再 embed"倒过来——① 整篇文档全部 token 送入长上下文 embedding 模型（8K 窗口）得到 token 级向量；② 之后再按 chunk 边界（transformer 之后、池化之前）划块；③ 每块内 token 向量做 mean pooling。每个 chunk 向量因此携带全文上下文，**零额外 LLM 调用、无需训练**。

**效果**：柏林例子中 "The city is also one of the states of Germany…" 与查询 "Berlin" 的相似度 naive 0.7535 → late **0.8498**；BeIR 四数据集平均 nDCG@10 相对提升约 **+3.5%**（52.2 → 54.0），单点最大 NFCorpus 23.5 → 30.0；文档越长收益越大。

【RG2.2】late chunking 需要长上下文 embedding 模型且复现结果有正有负（社区有低于 naive 的报告），**作为候选优化跑 eval 验证，不作默认**；它与路线 A 的关系：论文实测两者都能纠正 naive 漏检（late 0.8516 vs contextual 0.8590 vs naive 0.6343），A 更强但更贵。

### 2.4 路线 C：改检索结构 —— Small-to-Big 家族（LlamaIndex 术语）

来源：https://developers.llamaindex.ai/python/examples/retrievers/auto_merging_retriever/ ✅、https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/ ✅

- **Hierarchical / Auto-merging**：`HierarchicalNodeParser` 建层级（默认 **2048 → 512 → 128** token），只有叶子（128）进向量索引；检索命中同一父块的叶子超过阈值时**自动"上卷"合并为父块**——检索用小（准）、生成用大（全）。
- **Sentence-window**：按单句检索，命中后替换回前后 N 句窗口（`SentenceWindowNodeParser(window_size=3)`）。
- **Chunk expansion**（Pinecone 术语）：检索到 chunk 后把邻居一起带回。来源：https://www.pinecone.io/learn/chunking-strategies/ ✅

**诚实的实测披露**：LlamaIndex 官方示例中 auto-merging 与 base retriever **基本打平**（Correctness 4.27 vs 4.21）。工程解读：small-to-big 的收益体现在**证据分散语料的答案合成完整性**，简单抽取型 QA 提升有限——不是银弹，但实现成本近乎为零（索引结构问题，不增加推理成本）。

### 2.5 三路线对比

| 路线 | 代表 | 成本 | 实测强度 | 适用 |
|---|---|---|---|---|
| 改索引（补上下文） | Contextual Retrieval | LLM 逐 chunk 调用（caching 后 ~$1/M doc tokens） | **最强**（-67% 组合） | 高价值语料、FP2 高发 |
| 改 embedding | Late Chunking | 零额外 LLM 调用 | 温和（nDCG 相对 +3.5%） | 长文档、指代密集 |
| 改检索结构 | Auto-merging / sentence-window | 近零 | 合成阶段收益，抽取型不明显 | 默认值得开 |

【RG2.3】三路线作用在管线的不同环节，**可叠加**；叠加顺序建议：结构（默认）→ 上下文增强（eval 证明需要时）→ 重排（见 02 §3）。

---

## 3. 跨章节/跨文档引用（复杂逻辑核心二）

### 3.1 为什么 naive RAG 做不了跨章节引用

1. **信息分散**：证据分布在不相关（embedding 距离远）的 chunk 里，一步召回不全。GraphRAG 论文的判例：问"整个数据集的 top-5 主题"，向量检索"There is nothing in the query to direct it to the correct information"。来源：https://arxiv.org/abs/2404.16130 ✅
2. **指代消解**：跨 chunk 的"它/上述条款/该公司"无法在单块内解析（§2 的柏林例子正是此类）。
3. **查询-答案词汇不匹配**：多跳问题第一跳的查询词与第二跳文档毫无交集（IRCoT 的出发点："what to retrieve depends on what has already been derived"）。来源：https://arxiv.org/abs/2212.10509 ✅

按投入从低到高，四层解法（**逐层升级，不要直接跳到最贵方案**）：

### 3.2 第一层：结构元数据（近乎免费，默认必做）

- 切块时保结构：unstructured `by_title` 保证"chunk 不跨章节"，`include_orig_elements` 保留页码；Markdown/PDF 按 heading 切。来源：https://docs.unstructured.io/open-source/core-functionality/chunking ✅
- 每个 chunk 携带 **标题路径 + 章节号 + 文档名 + 页码** 元数据：检索时可作 pre-filter（"只搜第 4 章"），生成时可引用（"根据《XX 政策》第 3.2 条"）。
- 中文社区同步实践：父子块回填（命中子块→回填父章节）、"标题拼在 chunk 前"自带上下文。来源：https://blog.csdn.net/u013970991/article/details/161995517 ✅

【RG3.1】**任何 RAG 库的 chunk 必须携带（来源文档、章节路径、位置）三元结构元数据**——这是跨章节引用、答案溯源、FP7 定位的基础设施，缺失视为规范违规。

### 3.3 第二层：层次摘要索引 —— RAPTOR（2024）

来源：https://arxiv.org/abs/2401.18059 ✅

**机制（step-by-step）**：① 文档切 ~100 token 短块并 embedding；② **UMAP 降维 + GMM 软聚类**（节点可属多簇，簇数由 BIC 选）；③ 每簇 LLM 生成摘要，摘要再 embedding，**递归向上建树**（平均压缩 72%）；④ 检索用 collapsed tree（全树压平做相似度检索，2000 token 预算最优）。

**效果（长文档 QA）**：QuALITY test 上 **RAPTOR+GPT-4 82.6%，超此前最好 62.3% 约 20 个绝对百分点**；消融显示检索命中节点中 **18.5%–57% 来自非叶子层**——"全文主题"类问题靠平铺 chunk 只能召回片段，RAPTOR 能召回整篇故事摘要。

**定位**：文档内/跨文档的**抽象层级**索引，不需要实体图谱，成本主要是摘要 LLM 调用。适合"这本书讲什么""这两章的结论是否一致"类问题。

### 3.4 第三层：实体图谱 —— GraphRAG / LightRAG / HippoRAG

**GraphRAG（Microsoft Research, 2024）** 论文 https://arxiv.org/abs/2404.16130 ✅、官方博客 https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/ ✅

- **索引管线**：LLM 按 600-token（overlap 100）切块 → 抽取实体/关系/claims（gleaning 自反思循环补漏，600-token 块比 2400-token 多抽近 2 倍实体）→ **Leiden 层次社区检测** → 自底向上生成多层级社区摘要。
- **两种查询**：**Global search**（社区摘要 map-reduce，答"全局综述"问题，带 `[Data: Reports (513, 241…)]` 引用）；**Local search**（从查询识别实体 → 图上扩散邻居 → 回溯原始文本块，带关系引用）。
- **效果**：LLM-as-judge 成对胜率，Comprehensiveness 维度 GraphRAG **72–83%** 胜向量 RAG（p<.001）；Diversity 62–82%。
- **成本（必须正视）**：Podcast 单查询上下文 C0 26,657 tokens vs 全文摘要 1,014,611 tokens（省 >97%），但全层级 C3 反而 746,100 tokens；索引用 gpt-4-turbo 建 281 分钟。
- **中文实践校准**：中文长文档 chunks 建议从默认 1200 调小到 **600/overlap 50**；`entity_types` 按业务定制；小语料 + 细节型问题上 GraphRAG 指标**反而落后** naive RAG（Local Query 检回大量"潜在相关"图数据拉低 precision；约 25% Global Query 回答无效）。来源：https://zhuanlan.zhihu.com/p/1974504156912952449 ✅、https://juejin.cn/post/7395849559370858508 ✅

**LightRAG（2024）** https://arxiv.org/html/2410.05779v3 ✅：同样是"实体关系图 + 双层检索（low-level 实体 / high-level 主题）"，关键优势是**增量更新无需重建全图**；检索成本 **<100 tokens + 1 次 API 调用** vs GraphRAG 的 610,000 tokens + 610 次调用（Legal 数据集）。Diversity 维度 59–77% 胜 GraphRAG，Overall 接近。

**HippoRAG（NeurIPS 2024）** https://arxiv.org/abs/2405.14831 ✅：LLM 抽 KG + **Personalized PageRank** 单步扩散做多跳关联；多跳 QA 较 IRCoT 最高 +20%，**成本低 10–30 倍、快 6–13 倍**。

【RG3.2】**图方案是"全局综述需求强度 × 索引预算"的决策，不是默认组件**：点查/细节型问题用向量 RAG 更划算；全局归纳/跨文档对比才值得付索引成本；预算紧、需增量更新时优先 LightRAG/HippoRAG 而非 GraphRAG。

### 3.5 第四层：查询侧多跳（迭代检索）

| 方案 | 机制 | 数字 | 来源 |
|---|---|---|---|
| **IRCoT** | 检索与 CoT **交替**：推理生成下一步查询，检索结果修正后续推理 | 检索指标最高 +21 点、QA 最高 +15 点 | https://arxiv.org/abs/2212.10509 ✅ |
| **FLARE** | **主动检索**：生成下一句的临时预测作 query，低置信 token 触发检索后重写 | 4 个长文本知识密集任务 superior/competitive | https://arxiv.org/abs/2305.06983 ✅ |
| **Self-RAG** | 训练模型发 **reflection tokens**（是否检索/证据是否支撑），按需检索 | 7B/13B 超 ChatGPT 与 Llama2-chat+retrieval | https://arxiv.org/abs/2310.11511 ✅ |
| **Query decomposition / multi-query** | 拆子问题/多角度并行检索再合并 | 中文社区实测比单查询召回更全；但见下方警告 | https://blog.csdn.net/qq_62915969/article/details/161899536 ✅ |

⚠️ **反直觉证据（Wang et al. 2024 系统实验）**：**query rewriting 与 query decomposition 在其基准上反而伤害性能**（低于裸查询 + 好的 embedding）；有效的是 query classification（先分类再路由：平均分 0.428→0.443 且延迟 16.4s→11.58s）与 HyDE（**1 篇伪文档就够**，8 篇延迟翻倍收益甚微）。来源：https://ar5iv.labs.arxiv.org/html/2407.01219 ✅

【RG3.3】查询侧增强**不要一刀切全开**：精确编号/错误码类查询直接走 BM25；改写/HyDE 只对效果差或口语化的长尾查询触发（中文实践：HyDE 仅对 >15 字、非精确匹配查询启用，+9pp、代价 +300ms）。来源：https://blog.csdn.net/qq_43407325/article/details/163386366 ✅

### 3.6 跨章节方案选型表

| 问题类型 | 例子 | 首选方案 | 备选 |
|---|---|---|---|
| 局部事实（单点查） | "第 3.2 条怎么说" | 向量 RAG + 结构元数据过滤 | — |
| 章节内上下文完整 | "安装步骤全流程" | small-to-big / 父子块回填 | chunk expansion |
| 跨章节综合 | "A、B 两章的口径差异" | RAPTOR / 层次摘要 | auto-merging |
| 全局综述 | "全部文档的五大主题" | GraphRAG global / LightRAG | 层次摘要粗答 |
| 多跳推理 | "A 的负责人的所在部门的主管" | HippoRAG / IRCoT | 查询分解（需 eval 验证） |
| 指代密集长文档 | "它的人口是多少" | late chunking / contextual retrieval | 标题路径前置 |

---

## 4. 矛盾文档处理（复杂逻辑核心三，重点）

### 4.1 三类矛盾

1. **时序矛盾**：同一主题不同时间点（2023 版政策 vs 2025 版政策；上季度数据 vs 本季度数据）；
2. **来源/视角矛盾**：不同来源对同一事实说法不一（两份报告数字不同；官方口径 vs 媒体口径；不同部门各执一词）；
3. **参数知识 vs 上下文矛盾**：LLM 训练记忆与检索到的新文档冲突——**最隐蔽的一类**，因为表面看是"检索正常、回答流畅"，实际答案来自模型记忆。

### 4.2 证据：为什么不能指望 LLM 自己搞定矛盾

**（a）模型会固执地回退到记忆**（Longpre et al., EMNLP 2021）https://arxiv.org/abs/2109.05052 ✅

- 向上下文注入替换实体后，开卷模型**回退到记忆答案的比例：NQ 最高 20%，NewsQA 高达 75%**；
- **模型越大越固执**：记忆占比从 <15%（T5-60M）升到 ≥50%（T5-11B），跨两个数量级无饱和；
- **数字型实体最危险：被替换时 ≥83% 概率仍答原值**。

**（b）识别冲突容易、定位冲突难，且有强烈"无冲突"偏置**（Wang et al., COLM 2024）https://arxiv.org/html/2310.00935v3 ✅

- 冲突**检测**：零样本 Precision 0.999 但 **Recall 仅 0.144**——模型默认假装没冲突；CoT 把 F1 提到 0.779，微调 0.953；
- 冲突**定位**（指出冲突的具体片段）：最好基线 F1 0.621——最薄弱环节；
- 生成**多版本答案**：零样本两答全对率仅 0.031，指令法 0.569，GPT-4 0.656——可用但远未解决。

**（c）时序矛盾不是边角案例**（SituatedQA, EMNLP 2021）https://arxiv.org/abs/2109.06157 ✅

- NQ-Open 中 **约 16.5% 的问题是情境（时间/地点）依赖的**；即使提供更新语料，回答"现在"时仍有约 15 个点准确率下降。

【RG4.1】**矛盾处理必须做成系统机制（元数据 + 检索策略 + 生成约束），不能依赖 prompt 里一句"请以最新文档为准"**——上述证据表明模型默认行为与期望相反。

### 4.3 时序矛盾：版本化知识库 + 时间感知检索

**参考实现：Zep/Graphiti 的双时间轴**（arXiv 2501.13956）https://arxiv.org/abs/2501.13956 ✅

- **每条事实边四个时间戳**：`t_valid / t_invalid`（事实真实成立区间，T 轴）+ `t'created / t'expired`（系统摄入/失效时间，T' 轴）；
- **矛盾失效机制**：新事实入库时，LLM 将其与语义相关的既有边对比；发现时间重叠的矛盾时，**把旧边的 t_invalid 设为新边的 t_valid——旧事实不删除、只标记失效区间**，图同时维护"当前状态 + 完整历史"；
- 三层子图：episodic（原始消息无损存储，可溯源）/ semantic（实体与事实）/ community（聚类摘要）；
- 效果：DMR 基准 94.8%；LongMemEval 准确率最高 **+18.5%**，延迟 **-90%**。

**时序矛盾工程清单（综合实操口径）**：

1. 【RG4.2】每个 chunk 强制携带 `source / doc_date / version / valid_from / valid_to / authority` 元数据；Zep 四时间戳是完备设计，普通系统**至少要有文档日期**；
2. 【RG4.3】查询侧解析时间意图（"现在/去年/合同生效时"）：有时间意图 → 按时间过滤或加权（TempRALM 路线：检索器同时考虑语义与时间相关性，"up to 74% improvement"，https://arxiv.org/abs/2401.13222 ✅）；无意图 → 默认 recency 优先，**且答案中注明所引文档日期**；
3. 【RG4.4】版本化入库：新版本 = 新事实 + 旧版本 valid_to 截止，**禁止物理覆盖删除**（保留审计与"按当时日期回溯"能力；中文实践同步口径：增量版本化 + 软删除 is_latest 字段、至少留 3 个历史版本，https://blog.csdn.net/weixin_43107715/article/details/158152876 ✅）；
4. 【RG4.5】去重逻辑必须豁免跨版本组：新旧版本 chunk 语义高度相似，会被去重/互相挤掉——需按（实体组 + 版本组）豁免，让多版本都能被召回。

### 4.4 来源/视角矛盾：权威度标记 + 显式对比 + 引用硬指标

1. 【RG4.6】元数据加 `authority`（来源权威度：官方 > 部门 > 媒体 > 社区帖）；检索融合时按权威度加权（中文实践样例：`final_score = α·rerank_score + β·tag_boost(source/version) + γ·freshness(effective_date)`，新版压旧版、官方压社区帖，https://blog.csdn.net/qq_62915969/article/details/161899536 ✅）；
2. 【RG4.7】**多版本/多来源都召回，由生成端显式并列对比**，prompt 结构模板：

```text
以下资料对「{问题}」给出不同说法：
[文档A] 来源:{X} 日期:{2024-01} 权威度:{高}
[文档B] 来源:{Y} 日期:{2025-06} 权威度:{中}
要求：
1. 先判断上述资料之间是否存在冲突（不要默认没有冲突）；
2. 逐条指出每份资料的说法，并标注出处；
3. 按"默认采纳日期最新且权威度最高"的策略给出结论，其余说法如实并列；
4. 只依据上述资料回答，资料不足以裁决时明确说明。
```

（第 1 条"先判断有无冲突"直接针对 4.2(b) 的 no-conflict 偏置 Recall 0.144；逐条引用针对"定位是最薄弱环节"的结论。）

3. 【RG4.8】"以最新为准/以官方为准"做成**可配置策略**（按业务域注入 system prompt），法规/政策类场景配合 valid_from/valid_to 硬过滤；裁决规则变更不需要改代码。
4. 【RG4.9】**把引用完整性当验收硬指标**：ALCE 基准显示"即使在 ELI5 上，最好的模型也有 50% 的时间缺乏完整引用支持"（https://arxiv.org/abs/2305.14627 ✅）；矛盾场景输出必须逐 claim 可归因（AIS 框架，https://arxiv.org/abs/2112.12870 ✅）。

### 4.5 冲突检测管线（入库时主动发现矛盾）

```
候选对预筛（便宜）:  同实体组（主语/宾语重叠） + embedding 高相似
    → NLI 判定（中）:  分句两两跑 entailment/contradiction（SummaC 路线,
                       六数据集 balanced accuracy 74.4% SOTA）
    → LLM 复核（贵）:  判定矛盾对 → 进入告警/对比队列/版本链接
```

来源：SummaC https://arxiv.org/abs/2111.09525 ✅（arXiv API 摘要）；Zep 用"同实体对内才做事实去重，显著缩小搜索空间"的预筛策略 ✅。

【RG4.10】冲突扫描放在**入库管线**（新 chunk vs 库内同实体既有 chunk），而不是查询时——查询时才发现矛盾已经晚了；扫描出的矛盾对落库为"冲突组"，供检索豁免与生成对比复用。

### 4.6 矛盾处理标准管线（本节小结）

```text
入库:  parse → chunk → 打元数据(source/date/version/validity/authority)
       → 冲突扫描(实体重叠预筛 → NLI/LLM 判定) → 版本链接(新事实使旧事实失效, 不删除)
检索:  时间/版本过滤 → 多路召回(含历史版本, 跨版本组豁免去重) → recency/authority 融合加权
生成:  显式并列冲突说法 → 逐条引用出处 → 策略化裁决(默认最新/最权威, 或交用户)
验收:  引用完整率(ALCE 式指标) + 冲突测试集(替换实体/时间情境用例)
```

---

## 5. 长上下文 vs RAG（边界判断）

### 5.1 三组关键数据

1. **Self-Route（Google DeepMind, EMNLP 2024）** https://arxiv.org/abs/2407.16833 ✅：资源给足时长上下文一致优于 RAG（+7.6% Gemini-1.5-Pro / +13.1% GPT-4o）；但"RAG 判 unanswerable 才升级长上下文"的两步路由，在性能持平前提下**省 39–65% 成本**；**18–43% 的查询必须走长上下文**，其余 RAG 可自答。
2. **Databricks（2000+ 实验、13 个模型）** https://www.databricks.com/blog/long-context-rag-performance-llms ✅：生成准确率随上下文变长**先升后降且各模型拐点不同**（Llama-3.1-405b 32k 后降、claude-3-sonnet 16k 后降、dbrx 8k 后崩 0.528→0.255）；长上下文特有失败：claude-3-sonnet **版权拒答 3.7%@16k → 49.5%@64k**。
3. **Chroma Context Rot** ✅（§1.3）：退化是普遍现象，连"找一句话"都随长度变差。

### 5.2 结论

【RG5.1】**长上下文解决"放得下"，RAG 解决"找得准、可溯源、可更新、成本可控"；生产系统是"RAG 检索 + 长上下文合成 + 自省路由"的组合，不是二选一。** 路由形态：Self-Route 式（RAG 先答，答不了升级 LC）、或按查询类型分类路由（Wang et al.：query classification 双赢——分数与延迟同时改善）。

---

## 6. 能力阶梯（本篇总图）

```text
L0 整库进 prompt          知识库 ≤ ~20 万 token（~500 页），不需要 RAG
L1 naive RAG              固定切块 + 单路向量 + top-k    —— Demo 级,生产必翻车
L2 生产基线               结构化切块 + 元数据 + 混合检索 + rerank + 评估闭环
L3 上下文增强             contextual retrieval / late chunking / small-to-big
L4 复杂逻辑               跨章节: RAPTOR / GraphRAG / 多跳迭代
                          矛盾:   版本化 + 时间感知 + 权威度加权 + 显式对比
L5 自适应系统             查询分类路由 + Self-Route(RAG/LC) + agentic retrieval
```

每一层的证据与参数见《02-最佳实践与踩坑.md》；L3/L4 的成本与容量核算见《03-选型与容量规划.md》。

---

*附：本篇所有 ✅ 标记的来源均在调研时实抓核验（详见 `research/` 底稿与《04-参考资料.md》）；中文社区数字（如 61%→94%）为作者自述实验结果，已核验文章内容一致但未独立复现，引用时注明"社区实践数据"。*
