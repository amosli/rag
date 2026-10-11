# I. 意图识别与模型路由 —— 中文/国内分册(平台实践 + 国产模型价格延迟)

> Lane: 国内平台意图识别实践 + 国产/闭源候选模型价格延迟,2026-09-27 检索。
> 标注说明:✅已验证 = 本次直接抓取页面核对原文;✅(摘要) = 搜索结果中直接引用了官方原文片段,未整页抓取;⚠️未验证 = 仅搜索结果可见、未抓原文。
> 定价月级变动,所有数字标注"截至 2026-09-27";与英文分册(H-意图识别与模型路由.md)互补,本册不重复英文论文基准。

---

## 0. TL;DR

1. **国内主流平台的"意图路由"全是同一条技术路线**:LLM 按"意图/分类描述"做单选(或多选),输出映射为工作流分支。Dify(描述式 + 指令)、FastGPT(function_call 枚举)、百炼(意图描述 + 单/多选 + 思考模式)、扣子(示例话术匹配)。选哪家差别不大,差别在**分类判定用什么机制、要不要模型支持工具调用**。
2. **官方档位建议一致收敛**:Dify 官方"简单分类用速度较快的模型即可;分类边界细微时,更强的模型表现更好"✅;百炼官方推荐"专用意图分类模型、通义千问 Max 系列模型"✅。即:常规 RAG 分流用免费/轻量档(国内有真免费档:GLM-4-Flash、GLM-4.7-Flash),边界细微/类目多再升档。
3. **价格(截至 2026-09-27,元/百万 tokens,输入/输出)**:意图路由档位可用 GLM-4-Flash(免费)/GLM-4.7-Flash(免费,200K)、DeepSeek-Flash(1/4,空闲)、MiniMax-M2(2.1/8.4)、GLM-5.3-Flash(0.8/2.8, 1M);对照 OpenAI gpt-4o-mini($0.15/$0.60)。国产免费 + 低价档对"路由模型"这一用途已经过剩。
4. **延迟口径**:官方公开 TPS 的不多——MiniMax 官方标注 M2.x 标准 ~60 TPS / highspeed ~100 TPS ✅,GLM-5.3-FlashX 官方 200 tokens/s ✅;DeepSeek 不公布官方 TPS,改用高峰/空闲分时计价 + 并发限速(Flash 2500 并发)✅。三方口径(Artificial Analysis)只实抓到方法论页,逐模型数字未能核验 ⚠️。
5. **社区选型共识(⚠️中文实践文章)**:意图分类/路由是"高频小决策",用规则 + 微调小模型(或语义路由)前置分流、只把复杂 query 给大模型,混合架构比纯大模型方案单次对话成本低约 42%;微调后意图准确率可到 85%+;BERT 领域预训练能带来 +27 个百分点的分类提升。

---

## 1. 国内平台的意图识别/路由实现(官方文档优先)

### 1.1 Dify「问题分类器」(Question Classifier)✅已验证

来源(✅直抓):https://docs.dify.ai/zh-hans/guides/workflow/node/question-classifier

**机制**
- 利用大语言模型的语义理解对输入做智能分类,"替代复杂的条件逻辑",不同分类路由到不同工作流分支。
- **模型选择**:下拉选择一个 LLM 作为分类模型。官方建议(原文照录):**"简单分类用速度较快的模型即可处理;分类边界细微时,更强的模型表现更好。"**(页面不列具体支持模型名单,实际为平台已接入的所有对话模型)
- **分类定义是双字段设计**:
  - **分类描述**(编辑器正文):模型判断分支的依据,下游经 `class_name` 输出。官方编写要点(照录):"写出能准确区分该分类内容的描述。分类边界相近时,使用「与……相关」「不包括……」等边界表述有助于模型做出判断。"
  - **分类标题**(画布小标签):默认"分类 N",下游经 `class_label` 输出。
- **输入变量**:通常 `sys.query`,也可接上游任意文本变量。
- **高级设置——指令字段**:可加详细分类指南,处理边界场景/歧义/业务规则,"这有助于大语言模型理解分类之间的细微差别"。本页未提及记忆/视觉(旧版有记忆配置,以现文档为准)。

**官方对描述质量的建议(客服机器人指南)**
来源(✅摘要,搜索结果直接引用 docs.dify.ai 原文):《使用知识库搭建智能客服机器人》——**"为了让分类更准确,你需要选择更优秀的 LLM,分类需要写的足够具体、区分度足够大"**,并给出"用户询问无关问题 / 用户询问和 Dify 相关问题 / 用户请求技术支持…"式的参考分类。URL:https://docs.dify.ai/zh-hans/guides/application-orchestrate/chatbot-application(标题页,内容见文档"知识库"章节)

**已知坑(⚠️社区实践,未逐一核原文)**
- 知乎《Dify 分类总翻车》:只写类别名(如"售后服务")模型只能靠猜;应在分类描述里**枚举该类别下所有典型问法**。
- 博客园(2025-05)分析问题分类器实际发出的提示词,发现**格式污染、冗余内容**干扰分类效果;有 GitHub 讨论认为该节点不够好用,社区改用 **LLM 节点自定义提示词**做分类替代。
- CSDN(2025-09):把分类提示词当"小型专家系统"设计——先讲分类目的,再给互斥边界定义。
- 中文分类不稳定的根因与英文一致:描述写得糊 + 模型档位低;平台机制本身无中文特调(官方文档为 AI 翻译,建议对照英文版)。

### 1.2 Coze(扣子)意图识别节点

**官方定义(✅摘要,docs.coze.cn 原文片段)**:"低代码工作流中的**意图识别节点**能够让智能体识别用户输入的意图,并将不同的意图流转至工作流不同的分支处理,提高用户体验,增强智能体的落地效果。" 官方文档站:https://docs.coze.cn (注:coze.cn 全站为 SPA,本次多次尝试无法整页抓取,以下配置细节标 ⚠️)

**配置细节(⚠️来自第三方教程交叉印证:zengqueling.com《扣子:意图识别节点》、53ai《扣子意图识别实战》、知乎/CSDN 教程)**
- 输入参数默认 `query`(通常连开始节点的用户输入)。
- 可**添加多个自定义意图**,每个意图配置**示例话术**(多条例句),系统按输入与示例话术的匹配程度流转;每个意图即一个分支出口。
- 有**默认分支("其他意图")**,未命中任何意图时走兜底(可接兜底回复)。
- 官方案例(docs.coze.cn《通过语音控制设备》,✅摘要):对话流里配"意图识别 + 输出"节点,识别到"调节音量"等设备控制意图时,扣子编程向设备发送 `conversation.message.completed` 事件。
- 官方文档另有"大模型节点 + 选择器(条件分支)"组合实现路由的通用做法(意图识别节点自带分支,一般无需再接选择器)。

### 1.3 FastGPT「问题分类」节点 ✅已验证

来源(✅直抓):https://doc.fastgpt.io/docs/introduction/guide/dashboard/workflow/question_classify/

**机制与配置项(原文照录)**
- 分类机制是 **function_call**:"配置各个分类条目(如:打招呼、计费常见问题、其他问题)。系统将其转为 function_call 形式的 Function,模型返回对应的枚举值从而实现分类。"——即生成一个带 `enum` 枚举值的参数(如 `type`),模型必返回其中一个枚举值。
- 配置项:**系统提示词**("被放置在对话最前面,可用于补充说明分类内容的定义")、**聊天记录**("适当增加一些聊天记录,可以联系上下文进行分类")、**用户问题**、**分类内容**(多条目)。
- 节点标注:"可重复添加 / 有外部输入 / 需要手动配置 / 触发执行 / function_call 模块"。
- 官方自认局限(照录):**"在一些较模糊的场景中,分类效果不是很明显。"**

**与 Dify 的机制差异(本册归纳)**:FastGPT 靠 function calling 枚举(要求所选模型支持工具调用,输出天然结构化);Dify/百炼/扣子靠描述文本 + LLM 自由判断(模型无关,但输出解析更依赖提示词质量)。

### 1.4 阿里云百炼「意图分类节点」✅已验证(官方文档最完整)

来源(✅直抓):https://docs.agent.bailian.aliyun.com/zh/flow-agent/applications/workflow/intent-node.md (Flow Agent 工作流节点,索引见 /llms.txt)

**机制**:基于"意图描述智能分类匹配",由模型判断输入最匹配哪个意图并路由到对应链路。**默认自带"其他意图"分支**:"当模型判断输入的所有意图分类均不满足时执行该分支。"

**配置项(照录)**
| 配置项 | 说明 |
|---|---|
| 输入变量(必填) | 用于意图分类判断的输入文本或变量引用 |
| 模型配置 | **"支持专用意图分类模型、通义千问系列模型、第三方模型(如 DeepSeek),实际支持的模型请以控制台为准"**;默认参数 top_p 0.80 / temperature 0.70 / 最长回复 1024 / enable_thinking 开启 / thinking_budget 4000 |
| 意图分类 | "添加意图"新增,**最多支持 60 个**,含意图名称 + 意图描述("模型将根据不同的意图描述匹配后续链路") |
| 意图模式 | **单选模式(默认)**挑"最匹配的一个";**多选模式**挑"所有匹配的"(并行执行所有命中分支) |
| 思考模式 | 节点级配置:快速模式(默认,`thought` 为空)/ 效果模式(含推理细节);与模型参数 enable_thinking 是两回事 |
| 记忆 | 默认关闭;本节点缓存 1–50 轮(默认 3)或自定义(如 historyList) |
| 提示词 | 非必填,"可提供分类案例与限制条件" |

**输出变量**:单选 `result.subject`(命中意图名)/`result.thought`;多选 `result.subjects`(数组)。

**官方模型档位建议(照录)**:推荐 **"意图分类模型、通义千问 Max 系列模型"**;复杂场景用效果模式,"简单/实时性场景用快速模式"。

**官方提示的坑**:① 意图描述要"避免重叠和歧义",提示词给分类案例;② **多分支下游易收到空值**——多选/未命中时分支可能不执行,用"变量处理节点的聚合分组模式"兜住,避免报错。

**百炼的确定性路由补位**:"条件判断节点"(conditional-node,规则式 if-else)与"意图分类节点"(模型驱动语义路由)分工;另有"智能体群组节点"内部用**决策模型**做子智能体调度(✅索引可见)。老版智能体应用/工作流的"路由器"节点在新版 Agent Studio 中由这两者承接。

### 1.5 Spring AI Alibaba / Eino(代码侧路由模式)

- **Spring AI Alibaba Graph**(参考 LangGraph 的 Java 实现):条件路由本质是**"有向图的条件边"**——Graph 引擎根据状态动态选择下一节点,适合意图分流(库存→查询链路、优惠→规则链路、退款→售后链路)(⚠️ smzdm《Spring AI Alibaba 多 Agent 协作:6 种模式怎么选》;CSDN《Spring AI Alibaba Graph 全解析》https://blog.csdn.net/BADAO_LIUMANG_QIZHI/article/details/161928620)。实战案例:ChatFlow 智能待办助手,具备多轮对话、**意图识别和子图调用**(⚠️ gitcode.csdn)。选型观点(⚠️ quanxiaoha.com):Graph 适合"强可控的图结构工作流","先单 Agent,再 Workflow,最后多 Agent"。
- **Eino(CloudWeGo,字节 Go 框架)**:`compose` 包提供 Chain(链)/Graph(图)两种编排;**BranchNode 内部包含一个函数,根据输入动态返回下一个应执行的节点名称**,实现条件路由;典型用法是自定义判断函数按用户问题选路径(简单常识问题直答,专业问题进检索 Agent)(⚠️ 掘金《使用 Eino 和 Ollama 构建智能 Go 应用》、golangstar.cn、segmentfault)。官方文档站 cloudwego.io/zh/docs/eino/(本次 404 未抓到 Branch 专页,标注未验证)。
- CSDN 亦有《基于 Spring AI Alibaba 的自校正 RAG(LangGraph Custom RAG 架构)》——把"是否需要重检索/换路"做成图上的判定节点(⚠️ 2026-04)。

### 1.6 LangChain/LangGraph 中文社区的意图路由实践

- **LangChain 侧**:中文教程常用 `RouterChain`/`MultiRouteChain`(LLM 路由)或引入 `semantic-router` 库做 embedding 相似度零样本路由;判断 query 走向量知识库还是 Text-to-SQL 再分链(⚠️ CSDN《LLM RAG 系列:一文详解 RAG》"路由与请求构造"章节;GitHub `rag-summary-and-practice` 含查询重写/拆解/路由实践)。
- **LlamaIndex 侧**:`RouterQueryEngine` 基于嵌入相似度自动路由("查向量库/查知识图谱/调外部 API"),有社区文章称在 10 万页 PDF 场景路由准确率较高(⚠️ CSDN 2026-09,无具体数字原文可核);官方中文站 docs.llamaindex.org.cn 有 Text-to-SQL 高级工作流(知识库 + SQL 混合路由)(⚠️)。
- **痛点清单**:英伟达架构师《细数 RAG 的 12 个痛点及解决方案》(51CTO 中文版,2024-07)把"路由"列为核心痛点之一——不该进知识库的 query 硬检索是首因。
- 混合架构观点(⚠️ CSDN/wenku):框架层做粗路由(安全/权限,稳定可靠),LLM/embedding 做细路由(知识库 vs SQL vs API),"不能只靠静态规则"。

### 1.7 微调小模型做中文意图识别的社区实践(有数字)

均为**社区实践数据**(⚠️,未复现):
- **规则 + NLU 混合 vs 纯大模型**(腾讯云开发者社区/SegmentFault,2026-07,《规则引擎+NLU 混合架构 vs 纯大模型方案:单次对话成本差 42%》):微调后**意图准确率提升至 85% 以上**;分层路由——简单问题走规则引擎、长尾走 NLU 小模型、复杂问题才调大模型;**单次对话成本相差 42%**(纯大模型更贵);"调用云端大模型 API 的往返延迟通常在秒级,直接全量调用将导致不可控的成本支出"。
- **BERT 领域预训练**(CSDN 论坛实测帖):BERT-base 直接微调电商意图效果不佳,换电商领域语料预训练版本后**准确率 +27 个百分点**——领域预训练对中文意图分类很关键。
- **Qwen 小模型 LoRA**(知乎):Qwen2-0.5B/1.5B 中文能力强,LoRA 微调显存需求极低、单张消费级显卡可训,适合中文语境意图/情绪等小任务;DB-GPT 微调框架提供 spider 数据集(文本→SQL)准确率实测,可用于"query 路由到 SQL"这一支的微调参照(GitHub)。
- 对照英文分册 H 的结论:SetFit(8~64 条/类)与微调 encoder 在 CLINC150/RAFT 上的数字,中文场景可用 CLUE 类基准(IFLYTEK 等)自行复测,本册未找到权威中文意图基准的公开 SOTA 表(如有请补)。

---

## 2. 候选模型做"意图识别/路由"的价格与延迟(官方定价页实抓,截至 2026-09-27)

### 2.1 定价对比总表

单位:元/百万 tokens(输入/输出),括号内为缓存命中输入价;"$"为美元;上下文=窗口。

| 模型 | 输入 | 输出 | 上下文 | 免费档 | 截至 | 来源 |
|---|---|---|---|---|---|---|
| **智谱 GLM-4-Flash-250414** | 0 | 0 | 128K | ✅免费 | 2026-09-27 | ✅ docs.bigmodel.cn/cn/guide/start/pricing |
| **智谱 GLM-4.7-Flash** | 0 | 0 | 200K | ✅免费 | 2026-09-27 | ✅ 同上(GLM-4.5-Flash 下线后自动路由至此,⚠️ GitHub "Free LLM Intel") |
| 智谱 GLM-4-FlashX-250414 | 0.1 | 0.1 | 128K | - | 同上 | ✅ 同上 |
| 智谱 GLM-4-Air-250414 | 0.5 | 0.5 | 128K | - | 同上 | ✅ 同上(缓存命中 0.25) |
| 智谱 GLM-4-Long | 1 | 1 | 1M | - | 同上 | ✅ 同上 |
| 智谱 GLM-5.3-Flash | 0.8 | 2.8 | 1M | - | 同上 | ✅ 同上(缓存命中 0.23;限时折扣约为 GLM-5.3 的 1/20,✅模型页) |
| 智谱 GLM-5.3-FlashX | 2 | 7 | 1M | - | 同上 | ✅ 同上(官方 200 tokens/s,✅模型页) |
| 智谱 GLM-5 | 4 / 6 | 18 / 22 | (未标注,分段按输入 <32K / ≥32K) | - | 同上 | ✅ 同上 |
| 智谱 GLM-5.2 / GLM-5.3 | 8 | 28 | 1M | - | 同上 | ✅ 同上(缓存命中 2) |
| 智谱 GLM-4.6 | 无按量价 | — | — | - | 同上 | ✅ 仅私有实例计价("200K-fp8,175 元/算力单元/天") |
| **DeepSeek deepseek-flash(V4.1-Flash)** | 1(命中 0.02) | 4 | 1M,输出上限 384K | - | 同上 | ✅ api-docs.deepseek.com/zh-cn/quick_start/pricing(高峰×2:输入 2/输出 8) |
| DeepSeek deepseek-v4-pro | 4.5(命中 0.15) | 13.5 | 1M | - | 同上 | ✅ 同上(高峰×2) |
| **MiniMax-M2** | 2.1(读 0.21) | 8.4 | 204.8K | - | 同上 | ✅ platform.minimax.cn/docs/guides/pricing-paygo |
| MiniMax-M2.5 / M2.7 | 2.1 | 8.4 | 204.8K | - | 同上 | ✅ 同上(M2.5 缓存读 0.21;M2.7 读 0.42) |
| MiniMax-M2.x-highspeed | 4.2 | 16.8 | 204.8K | - | 同上 | ✅ 同上(官方 ~100 TPS vs 标准 ~60 TPS,✅API 页) |
| MiniMax-M3(≤512K 输入) | 2.1(永久五折,原 4.2) | 8.4 | 1M | - | 同上 | ✅ 同上(>512K 档输入 4.2/输出 16.8;priority 档 = 标准价 1.5 倍) |
| MiniMax-M1 / Text-01 | 已不在国内价目表 | — | — | - | 同上 | ✅(国内平台按量列表无;历史型号) |
| **OpenAI gpt-4o-mini** | $0.15 | $0.60 | (页未标注) | - | 同上 | ✅ developers.openai.com/api/docs/pricing(缓存输入 $0.075) |
| OpenAI gpt-5-nano | $0.05 | $0.40 | (页未标注) | - | 同上 | ✅ 同上(缓存输入 $0.005) |
| OpenAI gpt-5-mini | $0.25 | $2.00 | (页未标注) | - | 同上 | ✅ 同上(缓存输入 $0.025) |
| OpenAI gpt-5 | $1.25 | $10.00 | <272K 档标注 | - | 同上 | ✅ 同上 |
| OpenAI gpt-6-luna(新小旗舰) | $0.10 | $0.50 | - | - | 同上 | ✅ 同上(长上下文 $0.20/$0.75) |

**通用折扣**:智谱 Batch API 标准价 5 折 ✅;OpenAI Batch 半价 ✅;MiniMax `service_tier=priority` 1.5 倍换优先处理 ✅。

### 2.2 延迟口径(能核实什么、核实不到什么)

- **官方公开 TPS**:
  - MiniMax:标准档 **~60 TPS**,highspeed 档 **~100 TPS**(✅ platform.minimax.cn/docs/api-reference/text-anthropic-api,原文"输出约 60 TPS"/"约 100 TPS");"highspeed 为极速版,效果不变,更快更敏捷"。
  - 智谱:GLM-5.3-FlashX 官方标注 **200 tokens/s**(✅ docs.bigmodel.cn GLM-5.3-Flash 模型页);另有 GLM-4-AirX 为 8K 上下文的加速档(10 元/百万)。
- **DeepSeek 不公布官方 TPS/TTF**✅(文档首页无限速表):改用**分时定价**——高峰(工作日 9:00–12:00、14:00–18:00)价格为空闲 2 倍 ✅,并发限速模型("一个请求从发出到响应完成记为一个并发"):deepseek-flash **2500 并发**、deepseek-v4-pro **500 并发**,按账号计,扩容工单免费 ✅(api-docs.deepseek.com/zh-cn/quick_start/rate_limit);服务状态页 status.deepseek.com。⚠️ 社区实测口径(自建/CPU 部署)多为个位数~两位数 TPS,与官方 API 不可比,不建议引用。
- **三方口径(Artificial Analysis,⚠️未核到逐模型数字)**:AA 以 output speed(TPS)、latency(TTFT)作为核心指标(✅ artificialanalysis.ai/models 方法论页:"quality, price, output speed, latency, context window"),但模型明细页为 JS 渲染,本次仅抓到方法论;**引用具体 TPS 对比时必须标注三方口径与抓取日期**。
- **路由链路实际延迟的构成(归纳)**:意图分类 = 一次 LLM 往返,延迟 ≈ TTFT + 输出(十几个 token)时间。用百炼意图分类节点时官方建议简单场景用"快速模式"(`thought` 为空,省掉推理输出)✅;FastGPT 的 function_call 枚举输出极短,主要等 TTFT。

### 2.3 意图路由用途的成本粗算(社区常用口径)

设一次分类:输入 ~500 tokens(系统提示 + 意图描述 + query)、输出 ~10 tokens:

| 方案 | 单次成本 | 每万次 | 说明 |
|---|---|---|---|
| GLM-4-Flash / GLM-4.7-Flash | 0 元 | 0 元 | ✅官方免费档;国内路由首选试用档 |
| GLM-5.3-Flash | ≈0.0004 元 | ≈4.2 元 | 0.8×0.0005 + 2.8×0.00001 |
| DeepSeek-flash(空闲) | ≈0.0005 元 | ≈5.4 元 | 1×0.0005 + 4×0.00001;高峰 ×2 |
| MiniMax-M2 | ≈0.0011 元 | ≈11.3 元 | 2.1×0.0005 + 8.4×0.00001 |
| gpt-4o-mini | ≈$0.000081(≈0.0006 元) | ≈5.8 元 | $0.15×0.0005 + $0.6×0.00001 |

结论量级:免费档之外,各家轻量模型做一次路由都在**万分之几元**级,差异在 2–3 倍以内;**真正的分水岭是"要不要免费/要不要私有化"与"边界细微类目要不要上强档"**。

### 2.4 国内可访问性/合规注意(⚠️一般性说明,未核原文)

- OpenAI API 官方不面向中国大陆提供服务(区域支持列表),直连需自备合规出境网络;生产系统通常改用 Azure OpenAI(企业协议)或国产模型替代。涉密/信创场景直接排除境外 API。
- 国产三家(bigmodel.cn / api.deepseek.com / minimax.cn 平台)均支持国内直连、对公充值与实名;智谱/MiniMax 另有 Coding Plan/Token Plan 订阅制(打包价更低,适合高频路由流量)⚠️。

---

## 3. 选型讨论素材(中文社区/厂商文章)

### 3.1 "大模型做意图识别 vs 小模型微调"的中文讨论

- 腾讯云开发者社区(2026-07)《规则引擎+NLU 混合架构 vs 纯大模型方案:单次对话成本差 42%》⚠️:核心论据见 1.7;结论是分层(规则→NLU→大模型)而非二选一。
- CSDN 论坛 BERT 领域预训练 +27pp 实测帖 ⚠️:说明"中文意图分类拼的是领域语料,不是模型大小"。
- 知乎 Qwen2-0.5B/1.5B LoRA 微调帖 ⚠️:小模型单卡可训,适合把意图分类做成私有组件。
- 对照英文一手分册(H):SetFit N=8 胜普通全量微调 19.3 分、RAFT 上 355M 模型超 GPT-3 175B 8.6 分、ModernBERT 比 DeBERTa 快 ~2 倍——中文社区的说法与英文基准一致:**定型类目上微调小模型精度/成本双优;LLM 路由的价值在"类目常变、零标注、冷启动"**。
- 火山引擎开发者社区(2025-08)⚠️:提高 Dify 类分类准确率的进阶手段——增加领域训练数据或微调预训练模型;分类后接条件分支路由到不同知识库,避免无关信息干扰(RAG 场景典型)。

### 3.2 RAG 里的查询路由(query → 知识库/SQL/Web)中文实践

- CSDN《LLM RAG 系列:一文详解 RAG》(2024-11)⚠️:"路由与请求构造"章节系统列了 semantic routing(按 embedding 相似度选源)与 logical routing(LLM 结构化输出选源)两条线。
- LlamaIndex 中文站(docs.llamaindex.org.cn)Text-to-SQL 高级工作流 ⚠️:SQL 查询引擎 + RAG 引擎由 RouterQueryEngine 分发,是"知识库 + SQL"混合路由的官方参照实现。
- GitHub `rag-summary-and-practice` ⚠️:本地(Ollama)复现查询重写/拆解/路由的中文 notebook。
- 知乎《大模型 RAG 有哪些好用的技巧?》⚠️:生产经验讨论——路由 + 混合检索 + 重排是前处理三件套。
- 英伟达《RAG 12 痛点》中文版(51CTO)⚠️:痛点 7~9(路由/微调/ Related)——"不该进知识库的 query 进了知识库"是检索质量差的常见首因。

### 3.3 "路由模型本身该多大/多快"的讨论

- Dify 官方(✅):**简单分类用快模型;边界细微用强模型**——按"类目边界是否细微"选档,而不是无脑上旗舰。
- 百炼官方(✅):推荐"专用意图分类模型、通义 Max 系列";**节点级"快速/效果"思考模式**是官方给的延迟旋钮(快速模式 thought 为空)。
- MiniMax 官方(✅):同模型给"标准/highspeed"两档(60→100 TPS,价格 2→4.2 元输入),等于官方承认"路由类高频小决策值得买速度"。
- 社区(⚠️腾讯云/segmentfault 42% 文):路由模型应该"快且便宜到可以忽略",单次对话成本主要省在"分流掉了多少本该进大模型的 query"——这比路由模型本身的档次更重要。

---

## 4. 核验清单(本次实抓 12 + 摘要引用 3 + 社区未验证 10+)

**✅ 官方直抓(12)**
1. docs.dify.ai/zh-hans/guides/workflow/node/question-classifier(Dify 问题分类器机制)
2. doc.fastgpt.io/.../question_classify/(FastGPT function_call 分类)
3. docs.agent.bailian.aliyun.com/zh/flow-agent/applications/workflow/intent-node.md(百炼意图分类节点)
4. docs.agent.bailian.aliyun.com/zh/flow-agent(工作流应用总览)+ /llms.txt(节点索引)
5. docs.agent.bailian.aliyun.com(Managed Agent 概览,确认 Flow Agent/Agent Studio 结构)
6. api-docs.deepseek.com/zh-cn/quick_start/pricing(DeepSeek V4 定价,高峰/空闲)
7. api-docs.deepseek.com/zh-cn/quick_start/rate_limit(并发限速 2500/500)
8. api-docs.deepseek.com/zh-cn/(模型名映射:旧名按 Flash 计价;status 页)
9. platform.minimax.cn/docs/guides/pricing-paygo(MiniMax 全量定价)
10. platform.minimax.cn/docs/api-reference/text-anthropic-api(M3/M2.7/highspeed 上下文与 TPS)
11. docs.bigmodel.cn/cn/guide/start/pricing(GLM 全表,含免费模型列表)
12. docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash(GLM-5.3-Flash 规格)
13. developers.openai.com/api/docs/pricing(OpenAI 全量定价)

**✅(摘要)官方原文片段(3)**
14. docs.coze.cn 意图识别节点定义句(搜索结果直接引用)
15. docs.dify.ai《使用知识库搭建智能客服机器人》分类建议句
16. docs.coze.cn《通过语音控制设备》意图识别节点事件用例

**⚠️ 社区/第三方(未核原文,引用需注明)**
17. zengqueling.com《扣子:意图识别节点》;18. 53ai 扣子意图识别实战;19. 知乎《Dify 分类总翻车》;20. 博客园 Dify 分类器提示词格式污染分析;21. CSDN《如何让 dify 分类器更精准》;22. 腾讯云开发者社区 42% 成本文;23. CSDN BERT 领域预训练 +27pp;24. 知乎 Qwen2 小模型 LoRA;25. CSDN/golangstar/掘金 Eino Branch;26. CSDN/smzdm Spring AI Alibaba Graph 条件路由;27. CSDN LlamaIndex RouterQueryEngine 10 万页 PDF;28. 51CTO RAG 12 痛点;29. GitHub rag-summary-and-practice;30. GitHub "Free LLM Intel"(GLM-4.5-Flash 下线路由);31. artificialanalysis.ai 方法论页(逐模型数字未核)

**未能核验/缺口**:① Artificial Analysis 逐模型 TPS/延迟数字(JS 渲染);② 扣子意图识别节点官方配置页全文(SPA 反爬);③ DeepSeek 官方 TPS(官方不发布);④ 权威中文意图分类基准的公开 SOTA 表;⑤ GLM-5 上下文长度(定价页未标注,勿引用猜测值)。
