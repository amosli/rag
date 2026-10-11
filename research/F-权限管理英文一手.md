# F - RAG 权限管理 / Document-Level Access Control（英文一手资料）

> Lane：【RAG 权限管理 / document-level access control】中文正文 + English terms；检索与取证均为英文一手来源。
> 取证日期：2026-09-27。标注：✅已验证 = 本次实际抓取到正文；⚠️未验证 = 仅搜索/记忆可见，或链接已迁移。
> 共核实 25+ 个一手来源（官方文档为主，论文/标准为辅）。

---

## 1. 为什么朴素 RAG 会破坏访问控制（Why naive RAG breaks access control）

核心论点：检索层（retrieval layer）如果不带身份信息，就会把"用户本无权看到的文档"作为 context 送给 LLM，LLM 再转述出来——这是绕过应用层 ACL 的**越权泄露通道**。因此 **query-time / retrieval-layer 的 ACL 过滤是强制性的（mandatory）**，权限必须在检索时就生效，而不是生成后再截。

### 1.1 Microsoft 官方口径："respect user permissions"

- ✅ Microsoft Copilot（M365）数据隐私与安全文档，原文两句（逐字摘录）：
  - "**Microsoft Copilot only surfaces organizational data to which individual users have at least view permissions.** It's important that you're using the permission models available in Microsoft 365 services, such as SharePoint, to help ensure the right users or groups have the right access to the right content."
  - "**Semantic Index honors the user identity-based access boundary so that the grounding process only accesses content that the current user is authorized to access.**"
  - 即：语义索引（Semantic Index）本身维护"基于用户身份的访问边界"，grounding 只能取到当前用户被授权的内容；权限模型沿用 SharePoint/Entra，Copilot 不新增授权。
  - 来源：https://learn.microsoft.com/en-us/microsoft-365/copilot/microsoft-365-copilot-privacy （✅，2026-07 版页面）
- 引申含义（培训机构可讲）：微软把"索引与源权限对齐"作为产品级承诺；自建 RAG 必须在索引侧冗余存储 ACL 元数据并在查询侧强制过滤，否则等价于把 SharePoint 权限模型降级为"全员可见"。

### 1.2 Azure 官方对"应用端后过滤 vs 检索内过滤"的成本/性能判断

- ✅ Azure AI Search《Document-Level Access Control》明确列出"在检索管线内过滤"优于"查询后裁剪（post-query trimming）"：
  - "**Improves performance versus post-query trimming:** Filtering inside the search pipeline is faster than loading larger result sets into your application and trimming there, especially at high query volumes."
  - 同时列出的收益：无需在应用里自实现嵌套组解析/多层 ACL 遍历/post-query trimming；复用既有身份基础设施（Entra/SharePoint）。
  - 来源：https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview （✅）
- ⚠️（分析性引用，非官方）Gartner 等第三方对"RAG 越权"的公开分析本次未取到正文，建议课堂上用 Microsoft/Azure 官方表述 + OWASP LLM08（见 §5）作为权威依据，不引用无法验证的博客。

### 1.3 检索层 ACL 的两种实现姿态（官方文档口径）

1. **把过滤条件推进检索请求（query-time filter，pre-filter 语义）**：Azure security filter（`search.in`）、Elasticsearch/OpenSearch DLS、Kendra UserContext、Bedrock KB `userContext` 全部属于此类。
2. **把"索引"本身按身份隔离**（per-tenant namespace/collection/corpus）：Weaviate 多租户、Pinecone namespace、Vectara corpus isolation。
   - Azure 原文点名警告：`retrievable=false` 的安全字段**不是**内容混淆或字段级安全机制，"document-level authorization is enforced by applying the security filter to every query"（每一查询都必须带过滤）。来源：https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search （✅）

---

## 2. 厂商原生方案（Vendor-native mechanisms，含精确配置名）

### 2.1 Azure AI Search（两个 GA/preview 机制 + 三类新 ACL 摄取）

**A. Security filters（GA，自建身份可用）**
- ✅ 《Security Filter Pattern》：机制 = 常规 OData filter 字符串匹配，"There's no authentication or authorization through the security principal. The principal is just a string, used in a filter expression."
- 精确配置名：
  - 索引字段：`group_ids`，类型 `Collection(Edm.String)`，`filterable=true`，`retrievable=false`
  - 查询过滤（推荐写法，避免长 OR 链）：`group_ids/any(g:search.in(g, 'group_id1, group_id2'))`
  - 官方性能注记：等值表达式 disjunction（`Id eq 'id1' or Id eq 'id2'…`）在数百上千个值时"slows down query response time by many seconds"；改用 `search.in` "you can expect **subsecond response times**"。
  - 来源：https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search （✅）

**B. 内置 Document-Level Access Control（preview，四条路线）**
- ✅ 《Document-Level Access Control》总览，四类 approach（逐字表头）：
  1. **Security filters**（字符串比较，API-agnostic，GA）
  2. **POSIX-like ACL / RBAC scopes (preview)**：ADLS Gen2 目录/文件 ACL + ADLS Gen2/Azure Blob 的 RBAC scopes；查询时把 query token 背后的 Microsoft Entra principal 与文档 permission metadata 比对
  3. **Microsoft Purview sensitivity labels (preview)**：indexer 从 Blob/ADLS Gen2/SharePoint/OneLake 抽取敏感度标签，查询时结合 Entra token + Purview policy 评估
  4. **SharePoint in Microsoft 365 ACLs (preview)**：SharePoint indexer 摄取权限元数据做 query-time access checks
- 精确配置名（重要，出题点）：
  - 索引 schema：`SearchIndexPermissionFilterOption`（启用 permission filter）、字段级 `permissionFilter`（REST: indexes create-or-update `#permissionfilter`）
  - 查询 header：**`x-ms-query-source-authorization`**（携带最终用户 Entra token；服务端两步检查：先 `Search Index Data Reader`/`Search Index Data Contributor` 的索引级 RBAC，再比对文档级 permission metadata）
  - 摄取选项：`ingestionPermissionOptions`（知识源，如 `sensitivityLabel`）；push API 用 2026-08-01-preview REST API
  - chunk 化时：permission metadata 字段从 indexer field mappings 迁移到 **index projections**（每个 chunk 行都要投影 ACL/label，否则 chunk 级不过滤）
  - 来源：https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview （✅）
- 强制层与新鲜度（原文要点）：
  - "Query-time enforcement evaluates the caller's Microsoft Entra claims against the permission metadata that's **already stored in the index**. Permission changes in the source system … are only reflected in search results **after that metadata is synchronized to the index**"——即权限漂移由"同步节奏"决定（SharePoint：2026-05-01-preview 起 unique permissions 增量同步；继承权限需显式 refresh）。
  - 来源：同上（✅）

### 2.2 Elasticsearch（Elasticsearch Security：核心 DLS + Connector DLS）

**A. 核心 Document Level Security（DLS）/ Field Level Security（FLS）**
- ✅ 《Controlling access at document and field level》：
  - 机制：role 的 `indices` 块内加 **`query`**（Query DSL，作用于 read 类请求）；FLS 用 **`field_security`**（`grant` / `except`）。
  - 模板注入身份（Mustache `_user` 变量）：`"query": {"template": {"source": {"term": {"acl.username": "{{_user.username}}"}}}}`；支持 `_user.metadata`、`{{#toJson}}`。配 **set security user** ingest processor 把 `username/roles/email/metadata` 打进文档，实现按用户过滤。
  - 多 role 合并：DLS role query 之间是 **OR**；某 role 未配 DLS 则对该用户**整体关闭 DLS**（危险点）。
  - 限制（官方逐条）：角色查询不支持 `has_child`/`has_parent`；relevancy 打分用全局统计、忽略 role query；**聚合（aggregations）可能泄露受限文档信息**（term counts、field names）；DLS/FLS 生效的账户不得执行写操作；terms-lookup `terms`、`percolate` 等不支持。
  - 来源：https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/controlling-access-at-document-field-level （✅）

**B. Connector Document-Level Security（企业连接器 ACL 同步）**
- ✅ 《Connector DLS overview》：精确机制名：
  - **Access control documents**：connector 把身份文档写入隐藏索引 **`.search-acl-filter-<INDEX-NAME>`**；每个身份文档（username/email/groups）含一个 query template，`access_control` params 列出可见身份。
  - 内容文档字段：**`_allow_access_control`**（身份字符串数组；"field names may vary across different connectors"）。
  - 同步：必须**先启用 DLS 再首次内容同步**（否则要删数据重同步）；**Access control sync** 定期跑保持 ACL 新鲜。
  - 查询时：template 过滤，"if at least one access control element in their access control document matches an item within the document's `_allow_access_control` field"。
  - 边界行为（安全默认值不对称）：**缺字段 = 无限制（人人可见）；字段存在但为空 = 无人可见**。
  - 多索引场景：为每个用户签发含 per-index role descriptor 的 ES API key（带 `read` + 对应 template），建议设置 `expiration`；用户权限变化后需更新 key（典型的 late-ACL-propagation 风险面）。
  - 来源：https://www.elastic.co/docs/reference/search-connectors/es-dls-overview （✅）

### 2.3 OpenSearch Document-Level Security（DLS + term lookup）

- ✅ 《Document-Level Security》（Security plugin）：
  - 配置名：role 的 index permission 块内 **`dls`** 参数（值为一个 query），常与 **`fls`** 同用；通过 security admin tool / security configuration APIs 应用。
  - **DLS with term lookup**（拉取式 ACL）：`term` 的 `source` 指定 `id`（lookup 文档 ID）、`index`（ACL 所在索引）、`field`；查询时 OpenSearch 取出该文档并把字段值注入 filter——经典模式是"每个用户一个小 ACL 索引，DLS query 动态解析当前用户权限"。
  - 强制层：Security plugin 作为 filter 应用于 search/get 操作，"users only ever see the documents their role permits, regardless of the query they issue"。
  - 来源：https://docs.opensearch.org/docs/latest/security/access-control/document-level-security/ （✅）
  - ⚠️ 该页未给 DLS 对 k-NN/向量检索的性能影响数据（旧版社区已知 DLS 对 ppy k-NN 有额外开销），教学中标注为"需自测"。

### 2.4 Amazon 阵营：Kendra / Amazon Q Business / Bedrock Knowledge Bases

**A. Amazon Kendra — User context filtering**
- ✅ 《Filtering on user context》：
  - 机制名：**User context filtering**；查询时通过 **`UserContext`** 对象传入 `Query` API（user token / UserId+Groups / 属性过滤三选一）。
  - 索引侧 ACL：文档级 **`AccessControlList`**（`Principal` 对象：`ALLOW|DENY` + `USER|GROUP` + 名称，最多 200 条）；S3 数据源用 `.metadata.json`；可重配 `AccessControlConfigurationId`（`CreateAccessControlConfiguration`）。
  - 内置属性：**`_user_id` / `_group_ids`**（+`AttributeFilter`）；组解析可走 `PutPrincipalMapping` 或 `UserGroupResolutionConfiguration`（IAM Identity Center）。
  - 关键免责声明（逐字）："**User context filtering isn't an authentication or authorization control for your content.** It doesn't do user authentication on the user and groups sent to `Query` API."——即"过滤 ≠ 认证"，认证责任在应用。
  - 数据源覆盖：SharePoint/Confluence/Google Drive/Quip/ServiceNow/Salesforce 等 40+ 连接器逐个给出 `_user_id`/`_group_ids` 映射规则（如 SharePoint 本地组 = SHA256(site)|groupName）。
  - 来源：https://docs.aws.amazon.com/kendra/latest/dg/user-context-filter.html （✅）

**B. Amazon Q Business — User Store / Principal mapping / ACL crawling**
- ✅ 《Understanding Amazon Q Business User Store》（`connector-principal-store.md`）：
  - "end users see Amazon Q Business chat responses generated **only** from the documents that they have access to"；内部 **User Store** 存 principal mapping（用户 ↔ 各数据源 user IDs/groups）。
  - 用户被分配 **UUID**（email 改名/复用不串权限；别名多登录 ID 可挂同一 UUID）；email 自 2024-12-17 起大小写不敏感、`+subaddress` 等价。
  - S3 与 Web Crawler 连接器**不支持** User Store（需另行处理 ACL）。
- ✅ 《Principal mapping》（`principal-mapping.md`）：user mapping（IdP 登录 ↔ 数据源 ID ↔ UUID）+ group mapping（跨数据源组、子组）。
- ✅ 《Using ACL Analyzer》（`acl-analyzer.md`）：排障工具，按 Document ID + User ID 校验 allow/deny 归因（permission 可观测性，值得在课程里作为"权限调试"案例）。
- ✅ 《What is Amazon Q Business?》：概述口径 "permissions-aware responses… responses to questions are based on the content that your end user has permissions to access"；且页面顶部声明 **Amazon Q Business 不再对新客户开放**（"explore Amazon Quick"）——产品漂移证据。
  - 来源：以上四篇均 https://docs.aws.amazon.com/amazonq/latest/qbusiness-ug/…md（✅，TOC 另证实每个连接器都有 "ACL crawling" 小节，如 `sharepoint-cloud-user-management.html`、`confluence-cloud-user-management.html`、`googledrive-v2-acl-crawling.html`）

**C. Amazon Bedrock Knowledge Bases（Managed KB）— ACL-aware retrieval（最新且最完整）**
- ✅ 《Access Control Lists awareness enablement》（`kb-managed-acl.md`）：
  - 功能名：**ACL-aware retrieval / ACL awareness**。摄取时爬取 allowed users / denied users / allowed groups / denied groups；查询时传 **user context**，只返回该用户有权访问的文档。
  - 两阶段强制：
    1. **Pre-retrieval filtering**（用最近一次 crawl 同步的 ACL 过滤候选）
    2. **Real-time ACL verification**（SharePoint/OneDrive/Google Drive/Confluence 支持：查询时实时回源验证权限，捕获 sync 间隙的权限变更；S3/Custom 不支持——ACL 来自客户配置文件）
  - 身份模型：**email 为全局用户标识**，无跨 IdP 别名解析；组关系从数据源爬取，"Group memberships are as fresh as the last sync"。
  - **官方免责（逐字，重要）**："**ACL awareness is not authorization** — Bedrock Managed Knowledge Base provides *ACL-aware filtering*, not a security boundary… You must not rely on this feature as a sole access control mechanism without upstream authentication."（deny 优先：deny always overrides allow）
  - 连接器矩阵：SharePoint（ENTRA_ID_APP_ONLY，2LO）、OneDrive（ENTRA_APP_ID）、Google Drive（SERVICE_ACCOUNT，域委派 2LO）、Confluence（BASIC + admin token）；Web Crawler 不支持。
- ✅ 《ACL-aware retrieval on managed knowledge bases》（`kb-test-retrieve-acl.md`）：
  - API：`Retrieve` 请求带 **`userContext`**（`userId` = universal email）。**"If you do not provide `userContext`… ACL-enabled data sources return zero results"**（fail-closed）。
  - "**Missing ACL metadata means inaccessible**"（无 ACL 视为受限，非公开）；real-time 验证拒绝部分文档时**不回填**（partial results）。
  - 新鲜度/延迟注记：第三方 IdP 凭据缓存**至多 1 小时**；"Changes to access control permissions are eventually consistent. Updates typically take effect within a few minutes."
- ✅ 《Document-level access controls - Confluence》（`kb-managed-ds-confluence-acl.md`）：
  - 开关：`connectorParameters` 里 **`aclEnabled: true`**（`BASIC` 认证 + Secrets Manager 需 `adminApiKey`/`organizationId`/`directoryId`）；**ACL 配置不可逆**（"ACL configuration is permanent"）。
  - 爬取结构：Space permissions / Page restrictions（嵌套继承）/ Blogs / Attachments（继承父级）。
  - 故障语义："Retrieval fails closed: affected documents are silently omitted"。
  - 来源：以上三篇均 https://docs.aws.amazon.com/bedrock/latest/userguide/…md（✅）

### 2.5 Google Vertex AI Search（ACL-enabled data stores）⚠️

- ⚠️ 未验证：原《About ACLs》（generative-ai-app-builder / agent-builder `docs/about-acls`）在本次抓取时已 404/重定向到 "Gemini Enterprise Agent Platform" 总览页——Google 正把 Vertex AI Search 文档迁入 Gemini Enterprise，ACL 文档（identity resolution、ACL-enabled data stores、第三方 IdP 同步 Okta/Ping/Auth0、`userPseudoId`）当前**没有可抓取的稳定官方 URL**。
- 教学建议：以"机制存在过（ACL-enabled data store + identity resolution at query time）"讲述，引用迁移前 URL 并标注 ⚠️；等 Gemini Enterprise 文档稳定后复核。
- 观察到的旧 URL（均重定向/404）：https://cloud.google.com/generative-ai-app-builder/docs/about-acls 、https://cloud.google.com/agent-builder/docs/about-acls 、https://docs.cloud.google.com/gemini-enterprise/docs/about-acls

### 2.6 Weaviate — Native Multi-tenancy + RBAC

- ✅ 《Multi-tenancy》：
  - 机制：一个 collection 内按 tenant 分 **shard**（"Each tenant is stored on a separate shard"，数据互相不可见）；不是 collection-per-tenant。创建时 `multiTenancyConfig`；每个 CRUD/查询必须指定 tenant（如 `.with_tenant("tenantA")`）。
  - 状态机：`ACTIVE` / `INACTIVE` / `OFFLOADED`（v1.26 起 HOT→ACTIVE、COLD→INACTIVE 改名）；`autoTenantActivation`、`autoTenantCreation`（注意 typo 风险）；OFFLOADED 租户需 S3 offload 配置且**不参与备份**。
  - 收益（原文）：privacy（租户互不可见）+ efficiency（查询只打一个小 shard）。
  - 来源：https://docs.weaviate.io/weaviate/manage-collections/multi-tenancy （✅）
- ✅ 《RBAC Overview》：
  - 版本：RBAC v1.29+ 引入，v1.30+ 默认开启（llms.txt 原文："RBAC with collection- and tenant-level permissions, user management API, API key + OIDC auth (v1.29+, on by default in v1.30+)"，https://weaviate.io/llms.txt ✅）。
  - 模型：Users → Roles → Permissions（Actions：create/read/update/delete/manage；Resources：collections/tenants/data objects/backups/cluster…；可选约束：collection name filter（string/regex）、tenant name filter）。预置角色 **`root`** / **`viewer`**（不可修改）；根用户经 `AUTHORIZATION_RBAC_ROOT_USERS` 指定；无角色的认证用户默认无权限（default-deny）。
  - 官方安全提示：`manage_roles` 权限可被用于提权（"can be used to escalate privileges"）。
  - 来源：https://docs.weaviate.io/weaviate/configuration/rbac （✅）
- 认证：OIDC（OpenID Connect）+ API key（见 llms.txt，✅）。

### 2.7 Qdrant — Payload 过滤 + JWT RBAC + 多租户

- ✅ 《Access Control / Security》：
  - 版本线：Admin API key（v1.2.0）→ Read-Only API key（`read_only_api_key`，v1.7.0）→ **Granular Access API keys（v1.9.0，JWT-based）**：配置 `jwt_rbac: true` + `api_key`（HS256 签名验证）。
  - JWT claims 精确名：**`exp`**（30s 时钟偏差容忍）、**`value_exists`**（校验 token 时要求指定 collection 中存在匹配 `matches` 的点——把 token 与真实数据绑定，改 role/删点即撤销）、**`access`**（全局 `"r"`/`"m"`；按 collection `{"collection": "...", "access": "rw"}`；列表操作 "Allowed, but filtered"）。
  - 说明：Qdrant 官方 JWT 目前做到 collection 级读写域 + `value_exists` 数据绑定；**按 payload 条件（如 user_id=…）限制读取仍需应用侧把 claim 转成 `filter` 下推**——课程里不要把 Qdrant JWT 说成 ABAC。
  - 来源：https://qdrant.tech/documentation/security/ （✅）
- ✅ 《Filtering》：过滤子句 **must / should / must_not**（可递归嵌套）；条件类型 Match / **Match Any**（v1.1.0，替代同字段多 should，"It's faster, especially when filtering on a large number of values"——ACL id 列表场景直接受益）/ Match Except（v1.2.0）/ Nested object filter / Has id 等；性能关键句："**For performant filtering, create payload indexes for the fields you plan to filter on.** For best results, create payload indexes before ingesting data."（https://qdrant.tech/documentation/concepts/filtering/ ✅）
- ✅ 《Multitenancy》：
  - 官方推荐位次：① **payload 分区（推荐，海量小租户）**——共享 collection + `group_id` 类 payload 字段过滤 + keyword payload index 加 **`is_tenant: true`**（v1.11.0+，把同租户向量放一起，顺序读）；调优：全局 HNSW `m: 0` + `payload_m: 16`（每租户独立图）。② collection-per-tenant："rarely the most efficient approach"（Cloud 默认上限 1000 collections）。③ User-defined sharding（`sharding_method: custom`，少量大租户）。④ Tiered multitenancy（v1.16.0+，小租户共享 shard，大租户 `replicate_points` 晋升专用 shard；专用 shard ≤1000，约 2 万 points 起晋升）。
  - 来源：https://qdrant.tech/documentation/guides/multiple-partitions/ （✅）

### 2.8 Milvus / Zilliz — Partition-key 多租户 + RBAC

- ✅ 《Implement Multi-tenancy》（`multi_tenancy.md`）：三种方案——database-per-tenant / collection-per-tenant / **partition-key based multi-tenancy**（一个 collection 用 partition key 字段路由 + 查询时对该字段过滤）；官方以"场景选型表"给出取舍（隔离强度 vs 资源开销）。
- ✅ 《RBAC Explained》（`rbac.md`）：
  - RBAC 模型四要素：**Resource**（instance/database/collection）→ **Privilege**（如 Collection 级 Load/Release/Insert/Upsert/Delete/Search/Query/Get/Flush 等，数据库级 ListCollections 等，实例级 ListDatabases 等）→ **Privilege Group**（内置 COLLECTION/DATABASE/INSTANCE/CLUSTER 级 RO/RW/ADMIN 组；可自定义）→ **Role / User**；默认用户 **`root`**（密码 `Milvus`，须改）。
  - API：`CreateRole`/`DropRole`/`ListRoles`；`GrantPrivilege`/`RevokePrivilege`（v2.5 起推荐 `GrantPrivilegeV2`，resource 粒度到 collection/database/instance）；内置角色 `public`（可 DescribeCollection/ShowCollections）。
  - 来源：https://milvus.io/docs/rbac.md （✅）；partition key 用法详见 https://milvus.io/docs/use-partition-key.md （⚠️ 未单独抓取正文，llms.txt ✅ 证实存在）

### 2.9 pgvector / Postgres — Row-Level Security（RLS）

- ✅ PostgreSQL 官方《Row Security Policies》：
  - 精确 SQL：`ALTER TABLE … ENABLE ROW LEVEL SECURITY` / `FORCE ROW LEVEL SECURITY`（owner 也受限）/ `CREATE POLICY … TO <role> USING (…) WITH CHECK (…)`；**USING = 可见行过滤，WITH CHECK = 写入校验**；permissive（OR）/ restrictive（AND）两种合并；superuser 与 `BYPASSRLS` 绕过。
  - 行为要点：启用 RLS 但无 policy = **default-deny**（0 行）；policy 表达式在每行上、先于查询自身条件求值 → 天然"查询前过滤"；**子查询策略在 READ COMMITTED 下有并发泄露窗口**（官方给出 `SELECT … FOR SHARE` 等缓解）；`row_security=off` 不会绕过而是报错（防备份静默丢行）。
  - 来源：https://www.postgresql.org/docs/current/ddl-rowsecurity.html （✅）
- ✅ Supabase《Row Level Security》（RAG 落地最佳实践）：
  - 文档向量表直接套 RLS：`create policy "Users can view their own documents." on documents for select to authenticated using ( (select auth.uid()) = user_id );` → 向量近邻查询（`order by embedding <=> query_vec`）只会扫到调用者自己的行。
  - 为什么不能只靠应用侧过滤：**defense in depth**——"RLS … also protects your data when it is reached through third-party tooling"（任何直连 SQL/裸 API 的路径都受保护）；并警告 Supabase 新表默认对 `anon/authenticated/service_role` 全放权，需 `revoke` 后按需 `grant`。
  - 性能注记：对过滤列建索引；`(select auth.uid())` 包一层让函数 per-statement 缓存（initPlan）而非逐行求值。
  - 来源：https://supabase.com/docs/guides/database/postgres/row-level-security （✅）

### 2.10 Vectara —（重要漂移案例）从 document ACL 转向 ABAC metadata filter

- ✅ 现行文档《Attribute-based access control》（ABAC）：
  - **原文声明**："Vectara does not currently offer native field-level or document-level access control levels (ACLs)."——不再有原生文档 ACL。
  - 现行机制：索引时把权限相关信息（`user_id`/`account_id`/`team`/`department`/`access_level` 等）作为**自定义 metadata 字段（须在 corpus schema 声明）**；查询时由后端把已验证的用户属性拼成 **`metadata_filter`** 表达式注入（如 `doc.owner in ("mary","global") OR ("history" IN doc.groups AND …)`）。
  - **强制层警告（逐字）**："Filters are **enforced per query, not at platform level** — No filter supplied = no restriction."（谁忘了拼过滤器就全量泄露）；Agent 会话里把过滤器固定在工具的 `argument_override`（`"metadata_filter": "${session.metadata.corpus_filter}"`）防 LLM 覆写。
- ✅ 《Authorization overview》：三层组合——RBAC（预置 `QueryService`/`IndexService` 角色，account/corpus 级粗粒度）+ ABAC（文档级 metadata filter）+ **Multi-Tenant Corpus Isolation**（每租户独立 corpus，"complete data isolation"）。
- ✅ 来源：https://docs.vectara.com/docs/security/authorization/attribute-based-access-control 、https://docs.vectara.com/docs/security/authorization/authorization-overview （均 ✅；sitemap 里已无 document-level-security 页，✅ 旁证功能下线）
- ⚠️ 旧机制（`read_policy` / `allowed_groups` / query `user_context`）在旧文档与 API 中存在过，但本次未在现行文档中找到——**教学中按"已移除/演进为 ABAC"处理**，这是一个绝佳的"权限模型平台漂移"案例。

### 2.11 Pinecone — Namespaces

- ✅ 《Indexing overview》：index 内数据划分进 **namespace**；所有读写"scan only the targeted namespace"；官方多租户用法："one namespace per customer"（"When you need to isolate data between customers"）；serverless 限额按 plan：Starter 100 → Enterprise 1,000,000 namespaces/index。
- 来源：https://docs.pinecone.io/guides/index-data/indexing-overview （✅）
- 评注：namespace 是**逻辑隔离 + 性能分区**，不是独立凭证体系；谁持有 API key 谁就能指定任意 namespace——租户隔离必须由应用层保证写入/查询 namespace 与身份绑定（官方未提供 per-namespace ACL）。

---

## 3. 权限模型：RBAC vs ABAC vs ReBAC；Zanzibar 与开源实现

### 3.1 Zanzibar（Google）

- ✅ USENIX ATC '19 官方页：《Zanzibar: Google's Consistent, Global Authorization System》（Pang, Caceres, Burrows 等）
  - 要点（原文级）：统一的共享数据模型与配置语言覆盖数百个 Google 服务（Calendar/Drive/Maps/Photos/YouTube…）；"Decisions honor the causal ordering of user actions, giving external consistency even as ACLs and object contents change"（**Zookie** token 记录因果一致性）；规模："trillions of ACLs and millions of authorization checks per second"；三年生产指标："95th-percentile latency of less than 10 milliseconds and availability of greater than 99.999%"。
  - 来源：https://www.usenix.org/conference/atc19/presentation/pang （✅）
  - ⚠️ 常见引用错误：arXiv:1909.09628 **不是** Zanzibar（实为天体物理论文，已核实）；讲义请以 USENIX 页为准，arXiv 预印本编号未验证。

### 3.2 开源实现（用于把权限判定变成"检索过滤"）

- ✅ OpenFGA（CNCF，Zanzibar-inspired FGA）：
  - 官方概念分类（llms.txt ✅）："Learn fine-grained authorization concepts: **ReBAC, RBAC, ABAC, PBAC, and Google Zanzibar**"（https://openfga.dev/docs/authorization-concepts.md）；核心对象：Authorization Model（type/relation 定义）、Store、Tuple（user, relation, object）、关系查询 API：**Check / Read / Expand / ListObjects / ListUsers**（https://openfga.dev/docs/interacting/relationship-queries.md，✅ 链接清单）。
  - RAG 用法：`ListObjects(user, "can_read", "document")` 一次拉回用户可读的文档 id 列表 → 转成 vector DB 的 `has id`/`terms` 过滤（或 `doc_id IN (...)`）下推。
- ✅ SpiceDB（authzed）：官方自我定位 "The open-source Zanzibar database"，"Relationship-based access control"；概念：**Schema**（建模语言）、**Relationships**（可带 caveats/过期）、核心 API **`CheckPermission`**、批量/Lookup 系（`LookupResources`）、一致性（fully-consistent vs minimize-latency）与 Read-After-Write。来源：https://authzed.com/docs/ （✅）
- ⚠️ Permify（开源 Zanzibar-style 授权引擎）、Ory Keto（Ory 家族的 Zanzibar-inspired FGA）本次未抓取正文，仅列名。
- 模型对比（教学口径，均有一手概念页支撑）：
  - **RBAC**：角色→权限，静态组（Milvus/Weaviate/Vectara RBAC、ES roles）；
  - **ABAC**：属性/元数据条件（Vectara ABAC、Kendra `_user_id`/`_group_ids`、Postgres RLS policy 表达式）；
  - **ReBAC**：关系图判定（Zanzibar/SpiceDB/OpenFGA），能表达"文档的父目录共享给了组 X"这类继承关系，最贴合企业内容权限。

---

## 4. 身份传播链路（Identity propagation flow）

综合官方文档可归纳的标准链路（每一步都有一手出处）：

```
用户 → IdP（OIDC/JWT: sub + groups/claims）
  → API Gateway / 应用后端（校验 token，取 Entra/userId/groups）
  → 授权服务（三种取法）：
      a) 组列表直推 filter（Azure search.in(group_ids)）
      b) 权限服务解析可读 doc id 集合（OpenFGA ListObjects / SpiceDB LookupResources → terms/has id filter）
      c) ACL 索引 lookup（OpenSearch DLS term lookup; Kendra PutPrincipalMapping/User Store）
  → 向量检索时把过滤下推进 ANN（pre-filter） 
  → 生成前/后再校验（post-verification）
```

一手支撑点：
- ✅ Azure：查询头 `x-ms-query-source-authorization` 携带**最终用户** Entra token，服务端"Extracts the user, group, and scope claims from the token → Compares those claims to the permission metadata stored alongside indexed documents"（https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview ✅）。
- ✅ ES：`{{_user.username}}`/`{{_user.metadata.*}}` 模板变量 = "IdP claims → role query" 的官方实现（Elastic DLS 页 ✅）。
- ✅ Kendra：User token（`UserContext`）"extracts and validates the token, pulls and checks the user and group information"；组映射可用 `PutPrincipalMapping` 预存，免去查询时枚举全部组（Kendra 页 ✅）。
- ✅ Bedrock KB：`userContext.userId` = universal email；**应用负责认证并把已验证身份传给 Retrieve**（"ACL awareness is not authorization"，✅）。
- ✅ Q Business：IdP 登录 → UUID → User Store 全量映射 → chat 时按 UUID 取权限（✅）。
- 设计铁律（各官方文档一致）：**绝不要把"客户端自报 user_id/groups"当作受信输入**——Kendra 原文"it is up to your application to ensure that the user and group information sent to Query API is authenticated and authorized"；Bedrock 原文同义。

---

## 5. RAG 特有的安全风险（papers / standards）

### 5.1 语义缓存跨用户泄露（Semantic cache cross-user leakage）

- ✅ arXiv:2409.20002《The Early Bird Catches the Leak: Unveiling Timing Side Channels in LLM Serving Systems》（2024-09-30）：通过时序侧信道从 LLM 服务系统窃取提示词/缓存信息（任务说明里提到的 arXiv 2402.02751 **不是**该文——2402.02751 实为钽电沉积材料学论文，已经 arXiv API 双通道核实为错引）。
  - 来源：https://arxiv.org/abs/2409.20002 （✅ arXiv API 核实标题/编号/日期）
- ✅ arXiv:2601.23088《From Similarity to Vulnerability: Key Collision Attack on LLM Semantic Caching》（2026-01-30）：语义缓存**键碰撞**攻击——相似度命中导致返回**他人缓存回答**，正是共享语义缓存（GPTCache 类系统）跨用户泄露的直接证据。
  - 来源：arXiv API 检索确认（✅ 条目级），正文未深读。
- ⚠️ "Whispering GPTCache: Unveiling Vulnerabilities in LLM Cache Side Channels"：多组关键词在 arXiv API（ti/all: "GPTCache"、"Whispering"+"cache"）中均未命中该标题——**无法核实其 arXiv 编号**（2402.02751 已排除）。讲义引用时标注"存疑/未验证"，或改引上述两篇。
- 缓解（教学口径）：按**用户/租户分命名空间**缓存；缓存键里掺入权限上下文（user/tenant id）；敏感语料禁用全局语义缓存；对命中做 ACL 复核。

### 5.2 OWASP Top 10 for LLM Applications（2025）

- ✅ 官方 Top-10 页（十项全名）：LLM01 Prompt Injection / **LLM02 Sensitive Information Disclosure** / LLM03 Supply Chain / LLM04 Data and Model Poisoning / LLM05 Improper Output Handling / LLM06 Excessive Agency / LLM07 System Prompt Leakage / **LLM08 Vector and Embedding Weaknesses** / LLM09 Misinformation / LLM10 Unbounded Consumption。来源：https://genai.owasp.org/llm-top-10/ （✅）
- ✅ LLM08 详情页（RAG 专属风险，逐条）：共享向量库中"**context leakage between users or queries**"（跨租户/跨用户上下文泄露）；"Weak or misaligned access controls"（对 vector/embedding DB 的越权访问）；**embedding inversion**（"invert embeddings and recover significant amounts of source information"）；数据/RAG 中毒（含隐写注入案例："Ignore all previous instructions… 以白字写入简历"）；缓解明点名："**Permission-aware vector stores** with fine-grained access control and strict logical partitioning between user groups/tenants"、检索活动不可变审计日志。来源：https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/ （✅）
- LLM02（Sensitive Information Disclosure）与 LLM01（间接提示注入：被检索文档携带指令）与本文主题强相关（Top-10 页 ✅，详情页未单独抓取）。

### 5.3 索引投毒 / RAG 投毒（index poisoning / RAG poisoning）

- ✅ arXiv:2402.07867《PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models》（Zou, Geng, Wang, Jia；USENIX Security 2025）：
  - 声明为 "the first knowledge corruption attack to RAG"；把投毒建模为优化问题（黑盒/白盒构造恶意段落）；**知识库百万级文档中每目标问题仅注入 5 条恶意文本即可达约 90% 攻击成功率**；现有防御不足。来源：https://arxiv.org/abs/2402.07867 （✅）
- ⚠️ 相关（未逐一抓正文，搜索/API 可见）：arXiv:2401.06377《Poisoning Retrieval Corpora by Injecting Adversarial Passages》等——如需引用请先核实。

### 5.4 GraphRAG 的权限/投毒问题（shared entities 泄露）

- ✅ arXiv:2501.14050《GraphRAG under Fire》（IEEE S&P 2026）：
  - 关键发现：传统 RAG 投毒对 GraphRAG **反而更难**（图索引/检索的天然鲁棒性），但图的**共享关系**开辟新攻击面——**GragPoison** 利用"多条查询共享同一 relation"，**一段投毒文本即可同时污染多个查询**（relation injection / relation enhancement / narrative generation 三策略），成功率至高 98% 且所需投毒文本不到传统攻击的 68%。
  - 对权限议题的启示：知识图谱把不同来源文档的实体/关系**合并进同一图**——实体级 ACL 无法简单映射到共享边/社区摘要上；一个用户对某文档有权限 ≠ 对其牵出的跨文档合并社区有权限。这是 GraphRAG 权限建模尚未有官方方案（Microsoft GraphRAG 文档未提供 ACL 支持，⚠️）的直接证据；投毒侧一手证据即本篇。
  - 来源：https://arxiv.org/abs/2501.14050 （✅）
- ⚠️ GraphRAG 权限相关的其他一手来源（KGP 权限、社区 ACL 模型）本次未找到权威出处；arXiv 检索 "GraphRAG+access control" 0 命中（✅ 检索已做）。

### 5.5 权限漂移 / 陈旧 ACL / 延迟传播（permission drift / stale ACLs / late propagation）

官方文档承认的漂移点（全部 ✅）：
- Azure：源权限变更"only reflected in search results **after that metadata is synchronized to the index**"（indexer 节奏决定）。
- Bedrock KB："Group memberships are as fresh as the last sync"；IdP 凭据缓存至多 1 小时；"eventually consistent… within a few minutes"；对策即 **real-time ACL verification**（查询时回源验证，代价是每次检索多一跳源系统调用——延迟/配额成本）。
- Elastic connector：ACL 靠 **Access control sync** 定期任务；多索引方案里"keys must be updated if user permissions change"（API key 过期/轮换管理）。
- Milvus RBAC：`REVOKE` 立即生效（内存缓存失效机制），但**自行实现索引层 ACL 时**，payload 里冗余的 ACL 字段要靠增量更新管道维护——同步窗口即泄露窗口。
- 讲义要点：任何"把 ACL 拷贝进索引"的方案都必须回答三个问题：变更事件从哪来（webhook/轮询）、同步 SLA 多长、同步间隙靠什么兜底（fail-closed？实时回源验证？）。

---

## 6. RAG 授权架构模式（Architecture patterns）

### Pattern A：共享索引 + 强制 ACL payload 过滤（pre-filter before ANN）
- 适用：租户多、单租户文档少。机制：每个 chunk 的 payload 冻结 ACL 元数据（owner/group_ids/doc_id），**每次查询必须**带 ACL 过滤并命中 payload 索引。
- 一手依据：Azure security filter（`search.in` sub-second，✅）；Qdrant（payload index + `is_tenant`，✅）；Milvus partition-key（✅）；Postgres RLS（policy 先于查询条件逐行执行，✅）。
- 关键点：**ACL 过滤必须是 pre-filter（在 ANN 遍历前生效）**。Qdrant 官方（filtering 页 ✅）明示为过滤字段建 payload 索引、且建议**摄取前**建好；其 HNSW 为 filter-aware（过滤图遍历），避免"先 ANN 后过滤"。
- Post-filter 的双重坑（讲义必讲，⚠️ 综合各家文档的一致性结论）：
  1. **召回塌陷/空结果**：先取 top_k 再过滤，被过滤掉的命中不回填 → 结果远少于 k 或为 0（Bedrock KB 的 partial results / no backfill 是官方对"验证后过滤导致 k 不足"的明示，✅）；
  2. **结果不可预测**：不同用户的同一问题看到不同的"有效语料规模"，评测与排序漂移。
  - ⚠️ 注：Qdrant 现行 concepts/filtering 页已不展开 pre/post-filter 论述（正文已迁移），教学中用其 payload index 建议 + Bedrock partial-results 官方描述佐证。

### Pattern B：per-tenant index / collection / namespace（硬隔离）
- 一手依据：Weaviate 多租户（tenant=shard，互相不可见，✅）；Pinecone namespace per customer（✅）；Vectara corpus isolation（✅）；Milvus collection/database 级（✅）；Qdrant collection-per-tenant "rarely the most efficient"（✅）。
- 取舍：隔离最彻底（越权=无法寻址）、缓存/审计天然分界；成本是碎片化管理、跨租户查询难、小租户资源浪费（Weaviate 用 ACTIVE/INACTIVE/OFFLOADED 分层缓解，✅；Qdrant/Pinecone 有数量上限，✅）。

### Pattern C：per-user index（罕见，成本型）
- 仅 Weaviate autoTenantCreation/Pinecone 自动建 namespace 这类"海量小分区"机制可低成本支撑；一般只在 B2C 强隐私场景。Qdrant 官方把它归类为 collection-per-tenant 反模式（✅）。无独立厂商文档页，⚠️ 定位为"模式命名"而非功能。

### Pattern D：Proxy/Policy 层重写查询过滤（"authorization as a filter"）
- 机制：OpenFGA/SpiceDB 在网关处把 `user→documents` 关系解析为过滤表达式，**改写**进向量查询；索引里不存 ACL（或只存关系无关字段）。
- 一手依据：OpenFGA 的 `ListObjects`/`Check` API（✅）、SpiceDB `LookupResources`（✅）——把"授权判定"输出为可下推的 id 集合；与 §2 的 ES template/query、Azure `search.in` 同构。
- 优点：ACL 与索引解耦，权限变更零同步（判定实时）；缺点：id 集合可能巨大（需组级收缩/分页过滤）、多一跳授权服务延迟（Zanzibar 生产 p95 <10ms，✅ 可引用其规模指标）。
- 失效模式（务必写入讲义）：忘记注入 filter = 全量泄露（Vectara 原文 "No filter supplied = no restriction"，✅）；filter 由 LLM 生成时必须服务端 pin（Vectara `argument_override`，✅）。

### 生成后校验（post-verification）
- Bedrock KB 的 real-time ACL verification 是"查询时回源复核"的官方实现（✅）；Kendra 官方免责声明 + Bedrock "ACL awareness is not authorization"（✅）共同确立：**最终授权永远在"已验证身份 + 源权限系统"，检索过滤只是性能与深度防御层**。

---

## 7. 企业连接器 ACL 同步（Connector ACL sync）

统一管线（各家官方文档拼合）：

```
源系统（SharePoint/Confluence/Google Drive/ADLS/…）
  → 连接器爬取内容 + 权限（ACL crawling / permission metadata）
  → 身份规范化（email 为全局键；组映射：Q Business User Store UUID、
     Kendra PutPrincipalMapping、Bedrock 实时组解析、ES .search-acl-filter- 索引）
  → chunk 级落盘（Azure: index projections 把 ACL 投影到每个 chunk；
     ES: _allow_access_control 字段；Bedrock: 文档级 ACL 元数据）
  → 查询时过滤（§2 各机制） + 同步与漂移兜底（§5.5）
```

逐家要点（均 ✅，来源见 §2）：
- **SharePoint**：Azure AI Search SharePoint ACL preview（unique permissions 增量、继承权限需显式 refresh）；Kendra 的 site 本地组映射规则 `SHA256(site)|localGroupName`；Bedrock SharePoint 连接器（ENTRA_ID_APP_ONLY，2LO，支持 real-time verification）。
- **Confluence**：Bedrock 爬 Space permissions / Page restrictions（嵌套继承）/ Blogs / Attachments；需 Atlassian 组织管理员 API（`adminApiKey` + `organizationId` + `directoryId`）做身份爬取与实时校验；Kendra 要求 Confluence 用户 email visibility = **Anyone** 才能正确过滤（易踩坑）。
- **Google Drive**：Bedrock 用 domain-wide delegation（SERVICE_ACCOUNT，2LO）；Kendra 把 user/group/domain email 映射到 `_user_id`/`_group_ids`。
- **ADLS Gen2 / Blob**：Azure "POSIX-like ACL / RBAC scopes" preview（permissionFilter 字段 + x-ms-query-source-authorization）。
- **S3 / 自定义**：ACL 由客户元数据文件提供（Kendra `.metadata.json` ACL、Bedrock S3/Custom ACL 配置文件）——天然**无实时验证**，同步间隙风险由客户承担（Bedrock 原文明示）。
- **同步事件源**：各家均以"周期性 crawl/indexer run"为主（Elastic Access control sync、Azure indexer run、Bedrock crawl），**权限变更 webhook 级实时同步在主流托管检索服务里仍未成为默认**——这是课程里"增量更新权限漂移"的现实依据（全部 ✅ 有出处）。

---

## 8. 快速结论（教学提炼）

1. **访问控制必须在检索层强制执行**：微软 Copilot（"only surfaces … at least view permissions"、Semantic Index 身份边界）与 Azure（"filtering inside the search pipeline is faster than … trimming"）给出权威背书。
2. **"过滤 ≠ 授权"是所有 AWS 官方文档的统一免责**（Kendra、Bedrock KB），认证与最终授权在应用与源系统；fail-closed（无 userContext → 0 结果；无 ACL → 视为受限）是正确默认。
3. **pre-filter 是硬要求**：ACL 过滤要与 payload/partition 索引配合在 ANN 前生效；post-filter 会同时造成召回塌陷与空结果。
4. **索引隔离（Pattern B）最省心但最贵；共享索引 + ACL 下推（Pattern A）是默认工程解；策略层重写（Pattern D）适合复杂关系授权（Zanzibar 系）**。
5. **漂移管理决定真实安全性**：同步窗口 = 泄露窗口；实时回源验证（Bedrock）与失效安全默认值（ES 空字段=全员不可见）是两种官方兜底范式。
6. **RAG 特有风险有论文与标准可引**：OWASP LLM02/LLM08（跨租户上下文泄露、embedding inversion、投毒）、PoisonedRAG（5 条文本 ~90% ASR）、GraphRAG under Fire（共享 relation 放大投毒）、语义缓存键碰撞/时序侧信道。
7. **平台漂移风险真实存在**：Vectara 从原生文档 ACL 收缩为"平台不强制、后端拼 filter"；Amazon Q Business 停止新客；Vertex AI Search ACL 文档迁移中——选型必须把"权限功能的生命周期"纳入评估。

---

## 附：来源清单（✅ = 本次抓取验证；⚠️ = 未验证/链接迁移）

| # | 来源 | URL | 状态 |
|---|------|-----|------|
| 1 | Azure AI Search – Security Filter Pattern | https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search | ✅ |
| 2 | Azure AI Search – Document-Level Access Control 概览 | https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview | ✅ |
| 3 | Microsoft Copilot – Data, Privacy, and Security | https://learn.microsoft.com/en-us/microsoft-365/copilot/microsoft-365-copilot-privacy | ✅ |
| 4 | Elasticsearch – Document & Field Level Security（核心） | https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/controlling-access-at-document-field-level | ✅ |
| 5 | Elasticsearch – Connector DLS overview | https://www.elastic.co/docs/reference/search-connectors/es-dls-overview | ✅ |
| 6 | OpenSearch – Document-Level Security | https://docs.opensearch.org/docs/latest/security/access-control/document-level-security/ | ✅ |
| 7 | Amazon Kendra – Filtering on user context | https://docs.aws.amazon.com/kendra/latest/dg/user-context-filter.html | ✅ |
| 8 | Amazon Q Business – User Store / Principal mapping / ACL Analyzer / What-is | https://docs.aws.amazon.com/amazonq/latest/qbusiness-ug/connector-principal-store.md 等 | ✅ |
| 9 | Bedrock KB – ACL awareness enablement | https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-acl.md | ✅ |
| 10 | Bedrock KB – ACL-aware retrieval（userContext） | https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve-acl.md | ✅ |
| 11 | Bedrock KB – Confluence Document-level access controls | https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-ds-confluence-acl.md | ✅ |
| 12 | Google Vertex AI Search – About ACLs（迁移中） | https://cloud.google.com/generative-ai-app-builder/docs/about-acls | ⚠️ |
| 13 | Weaviate – Multi-tenancy | https://docs.weaviate.io/weaviate/manage-collections/multi-tenancy | ✅ |
| 14 | Weaviate – RBAC Overview | https://docs.weaviate.io/weaviate/configuration/rbac | ✅ |
| 15 | Weaviate – llms.txt（版本/能力摘要） | https://weaviate.io/llms.txt | ✅ |
| 16 | Qdrant – Security / JWT RBAC | https://qdrant.tech/documentation/security/ | ✅ |
| 17 | Qdrant – Filtering（payload index） | https://qdrant.tech/documentation/concepts/filtering/ | ✅ |
| 18 | Qdrant – Multitenancy（is_tenant 等） | https://qdrant.tech/documentation/guides/multiple-partitions/ | ✅ |
| 19 | Milvus – Multi-tenancy | https://milvus.io/docs/multi_tenancy.md | ✅ |
| 20 | Milvus – RBAC Explained | https://milvus.io/docs/rbac.md | ✅ |
| 21 | PostgreSQL – Row Security Policies | https://www.postgresql.org/docs/current/ddl-rowsecurity.html | ✅ |
| 22 | Supabase – Row Level Security（RAG/pgvector 实践） | https://supabase.com/docs/guides/database/postgres/row-level-security | ✅ |
| 23 | Vectara – ABAC（现行） | https://docs.vectara.com/docs/security/authorization/attribute-based-access-control | ✅ |
| 24 | Vectara – Authorization overview | https://docs.vectara.com/docs/security/authorization/authorization-overview | ✅ |
| 25 | Pinecone – Indexing overview / namespaces | https://docs.pinecone.io/guides/index-data/indexing-overview | ✅ |
| 26 | USENIX – Zanzibar | https://www.usenix.org/conference/atc19/presentation/pang | ✅ |
| 27 | OpenFGA – docs/llms.txt（ReBAC/Zanzibar 概念） | https://openfga.dev/docs 、https://openfga.dev/llms.txt | ✅（llms.txt 链接清单）/ ⚠️（正文 JS 渲染未抓全） |
| 28 | SpiceDB – docs | https://authzed.com/docs/ | ✅ |
| 29 | OWASP – Top 10 for LLM Apps（2025） | https://genai.owasp.org/llm-top-10/ | ✅ |
| 30 | OWASP – LLM08 Vector and Embedding Weaknesses | https://genai.owasp.org/llmrisk/llm082025-vector-and-embedding-weaknesses/ | ✅ |
| 31 | PoisonedRAG（USENIX Sec'25） | https://arxiv.org/abs/2402.07867 | ✅ |
| 32 | GraphRAG under Fire（S&P'26） | https://arxiv.org/abs/2501.14050 | ✅ |
| 33 | The Early Bird Catches the Leak（时序侧信道） | https://arxiv.org/abs/2409.20002 | ✅（arXiv API 核实） |
| 34 | Key Collision Attack on LLM Semantic Caching | https://arxiv.org/abs/2601.23088 | ✅（条目级） |
| 35 | "Whispering GPTCache"（待考） | arXiv 2402.02751 **已排除**（实为材料学论文） | ⚠️ 编号未证实 |
| 36 | Permify / Ory Keto | github.com/Permify/permify 、github.com/ory/keto | ⚠️ 未抓取 |
