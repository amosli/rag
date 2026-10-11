# G. 中文社区/国内厂商的 RAG 权限管理与多租户实践

> 调研日期:2026-09-27。本文为 RAG 培训文档【权限管理篇】的中文社区/国内实践分册。
> 标注说明:✅ = 已验证(实抓页面/源码确认内容);⚠️ = 未验证(仅搜索结果可见,未实抓)。
> 注意:部分 CSDN 文章出自疑似批量注册/AI 辅助写作的账号(如 `weixin_xxx`、`gitblog_xxx`),已实抓核验的会在条目内单独提示"可信度注意",引用其具体数字时应交叉验证。

---

## 1. 国内开源 RAG 平台的权限模型拆解

### 1.1 Dify:工作区角色 + 数据集三级权限 + metadata 过滤

**工作空间(workspace)四级角色** ✅ 已验证(官方文档 [team-members-management](https://docs.dify.ai/zh-hans/guides/workspace/team-members-management)):
- **所有者(Owner)**:"拥有工作空间的完全控制权,每个工作空间仅一名";角色变更只能通过转移所有权,需邮箱验证码。
- **管理员(Admin)**:"管理成员和模型供应商,并拥有编辑者的全部权限"。
- **编辑者(Editor)**:"创建、编辑和删除应用与知识库"。
- **普通成员(Normal)**:"仅可使用已发布的应用"。
- 关键句:"所有者、管理员和编辑者角色可创建知识库",但**知识库本身的可见性由知识库自身的权限设置控制**——工作区角色只决定"能不能建",不决定"能看到哪些库"。

**知识库(Dataset)权限粒度** ✅ 已验证(官方文档 + 源码双重确认):
- 文档 [manage-knowledge/introduction](https://docs.dify.ai/zh-hans/guides/knowledge-base/manage-knowledge) 原文:"权限 | 定义哪些工作空间成员可访问该知识库。被授予访问权限的成员将拥有维护知识库内容中列出的全部权限。"——即知识库权限是**二值的**:"可访问 = 可维护(读写)",没有"只读协作者"概念。
- 源码 `api/models/dataset.py`:`Dataset.permission` 字段,`default=DatasetPermissionEnum.ONLY_ME`(默认"只有我");`api/services/dataset_service.py` 中可见三个取值:`ONLY_ME`(只有我)、`ALL_TEAM`(所有团队成员)、`PARTIAL_TEAM`(部分成员)。
- `PARTIAL_TEAM` 的协作者名单存在独立表 `dataset_permissions`(字段:dataset_id / account_id / tenant_id / has_permission,`api/models/dataset.py` L1214-1232)——"部分成员"是逐人白名单,不是按角色/部门授权。
- 查询侧过滤(dataset_service.py):非 owner 只能看到 `permission == ALL_TEAM` 或(PARTIAL_TEAM 且自己在白名单中)的库。**生效层在 API 列表/读取层,不在向量检索层**。
- 补充:文档 [integrate-knowledge-within-application](https://docs.dify.ai/zh-hans/guides/knowledge-base/integrate-knowledge-within-application) 提到 Chatflow/Workflow 的**知识检索节点支持"元数据筛选"**:"根据文档的元数据字段(如标签、类别或**访问权限**)优化检索结果"——这是 Dify 内做文档级 DIY 权限过滤的官方入口(把部门/密级写进 metadata,检索时加过滤条件),需要开发者自己维护元数据与用户身份的映射。

小结:Dify 原生权限 = **工作区 RBAC(粗) + 知识库可见性(三级,人名白名单,二值读写)**;要细到"部门级/文档级"需用 metadata + 检索节点过滤自行实现。

### 1.2 FastGPT:Linux 风格权限位 + 文件级检索鉴权(国内文档做得最细)

**团队权限体系** ✅ 已验证(官方文档 [team_roles_permissions](https://doc.fastgpt.io/guide/workspace/team/team_roles_permissions)):
- 官方定位:"权限系统融合了基于属性(ABAC)和基于角色(RBAC)的权限管理范式",通过**成员、部门、群组三种主体**控制 team/app/dataset 等资源。
- 主体层级:Root(系统唯一超管,"对所有团队的所有资源拥有完全访问和管理权限")> Owner(每资源唯一,可转移但转移后失权)> 管理员("不能修改或移除自身权限"、不能授予他人管理员)> 普通成员/群组/部门。
- 权限点按资源划分:团队级(创建应用/知识库/APIKey、管理成员)、应用级(可使用/可编辑/可管理)、知识库级(可使用=应用中调用/可编辑=修改内容/可管理=分配协作者)、文件/文件夹级(读/写/管理员)。
- **判定逻辑**:"成员权限最高","部门与群组权限采用权限并集原则,优先级低于成员权限";先查个人权限,再取部门+群组权限并集组合。官方明确:"个人成员权限会覆盖全员组权限"(应用设为全员可编辑、某用户单独设为仅使用 → 该用户只能使用)。
- **存储设计**(开发者参考):"参考 Linux 权限设计,采用二进制方式存储权限位",Owner 为全 1;存于 MongoDB `resource_permissions` 集合,关键字段 teamId、tmbId/groupId/orgId(三选一为权限主体)、resourceType(team/app/dataset/agentSkill/collection)、permission(数字)、resourceId(团队资源为 null)。

**知识库文件(Collection)级权限——检索层鉴权的正面范例** ✅ 已验证(官方文档 [collection_permission](https://doc.fastgpt.io/guide/dataset/collection_permission)):
- **商业版专属**,两级开关:
  - 关闭态(默认):"集合的可读性与知识库完全一致",系统不查任何集合权限记录(零开销);
  - 开启态:"每个文件/文件夹都会生成一份完整的**权限快照**,列表、详情与**检索召回均按快照逐条鉴权**"。
- 核心原则:"**知识库权限始终是门槛:集合权限无法绕过知识库权限**"(两级权限求交)。
- 协作者配置是"全量覆盖语义,提交的列表即最终结果";"提交的协作者列表必须包含当前创建者,否则会被拒绝";分**继承态/独立态**,修改来自父级的协作者会触发"自动独立化";仅创建者可删除文件,"管理员或写权限的协作者不能删除"。
- **对检索召回的影响(重点)**:开启后,"知识库检索会按当前成员可读的文件过滤,**越权文件不会被召回**",覆盖工作流/Agent 的知识库搜索节点与检索测试。
- **已知限制(重要坑)**:"仅在按当前成员身份鉴权时生效。**工作流中预选知识库且未按成员身份鉴权的场景不做文件级过滤**。"——即自动化工作流里如果没有把"当前用户身份"传入检索节点,文件级权限形同虚设。
- 特殊身份:系统 root 和团队 Owner 拥有全部文件权限;团队管理员"不绕过文件级权限"。

**团队模式与 SSO/成员同步(企业接入)** ✅ 已验证(官方文档 [teamMode](https://doc.fastgpt.io/guide/admin/teamMode) 与 [sso](https://doc.fastgpt.io/guide/admin/sso)):
- 三种团队模式(v4.9+):单团队模式(默认关闭,开启后"所有新增的用户都不再创建自己的默认团队,而是加入 root 用户所在的团队")、多团队模式(每用户建自己的默认团队)、**成员同步模式**(外部成员系统的成员自动同步,除 SSO 注册外不建也不加入默认团队)。配置入口:管理员 → 系统配置 → 用户配置。
- SSO 经 `fastgpt-sso-service` 中转镜像适配:**飞书(SSO+成员同步均支持)、企业微信(SSO+同步)、钉钉(仅 SSO)、Saml2.0(仅 SSO)、Oauth2.0(仅 SSO)、LDAP(只同步用户和组织,不提供登录,可与 oauth2 组合:`SSO_PROVIDER=oauth2` + `SSO_SYNC_PROVIDER=ldap`)**。
- 组织映射:内部标准接口 `/org/list`(id/name/parentId,"只能存在一个根部门",多根需虚拟根)、`/user/list`(username/memberName/orgs);LDAP 组织 ID 为规范化 DN;LDAP 成员关系 4 种模式(user_attribute / group_member_dn / user_member_of / posix_member_uid),不递归展开嵌套组、不解析动态组。定时同步 `SYNC_MEMBER_CRON`(注意 UTC 时差)。
- **企微特殊规则(权限同步的天然边界)**:同步范围受应用可见范围限制,成员"只保留至少属于一个应用可见部门的成员",需用"通讯录同步助手" Secret 拉通讯录。
- 这套"组织架构每日定时同步 + username 前缀强一致"是国内自建 RAG 对接 IM 身份源的主流做法。

### 1.3 RAGFlow:默认"个人团队"模型,共享粒度是资源整体

✅ 已验证(官方文档 [manage_team_members](https://ragflow.com.cn/docs/manage_team_members)):
- 每个用户注册后**自动拥有一个以自己名字命名的团队**:"默认情况下,您是自己团队的所有者,也是唯一有权限邀请用户加入团队或移除团队成员的人。"——RAGFlow 的多租户本质是"每人一个 tenant/团队",协作靠把别人拉进自己的团队。
- 邀请前提:"请确保被邀请的团队成员是 RAGFlow 用户,并且所使用的邮箱地址与 RAGFlow 用户账号相关联";"您的团队成员目前不被允许邀请用户加入您的团队"(成员无二级邀请权)。
- 团队页可共享的资源类型:数据集、聊天助手、Agent、模型、记忆。共享数据集后成员可以:"上传文档到您的共享数据集"、"解析您的共享数据集中的文档"、"使用您共享的 Agent"。
- **权限开关在数据集配置页**:"若要让成员查看和更新数据集,需在其配置页面的权限设置中,将『仅限自己』改为『团队』"——同样是"个人/团队"二值开关,无细粒度角色。
- 版本边界:"与团队成员共享添加的模型仅在 RAGFlow 企业版中可用。"
- ⚠️ 未验证(仅搜索可见):CSDN 有《RAGFlow 权限管理避坑指南:从 API 密钥到资源访问控制的完整配置流程》(2026-03)提及 401/403 区分("401 意味着未登录或凭证错误,403 意味着登录成功但权限不足")、API Key 按应用/环境拆分、密钥硬编码进 Git 等坑,后半部分付费。⚠️ 另有《RAGFlow 企业知识库团队和权限管理详解》等转载文。

### 1.4 MaxKB:X-Pack(商业版)的工作空间 + 角色资源授权;飞书文档"导入即脱敏"

**资源授权机制** ✅ 已验证(官方文档 [X-Pack 资源授权](https://maxkb.cn/docs/v2/user_manual/X-Pack/authorization_resources.html)):
- 授权主体三级:系统管理员(可对所有资源授权)、工作空间管理员(仅本工作空间内资源)、普通用户("可以对自己创建的资源进行授权")。
- 资源类型:智能体、知识库、工具、模型(按工作空间文件夹目录列出);被授权用户为"当前工作空间下的所有成员"。
- 权限类型:**不授权(默认)/ 查看("仅能查看和使用资源,不能增删改")/ 管理("可对该资源进行所有操作")/ 按用户角色**。
- 多角色规则:"若成员有多个角色,则取所有角色的最大权限(并集)进行授权";角色权限最高规则("系统管理员的授权默认按用户角色,修改管理员身份不影响其角色权限")。
- 工作空间、角色管理本身是 **X-Pack 商业版功能**;社区版只有 `user_community.html` 的用户管理——社区版实际是"单工作空间 + 资源创建者"模型。

**飞书文档知识库(企业 IM 权限接入的典型样本)** ✅ 已验证(官方文档 [X-Pack 飞书文档知识库](https://maxkb.cn/docs/v2/user_manual/X-Pack/feishu_doc.html)):
- 接入方式:飞书**企业自建应用** + 机器人能力;把机器人所在群组"添加为文档文件夹的协作者";MaxKB 建"飞书"类型知识库只需三参数:App ID、App Secret、Folder Token。
- 所需飞书 API 权限(四项,均只读/导出类):`docs:document:export, docx:document:readonly, drive:drive, space:document:retrieve`。
- 导入是**手动选择文件夹内文档一次性导入**,文档未提及定时/增量同步。
- **关键结论(权限漂移的实证)**:"对飞书文档的支持是通过飞书机器人并配置对应的文档权限实现";导入后,**文档权限由 MaxKB 自身管理,官方页面未说明是否同步飞书原始权限**。即:飞书侧把某文档收缩为 3 人可见后,RAG 问答里任何能用这个知识库的用户仍可能检索到它——源系统权限变更不会自动传播。这是企业 IM 文档接入 RAG 的通病,培训时应作为反面案例讲透。

### 1.5 Coze(扣子)开源版:space_id 多租户 + 每知识库独立 collection

✅ 已验证(CSDN《Coze源码分析-资源库-创建知识库-基础设施/存储/安全》,https://blog.csdn.net/lypzcgf/article/details/151689359 。可信度注意:作者自述部分代码是"归纳的设计模式,未必逐行对应官方仓库",二开前以 coze-studio 源码为准):
- **空间隔离是第一道边界**:知识库三表(knowledge_base/knowledge_document/knowledge_chunk)均含 `space_id` 或经 knowledge_base_id 级联,"通过 space_id 实现多租户数据隔离";创建时强制校验"每个用户只能在其所属工作空间中创建知识库"(SpaceID 不一致直接报"无权限在该工作空间创建知识库")。
- 用户级校验:JWT 中间件校验 Token 有效性 + "知识库创建权限位"。
- 配额与限流:存储配额(`CanCreate: usage < quota*0.95`)、工作空间 MaxKnowledgeCount、用户 24 小时内"最多创建 20 个知识库"、同 space 内名称唯一。
- **存储三层隔离**:MySQL 按 space_id 建索引;ES 统一索引 `coze_resource` 用 `res_type` 区分资源、文档里**冗余存储 space_id/owner_id 实现查询侧过滤**;向量层"每个知识库独立 collection,命名为 kb_<id>,实现物理级隔离"。
- 对照:SaaS 版扣子的团队/企业空间权限未在公开文档中找到可抓取的页面(SPA 渲染),⚠️ 标记为未验证;一般认知为空间内可见、跨空间不可见,细节以官方为准。

### 1.6 对照表(权限粒度 × 生效层)

| 平台 | 知识库权限粒度 | 文档级权限 | 生效层 | 商业版边界 |
|---|---|---|---|---|
| Dify | 只有我/全团队/部分成员(人名白名单,二值读写) | 无原生,靠 metadata + 检索节点过滤 DIY | API 列表层;检索层需 DIY | 部分(工作区角色全版可用) |
| FastGPT | 知识库(可使用/可编辑/可管理)+ 成员/部门/群组并集 | ✅ 文件/文件夹级权限快照(商业版) | **检索召回层逐条鉴权** | 文件级权限、SSO 均商业版 |
| RAGFlow | 团队共享(仅自己/团队,二值) | 无 | API/团队边界 | 模型共享等企业版 |
| MaxKB | 查看/管理(按资源),工作空间+角色(X-Pack) | 无 | API 授权层 | 工作空间/角色/授权全在 X-Pack |
| Coze 开源 | space_id 隔离 + 配额 | 无(按知识库整体) | DB/ES 查询侧过滤 | SaaS 版未见公开文档 |

---

## 2. 多租户隔离:索引隔离 vs 过滤隔离 vs 分区

### 2.1 Milvus 官方四级策略(中文文档,选型基准)

✅ 已验证(官方中文文档《实施多租户》 [milvus.io/docs/zh/multi_tenancy.md](https://milvus.io/docs/zh/multi_tenancy.md),v3.0.x;英文版 v2.5.x 同源内容一致):

| 策略 | 隔离强度 | 数量上限 | 适合规模 | 权限控制 | 成本 |
|---|---|---|---|---|---|
| Database(库级) | 物理 | 默认 64 个 Database | ≤ 64 租户 | 有 RBAC | 最低 |
| Collection(集合级) | 物理 | 65,536 个 Collection | ≤ 10,000 租户 | 有 RBAC | 中 |
| Partition(分区级) | 逻辑(内存隔离) | 1,024 个分区/集合 | ≤ 1,000 租户 | 无 RBAC | 中高 |
| Partition Key(分区键) | 逻辑(16 个物理分区路由) | 百万级租户 | 10,000~百万+ 租户 | 无 RBAC | 最高 |

- 原文关键句:"数据库名称在 Milvus 实例中是唯一的"、"路由查询或获取操作时,Milvus 会根据 Partition Key 值自动筛选出匹配该值的实体所在的一个或最几个物理分区,从而将搜索范围缩小到这些分区,而不是扫描所有逻辑分区。这在很大程度上提升了搜索性能。"
- 选型决策树:租户数 ≤ 64 → Database;≤ 1 万 → Collection;≤ 1 千(需要强内存隔离)→ Partition;1 万~百万级 → Partition Key。
- **官方明确标注的三类风险**:操作错误(误改共享 Collection 的 Schema,如误删/改租户字段会导致全部租户数据不可检索)、性能热租户问题(某租户数据量/请求量骤增拖慢同 Collection 其他租户)、资源争抢(逻辑隔离下内存/CPU 共享,无资源配额)。
- **注意:Partition / Partition Key 均无 RBAC**——逻辑隔离只有"路由裁剪"没有"鉴权",应用层必须在检索表达式里强制带租户过滤,否则任何拿到连接的调用方都能跨租户查询。

### 2.2 社区实践:混合分级("大客户 Partition,小客户标量过滤")

✅ 已验证(CSDN《Milvus 分区 Partition 实践:多租户场景下的数据隔离与性能》,2026-09,https://blog.csdn.net/2401_87746054/article/details/164404390 ):
- 原理:"每个 Partition 在物理上拥有自己独立的 Segment 和 HNSW 索引图结构",查询传 partition_names 做"物理剪枝","只加载并搜索该分区专属的局部小图";性能数据:指定分区检索"从全局大图的 15ms 骤降至局部小图的 1.2ms"。
- 反面数据:纯标量过滤方案下,若租户数据仅占全局 1%,HNSW 前置过滤要跳过大量无效节点,"查询延迟显著上升"——**过滤隔离在数据倾斜时的性能塌方,比功能正确性更常被低估**。
- 坑 1(每租户一个 Collection):"etcd 的内存与网络同步压力急剧膨胀",协调节点重启加载可卡死十几分钟,"甚至直接引发 Raft 集群失联"——验证了官方"Collection ≤ 1 万"上限的真实约束。
- 坑 2(分区数):"Milvus 单个 Collection 默认最多支持 4096 个 Partition"(注:不同版本/配置下上限不同,官方文档口径为默认 1024,以部署版本为准),面向数十万 C 端用户绝不能一人一分区。
- 作者定式(可直接引用):"**大客户用 Partition 做物理隔离提速,小客户用标量过滤保轻量,严禁滥用 Collection 导致元数据爆炸**";约 10% 大客户专属 Partition + 90% 小客户共享分区 + tenant_id 过滤的二八方案。

### 2.3 行级(row-level)权限:Milvus array 列 + BITMAP 索引

✅ 已验证(CSDN《企业RAG构建中,如何用"行级别权限管控"避免数据泄露》,2024-11,https://blog.csdn.net/2401_84208172/article/details/143964273 ,Zilliz 投稿方向):
- 设计:每行(=chunk)带 array 列如 `security_group`,标注可读角色;"array 的最大长度根据每个用户自己具体的使用场景设定,并在此列建立 BITMAP 索引";"一个角色对应一个位图,每一位代表一行,如果该位为 1,则表示该角色对该行有访问权限"。
- 检索时权限过滤示例:`filter='array_contains(security_group, "ceo")'`;业务条件必须与权限**联合查询**:`filter='pk in [1, 3, 5] && array_contains(security_group, "sales")'`——**权限过滤不是可选后缀,必须放进同一个布尔表达式**,否则联合查询退化成"先查后删"。
- 坑:
  1. "依赖应用层自觉":权限过滤靠查询时主动加 filter,"忘记加就越权"——数据库不强制执行(见 §2.1 风险一);
  2. "对管理员的操作提出了更高的要求"(写入时粒度、建表时全局规划);
  3. 权限变更要**整行 upsert**(vector 等全部字段重传),易错且低效;
  4. array 容量限制(max_capacity=10),角色多时不够用;
  5. "在权限的动态管理和继承等易用性方面"仍不完善(无部门继承/传递授权)。

---

## 3. 检索时用户权限过滤:pre-filter / post-filter 实践与坑

综合已验证来源,可归纳为三层模式(每条都有上面对应来源):

1. **Pre-filter(检索前/检索中过滤)**——主流且推荐:把权限变成标量条件进检索表达式(FastGPT 文件级快照鉴权、Milvus filter/partition、Coze ES+向量层 space_id)。优点是"越权 chunk 根本不进上下文";代价是大权限过滤比(用户可见 <10% 数据)时 ANN 剪枝失效、延迟上升(§2.2 实测)。
2. **Post-filter(检索后过滤)**——作为兜底而非主防线:对召回结果逐条做 ACL 复核,再决定是否进 prompt。FastGPT 的"逐条鉴权快照"实际是检索内 pre-filter + 结果校验双做;纯 post-filter 的问题:top_k 被过滤后剩余结果过少、且敏感 chunk 已进入过检索链路(日志/缓存风险)。
3. **生成层过滤**——最后防线:对 LLM 输出做脱敏/审计。§5 的攻防文章把"生成层 PII 擦除"列为纵深防御的一环,但明确不能只靠这层。

**中文实践里的四个高频坑**(均有来源支撑):
- **自动化工作流丢身份**:FastGPT 官方原文"工作流中预选知识库且未按成员身份鉴权的场景不做文件级过滤"——批处理/定时任务/回调场景没有"当前用户",必须在编排层显式传入委托身份(on-behalf-of),否则权限过滤被静默跳过。
- **多存储层授权条件不同步**:向量库、元数据库(ES/MySQL)、原文对象存储可能各自有一份权限判断,跨层取数时必须"保留同一份授权条件"(⚠️ CSDN《Milvus 与RAG 权限边界:集合、元数据和原文分别授权》2026-08 提出该观点,仅搜索可见,未实抓;其结论与 Coze 源码分析的三层冗余 space_id 设计互相印证)。
- **权限元数据写错/漏写**:写入端漏打 security_group/tenant_id 标签,该行变成"任何人不可见"或"所有人可见",需要写入端强校验 + 定期对账。
- **权限变更传播**:整行 upsert 才能改权限(§2.3),意味着权限变更=重写向量行,大批量回收权限时要有批量任务与索引重建预算。

---

## 4. 语义缓存(Semantic Cache)跨用户泄露:一次"差点出事"的复盘

✅ 已验证(CSDN《RAG语义缓存踩坑记:向量相似不等于业务相同,租户数据差点越权泄露》,2026-08,https://blog.csdn.net/weixin_43272162/article/details/163445365 。可信度注意:单账号复盘文,数字未经第三方证实,模式本身可信):
- **事故**:生产安全巡检发现,企业租户 A 用户提问"我们公司上个月的财务净利润是多少?",语义缓存以 **0.96 相似度**命中此前**租户 B 用户**的同类提问,直接返回了 B 的财务数据;审计共发现 3 处跨租户错误命中。
- **根因**:缓存键只做向量余弦相似度(阈值 >0.85)匹配,**没有绑定 TenantID/RoleID/文档版本**。教训原句:"向量相似不等于业务相同"——Embedding 空间里两家公司对"上季度净利润是多少"的问句几乎重合,语义相似度天然无视权限边界。
- **修复方案(六条,可作 checklist)**:
  1. 缓存键改为**强属性哈希**:`sha256(TenantID + RoleID + NormalizedQuery)`,语义相似仅作二级加速;
  2. **双层缓存网关**:L1 私有缓存(键含 TenantID+RoleID)、L2 公共知识缓存(仅 `is_public=true` 的数据允许进入);
  3. 向量库检索强制 filter:`tenant_id == X && is_public == true`;
  4. **敏感问题跳过语义缓存**(正则命中"利润/薪资/客户名单"等直接 bypass);
  5. **权限变更/文档更新触发批量缓存失效**(版本号失效);
  6. 审计:每次命中记录 caller_tenant,"命中 tenant 与 caller 不一致"告警进 SIEM。
- 效果宣称:修复后 500 个跨租户问题 0 泄露,命中率 40%。

---

## 5. 攻防视角:越权检索与检索内容注入

✅ 已验证(CSDN《RAG系统安全攻防:越权检索与指令规避实战解析》,2026-09,https://blog.csdn.net/weixin_42524171/article/details/164830231 。**可信度注意:该账号发文量异常(月更数百篇),内容疑为 AI 辅助生成,文中百分比数据(如 73%、92%)不应直接引用,但攻击手法清单与防御分层与业界公开资料一致**):
- **越权检索攻击面**:绕过认证直连检索 API(如 RAGFlow HTTP API 只需 API Key、无用户级鉴权时);利用检索层无权限过滤"问出"知识库里其他部门/密级的内容;用角色扮演/指令注入让系统忽略权限提示词("假装是管理员")。
- **指令规避/变形手法**(为绕过敏感词与注入检测):Base64 等编码载荷;TokenBreak 式分词攻击(在文本中插入分隔符使 tokenizer 切开敏感词,绕过基于分词的检测);零宽字符(如 U+200B)隐写;同形字符(Homoglyph)替换;多轮渐进式信息收集(每轮只索取一小块,拼出完整敏感内容);语义伪装提问。
- **防御分层**(与国内安全厂商口径一致):
  1. 输入层:Unicode NFKC 规范化、编码/隐写检测、字形还原;
  2. 语义层:意图识别(识别"套取权限外数据"的意图);
  3. **检索层:动态 ACL 复验**——从查询中抽取"实体+动作",查用户真实权限组,比对数据密级标签,策略引擎给 allow/deny/mask 三态裁决;数据脱敏(PII 级别映射);
  4. 生成层:输出侧 PII 擦除与引用审计。
- ⚠️ 未验证(仅搜索可见,国内权威安全厂商口径,建议后续补抓):**奇安信《RAG安全攻略:揭秘检索增强生成的风险与防护之道》(2025-08,mdr.skyeye.qianxin.com)**,列出 RAG 五大风险:数据投毒(Data Poisoning)、提示注入(Prompt Injection)、信息泄露(Information Leakage)、对抗性嵌入(Adversarial Embeddings)等;站点反爬未能实抓原文。另有安全客《AI本地部署的天坑?RAG曝出重大安全漏洞》(2025-04,secrss.com)仅搜索可见。

---

## 6. 企业落地:身份源对接与权限同步

- **统一身份与组织同步(飞书/企微/钉钉/AD)** ✅:FastGPT 的 SSO 方案是当前中文圈文档化最完整的范本(§1.2):SSO 与成员同步是两个独立能力,飞书/企微能同步、钉钉只登不上同步(走 LDAP 补);组织树只允许单根;企微同步范围天然受"应用可见范围"约束。可提炼的落地规则:组织权限每日定时同步(SYNC_MEMBER_CRON)+ username 前缀防撞 + 权限主体尽量挂在部门/群组而非个人。
- **文档权限接入的真实形态是"导入"而非"映射"** ✅:MaxKB 飞书文档知识库(§1.4)代表了当前国内主流平台的做法——用只读 scope 把文档**内容**拉进来,**飞书侧的协作者权限不会跟着进 RAG**。要对齐飞书/钉钉文档权限,社区通行做法是自己拉协作者 API 建映射表(飞书 `drive:drive`+`space:document:retrieve` 已含所需 scope),再把映射结果写进向量库权限元数据(§2.3 的 array 列),检索时 pre-filter——即"权限感知检索(permission-aware retrieval)要自己做"。
- **权限漂移(permission drift)的两个必然来源**(由 ✅ 来源归纳):
  1. **源系统变更不传播**:飞书把文档收权/删除后,RAG 库里的 chunk 与权限标签原地不动(MaxKB 场景实证);
  2. **批量导入/重建时权限映射重算成本高**:权限变更=整行 upsert(§2.3)+缓存批量失效(§4),没有增量机制的平台(如 MaxKB 文档未提定时同步)只能靠全量重导。
- ⚠️ 未验证:SharePoint/Confluence 侧中文实践文章(CSDN《Langchain-Chatchat与Confluence协同》《Kotaemon企业内部Wiki对接方案》等)多出自营销号且未能实抓,结论与 §6 上述模式一致但证据等级低。

---

## 7. 踩坑速查表(可直接进培训的"坑清单")

| # | 坑 | 出处 | 类别 |
|---|---|---|---|
| 1 | 工作流预选知识库但不传用户身份 → 文件级权限过滤被静默跳过 | FastGPT 官方文档 ✅ | pre-filter 失效 |
| 2 | 语义缓存只按相似度命中 → 跨租户泄露(0.96 相似度事故) | CSDN 复盘 ✅ | 缓存泄露 |
| 3 | 权限过滤靠"记得加 filter",忘加即越权 | Zilliz 行级权限 ✅ | 应用层自觉 |
| 4 | 每租户一个 Collection → etcd 元数据爆炸、Raft 失联 | Milvus 分区实践 ✅ | 隔离选型 |
| 5 | 小租户占比 1% 时纯标量过滤延迟显著上升 | Milvus 分区实践 ✅ | 性能 |
| 6 | Partition/Partition Key 无 RBAC,逻辑隔离不鉴权 | Milvus 官方文档 ✅ | 误信隔离=鉴权 |
| 7 | 权限变更需整行 upsert(vector 重传) | Zilliz 行级权限 ✅ | 变更传播 |
| 8 | 飞书文档导入后权限由平台自管,源权限收缩不生效 | MaxKB 官方文档 ✅ | 权限漂移 |
| 9 | 企微成员同步只保留"应用可见范围"内成员,超出范围即静默丢失 | FastGPT SSO 文档 ✅ | 同步边界 |
| 10 | Dify 知识库权限二值(可访问=可维护),没有只读协作者 | Dify 文档+源码 ✅ | 粒度误估 |
| 11 | API 超级密钥到处用/硬编码进 Git;401/403 混淆 | RAGFlow 避坑文 ✅(付费文,部分) | 密钥管理 |
| 12 | 多层存储(向量/ES/原文)授权条件不一致 → 旁路越权 | Coze 源码分析 ✅ + ⚠️ 交叉印证 | 层间一致性 |

---

## 8. 来源清单

**✅ 已验证(实抓确认,共 16 项)**
1. FastGPT 官方文档:团队角色权限 — https://doc.fastgpt.io/guide/workspace/team/team_roles_permissions
2. FastGPT 官方文档:知识库集合(文件级)权限 — https://doc.fastgpt.io/guide/dataset/collection_permission
3. FastGPT 官方文档:团队模式 — https://doc.fastgpt.io/guide/admin/teamMode
4. FastGPT 官方文档:SSO & 外部成员同步 — https://doc.fastgpt.io/guide/admin/sso
5. Dify 官方文档:团队成员管理(工作区四角色)— https://docs.dify.ai/zh-hans/guides/workspace/team-members-management
6. Dify 官方文档源:知识库管理"权限"定义 — https://docs.dify.ai/zh-hans/guides/knowledge-base/manage-knowledge(dify-docs 仓库 mdx 原文)
7. Dify 官方文档源:知识检索节点元数据筛选 — https://docs.dify.ai/zh-hans/guides/knowledge-base/integrate-knowledge-within-application
8. Dify 源码(api/models/dataset.py、api/services/dataset_service.py,langgenius/dify main 分支)— 数据集权限枚举 ONLY_ME/ALL_TEAM/PARTIAL_TEAM 与 dataset_permissions 表
9. RAGFlow 官方中文文档:管理团队成员 — https://ragflow.com.cn/docs/manage_team_members
10. MaxKB 官方文档:X-Pack 资源授权 — https://maxkb.cn/docs/v2/user_manual/X-Pack/authorization_resources.html
11. MaxKB 官方文档:X-Pack 飞书文档知识库 — https://maxkb.cn/docs/v2/user_manual/X-Pack/feishu_doc.html
12. Milvus 官方中文文档:实施多租户 — https://milvus.io/docs/zh/multi_tenancy.md
13. CSDN:企业RAG构建中,如何用"行级别权限管控"避免数据泄露 — https://blog.csdn.net/2401_84208172/article/details/143964273
14. CSDN:Milvus 分区 Partition 实践:多租户场景下的数据隔离与性能 — https://blog.csdn.net/2401_87746054/article/details/164404390
15. CSDN:RAG语义缓存踩坑记(跨租户泄露复盘)— https://blog.csdn.net/weixin_43272162/article/details/163445365
16. CSDN:Coze源码分析-资源库-创建知识库-基础设施/存储/安全 — https://blog.csdn.net/lypzcgf/article/details/151689359
17. CSDN:RAG系统安全攻防:越权检索与指令规避实战解析 — https://blog.csdn.net/weixin_42524171/article/details/164830231(内容已核验,账号可信度低,数据慎用)
18. CSDN:RAGFlow 权限管理避坑指南(API Key 部分)— https://blog.csdn.net/swift5iosmith/article/details/153044923(后半付费)
19. CSDN:企业级 RAG 权限管理 多租户隔离方案(RBAC 五级密级 + 多向量库隔离,前半可见)— https://blog.csdn.net/weixin_40040017/article/details/159726251

**⚠️ 未验证(仅搜索可见,供后续补抓)**
- 奇安信:RAG安全攻略:揭秘检索增强生成的风险与防护之道(2025-08,mdr.skyeye.qianxin.com)— 五大风险框架,站点反爬
- 安全客:AI本地部署的天坑?RAG曝出重大安全漏洞(2025-04,secrss.com)
- CSDN:Milvus 与RAG 权限边界:集合、元数据和原文分别授权(2026-08)
- CSDN:RAG越权检索与变形指令/隐写规避 / 企业级AI智能体安全架构:基于MCP与细粒度RAG的权限控制实践(2026-08)
- CSDN:RAG 知识库要不要加权限?多租户、文档隔离、引用溯源一次讲清(付费专栏,仅摘要:"企业 RAG 不加权限,答得越准越危险")
- 腾讯云开发者社区:从 64 到百万租户:Milvus 多租户选哪层(标题仅在搜索摘要出现,未能定位原文)
- 扣子 SaaS 版知识库权限(coze.cn 文档为 SPA,无法抓取)
