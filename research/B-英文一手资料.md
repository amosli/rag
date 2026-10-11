# B 线调研笔记:英文一手工程资料 —— RAG 的陷阱(Failure Modes)与最佳实践

> 调研日期:2026-09-26
> 调研方法:以 WebSearch 定位 + WebFetch 逐条抓取原文验证;每条关键结论均附来源 URL。
> 验证标记:✅已验证 = 已抓取原文核实内容;⚠️未验证 = 仅搜索结果可见、未抓到正文。
> 本笔记面向 RAG 工程培训文档编写:目标是把"有数字、可操作"的一手证据沉淀下来。

---

## 0. 一页速览(Top Findings)

1. **RAG 失败模式有系统化的分类**:Barnett et al. (2024) 基于三个真实案例(科研、教育、生物医学)归纳出 7 个失败点(Missing Content、Missed Top Ranked、Not in Context、Not Extracted、Wrong Format、Incorrect Specificity、Incomplete),并强调 **RAG 验证只能在运行时(operation)进行,健壮性是"演化"出来的而非一开始设计出来的**。[arXiv:2401.05856](https://arxiv.org/abs/2401.05856) ✅
2. **位置偏差是硬约束**:GPT-3.5-Turbo 在 20 文档设置下,gold 文档放开头 75.8%、放中间 53.8%;30 文档时中间位置跌到 50.5%,甚至低于 closed-book 的 56.1%。超过 20 篇文档收益 <2%。→ **排序策略必须把最相关内容放在头尾,即"reverse repacking"**。[Lost in the Middle, arXiv:2307.03172](https://ar5iv.labs.arxiv.org/html/2307.03172) ✅
3. **性价比最高的检索增强三板斧(有实测数字)**:① Hybrid search(稠密+稀疏,Wang et al. 实测最优混合权重 α=0.3);② Reranker(Anthropic:加 reranker 后 top-20 检索失败率从 5.7% 降到 1.9%,累计 -67%);③ Contextual chunk enrichment(Anthropic Contextual Embeddings + BM25:-49%)。[Anthropic](https://www.anthropic.com/news/contextual-retrieval) ✅、[arXiv:2407.01219](https://ar5iv.labs.arxiv.org/html/2407.01219) ✅
4. **chunk size 的经验共识**:512 token 上下最稳(Wang et al. faithfulness 97.59;Weaviate 生产基线"512 token + 50~100 overlap";LlamaIndex 在 Uber 10-K 上实测 1024 为 faithfulness/relevancy 峰值)——**没有一个万能数字,必须用自己的数据跑评估**。
5. **评估是最大瓶颈**:Hamel Husain"你的 AI 产品需要 Evals"——失败产品几乎都缺评估系统;方法论 = 日志先行 + 二元标注 + LLM-as-judge 校准(每次 25~50 条对齐)+ 三级评估(单测/人评+机评/A/B)。[hamel.dev](https://hamel.dev/blog/posts/evals/) ✅
6. **"长上下文杀死 RAG"的争论有数据结论**:Databricks 2000+ 实验、13 个模型:准确率随上下文变长先升后降(Llama-3.1-405b 32k 后下降、claude-3-sonnet 16k 后下降),且出现版权拒答激增(3.7%→49.5%)等长上下文特有失败;结论是**长上下文与 RAG 互补而非替代**。[Databricks](https://www.databricks.com/blog/long-context-rag-performance-llms) ✅;Chroma Context Rot (2025) 用 18 个模型证明**输入越长性能越差,即使任务极简单**。[Chroma](https://www.trychroma.com/research/context-rot) ✅

---

## 1. 文档化的 RAG 失败模式(Failure Modes)

### 1.1 七个失败点 —— Barnett et al. 2024(CAIN'24 经验报告)

来源:[arXiv:2401.05856](https://arxiv.org/abs/2401.05856) ✅已验证(注意:正确 ID 是 **2401.05856**;2401.05859 是一篇编码理论论文,引用时勿混淆)

三个案例研究:① Cognitive Reviewer(科研 PDF 文献综述辅助);② AI Tutor(教育场景 PDF/视频/HTML,Deakin 大学试点);③ BioASQ(生物医学,4017 篇开放获取论文 + 1000 道专家问答,GPT-4 生成)。评估工具用 **OpenAI Evals** + 人工复核;作者发现自动评估比人工更悲观。

| # | 失败点 | 说明 |
|---|--------|------|
| FP1 | **Missing Content**(内容缺失) | 答案根本不在知识库里,但系统"硬答",不说不知道 |
| FP2 | **Missed Top Ranked**(该中的没进 Top-K) | 答案在语料里,但相关 chunk 排名不够高,没被检索进 top-K |
| FP3 | **Not in Context**(合并策略丢内容) | 检索到了,但在组装 prompt 的"整合(consolidation)"环节被丢掉 |
| FP4 | **Not Extracted**(没抽出来) | 答案就在上下文里,但 LLM 因为噪声/矛盾信息没提取对 |
| FP5 | **Wrong Format**(格式错) | 忽略格式指令(该出表格/列表/代码时给成散文) |
| FP6 | **Incorrect Specificity**(粒度不对) | 答了,但过于笼统或过于具体,不符合用户真实需要 |
| FP7 | **Incomplete**(答一半) | 答案不算错但漏掉了可用信息;典型场景:一次问多个文档效果差,分开问更好 |

论文给出的缓解经验:
- **更大上下文窗口改善 FP4**(8K vs 4K);
- 给检索结果**加元数据(文件名 + chunk 编号)**,帮助 FP2/FP4 和多轮对话;
- **Semantic caching**(用 FAQ 预填充)降成本/延迟,缓解 FP1 与并发限制;
- 小文本上**开源 embedding 不输闭源**(利于 FP2 的成本控制);
- 端到端训练(pipeline 整训)优于手工拼装(利于 FP2/FP4);
- 核心论断:**"RAG validation is only feasible during operation"** —— 鲁棒性是演化出来的(evolves rather than designed in),必须运行时持续监控与校准。

### 1.2 Lost in the Middle —— 位置偏差(Liu et al., TACL 2023)

来源:[arXiv:2307.03172](https://ar5iv.labs.arxiv.org/html/2307.03172) ✅已验证(ar5iv 全文,含表格数字)

- 多文档 QA 中,**性能与 gold 文档位置呈 U 形曲线**:开头和结尾好,中间差。
- GPT-3.5-Turbo 实测(多文档 QA 准确率):
  - 10 篇:开头 76.8% / 中间(第 5 篇)61.2% / 末尾 62.4%
  - 20 篇:开头 75.8% / **中间(第 10 篇)53.8%** / 末尾 63.2%
  - 30 篇(16K 版):开头 73.4% / **中间 50.5%** / 末尾 63.7%
  - 论文原话:GPT-3.5-Turbo 多文档 QA 性能可下降"more than 20%"
- **closed-book(不给文档)56.1%,oracle(只给 gold)88.3%**;20/30 篇时中间位置的成绩甚至低于不给任何文档。
- **检索文档数 >20 后收益只有 ~1.5%(GPT-3.5)/ ~1%(Claude-1.3)** —— 盲目加大 top-k 没有意义,该做的是排序与压缩。
- 工程含义:① top-k 不是越大越好;② 上下文组装时把最相关的放头尾(reverse repacking,Wang et al. 实测同样结论,见 §3.3);③ 长 prompt 中间藏关键信息 = 高危。

### 1.3 Context Rot —— 长上下文普遍退化(Chroma Research, 2025-07)

来源:[Chroma "Context Rot"](https://www.trychroma.com/research/context-rot) ✅已验证

- 18 个模型(GPT-4.1、Claude 4、Gemini 2.5、Qwen3 等),NIAH 变体 + LongMemEval + 重复词任务。
- 关键发现:
  - **输入 token 越多性能越低,即使是最简单的"找一句话"任务**;
  - **针-问题相似度越低,随长度退化越快**(余弦相似度 0.445~0.829 区间实验);
  - **1 个干扰项(distractor)就显著伤害性能,4 个叠加**;干扰项 2、3 最常出现在幻觉答案里;GPT 系幻觉最多,Claude 系倾向拒答/弃权;
  - 反直觉:**打乱句子顺序的 haystack 反而比逻辑连贯的 haystack 成绩更好(18 个模型一致)** —— 连贯文本让模型更被"噪声相似性"干扰;
  - LongMemEval:~113k token 全量输入 vs ~300 token 聚焦输入,所有模型在聚焦输入上显著更高;
  - 结论:关键在 **context engineering —— 信息如何呈现比是否存在更重要的**。

### 1.4 幻觉与"答非所问"的结构化理解:RAG Triad(TruLens)

来源:[TruLens RAG Triad](https://www.trulens.org/getting_started/core_concepts/rag_triad/) ✅、[TruLens GitHub README](https://github.com/truera/trulens) ✅

Triad 沿 RAG 架构的三条边定位幻觉/失败:
1. **Context Relevance**:检索的每个 chunk 与 query 是否相关 —— 挂了 = 检索问题;
2. **Groundedness**:答案是否逐 claim 有检索上下文支撑 —— 挂了 = LLM 在"夸大或扩展出一个听起来正确的答案"(hallucination);
3. **Answer Relevance**:最终回答是否切题 —— 挂了 = 检索和 grounding 都对但没回答问题。
三者全过 → "在知识库范围内 hallucination-free"。

TruLens 官方基准数字(README 披露):Groundedness 在 **LLM-AggreFact 上 F1 0.81**(超过 Bespoke-MiniCheck-7B);Context relevance **NDCG@5 0.93**(AIMultiple 2026-03 对比 RAGAS/DeepEval/UpTrain/W&B Weave 居首);Agent GPA 在 TRAIL/GAIA 上 **抓住 95% 的 agent 错误(267/281,基线 55%)**;对抗样本(近拷贝换一个事实)胜负比 **4.2:1**。

### 1.5 PDF / 表格 / 企业内容抽取的坑(一手建议汇总)

- **Azure AI Search(官方 RAG 指南)**:内容准备是 RAG 质量之本 —— 大文档要 chunking、PDF/图片需 **OCR、image analysis、image verbalization、document extraction skills**,术语不匹配用 **synonym maps + semantic ranking** 补。[Azure AI Search RAG overview](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) ✅
- **Pinecone**:结构感知切分用 NLTK/spaCy 分句;LangChain `RecursiveCharacterTextSplitter`(默认分隔符 `"\n\n"`、`"\n"`、`" "`、`""`);PDF/HTML/Markdown/LaTeX 各有专用 parser。[Pinecone chunking](https://www.pinecone.io/learn/chunking-strategies/) ✅
- **Weaviate**:Document-based chunking 按 Markdown 标题/HTML 标签/代码函数/PDF 表头表格切;Code chunking 按函数/类/模块并保语法。[Weaviate chunking](https://weaviate.io/blog/chunking-strategies-for-rag) ✅
- **Seven Failure Points** 的三个案例都从 PDF 起家,FP4(Not Extracted)与表格/噪声直接相关;FP7(一次问多文档答一半)是"抽取类"失败的典型表现。[arXiv:2401.05856](https://arxiv.org/abs/2401.05856) ✅
- (⚠️未验证补充)Qdrant 文章索引中出现 "SHIFTing Languages in Multilingual RAG"(多语言 embedding 偏移修复)等,可作为延伸线索。[Qdrant articles](https://qdrant.tech/articles/) ✅(索引已验证,具体文章未抓取)

---

## 2. Chunking(分块)最佳实践:带证据

### 2.1 尺寸与重叠:各来源的实测数字

| 来源 | 设置 | 结果/建议 |
|------|------|-----------|
| [Wang et al. 2024, arXiv:2407.01219](https://ar5iv.labs.arxiv.org/html/2407.01219) ✅ | 256/512/1024/2048 token,overlap 20 token | **512 最优**(faithfulness 97.59 / relevancy 97.41),256、1024 接近,**2048 明显最差**(80.37);句子级切分 + 20 token overlap 是推荐配置 |
| 同上(small-to-big / sliding window) | 小块 175 + 大块 512 | **sliding window 最高分**(97.41/96.85),small-to-big 优于朴素切分 |
| [LlamaIndex chunk-size 评估](https://www.llamaindex.ai/blog/evaluating-the-ideal-chunk-size-for-a-rag-system-using-llamaindex-6207e5d3fec5) ✅ | Uber 10-K,128~2048 五档,20 个问题,GPT-3.5 生成 + GPT-4 评 Faithfulness/Relevancy | **1024 处 faithfulness 与 relevancy 双峰**,"1024 是响应时间与质量的平衡点";128 太小会在 top-k 里丢关键信息 |
| [LlamaIndex Boosting RAG(2023-11)](https://www.llamaindex.ai/blog/boosting-rag-picking-the-best-embedding-reranker-models-42d079022e83) ✅ | Llama 2 论文 36 页,chunk 512,retrieval top-10 → rerank top-5 | 评估方法论标杆:每 chunk 用 LLM 生成 2 个问题避免模型偏好;指标 Hit Rate + MRR |
| [Weaviate](https://weaviate.io/blog/chunking-strategies-for-rag) ✅ | — | 生产基线:**512 token chunk + 50~100 token overlap 起步**,再用 hit rate / precision / recall 迭代;fixed-size 的 overlap 建议 **chunk 大小的 10~20%** |
| [Pinecone](https://www.pinecone.io/learn/chunking-strategies/) ✅ | — | 试探区间:小块 ~128–256 token(细粒度),大块 ~512–1024 token(多上下文);上限对齐 embedding 模型窗口;**不必死磕 overlap 参数,可用检索后"chunk expansion"(把邻居 chunk 一起带回)** |
| [Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) ✅ | chunk ~800 token(示例),生成 50–100 token 上下文 | "chunks typically a few hundred tokens";top-5/10/20 中 **top-20 表现最好**;知识库 <200K token(~500 页)可能根本不需要 RAG |

**共识与分歧**:512~1024 token 是常见起点;"最优值"强依赖数据与查询,**必须像 LlamaIndex 那样用自己的语料生成问题集来 A/B**(Pinecone 同样建议:多 namespace 跑代表性查询做 A/B)。

### 2.2 策略谱系(Weaviate 十类 + LlamaIndex 实现)

来源:[Weaviate chunking strategies](https://weaviate.io/blog/chunking-strategies-for-rag) ✅、[LlamaIndex node parsers](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/) ✅

1. **Fixed-size**(token/字符定长):简单,配 10–20% overlap;
2. **Recursive**(按分隔符优先级递归切,LlamaIndex 默认,示例 max 1000 字符):非结构化文本首选;
3. **Document-based**(按 Markdown 标题/HTML/代码结构):结构化文档;
4. **Semantic chunking**(逐句 embedding,相似度骤降处切开):LlamaIndex `SemanticSplitterNodeParser(buffer_size=1, breakpoint_percentile_threshold=95)`,阈值需调;源出 Greg Kamradt "5 levels of embedding chunking";
5. **LLM-based chunking**:LLM 定边界/加摘要,最强但最贵最慢;
6. **Agentic chunking**:agent 逐文档选策略,成本最高;
7. **Late chunking**(见 §2.3);
8. **Hierarchical chunking**:LlamaIndex `HierarchicalNodeParser(chunk_sizes=[2048, 512, 128])`;
9. **Adaptive chunking**:按内容密度动态调 size/overlap;
10. **Code chunking**:按函数/类/模块,保语法。

### 2.3 解决"chunk 失去上下文"的三种带证据方案

**(a) Contextual Retrieval(Anthropic, 2024-09)** ✅ [来源](https://www.anthropic.com/news/contextual-retrieval)

做法:预处理阶段用 Claude(示例用 3 Haiku)读**整篇文档**,为每个 chunk 生成 50–100 token 的"该 chunk 在文中的位置/作用说明",**拼在 chunk 前再做 embedding 与 BM25 索引**。

实测(top-20 检索失败率 = 1 − recall@20):
- Contextual Embeddings 单独:**失败率 -35%**(5.7% → 3.7%)
- + Contextual BM25:**-49%**(5.7% → 2.9%)
- 再加 reranking:**-67%**(5.7% → 1.9%)
- 成本:800-token chunk、8K 文档、prompt caching 下 **$1.02/百万文档 token(一次性)**;prompt caching 带来 **>2x 延迟降低、最高 90% 成本节省**;
- 其他结论:embedding+BM25 > 纯 embedding;测试中 **Voyage、Gemini 是最好的 embedding**;每种 embedding 组合下 contextualization 都有提升;建议自定义 contextualizer prompt(领域词典)并"always run evals"。

**(b) Sentence Window Retrieval(LlamaIndex)** ✅ [node parsers 文档](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/)、[AutoMerging 示例](https://developers.llamaindex.ai/python/examples/retrievers/auto_merging_retriever/)

`SentenceWindowNodeParser(window_size=3)` 把文档切成单句节点,每句的 metadata 存前后 3 句窗口;检索按**单句**匹配(精度高),用 `MetadataReplacementPostProcessor` 把命中句替换回窗口再喂 LLM(上下文足)。思想 = **"embed 小、喂大"**。

**(c) Late Chunking(Jina AI, 2024)** ✅ [来源](https://jina.ai/news/late-chunking-in-long-context-embedding-models)

做法:先用长上下文 embedding 模型(如 jina-embeddings-v2,8K)对**整篇文档**产出 token 级向量,再**对每个 chunk 的 token 段做 mean pooling** —— 每个 chunk 向量都携带全文上下文,天然解决"its / the city 指代 Berlin 但与人口数字不在同一 chunk"的问题。

BeIR 实测(jina-embeddings-v2-small-en,256-token chunks,nDCG@10):
- NFCorpus:naive 23.46% → late chunking **29.98%**(+6.5);
- SciFact:64.20% → **66.10%**;TRECCOVID:63.36% → 64.70%;FiQA2018:33.25% → 33.84%;
- 所有测试集全部提升;**文档越长收益越大**;定性例子:"the city"与"Berlin"相似度 0.753 → 0.850。
- 对应论文式描述:naive chunking 是 i.i.d. 独立编码,跨 chunk 指代必然断裂。

**(d) Auto-Merging Retriever(LlamaIndex)** ✅ [示例](https://developers.llamaindex.ai/python/examples/retrievers/auto_merging_retriever/)

层级 `HierarchicalNodeParser` 默认 **2048/512/128**;只索引叶子块(示例中 1029 个节点中 795 个叶子),检索命中同一父块的叶子超过阈值就**"合并回父块"**。例:top_k=6 → 4 个叶子合并成 1 个父块,最终返回 3 个节点。注意:官方示例在 60 题(Llama 2 论文)上的对比结果是 **auto-merging 与 base retriever 基本打平**(Correctness 4.267 vs 4.208,GPT-4 pairwise 偏好 0.525)—— 该技术在"证据密集"的语料上才明显占优,不要当银弹。

---

## 3. 检索提升技术(带实测增益)

### 3.1 Hybrid Search(稠密 + 稀疏)

- **Wang et al. 2024** ✅ [arXiv:2407.01219](https://ar5iv.labs.arxiv.org/html/2407.01219):TREC DL19/20 上无监督稀疏(BM25/Contriever)明显落后于监督稠密;**Hybrid + HyDE 最优**(DL19 mAP 52.13 / DL20 53.13;单方法最高为 LLM-Embedder+HyDE 50.87);**混合权重 α=0.3**(score = α·sparse + dense)最优。
- **Eugene Yan(生产经验)** ✅ [LLM Patterns](https://eugeneyan.com/writing/llm-patterns/):自己做 Obsidian-Copilot 的结论 —— **"hybrid retrieval(传统搜索索引 + embedding)优于任何单打"**(BM25/OpenSearch + e5-small-v2);纯向量对**人名("Eugene")、缩写("RAG, RLHF")、ID("gpt-3.5-turbo")** 无能为力,而关键词搜索搞不定同义词/上位词 → 互补;另可加元数据过滤(日期/评分等)精化。
- **Anthropic** ✅:BM25 精确匹配 + embedding 语义匹配,双路检索后 **rank fusion(RRF 思路)去重合并**。
- **Azure AI Search** ✅ [RAG overview](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview):官方建议 **hybrid queries(keyword+vector)最大化 recall + semantic ranking 重排**;向量查询可加权重与最低阈值。
- **DPR 里程碑数字**(Eugene Yan 综述引用):dense retrieval top-5 accuracy **65.2% vs BM25 42.9%**,且"检索精度越高,端到端 QA 越准"—— 上游决定下游。

### 3.2 Rerankers(重排器)

- **Anthropic** ✅:150 候选 → Cohere rerank → 取 top-20 生成;把整体失败率从 -49% 推到 **-67%**。
- **LlamaIndex Boosting RAG** ✅:top-10 → rerank top-5;最强组合:Jina-base + bge-reranker-large **Hit Rate 0.938 / MRR 0.869**,OpenAI embedding + CohereRerank 0.927/0.866;结论:**几乎所有 embedding 加 reranker 都显著变强,reranker 能把平庸 embedding 变成有竞争力**;但同时警告 **"基础检索太差时,再好的 reranker 也救不回来"**,且数字是数据集特定的。
- **Wang et al.** ✅(重排方法横向对比,DL19):**RankLLaMA(Llama-2-7B)精度最高**(MRR@10 32.35)但 **~82s/查询**;**monoT5(T5-base)是精度/效率平衡推荐**(31.78,4.5s);monoBERT 同级但 15.8s;**TILDEv2 最快(~0.02s)**但需文档在索引内;无 reranking 平均分 0.478→0.470,reranking 是必要项;通常 **top-50 送 rerank**。
- **Qdrant "When Is a Reranker Worth It?"** ✅ [来源](https://qdrant.tech/documentation/search-tuning/when-a-reranker-is-worth-it):工程上最实用的一篇 ——
  - 先测 **oracle gap**(完美排序 vs 当前输出,nDCG@10):5 个数据集上 200 候选时 gap 为 0.247–0.487,没 gap 就别加 reranker;
  - **先调好融合(RRF)再上 reranker**,否则你只是追平了调好的融合;tuned fusion 会让部分数据集的 reranker 增益归零(WANDS:0% 有效);
  - **从 10 个候选测起**:10 个输就不会在 200 个赢;DBPedia-entity 在 50 候选时已拿到 96% 的收益,200 候选是"4 倍工作量买 4%";
  - 选型看**匹配度(窗口长度/领域)**不是大小:5 个数据集里只有 jina-reranker-v2(1024 窗口、代码语料训练)全部复验成功;
  - CPU 实测延迟:MiniLM-L6 64–212 docs/s;bge-reranker-base 16–45 docs/s;jina-reranker-v2 CPU <2 docs/s(需 GPU,32–310 docs/s);100 候选时 CPU 上约 0.5–5s/查询。

### 3.3 Query 改写 / 扩展 / HyDE / 上下文打包

- **Wang et al. 2024** ✅(重要的"反直觉"证据):
  - **Query rewriting 与 query decomposition 反而伤害性能**(比 LLM-Embedder 裸查询更差)—— 不是所有"聪明"技巧都涨分;
  - **HyDE 有效但一篇就够**:1 篇伪文档拼 query 最优;8 篇延迟 ~2x 收益甚微;
  - **Query classification(先分类再路由)双赢**:平均分 0.428→0.443,延迟 16.4s→11.58s;
  - **Reverse repacking**:相关上下文放离 query 更近的位置最好(与 Lost in the Middle 一致);Recomp 摘要式压缩平均分最高(0.483),去掉几乎不损分但省延迟;
  - 推荐组合:**最高性能 = 分类 + Hybrid+HyDE + monoT5 + reverse repacking + Recomp(0.483,~11.7s/查询);均衡版 = 分类 + Hybrid + TILDEv2 + reverse + Recomp**;
  - 生成器微调技巧:用"相关 + 随机文档混合"训练,模型对无关上下文最鲁棒。
- **HyDE 出处与直觉**(Eugene Yan)✅:没有标注时,让 LLM 先写"假想答案文档",embed 它去检索真实文档,弥合 query-document 的 embedding gap。
- **Azure AI Search(Agentic retrieval)** ✅:官方产品化路径 —— LLM 把复杂问题拆成多个子查询并行执行,用对话历史理解指代,返回带引用的结构化结果;解决"用户问 PTO policy、文档写 time off/telecommute"的词表不匹配。
- **OpenAI Cookbook(QA with embeddings)** ✅ [来源](https://developers.openai.com/cookbook/examples/question_answering_using_embeddings):基线范式 —— 分节 embedding(text-embedding-3-small)→ cosine 相似度(`1 - cosine`)→ 默认 top_n=100 → **贪心装填到 3596 token 预算**(gpt-4o 有 128K 也不塞满)→ temperature=0;排障要点:失败先分清是检索失败还是推理失败(`print_message=True` 看喂了什么);换更强模型修好了"混合双人奖牌漏答"这类 FP7 不完整问题;成本:gpt-4o ~$0.0025/查询,gpt-4o-mini ~$0.00015/查询。
- 更多技术目录(NirDiamant/RAG_Techniques,42+ notebook)✅ [GitHub](https://github.com/NirDiamant/RAG_Techniques):Query Transformations(改写/step-back/子问题分解)、HyDE、**HyPE**(预计算每个 chunk 的假想问题,问题-问题匹配,零查询时 LLM 成本)、Contextual Chunk Headers、Relevant Segment Extraction、Contextual Compression、Fusion Retrieval、Reranking、Hierarchical Indices、Dartboard(相关性+多样性)、RAPTOR、Self-RAG、CRAG、GraphRAG/LightRAG、评估(DeepEval/RAGAS/GroUSE)等 —— 适合做训练课程的 technique 地图。

### 3.4 Embedding 模型选择

- Wang et al. ✅:MS MARCO 上 **LLM-Embedder ≈ BAAI/bge-large-en 且模型小 3 倍**,故选 LLM-Embedder;gte 系、jina、e5、all-mpnet 明显落后(该基准下)。
- Anthropic ✅:自家测试中 **Voyage 与 Gemini 最佳**;contextualization 在所有组合下都加分。
- LlamaIndex benchmark ✅(2023-11):OpenAI、Voyage、Cohere v2/v3、Jina、BGE、PaLM 横评,单 embedding 档 top-10 Hit Rate ~0.85–0.88 区间,组合 reranker 后全部 0.88–0.94。
- **元结论:embedding 排名因数据集而异,唯一可靠的排名来自你自己的评估集**(LlamaIndex、Qdrant、Wang et al. 三方一致)。Qdrant 有专文 "How to Choose an Embedding Model: Evaluation & Tradeoffs"(⚠️文章未抓取,标题见 ✅已验证的[索引页](https://qdrant.tech/articles/))。

### 3.5 自主/纠错式检索:Self-RAG、CRAG、Agentic Retrieval

- **Self-RAG(Asai et al., ICLR'24)** ✅ [arXiv:2310.11511](https://arxiv.org/abs/2310.11511):训练 7B/13B 模型按需检索,并用 **reflection tokens** 自评"检索到的段落是否相关、输出是否有支撑";在开放域 QA、推理、事实核查上**超过 ChatGPT 与检索增强的 Llama2-chat**,长文生成的引用正确率显著提升。针对的正是 FP1/FP4 类问题("固定检索 K 篇、不管是否相关")。
- **CRAG(Corrective RAG)**:评估检索质量,不行就回退 web search(见 NirDiamant 仓库 ✅;原始论文本次未单独抓取,标 ⚠️)。
- **LlamaIndex "RAG is dead, long live agentic retrieval"(2025-05-29)** ✅ [来源](https://www.llamaindex.ai/blog/rag-is-dead-long-live-agentic-retrieval):论点是**"naive top-k 向量检索已过时,agentic 检索是桌上筹码"**,不是 RAG 该死 —— 现实知识库异构(SEC 文件/会议纪要/工单/PPT),单一索引一套解析配置必然失败;生产里查询类型不可预知,需要 LLM 路由:Layer 1 选索引(LLM 分类),Layer 2 选检索方式(chunk 级/按文件名/按内容);文中点名当代技术栈:hybrid search、CRAG、Self-RAG、HyDE、reranking、多模态 embedding、RAPTOR。

---

## 4. RAG 评估方法论

### 4.1 RAGAS:事实标准指标集

来源:[RAGAS Available Metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) ✅

| 指标 | 测什么 | 需要参考答案? |
|------|--------|----------------|
| **Faithfulness** | 答案的 claims 有多少被上下文支持(支持数/总数) | 否 |
| **Answer/Response Relevancy** | 答案与问题的相关度(常用"从答案反生成问题再算相似度") | 否 |
| **Context Precision** | 相关 chunk 是否排在检索结果前列(平均精度思路) | 是 |
| **Context Recall** | 参考答案的要点有多少能归因到检索上下文 | 是 |
| Context Entities Recall | 上下文与参考答案的命名实体重叠 | 是 |
| Noise Sensitivity | 加噪声上下文后性能掉多少 | 是 |
| Aspect Critic / Rubrics | 按自定义维度做二元/打分评判 | 视定义 |
| Factual Correctness / Semantic Similarity / BLEU/ROUGE | 与参考的传统/语义比对 | 是 |

要点:LLM-based 指标可能一次到多次 LLM 调用;可写自定义指标;**Faithfulness/Relevancy 无需标注、可上线上跑**(对应"运行时验证"论断),Precision/Recall 需要金标、适合离线回归。

### 4.2 RAG Triad(TruLens/Snowflake)

见 §1.4。评估侧的互补定位:Triad 是**诊断三角形**(哪条边挂了就知道是检索/生成/任务哪个环节),RAGAS 是**指标全家桶**。两者都以 LLM-as-judge 实现地基。Snowflake 官方基准博客(⚠️正文超时未抓取,数字经 ✅已验证的 [TruLens README](https://github.com/truera/trulens) 披露):Groundedness F1 **0.81**@LLM-AggreFact、Context relevance NDCG@5 **0.93**。

### 4.3 DeepEval Faithfulness

来源:[DeepEval docs](https://deepeval.com/docs/metrics-faithfulness) ✅

- 定义:衡量 `actual_output` 是否与 `retrieval_context` 事实一致,**关注矛盾**而非"LLM 本身幻觉"。
- 计算:LLM 从输出抽 claims → 逐 claim 对照检索上下文判定 → **Faithfulness = 真实 claims / 总 claims**;默认 LLM 模式,另有 system_one(加权三问,本地小模型)模式;阈值常用 `threshold=0.7`。
- 实用排障规则(直接可写进培训文档):
  - **低 faithfulness = 生成问题,不是检索问题**(检索看 Contextual Recall/Precision);
  - **高 relevancy + 低 faithfulness = "自信地错"**:模型在用参数知识而不是检索上下文 → 收紧 prompt 强制 grounded;
  - 真话被判不忠实:因为真实世界事实不在检索文本里 → 调 `truths_extraction_limit`/`penalize_ambiguous_claims`;
  - 跨版本对比时保持同一 eval mode。

### 4.4 工程化评估流程(把评估当产品做)

**Hamel Husain "Your AI Product Needs Evals"** ✅ [来源](https://hamel.dev/blog/posts/evals/)(两次抓取交叉验证)

- 判断:失败的 LLM 产品几乎都共享一个根因 —— **没有评估系统**;成功 = 迭代速度,评估是"软件工程里的测试"。
- 三级评估:Level 1 单测(pytest 式断言,每次改代码就跑;Rechat 有数百条,含"输出不得泄漏 UUID"这类通用断言;**LLM 产品的单测不要求 100% 通过,阈值是产品决策**)→ Level 2 人评 + LLM-as-judge(前提是先记录 traces)→ Level 3 A/B(产品成熟后再做,最贵)。
- **先看数据(误差分析)**:"你必须把看数据的摩擦降到零";自建领域查看工具(Shiny/Gradio,<1 天),一屏看全 trace+场景标签+合成/真实;早期把测试集 trace 全看一遍,之后抽样,"直到看不出新东西为止"。
- 标注用**二元好/坏**,不要细粒度打分("更难管理");从失败模式反推断言,失败模式固化为新单测,治疗 whack-a-mole。
- LLM-as-judge 校准:用你能负担的最强模型;**每次人工标 25~50 条,用表格对齐 judge 与人**,持续监控一致性;⚠️ 类别不平衡时 **raw agreement 会骗人,要分别看 precision/recall**。
- 合成数据:直接让 LLM 生成测试输入(一次 50 条指令/查询);别等生产数据;模型过不了的难测试 = 微调数据的来源;**微调 99% 的工作量是数据治理,评估系统顺手就把它干了**。

**Eugene Yan** ✅ [LLM Patterns](https://eugeneyan.com/writing/llm-patterns/)(索引页 [writing](https://eugeneyan.com/writing/) 亦 ✅):
- "Eval Driven Development (EDD)":构建任务专属评估,别信 off-the-shelf 榜单(引 QLoRA 作者:"不要用/不要信 MMLU 分数");
- 检索上游决定下游(DPR:检索精度↑ → 端到端 QA↑);
- LLM-as-judge 已知偏差清单:位置偏差、冗长偏差、自评偏差(self-enhancement),需缓解设计;
- ANN 索引选型评估维度:recall vs 精确近邻、延迟/吞吐、内存、插入便利 —— "先定义功能与非功能需求再跑分"。他的相关深文:[evals](https://eugeneyan.com/writing/evals/)、[llm-evaluators](https://eugeneyan.com/writing/llm-evaluators/)、[qa-evals](https://eugeneyan.com/writing/qa-evals/)(⚠️深文未逐篇抓取)。

**其他可引用的评估实践**:
- Seven Failure Points ✅:OpenAI Evals 自动评 + 人工复核,自动评估**更悲观** —— 提醒别把 LLM-judge 分数当绝对值;
- LlamaIndex chunk-size 实验 ✅:小样本(20 题)+ GPT-4 评 Faithfulness/Relevancy 的轻量法,任何团队当天可复制;
- Qdrant reranker 文档 ✅:引入"oracle gap / held-out validation"作为**要不要上某个检索组件**的决策程序,这是检索调参的实验设计范本。

---

## 5. 生产经验与"RAG is dead / long live RAG"之争

### 5.1 长上下文 vs RAG:数据说话

- **Databricks "Long Context RAG Performance of LLMs"** ✅ [来源](https://www.databricks.com/blog/long-context-rag-performance-llms):
  - 设置:2000+ 实验、13 个 LLM、4 个数据集(Databricks DocsQA、FinanceBench、HotPotQA、NQ);检索用 text-embedding-3-large、**512 token chunk + 256 stride**、FAISS。
  - 检索侧:embedding recall 随 token 预算 0.468@2k → 0.788@8k → 0.894@32k → **~0.95 饱和@96k**;NQ 在 8k 就 1.0。
  - 生成侧:准确率**先升后降**,且**每个模型的拐点不同**:Llama-3.1-405b 32k 后下降,gpt-4-turbo/claude-3-sonnet 16k 后下降,dbrx 8k 后崩(0.528→0.255@32k);新模型(gpt-4o、claude-3.5-sonnet、gpt-4o-mini)"little to no deterioration"。
  - 长上下文特有失败模式:claude-3-sonnet **版权拒答 3.7%@16k → 49.5%@64k**;DBRX 指令失效率 5.2%@8k → 50.4%@32k;Mixtral 复读/乱写;GPT-4/Llama-405b 主要是答错。
  - 结论:**长上下文是 RAG 的组件而非替代品;"长上下文取代 RAG"仍需模型大幅进步**。
- **Chroma Context Rot** ✅:从"退化是普遍现象"角度给同一争论补刀(§1.3)。
- **OpenAI Cookbook** ✅:即使 gpt-4o 有 128K 窗口,官方示例也只给 ~3.6K token 预算 —— 检索+裁剪而非倾倒,是默认工程姿态。
- (⚠️未验证,标题经 ✅索引确认)Qdrant 亦有专文 "Is RAG Dead? Why Long Context Windows Don't Replace RAG"。[Qdrant articles 索引](https://qdrant.tech/articles/) ✅

### 5.2 演化方向:Agentic Retrieval(LlamaIndex 观点)✅

见 §3.5:检索从"固定管线"走向"LLM 决策的多层路由";生产系统的核心不再是"选一个 chunk size",而是**索引拆分 + 查询路由 + 组件编排**。与 Seven Failure Points 的"持续演化论"、Wang et al. 的"query classification 先行"互为印证。

### 5.3 生产落地检查单(综合一手建议)

1. 语料 < 200K token(~500 页)时先问要不要 RAG(Anthropic ✅);
2. 基线:句子/递归切分,512 token + 20~100 overlap;用自建问题集跑 hit rate/MRR/faithfulness(Weaviate ✅、LlamaIndex ✅、Wang ✅);
3. 上 hybrid(dense+BM25,α≈0.3 起)+ rank fusion(Anthropic ✅、Wang ✅);
4. 上下文增强:contextual retrieval(预算允许)、sentence-window、late chunking(§2.3);
5. reranker 当作**需持证上岗的组件**:先测 oracle gap,10 候选起步,held-out 复验(Qdrant ✅);送排 50 候选、产出 top-20(Wang ✅、Anthropic ✅);
6. 上下文打包:reverse repacking,相关内容放头尾,控制预算不塞满(Lost in the Middle ✅、Wang ✅、OpenAI ✅);
7. 元数据(文件名+chunk 号)进 prompt,formatting 指令显式化(Barnett ✅);
8. 评估:无金标先上 faithfulness/relevancy,有金标加 context precision/recall;建 trace + 三级评估 + judge 校准(RAGAS ✅、TruLens ✅、DeepEval ✅、Hamel ✅);
9. 把 RAG 失败点清单(FP1–FP7)做成运行时监控面板(Barnett ✅)。

---

## 6. 资源索引(用于培训课程扩展)

- **LangChain rag-from-scratch**(配套 YouTube 播放列表)✅ [GitHub](https://github.com/langchain-ai/rag-from-scratch):5 个 notebook 覆盖 topics 1–18,从 indexing/retrieval/generation 基础建到高级技术(README 本次抓取未列出各 topic 名称,课程细节见视频列表 ⚠️);[播放列表](https://www.youtube.com/playlist?list=PLfaIDFEXuae2LXbO1_PKyVJiQ23ZztA0x)
- **NirDiamant/RAG_Techniques**(42+ notebook,CC BY-NC)✅:按"基础→查询增强→上下文增强→高级检索→迭代自适应→图架构→评估"组织,是现成的实操课大纲(清单见 §3.3)。
- **LlamaIndex 博客地图** ✅ [blog 索引](https://www.llamaindex.ai/blog):评估类(Evaluating RAG with DeepEval、GPT-4 vs Prometheus、OpenAI Cookbook: Evaluating RAG systems)、混合检索 α 调参、reranker 微调、LlamaParse 抽取、"RAG is dead, long live agentic retrieval"、"Is grep all you need?"(词法 vs 语义检索)等。
- **Chip Huyen 博客地图** ✅ [blog 索引](https://huyenchip.com/blog/):无 RAG 专文,但 [Common pitfalls when building generative AI applications](https://huyenchip.com/2025/01/16/ai-engineering-pitfalls.html)、[Building A Generative AI Platform](https://huyenchip.com/2024/07/25/genai-platform.html)、[Building LLM applications for production](https://huyenchip.com/2023/04/11/llm-engineering.html) 覆盖检索组件与评估(⚠️各篇正文未抓取)。
- **Eugene Yan 写作地图** ✅ [writing 索引](https://eugeneyan.com/writing/):评估/检索/LLM 模式三大板块,详见 §4.4。
- **Microsoft/Azure**:RAG 官方设计评估指南(✅见 §1.5/§3.1);**Google Cloud / Databricks**:Databricks 长上下文研究 ✅(§5.1);Google Cloud RAG 指南本次未抓取 ⚠️。
- **Vectara 博客** ✅(索引页已验证):近期偏企业搜索/多模态(Boomerang V2:8192-token 多模态 embedding、Matryoshka 截断),RAG 落地类旧文未抓取 ⚠️;其 HHEM 幻觉榜单是已知延伸资源 ⚠️。

---

## 7. 来源清单与验证状态(汇总)

| # | 来源 | URL | 状态 | 关键贡献 |
|---|------|-----|------|----------|
| 1 | Anthropic: Contextual Retrieval | https://www.anthropic.com/news/contextual-retrieval | ✅ | -35%/-49%/-67% 失败率;管道;成本 |
| 2 | Barnett et al., Seven Failure Points (arXiv:2401.05856) | https://arxiv.org/abs/2401.05856 | ✅(全文抓取) | 7 个失败点 + 运行时验证论 |
| 3 | Wang et al., Searching for Best Practices in RAG (arXiv:2407.01219) | https://ar5iv.labs.arxiv.org/html/2407.01219 | ✅(全文抓取) | 512 token、α=0.3、monoT5、HyDE、反向打包 |
| 4 | Liu et al., Lost in the Middle (arXiv:2307.03172) | https://ar5iv.labs.arxiv.org/html/2307.03172 | ✅(全文抓取) | U 形曲线精确数字;closed-book 56.1% |
| 5 | Asai et al., Self-RAG (arXiv:2310.11511) | https://arxiv.org/abs/2310.11511 | ✅ | reflection tokens;超 ChatGPT |
| 6 | RAGAS docs | https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/ | ✅ | 指标全家桶 |
| 7 | LangChain rag-from-scratch | https://github.com/langchain-ai/rag-from-scratch | ✅ | 课程系列入口 |
| 8 | Pinecone: Chunking Strategies | https://www.pinecone.io/learn/chunking-strategies/ | ✅ | 尺寸区间;chunk expansion;4 因素 |
| 9 | Weaviate: Chunking Strategies for RAG | https://weaviate.io/blog/chunking-strategies-for-rag | ✅ | 10 类策略;512+50~100 基线 |
| 10 | OpenAI Cookbook: QA using embeddings | https://developers.openai.com/cookbook/examples/question_answering_using_embeddings | ✅ | 基线管线;预算装填;排障 |
| 11 | LlamaIndex: Auto-Merging Retriever 示例 | https://developers.llamaindex.ai/python/examples/retrievers/auto_merging_retriever/ | ✅ | 2048/512/128;结果打平的诚实披露 |
| 12 | LlamaIndex: Node Parsers 文档 | https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/ | ✅ | sentence-window/层级/语义切分参数 |
| 13 | LlamaIndex: Boosting RAG(embedding×reranker 横评) | https://www.llamaindex.ai/blog/boosting-rag-picking-the-best-embedding-reranker-models-42d079022e83 | ✅ | Hit Rate/MRR 全表 |
| 14 | LlamaIndex: Evaluating the Ideal Chunk Size | https://www.llamaindex.ai/blog/evaluating-the-ideal-chunk-size-for-a-rag-system-using-llamaindex-6207e5d3fec5 | ✅ | 1024 峰值;评估法 |
| 15 | LlamaIndex: RAG is dead, long live agentic retrieval | https://www.llamaindex.ai/blog/rag-is-dead-long-live-agentic-retrieval | ✅ | agentic 路由;技术栈地图 |
| 16 | LlamaIndex 博客索引 | https://www.llamaindex.ai/blog | ✅ | 扩展文章地图 |
| 17 | Hamel Husain: Your AI Product Needs Evals | https://hamel.dev/blog/posts/evals/ | ✅ | 三级评估;judge 校准 25~50 条 |
| 18 | Eugene Yan: LLM Patterns | https://eugeneyan.com/writing/llm-patterns/ | ✅ | hybrid 生产经验;EDD;DPR 数字 |
| 19 | Eugene Yan: writing 索引 | https://eugeneyan.com/writing/ | ✅ | 评估深文地图 |
| 20 | Databricks: Long Context RAG Performance | https://www.databricks.com/blog/long-context-rag-performance-llms | ✅ | 2000+ 实验;各模型拐点数字 |
| 21 | Chroma: Context Rot | https://www.trychroma.com/research/context-rot | ✅ | 18 模型普遍退化;distractor 效应 |
| 22 | Jina: Late Chunking | https://jina.ai/news/late-chunking-in-long-context-embedding-models | ✅ | BeIR +6.5;nDCG 表 |
| 23 | TruLens: RAG Triad | https://www.trulens.org/getting_started/core_concepts/rag_triad/ | ✅ | 三角定义 |
| 24 | TruLens GitHub README(含基准数字) | https://github.com/truera/trulens | ✅ | F1 0.81 / NDCG@5 0.93 / 95% |
| 25 | DeepEval: Faithfulness | https://deepeval.com/docs/metrics-faithfulness | ✅ | 公式;排障规则 |
| 26 | Qdrant: When Is a Reranker Worth It? | https://qdrant.tech/documentation/search-tuning/when-a-reranker-is-worth-it | ✅ | oracle gap;10 候选;延迟表 |
| 27 | Qdrant: articles 索引 | https://qdrant.tech/articles/ | ✅ | 文章地图(Is RAG Dead 等) |
| 28 | Microsoft/Azure AI Search: RAG overview | https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview | ✅ | hybrid+semantic ranking;agentic retrieval |
| 29 | NirDiamant/RAG_Techniques | https://github.com/NirDiamant/RAG_Techniques | ✅ | 42+ 技法实操大纲 |
| 30 | Vectara 博客索引 | https://vectara.com/blog/ | ✅(索引) | 企业搜索方向 |
| 31 | Chip Huyen 博客索引 | https://huyenchip.com/blog/ | ✅(索引) | pitfalls/platform 文章定位 |
| 32 | Snowflake: Benchmarking RAG Triad LLM-judge | https://www.snowflake.com/en/engineering-blog/benchmarking-LLM-as-a-judge-RAG-triad-metrics/ | ⚠️(超时,数字经 #24 交叉) | F1/NDCG 基准 |
| 33 | Qdrant: Is RAG Dead?(具体 URL 未定位) | —(见 #27 索引) | ⚠️ | 长上下文 vs RAG 辩题 |
| 34 | Google Cloud RAG 指南 | —(未抓取) | ⚠️ | 未覆盖 |

> 复现提示:统计口径 —— #1–#31 为本次会话实际抓取并核实内容的来源(其中 #30/#31 为索引页级验证);共 **29 个来源内容级验证 + 2 个索引级验证**,另 3 条 ⚠️。
