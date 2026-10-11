# H. 意图识别 (Intent Classification) 与模型路由 (Model Routing)

> Lane: 英文一手来源 (论文 + 官方文档 + 官方定价页)，2026-09-27 检索。
> 标注说明：✅已验证 = 本次直接抓取页面核对；⚠️未验证 = 仅搜索结果可见、未抓原文。

---

## 0. TL;DR 决策表

| 场景 | 推荐方案 | 依据 |
|---|---|---|
| 意图/路由，低延迟 (<100ms)、无 GPU | Semantic router (encoder + cosine 阈值) | ✅ Aurelio semantic-router：纯 encoder，社区 10ms 级实测 |
| 意图类目多 (50+ 类)、每类只有 8~64 条标注 | SetFit few-shot 微调 | ✅ arXiv 2209.11055：N=8 胜普通全量微调 19.3 分 |
| 大规模固定意图集、追求最高准确率 | 微调 encoder (BERT/ModernBERT/DeBERTa)，配 OOS 检测 | ✅ CLINC150、✅ ModernBERT (比 DeBERTa 快约 2 倍) |
| 快速原型 / 意图经常变 / 类目数十个 | LLM zero/few-shot + structured output + logprobs 置信度 | ✅ OpenAI Cookbook；成本约小模型百倍 |
| 高频定型微决策 (路由/工具选择)，TypeSafe 式 | Typed decision model (Jev / Laya) | ✅ Laya 32.8ms vs Jev 236–276ms，输出概率分布不可幻觉 |
| 省钱：大模型池中按 query 挑便宜模型 | RouteLLM / Hybrid LLM / GraphRouter / 商用 router | ✅ RouteLLM README：最多省 85% 成本保 95% GPT-4 质量 |
| 降碳/降本：简单 query 不进大模型 | LLM cascade (FrugalGPT) | ✅ 最多省 98% 成本追平 GPT-4 |

核心结论：**路由收益取决于流量分布** (多少比例 query 本可以给弱模型) —— RouteLLM 用 APGR 指标显式刻画；MMLU/GSM8K 类任务仅靠人类偏好数据路由近乎随机 (必须加金标数据)；语义路由 ms 级 vs LLM 路由百 ms~s 级，两者是"意图识别"和"模型选择"两个层次。

---

## 1. 意图识别方法谱系 (Intent Classification)

### 1.1 方法对比总表

| 方法 | 准确率 (代表数字) | 延迟 | 单条成本 | 标注数据需求 | 备注 |
|---|---|---|---|---|---|
| 关键词/正则规则 | 无公认基准；独立评测中"两行正则 91.8% 胜过直问 LLM 62.6%"(钓鱼任务, ⚠️ jev-radar CASEBOOK 引述) | <1ms | ~0 | 无 (人工写规则) | 召回低、难维护；适合做第一层粗筛/白名单 |
| Embedding kNN / Semantic router | 依赖阈值调优 (route 阈值可训练) | ~10ms 级 (✅社区 10ms hotline challenge) | 仅 embedding 费 | 每路由十几条 utterance | 无 LLM 参与决策，返回 None 走兜底 |
| SetFit (sentence-transformers few-shot 微调) | N=8/类平均 62.3 (5 数据集)，超普通微调 19.3 分；RAFT 71.3 (355M) 超 GPT-3 175B 的 62.7 | 训练 ~30s / 25 美分；推理比 T-Few 3B 快 ~19x (MPNet) ~123x (MiniLM) | ~0 (本地) | 8~64 条/类 | 无需 prompt/verbalizer；checkpoints 小 26–163x |
| 微调 encoder (BERT/ModernBERT/DeBERTa) | CLINC150/全量数据 SOTA 档；Jev 77 类 Banking77 达 0.870 | ModernBERT 比 DeBERTa 快 ~2x (混合长度 4x)，<1/5 显存 | ~0 | 全量 (数百~数千/类) | 生产线最成熟；必须配 OOS 检测 (CLINC150 教训) |
| LLM zero/few-shot + structured output | 未在 CLINC150 出正榜；RAFT 上 GPT-3 175B 仅 62.7 (远低于 SetFit 71.3) | 100ms~s 级 | $0.05~$0.6/1M in (小模型) | 0~几十条示例 | 类目变更零成本；用 logprobs 拿置信度 |
| Typed decision model (Jev/Laya) | Laya 微调 0.766 vs Jev 0.727；宽类目 Jev 0.870 胜 Laya 0.425 (Banking77) | Laya 32.8ms vs Jev 236–276ms | Laya 本地 / Jev API | Laya zero-shot 弱 (0.362)，需微调 | 单次前向输出 choice/score/bool 概率分布，物理上无法幻觉散文 |

### 1.2 CLINC150：OOS (out-of-scope) 是生产线真正难点

- 论文：Larson et al., "An Evaluation Dataset for Intent Classification and Out-of-Scope Prediction", EMNLP-IJCNLP 2019。✅ https://aclanthology.org/D19-1131/
- 数据构成 (✅ https://github.com/clinc/oos-eval)：`data_full.json` 含 150 个 in-scope 意图类，每类 100 train / 20 val / 30 test (= 22,500 条) + 100 OOS train + 100 OOS val + 1,000 OOS test → 合计约 23,700 条；10 个领域 (银行/旅游/公用事业等)；英文、单意图。
- 官方结论 (✅ Anthology 页)：分类器 "perform well on in-scope intent classification" 但 "struggle to identify out-of-scope queries" —— 低数据场景下 OOS 检测尤其差。
- **工程含义**：任何意图分类器都要配 OOS 通道 (相似度阈值 / logprobs 低于门限 / router 返回 None → 转大模型或人工)，这正是"fail-open 到大模型"决策表的出处。

### 1.3 SetFit：few-shot 微调的基准数字

论文 arXiv 2209.11055 (✅ 摘要 + ✅ ar5iv 全文) + HuggingFace 博客 (✅ https://huggingface.co/blog/setfit)：

- 机制：对比学习 (contrastive Siamese) 微调 Sentence Transformer → 再训分类头；无需 prompt 或 verbalizer；摘要称 "orders of magnitude less parameters"、"an order of magnitude faster to train"。
- N=8/类，5 数据集平均：**SetFit_MPNet 62.3** vs T-Few 3B 63.4 (差距 ~1 分) vs Adapet 58.3 vs 普通微调 43.0 —— **N=8 时比普通微调平均高 19.3 分**。分项：SST-5 43.6 / AmazonCF 40.3 / CR 88.5 / Emotion 48.8 / EnronSpam 90.1 / AGNews 82.9。
- N=64/类：SetFit 75.3 > T-Few 3B 70.3、Adapet 73.8、全量微调基线 69.7。
- 全量微调参考：84.8 (Emotion 92.6 / EnronSpam 99.0 / AGNews 93.8)。
- RAFT：SetFit_RoBERTa (355M) 71.3，**超 GPT-3 175B (62.7) 8.6 分**，距 T-Few 11B (75.8) 4.5 分，human 73.5。
- 成本：p3.2xlarge 训练 ~30 秒 (每 split $0.025) vs T-Few ~700s on p4d.24xlarge ($0.7)；存储 checkpoint 70MB/420MB，比 T0-3B 的 11.4GB 小 163~26 倍；参数量 MPNet 110M，比 T-Few 3B 小 27+ 倍。
- HF 博客补充：CR 情感集上 8 例/类即可匹敌 RoBERTa-large 全量 3k 例微调；蒸馏可获得 123x 推理加速。

### 1.4 微调 encoder：ModernBERT / DeBERTa

- ModernBERT (arXiv 2412.13663, ✅摘要 + ✅ HF 博客 https://huggingface.co/blog/modernbert)：149M/395M 两档；原生 8192 token 上下文；**约 2 倍快于 DeBERTa，混合长度输入下最多 4x**；首个在 GLUE 上击败 DeBERTaV3 的 base 级模型，显存不足其 1/5；unpadding (Flash Attention RoPE) 带来 10–20% 额外加速；官方推荐场景第一条就是 classification (内容审核类)。
- Jev 的 Banking77 0.870 表明：**宽类目 (77 类) 上微调 encoder 仍是天花板** (Laya 421M 仅 0.425，其论文自认 zero-shot 弱)。

### 1.5 LLM 当分类器：structured output + logprobs 置信度

OpenAI Cookbook "Using logprobs" (✅ https://github.com/openai/openai-cookbook/blob/main/examples/Using_logprobs.ipynb)：

- 参数：`logprobs=true` + `top_logprobs=2..5`；置信度 = `exp(logprob) * 100`。
- 多分类示例：新闻标题 4 类 (Technology/Politics/Sports/Art)，prompt 要求"只返回类名"；清晰样本置信度 ~100%，歧义样本 "Art" 97.23% vs "Sports" 1.39%。
- 检索可用性判断：让模型输出 boolean `has_sufficient_context_for_answer`，该 token 的 logprob 即置信度 —— **这就是 RAG "该不该信检索结果" 的置信门控范式**。
- 官方警告 (原文)："confidence measures certainty in the classification, not whether the classification is correct"；"validate its classifications and thresholds with task-specific evals"。
- 工程模式：高置信自动分类，低置信转人工/转大模型 (阈值由业务评测定)。

### 1.6 Semantic router (语义路由)：encoder + cosine 阈值 + 动态路由

Aurelio Labs `semantic-router` (✅ https://github.com/aurelio-labs/semantic-router, MIT, ~3.9k stars)：

- 机制：定义 Route (name + 示例 utterances) → encoder (Cohere/OpenAI/HuggingFace/FastEmbed) 向量化 → RouteLayer 按 cosine 相似度匹配；低于阈值返回 `None` (天然 OOS 兜底)。
- 延迟：README 定位 "superfast decision-making layer"；社区引用 "real life 10ms hotline challenge" (ollama/gemma2) —— **毫秒级，无 LLM in the loop**。
- Dynamic routes：匹配后可再调 LLM 抽参数/函数调用 (此时才有 LLM 成本)；阈值可按 route 训练优化 (Route Optimization notebook)；支持本地 (`semantic-router[local]`，HuggingFaceEncoder + LlamaCppLLM)、混合检索、图像路由、Pinecone/Qdrant 集成、LangChain agent 集成。
- 应用样例：7 个合成医疗行政工作流的 routing + 阈值优化 + 评测 notebook；IEEE GlobeCom 2024 论文用于 5G 核心网管理。

### 1.7 Typed decision models (Jev / Laya)

flowtivity 博客 (✅ https://flowtivity.ai/blog/laya-open-source-jev-alternative)：

- **Jev** (TypeSafe AI，闭源 API)："typed decision model"，非自回归 System 1 引擎，单次前向回答"你定义的问题"：`choice` (选选项+完整概率分布) / `score` (序数打分) / `boolean` (校准 P(true))；"physically cannot hallucinate prose or emit malformed JSON"。
- **Laya** (Convai Innovations，Apache 2.0，2026-09-18 发布 3 个 HF checkpoint)：`laya` ModernBERT-large 421M/512ctx 英文 triage；`laya-multilingual` mmBERT-base 322M/1024ctx 100+ 语言；`laya-typed-decisions` 421M/1024ctx。作者声称先于 Jev 发布核心思想 (arXiv 2503.23303 / 2510.01237 ⚠️未验证内容)。
- 对比数字 (该博客引述)：
  | 指标 | Laya | Jev |
  |---|---|---|
  | hard-label 准确率 | 0.766 (微调后) / 0.362 (zero-shot) | 0.727 |
  | 延迟 | **32.8 ms** | 236–276 ms |
  | 校准 ECE | 0.081 (按题重拟合温度后) | 0.246 |
  | Banking77 宽类目 | 0.425 | **0.870** |
- 另有 Reddit 头对头 "Jev 92.9% vs Laya 421M 65.3%" ⚠️未验证。
- **结论**：typed decision model ≈ "微调 encoder 分类器 + 校准概率输出" 的产品化；小类目快而稳，宽类目仍是 encoder 微调/大模型更强。

### 1.8 置信度校准 (calibration)：级联兜底的数学基础

- Guo et al. 2017, "On Calibration of Modern Neural Networks" (✅ https://arxiv.org/abs/1706.04599)：现代神经网络普遍 miscalibrated (过度自信)；深度/宽度/weight decay/BatchNorm 影响校准；**temperature scaling (Platt Scaling 单参数版) "surprisingly effective"**，几乎所有数据集有效。
- 工程含义：置信门控级联前，先用验证集拟合温度，让 P(correct) 有意义；Laya 的 ECE 0.081 也是温度重拟合后的结果。

---

## 2. 模型路由 (Model Routing: query → which LLM)

### 2.1 RouteLLM (LMSYS) —— 核心数字

论文 arXiv 2406.18665 (✅摘要 + ✅ html v2 全文) + GitHub README (✅ https://github.com/lm-sys/RouteLLM)：

- **头号声称 (README)**："reduce costs by **up to 85%** while maintaining **95% of GPT-4 performance**" (MT Bench 类基准) —— 用户要求核对的 "85%/95%" 出处就是 README；**论文正文的精确数字见下表 (85% 对应不了论文表，引用时建议用论文表)**。
- 论文 Table 6 (最佳 router，质量锚点 → 成本节省)：
  | 基准 | 95% 质量档 (CPT 50%) 节省 | 质量锚点 | 80% 档节省 |
  |---|---|---|---|
  | MT Bench | **3.66x (≈73%)** | 95% of GPT-4 | 2.49x |
  | MMLU | 1.41x (≈29%) | 92% of GPT-4 | 1.14x |
  | GSM8K | 1.49x (≈33%) | 87% of GPT-4 | 1.27x |
- MT Bench (Table 1)：Matrix Factorization 最优 —— CPT(50%)=13.40% (只需 13.4% 额外 GPT-4 调用即可让一半 query 升级到 GPT-4 质量)，CPT(80%)=31.31%，APGR=0.802 (比随机路由 +60.4%)；BERT 分类器 19.58%；causal LLM 31.50%；SW ranking 23.21%。
- **流量分布依赖的关键发现**：MMLU 上仅用 Arena 偏好数据训练的 router 接近随机；加约 1,500 条金标 (<2% 训练量) 后所有 router 提升 ~20%；GSM8K 更差 (Arena 数据单独低于随机)，需 GPT-4-judge 增广 (~120K 条，总成本 ~$700)。→ **路由器质量 = 训练数据与目标任务的匹配度**。
- 训练数据：80K Chatbot Arena 对战 (65K 成对偏好，64 个模型聚成 10 个 Elo tier，tier 0–1 为强模型、tier 2 为弱)；强=GPT-4-1106-preview，弱=Mixtral-8x7B。
- 成本假设：$24.7/M tokens (GPT-4) vs $0.24/M (Mixtral)，**约 100 倍价差** —— 这是路由存在的前提。
- 路由开销可忽略：Matrix Factorization **155 req/s，$1.42 每百万请求**；连 SW ranking 也 ≤GPT-4 生成成本 0.4%。
- 泛化：router 不重训直接换 Claude 3 Opus / Llama 3 8B 对，APGR 0.703 (+42.2%)。
- Router 类型：SW ranking (Bradley-Terry Elo，免训练) / **matrix factorization (推荐)** / BERT 分类器 / causal LLM (Llama 3 8B) / random 基线。
- 用法：OpenAI 兼容 server，`model="router-mf-0.11593"`；`routellm.calibrate_threshold` 按"强模型使用占比"目标校准阈值 (50% 目标 → 0.11593)。注意 mf/sw_ranking 仍需 embedding key。
- 对比商用：论文附录 E 称对 Unify AI、Martian 在 MT Bench 上以"最多少 40% 的 GPT-4 调用"追平表现。

### 2.2 FrugalGPT (cascade 鼻祖)

arXiv 2305.05176 (✅摘要)：

- LLM API 定价可差**两个数量级**。
- 三个降本策略：prompt adaptation / LLM approximation / **LLM cascade**。
- FrugalGPT (cascade 实例)：**追平 GPT-4 性能最多省 98% 成本；同等成本下比 GPT-4 高 4% 准确率**。
- 机制：多个 LLM 按价格排序串联，便宜模型先答，判分器 (评分/一致性) 认为不可靠才升级 —— 与置信门控级联同构。

### 2.3 Hybrid LLM：路由器延迟数字 (DeBERTa router)

arXiv 2404.14618 (✅摘要 + ✅ ar5iv 全文)：

- 路由器：**DeBERTa-v3-large (300M)，单 query 0.036s** —— 比 FLAN-t5-800m (0.46s) 快约 10 倍，比 Llama-2 7B (7.99s) 快 200+ 倍；10k 训练样本 (MixInstruct，每样本 10 个 LLM 回答)。
- 总声称：**最多减少 40% 大模型调用且质量无显著下降**；导语图示：减少 22% GPT-3.5-turbo 调用、BART 分降幅 <1%。
- 按模型对价差分解 (质量降幅约束 vs 成本收益)：小价差对 (Llama-2 7b→13b) 40% 成本优势仅 0–0.2% 降幅；中价差 (Llama-2 13b→GPT-3.5) 20% 优势 ≤1% 降幅；**大价差对 (FLAN-t5-800m→Llama-2 13b) 拿 40% 成本优势要付 10.3% 质量代价** → 级联收益强依赖强/弱模型差距。
- 阈值校准：仅用 500 条验证样本、≤1% 降幅约束下定的阈值可泛化到测试集 (如 7b/13b 对拿到 96–98.6% 成本优势、测试降幅 1.4–1.6%)。

### 2.4 2024–2026 新路由研究

- **GraphRouter** (arXiv 2410.03834, ✅摘要)：task/query/LLM 异构图 + 边预测预测"响应效果与成本"；3 种 effect-cost 权重场景下**比现有 router 至少 +12.3% 质量**；新 LLM 泛化 **+9.5%**、无需重训。代码 github.com/ulab-uiuc/GraphRouter。
- **RouterBench** (arXiv **2403.12031** —— 注意不是 2411.12626，后者是扩散几何论文, ✅摘要)：Martian 团队出品，**405k+ 推理结果**的 (prompt, 模型, 响应, 成本, 质量) 数据集，评测任意 router 的 quality-cost Pareto 前沿；11 个 LLM、覆盖 GSM8K/HumanEval/ARC/HellaSwag/MMLU/QuALITY (11 个模型与覆盖面来自搜索摘要 ⚠️部分)；结论：没有单一模型全任务最优，路由有真实收益，但当时 SOTA router 离理论最优前沿还很远。
- **Routing survey** ("A Survey of Model Routing", arXiv 2503.10657, ✅摘要)：定义 routing = "用 router 从候选池为给定输入推荐最佳 LLM"；实测 **8,500+ LLM、2 亿+ 性能记录、12 种评测**；发现 "model-level scaling up"：候选池越大路由收益越大，**可超过池中最佳单模型和许多强 LLM**；同时"现有方法仍有显著提升空间"。
- **CARROT** (arXiv 2502.03261, ✅摘要)：成本感知 rate-optimal router，minimax 分析证明"同时估计每模型成本与准确率的简单 router"是 minimax 最优；发布 **SPROUT 数据集** (Smart Price-aware ROUTing)；在 SPROUT/RouterBench/Open-LLM-Leaderboard-v2 上验证，v2 (2025-05) 加 o3-mini。
- **SPROUT (碳效)** (arXiv 2403.12900, EMNLP 2024, ✅摘要)：MIT-LL 的 carbon-efficient 推理，用 "generation directives" + 离线质量评估器，Llama2 实测**碳排放降 40%+**。注意：名字与 CARROT 的 SPROUT 数据集是两回事。

### 2.5 商用/Agent 路由器

- **OpenRouter Auto Router** (`openrouter/auto`, ✅ https://openrouter.ai/auto)：2026-09 现状 —— "**powered by the wisdom of the market**"：按 OpenRouter 社区对"同类任务"近 7 天实际付费选择模型，自动跟进新模型；`cost_tier` = low/medium/high/xhigh/max (默认 low = 最省)；遵守 guardrails/隐私 ZDR/模型限制；多轮对话固定同一模型直到不再是该任务优选；**router 本身免费**，按被路由到的模型计价。(历史：曾由 Not Diamond 驱动，⚠️第三方资料如此说；Not Diamond 官网亦宣称 "powers OpenRouter's intelligent routing" ✅。)
- **Not Diamond** (✅ https://notdiamond.ai/)：定位 "intelligent model router for coding agents"，按输入预测该用哪个模型 (权衡 provider/model/domain/reasoning/complexity/sub-agents/cache/context/harness)；官网声称 **准确率 +5%、成本 −20%、开发周期 2x**；案例推演 $4.8M/年 → $3.6M/年 (25%，等效输出质量)；客户 Rootly 称 SRE 基准平均准确率 +39%；SOC-2/ISO 27001。
- **Martian** (✅ https://withmartian.com/，官网 2026 已转向 "Understanding Intelligence"/Thesean 品牌)：RouterBench 是其历史产出；RouteLLM 论文把 Martian 当商用对比对象 ("最多少 40% GPT-4 调用即追平")。当前无公开路由质量/成本新数字。
- **Unify** ⚠️：仅在 RouteLLM 附录对比中出现，未另行验证。

### 2.6 框架内建路由 (LlamaIndex / LangChain)

- **LlamaIndex RouterQueryEngine** (✅ https://developers.llamaindex.ai/python/framework/module_guides/querying/router/)：
  - Router = "接收 query + 一组带元数据的 choices，返回一个或多个选择"，**用 LLM 做决策** (所以这是"慢路由"，与 semantic router 的"快路由"相对)。
  - Selector 四种：`LLMSingleSelector` / `LLMMultiSelector` (文本补全) 与 `PydanticSingleSelector` / `PydanticMultiSelector` (function calling 返回 Pydantic 对象)。
  - 典型用途：从多个数据源选一；**summary index vs vector index (摘要问题走列表式、细节问题走向量检索)**；multi-routing 并行取多路再合并。配套 `RouterRetriever`；docs 还列出 Router Query Engine / Retriever Router / **SQL Router Query Engine** 等示例 (SQL+向量混合检索即 RAG 查询路由)。
  - 大量 choices 时可用 `ToolRetrieverRouterQueryEngine` (检索式路由, beta)。
- **LangChain** (✅ source docstring, langchain_core/runnables/branch.py)：
  - `RunnableBranch((condition, runnable), ..., default)`：按序求值条件，第一个 True 的分支执行，否则跑 default；条件可为任意谓词 (含 embedding 相似度 → 语义路由)。
  - 新版官方文档把条件编排重心移向 LangGraph (middleware 也提供 routing 能力)。

---

## 3. 延迟与价格基准 (路由的 cost-latency 地板)

### 3.1 OpenAI 官方定价 (per 1M tokens, ✅ platform.openai.com/docs/pricing, 2026-09-27 抓取)

| 模型 | Input | Cached input | Output | 路由角色 |
|---|---|---|---|---|
| gpt-5 | $1.25 | $0.125 | $10.00 | 强模型档 |
| gpt-5-mini | $0.25 | $0.025 | $2.00 | 均衡档 |
| gpt-5-nano | $0.05 | $0.005 | $0.40 | 意图分类/路由档 |
| gpt-4.1 | $2.00 | $0.50 | $8.00 | 强模型档 |
| gpt-4.1-mini | $0.40 | $0.10 | $1.60 | 均衡档 |
| gpt-4o-mini | $0.15 | $0.075 | $0.60 | 意图分类/路由档 |
| gpt-4.1-nano | $0.10 | $0.025 | $0.40 | 极省档 |
| 微调 gpt-4.1-mini | 训练 $5.00/1M | — | — | 定制小分类器 |

- "ChatGPT 当路由器/意图分类器" 的成本基线：一条短意图分类请求 (≈300 in + 5 out tokens, gpt-4o-mini) ≈ **$0.00005**，但延迟 100ms~1s、且无离线批量优势；对比本地 encoder (~0 边际成本, 5–30ms) 或 semantic router (~10ms)。
- 强弱价差是路由收益之源：RouteLLM 假设 GPT-4 vs Mixtral 100x；OpenAI 体系内 gpt-5 vs gpt-4o-mini 输出价差 16.7x、输入 8.3x。

### 3.2 实测速度 (✅ Artificial Analysis, 2026-09 抓取)

| 模型 | 输出速度 | AA Intelligence Index | 价格 (in/out per 1M) |
|---|---|---|---|
| gpt-4o-mini | **111.7 tok/s** (快于均值 104) | 7 | $0.15 / $0.60 |
| gpt-5 mini (high, reasoning) | 107.0 tok/s | 17 | $0.25 / $2.00 (cache 折扣 90% → $0.025) |

- 两页均已标记 deprecated (AA 转向 GPT-5.4 mini 等)，数字为停更前基线，作数量级参考。
- 级联延迟算术：弱模型先行 + 不确定升级 = 最坏情况两级延迟；RouteLLM 类 router 开销 (MF: 155 req/s) 基本无感，LLM-based router (causal LLM / Hybrid LLM 用 LLM 判难度) 开销可达 0.4s+，要计入预算。

### 3.3 置信度来源三件套 (成本从低到高)

1. encoder/typed-model 自带校准概率 (Laya `boolean` 返回校准 P(true), ECE 0.081 ✅)；
2. LLM 单 token logprobs (Cookbook ✅，top_logprobs≤5，注意"自信≠正确")；
3. self-consistency / judge (FrugalGPT 判分器 ✅，成本×N)。

---

## 4. 最佳实践 (综合推导，每条挂来源)

1. **先算流量分布再上路由**。RouteLLM 的收益全部以 "P% 质量 → x 倍省钱" 曲线表达，且 MT Bench (闲聊) 3.66x、GSM8K (数学) 仅 1.49x —— 同一 router 在不同任务分布上收益差 2.5 倍 (✅ arXiv 2406.18665 Table 6)。做法：先统计 query 难度/类型分布，再估可下放比例。
2. **路由器要喂对口数据**。Arena 偏好数据路由 MMLU 近乎随机，加 1,500 条金标 (<2% 数据) 提升 ~20%；GSM8K 需 ~$700 的 judge 增广 (✅同上)。**用你自己的历史 query + 领域金标微调 router**，别指望通用偏好数据。
3. **错路由代价不对称时用级联而非硬路由**。弱模型答错 → 升级重答 (FrugalGPT cascade, 98% 省钱追平 GPT-4 ✅)；Hybrid LLM 显示强/弱差距越大，级联省越多但质量风险越大 (大价差对 40% 省钱=10.3% 质量损失 ✅) —— 级联用"可退回"换掉"不可逆错误"。
4. **置信门控必须校准**。神经网/LLM 自信≠正确 (Cookbook 原文警告 ✅；Guo et al. 2017 ✅)；上线前温度缩放 + 按类目设阈值；Jev 类产品宣称校准但 ECE 0.246 仍偏高，自建时务必自测 ECE。
5. **给 OOS/未知留 fail-open 出口**。CLINC150 的核心教训：in-scope 好 ≠ OOS 好 (✅ D19-1131)；semantic router 返回 None、logprobs 低于门限、cascade 判分低 → 全部导流到大模型或人工 (对应 jev/05 决策表的 fail-open 行)。
6. **监控路由器本身**。监控三件套：router 决策分布 (强/弱占比漂移)、抽检 router 准确率 (对拍金标或 judge)、级联触发率/重试率；RouterBench 的存在意义就是"路由系统没有基准就没法监控" (✅ 2403.12031)。
7. **路由开销别超过收益**。MF router $1.42/百万请求、≤0.4% 生成成本 ✅；LLM-based router (0.46s+ ✅Hybrid LLM) 在高频低价值决策上不划算 —— 这正是 Jev/Laya (32.8ms ✅) 与 semantic router (10ms ✅) 的生态位。
8. **大候选池 + 好路由 > 单一最强模型**。survey 的 model-level scaling：池子越大路由越强，可超池中最佳单模型 (✅ 2503.10657)；OpenRouter auto 用"社区 7 天付费分布"免费实现近似 ✅。

---

## 5. RAG 专属查询路由 (route to retrieval strategy)

- **向量检索 vs 摘要 vs SQL**：LlamaIndex RouterQueryEngine 的标准用法 —— `QueryEngineTool` 各带 description ("useful for summarization..." / "useful for retrieving specific context...")，selector 按问题路由；官方示例含 SQL Router Query Engine (自然语言问题 → SQL 引擎 or 向量引擎) ✅ https://developers.llamaindex.ai/python/framework/module_guides/querying/router/
- **检索质量门控**：Cookbook 的 `has_sufficient_context_for_answer` + logprob 置信度 = "检索结果够不够答" 的轻量判别 (✅ Using_logprobs.ipynb)，可接在 retriever 后决定是否改写 query/换检索源/直答。
- **语义路由做 RAG 分流**：semantic-router 的 Route 即检索策略 (politics→web, chitchat→不检索直答, healthcare 7 工作流示例 ✅ GitHub)；本地 encoder 部署时为 0 API 成本。
- **LangChain**：RunnableBranch + embedding 相似度条件实现 query routing；重型条件编排官方建议 LangGraph ✅。
- 组合建议：第一层 semantic router/SetFit (ms 级) 定检索策略与 OOS；第二层可选 RouteLLM/Hybrid LLM 定生成模型；两层都用 logprobs/校准概率做门控。

---

## 6. 来源清单 (全部)

| # | 来源 | URL | 状态 |
|---|---|---|---|
| 1 | SetFit 论文摘要 | arxiv.org/abs/2209.11055 | ✅ |
| 2 | SetFit 全文 (ar5iv) | ar5iv.labs.arxiv.org/html/2209.11055 | ✅ |
| 3 | SetFit HF 博客 | huggingface.co/blog/setfit | ✅ |
| 4 | RouteLLM 摘要 | arxiv.org/abs/2406.18665 | ✅ |
| 5 | RouteLLM 全文 v2 | arxiv.org/html/2406.18665v2 | ✅ |
| 6 | RouteLLM GitHub README | github.com/lm-sys/RouteLLM | ✅ |
| 7 | FrugalGPT 摘要 | arxiv.org/abs/2305.05176 | ✅ |
| 8 | Hybrid LLM 摘要 | arxiv.org/abs/2404.14618 | ✅ |
| 9 | Hybrid LLM 全文 (ar5iv) | ar5iv.labs.arxiv.org/html/2404.14618 | ✅ |
| 10 | CLINC150 论文页 (D19-1131) | aclanthology.org/D19-1131/ | ✅ |
| 11 | CLINC150 数据 repo | github.com/clinc/oos-eval | ✅ |
| 12 | GraphRouter 摘要 | arxiv.org/abs/2410.03834 | ✅ |
| 13 | RouterBench 摘要 (正确 ID 2403.12031) | arxiv.org/abs/2403.12031 | ✅ |
| 14 | Model Routing Survey 摘要 | arxiv.org/abs/2503.10657 | ✅ |
| 15 | CARROT/SPROUT 数据集摘要 | arxiv.org/abs/2502.03261 | ✅ |
| 16 | SPROUT 碳效摘要 | arxiv.org/abs/2403.12900 | ✅ |
| 17 | 温度缩放摘要 | arxiv.org/abs/1706.04599 | ✅ |
| 18 | ModernBERT 摘要 | arxiv.org/abs/2412.13663 | ✅ |
| 19 | ModernBERT HF 博客 | huggingface.co/blog/modernbert | ✅ |
| 20 | semantic-router GitHub | github.com/aurelio-labs/semantic-router | ✅ |
| 21 | OpenAI Cookbook Using logprobs | github.com/openai/openai-cookbook/.../Using_logprobs.ipynb | ✅ |
| 22 | OpenAI 官方定价页 | platform.openai.com/docs/pricing | ✅ |
| 23 | Artificial Analysis GPT-4o-mini 页 | artificialanalysis.ai/models/gpt-4o-mini | ✅ |
| 24 | Artificial Analysis GPT-5 mini 页 | artificialanalysis.ai/models/gpt-5-mini | ✅ |
| 25 | LlamaIndex Router docs | developers.llamaindex.ai/python/framework/module_guides/querying/router/ | ✅ |
| 26 | LangChain RunnableBranch 源码 docstring | github.com/langchain-ai/langchain .../runnables/branch.py | ✅ |
| 27 | OpenRouter Auto Router 页 | openrouter.ai/auto | ✅ |
| 28 | Not Diamond 官网 | notdiamond.ai | ✅ |
| 29 | Martian 官网 (已转型) | withmartian.com | ✅ |
| 30 | Laya/Jeva 对比博客 | flowtivity.ai/blog/laya-open-source-jev-alternative | ✅ |
| 31 | OpenRouter Auto 历史由 Not Diamond 驱动；Jev 92.9% vs Laya 65.3% Reddit 基准；RouterBench 11 模型/数据集覆盖细节；Unify | (搜索摘要) | ⚠️ |
| 32 | CLINC150 名字由来、23,700 总数 | (repo+搜索综合; repo 列出分项可加总得 23,700) | ✅/⚠️ |

> 排错记录：CLINC150 论文不是 P19-1535 而是 **D19-1131** (EMNLP-IJCNLP 2019)；RouterBench 不是 2411.12626 而是 **2403.12031**；"AutoRoute" 作为独立路由论文不存在，近义工作为 CARROT/Causal LLM Routing (搜索确认 ⚠️)。
