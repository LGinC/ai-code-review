# L0/L1 技术实施方案（VTR-6，评审稿 v0.1）

> 面向实施代理：由队长按任务卡逐项安排执行与复核；本阶段不启动开发，不创建后续子任务。技能建议的执行子技能不构成本阶段启动开发的授权。

**目标：** 先完成 L0 的真实证据与范围签署，再实现可在真实 PostgreSQL 上验证的 L1 最小正确性内核。

**架构：** 沿用单 Go module、Gin + GORM SQL-first、单 API + 单 Worker；workflow、maintenance、Coordinator 在同一 Worker。只安装 core，所有注册来自同一 release-scope；外部效果、观察事实和业务完成保持独立。

**技术栈：** Go、Gin、GORM PostgreSQL Driver、PostgreSQL、log/slog；固定版本 depguard/架构检查。具体项目依赖版本须在 NOW-01/05 登记，不把本机版本冒充发布锁定。

**唯一设计：** ai-devops-platform-design-v1.7.md；HEAD=f5a815bf08d356a6b64bdf2a2fbe0161d28dda2e；文件 SHA256=f7b8225ad26a5d390b06380fd627e3b292fd609e3c25539b5b62ba75628cbb85。核查日期 2026-09-16。

**状态：** 方案已细化为可评审任务和契约；外部实施事实及本稿第 10 节决策未关闭，不能称为已冻结或准予开发。全部 L0/PG 运行门禁均未执行。本稿不是探针报告、签署、DDL 或产品实现。

## 1. 固定约束与依据

| 约束 | 依据 |
|---|---|
| 当前仅 L0+L1；L2 产品链路、前端和真实 Review 发布排除 | VTR-5 最终 PRD 评论 01a0a858-2a84-7566-a68c-b032723457ff；用户 Q6 评论 01a0a858-c88c-71ac-baa4-62535a8be4ec |
| Scan=DEFERRED；不装配 scan.schedule/run/report/auto_issues；进入 L3 冻结范围时复评 | 用户 Q2；主设计:1262 |
| 现有企业 OIDC + 飞书自定义机器人，具体实例不由代理代选 | 用户 Q3/Q4；主设计:6848 |
| Q5=1 允许技术设计/L0 采集事实，缺失仍是门禁；不是签署授权 | 评论 01a0a856-c09b-794a-92f7-216d64ebbbde |
| 一个状态所有者、所有外部业务写入走 Operation | 主设计:179、:3152 |
| 只安装 core；不提前建 review/notification/SSO 扩展、repair、scan、Intake、Feed | 主设计:4188 |
| AuthContext 贯穿业务读写/执行/查证，形状检查不等于授权 | 主设计:4242、:5246 |
| 不实现完整 Wait/Signal/Timer、YAML、NATS、Temporal、HA、独立 Coordinator | 主设计:2888、:3291、:6823 |
| 数据库失败必须失败；未跑为 not_run，非适用有 scope 原因，不能 skip-green | 主设计:4186、:6697 |
| 当前无既有产品安装或在线表；不虚构旧系统迁移项目 | 当前 git ls-tree；主设计:6866 |

顺序固定：NOW-01 → NOW-02 与 NOW-03 → NOW-04 → NOW-05 → NOW-06。NOW-04 的文档规范属于 L0；它的工程门禁实现属于 L1，必须先通过全部 L0 退出审查。NOW-05 包含最小真实恢复路径及 Harness，不以等 NOW-06 完整 Worker 为由拖后数据库验证。

本稿沿用现有根目录设计文档布局，放在指定隔离 worktree，目标分支 feature/vtr-4-l0-l1。术语另见 CONTEXT.md。未创建第二份架构标准、ADR 或运行 scope；release-scope.yaml 是后续唯一批准清单，不与本稿并行充当运行配置。

## 2. 已采集事实与 L0 缺口

### 2.1 已核验事实

| 来源 | 实际值与边界 |
|---|---|
| git rev-parse HEAD、git ls-tree、git status | 提交与 Stage 0 一致；跟踪文件仅设计/合同/登记 YAML、README、LICENSE、gitignore；没有 go.mod、cmd、internal、迁移、测试、CI |
| 当前主工作区 | main；未跟踪 AGENTS.md 与 .multica/ 保留；无产品代码改动 |
| multica project get / project resource list；.multica/project/resources.json | 项目仅绑定仓库及本地目录；lead_id=null；未提供环境证据位置 |
| 原生 OCR --version、Get-FileHash | 本机候选 v1.12.1 (1f5caf4d5)，windows/amd64；built 2026-09-14T11:09:44Z；二进制 SHA256=6033c3af93a8da41ab73d6013c562b57d197bebf614e79904410c1c4359ce6a1 |
| OCR 原生文件来源 | npm 安装包 @alibaba-group/open-code-review → @alibaba-group/ocr-win32-x64/bin/opencodereview.exe；只记录本机事实，非获准 Runner 发行物 |
| OCR review --help | 存在 --from/--to（merge-base 语义）、--format json、--output、--repo、--audience agent、--color never、--timeout、--concurrency、--max-tokens-budget |
| OCR 非交互证据 | help 没有 --non-interactive/--yes；--audience agent 只是摘要输出模式，未证明配置加载、凭据与非交互执行；未运行 Review |
| 工具发现 | git/go/docker/multica 可发现；psql 未在 PATH；没有据此推断 Docker daemon、PG 服务或目标系统可用 |
| 设计:5615、:5872 | issuer、redirect_uri、Secret/Profile、飞书 destination 均明确是示例；不能从 .example 或示例 ready 字段推出已部署 |
| frontend-build-L2-registry.yaml:21、:38 | 前端签署字段为空，只能证明 L2 尚未冻结，不能充当 G02/G04 签署 |

实际源码来源完整 commit、候选 Runner 平台及 digest 尚未固定。npm launcher 含更新检查逻辑，后续探针调用批准的原生二进制或不可变镜像，避免 launcher 变更扰动版本。不得使用当前文档仓 HEAD 自动充当 OCR 样本提交。

### 2.2 G01～G04 退出记录

| 门禁 | 必须收集/冻结 | 当前事实与缺口 | 职责与阻断 |
|---|---|---|---|
| G01 Review | release/source commit、binary hash、Runner digest、固定样本仓与 base/head SHA、规则/模型/配置 hash、执行身份/出站范围、argv、stdout/stderr/exit code、结构化成功与失败原件、解析版本、探针预算 | 只有本机版本/hash/help；批准发行物、镜像、样本、运行配置、负责人及真实结果未找到 | OCR 探针执行人具名缺失；阻断 NOW-02 退出和全部 L0 完成 |
| G02 Scan | decision_owner、technical_owner、signed_at、scope_manifest_revision、reason、excluded_capabilities、revisit_trigger | 用户选定 DEFERRED；未找到绑定具体 scope revision 的具名产品/技术双签记录 | 产品/技术签署人具名缺失；不能把 Q2 选择或代理署名当完整签署 |
| G03 外部范围 | 实际 IdP Provider/版本或托管服务版本策略、issuer/Discovery、精确 callback、客户端类型、claims/准入规则、停用语义、测试环境与账户负责人；飞书模式、目标引用、安全设置、发送/查证/验证角色与能力证据 | 已选企业 OIDC、飞书自定义机器人；真实值与证据未找到 | OIDC 环境负责人、飞书管理员、查证身份责任人具名缺失；阻断 NOW-03 退出 |
| G04 工程纪律 | 单 scope revision、目录/表 owner、注册清单、等待评审模板、PG-G 计划、真实 PG/角色/版本、阈值负责人、各门禁退出条件、签署引用 | 本稿给出执行规则与测试计划；批准清单、具名 owner、PG 环境和阈值未找到 | 后端/DevOps/测试及 PG 阈值 owner 需队长落实；阻断 NOW-01/04 冻结及 L1 开发 |

G02 签署原因应由签署人确认：本次仅 L0/L1，扫描不在交付范围；候选尚未完成扫描实验，不称不支持。进入 L3 时继续排除则更新签署，选择自动扫描才运行 SCAN-P01～06。DEFERRED 不授予 scan.report，也不要求本次运行 SCAN-P。

G03 飞书不具备 Lookup/回执时允许如实冻结 unsupported/ack_only；不要求伪造不存在的“只读机器人凭据”。实际能力没有查证接口时记录人工查证责任与证据渠道，unknown 不能自动重发。发送凭据与能够提供的只读查证角色分开；任何秘密只存批准 Secret 管理系统，本稿和 issue 只接收非秘密引用。

### 2.3 证据交付约定

沿用根目录登记模式：release-scope.yaml、l0-evidence-index.yaml 为后续唯一 scope 与证据索引，具体原件以受控附件/制品引用交付。索引逐条存 gate_id、scope revision/hash、执行人、UTC 时间、输入/输出 hash、命令版本、结果（pass/fail/not_run/blocked）、证据引用和审核人；签署绑定准确 revision/hash。敏感 stdout 原件限制访问，公开摘要脱敏，不能把脱敏空文件当原始执行成功。

阈值登记必须有环境/数据规模、并发槽/连接预算、P95 与最大等待、恢复扫描上界、负责人、批准时间及原始报告引用。G04 冻结测试输入/负责人/判定方法；进入公平性验收前锁定数值，不能看完结果后移动阈值。初始实验建议值见第 6 节，尚未批准。

## 3. 模块、文件与表归属

下表是实施职责归属，具名负责人与审核人由 NOW-01 登记。一个文件只有一个修改负责人；架构师仅审阅文档，后续产品代码由后端负责人实现。

| 文件范围（后续创建） | 唯一职责/输出 | 依赖与表写权 |
|---|---|---|
| cmd/api-server/main.go、internal/delivery/ginhttp/server.go | Gin 生命周期、绑定/恢复/上限/超时、部署探测；不运行 OCR/SQL | delivery → application/ports；无表直写权 |
| cmd/worker/main.go、internal/bootstrap/api.go、worker.go、registry.go | 唯一组装点，单 Worker 启动与关闭，核验已安装清单/Schema hash/所有注册依赖 | 可导入具体 Adapter；只组装不拥有业务状态 |
| internal/domain/identity/auth.go、internal/application/identity/authorizer.go | 可信身份、当前权限与决策边界；L1 受限 workload，human 测试身份仅测试 binary | tenant/users/memberships、Integration 准入与策略只由 identity 应用入口变更 |
| internal/domain/externalop/{operation,attempt,observation}.go；internal/application/externalop/{planner,coordinator,watch}.go | 原效果键、合法转移、三类领取、封存/迟到事实、已知 Job 观察 | external_operations/attempts/watches/observations、resource_write_leases、operation_effect_links |
| internal/domain/workflow/{run,step,event}.go；internal/application/workflows/{registry,checkpoint,recovery}.go | 固定 Handler 注册及短步骤检查点；本地引用恢复；不含私有调度线程 | workflow_runs/step_runs；恢复器只读本地投影 |
| internal/ports/{auth,unitofwork,operation,queue,workflow,schema}.go | 标准 context、领域 DTO 和显式事务端口，无 Gin/GORM/SDK 类型 | ports → domain |
| internal/adapters/storage/gormpostgres/{unitofwork.go,identity/,workflow/,externalop/,outbox/} | 唯一 SQL-first Repository；参数化 SQL 放所属 sql/；UoW 绑定同连接 | 只作为所属 owner 的持久化实现；无跨域任意 Save |
| internal/adapters/queue/postgres/{queue,relay}.go | Outbox 路由与逐订阅者 delivery、ACK/重投 | outbox_events 的路由字段、outbox_deliveries、consumer_receipts；不判定业务完成 |
| internal/adapters/schema/{registry,validator}.go | 内嵌静态 Schema、严格解码/hash/quarantine | 不允许网络 $ref 或动态加载执行逻辑 |
| internal/adapters/system/{clock,secrets}.go；storage/gormpostgres 下 artifact/audit 持久化 | 受信时间、按身份/目标取密钥引用、最小制品/审计 | artifacts/audit_events；未提供真实 Secret 服务则外部能力不可启用 |
| architecture/{modules,exceptions}.yaml、handler-review.md；.golangci.yml | 唯一目录/表 owner、禁边、到期例外、等待事实模板 | 后端边界实施人修改，架构角色复核 |
| tools/{archcheck,check-architecture-fixtures}/main.go | depgraph/AST/目录/传递边界及正反 fixture 校验 | 无业务 Repository；读取真实 module prefix |
| db/migrations/core/{manifest.yaml,0001_identity.sql,0002_workflow_outbox.sql,0003_externalop.sql} | 有序 SQL/checksum/约束/索引，scope 只选择 core | 由后端数据库负责人独占；DevOps 不修改同一迁移 |
| tests/architecture/、tests/contract/、tests/integration/、tests/security/、tests/fixtures/ | 一套参数化机械 Harness 与隔离测试 Provider | 测试负责人独占，允许真实 Adapter 测试依赖，禁止全局 _test.go 排除 |

现阶段不创建 cmd/runner-controller 产品服务；最小 Job 观察通过测试 Provider/Job 本地事实夹具验证，实际 Runner 与 Review 业务在 L2。测试 Provider 不注册进生产 binary，不能以 L1 测试通过声称真实 OCR/GitHub/飞书集成完成。

### core 最小表与约束

| 表/组 | 关键字段和必须的约束 | 理由与边界 |
|---|---|---|
| tenants、users、memberships、integrations、policy_versions、模块登记 | UUID；业务关联带 tenant；membership 与策略版本可复验；Integration 只存 Secret 引用；模块名唯一且有迁移 checksum | 不装 OIDC Session/SSO 扩展产品；模块登记存安装事实，不等同能力授权 |
| workflow_runs、step_runs | definition/version/hash、逻辑 step、effect slot/generation、input hash、deadline、state_version/owner epoch；同 tenant FK；状态 CHECK | 将主设计示例欠缺的稳定效果字段/期限补入正式迁移；机械 attempt 不改变身份 |
| api_idempotency_records | unique tenant+principal+operation+key_digest；request_hash、canonicalizer/hash key 版本、receipt schema、replay/tombstone 窗口 | 同键异 hash 冲突；回放时重新授权，不存敏感响应正文；不反向引用 Intake |
| outbox_events、outbox_deliveries、consumer_receipts | typed envelope、payload/hash；event+subscriber+route generation 唯一；receipt tenant+subscriber+event 唯一；lease epoch/到期/错误原因 | routed、传输确认和业务消费分开；保留路由世代，不靠 MAX(id) |
| external_operations | 主设计:3652 的字段；unique tenant+integration+type+key；所有 parent/workflow/artifact 同 tenant FK；confirmed 必须证据；sending 必须租约 | 未知不能通过状态变化释放写入权；不执行整章 DDL |
| external_operation_attempts | unique tenant+operation+attempt_no；action 枚举、epoch、request hash、开始/封存时间/证据 | 开始先落库，封存单次 CAS；无 UPDATE 覆盖封存事实 |
| external_watches、external_observations | Watch 绑定 tenant+integration+target+purpose+授权范围+generation 唯一；观察版本/来源/证据/时间、同租户关联 | 只观察已知 Job；未知提交由 Operation 查证；迟到事实追加 |
| resource_write_leases、operation_effect_links | tenant+integration+resource_write_key 唯一；owner/operation FK、epoch；活动效果槽受唯一约束；workflow/step owner 采用显式同租户关联 | owner_kind/id 字符串不是 FK；独立 core 操作可自持账本身份，不伪造 Workflow；L2 再加具体业务关联 |
| artifacts、audit_events | tenant、hash/size/classification、存储引用、actor/action/decision/版本/时间；制品引用同 tenant | 大原件不进任务表；活跃 unknown/观察/取证依赖不能被普通 TTL 删除 |

不得新建 L3 Remediation 来通过 PG-G09；用核心资源租约/操作模型。枚举/复合键/部分唯一索引都进入正式 SQL，GORM 标签不能代替。无在线安装，0001～0003 是新库计划，不宣称旧版本 downgrade 可以删除数据；兼容回退指旧兼容应用读取新增结构，已使用字段不得自动 DROP。

关键索引：external_operations_due_idx、external_operations_sending_expiry_idx（主设计:3695）；step_runs_due_idx、outbox_pending_idx、delivery 到期+订阅者索引、Watch 到期+类型索引、幂等唯一键与 GC 索引。索引 predicate 不使用 now()；时间谓词放查询。公平领取 SQL 必须包含 tenant/type/action 与对应状态条件；按 action 的索引增补以代表性 EXPLAIN 为依据。JOIN ON 不函数包裹索引列，不 CAST 业务键来补类型错误。静态 Registry/Schema 在启动时内嵌缓存，动态授权不可无期限缓存。

## 4. 内部契约、HTTP 边界与事务

### 4.1 沿用的类型化端口

主设计第 23 章为类型字段唯一依据，按下表抽取实际使用的类型；不复制全产品 ports。下列签名中的 AuthContext/OperationSpec 等均采用主设计定义，字符串状态在所属 domain 中收紧为枚举。

| 文件/端口 | 输入 → 输出与约束 |
|---|---|
| ports/auth.go：AuthContext | 主设计:4275；tenant/principal/kind/scope/decision/policy/可选 grant version/expiry；禁止从请求 JSON 绑定，空范围授予零权限 |
| ports/operation.go：OperationPlanner | Plan(context.Context, AuthContext, OperationSpec) (OperationReceipt, error)；Get(context.Context, AuthContext, string) (OperationReceipt, error)；RequestReconcile(context.Context, AuthContext, string) error；均来自主设计:4925 |
| OperationExecutor | Execute(context.Context, AuthContext, AuthorizedOperation) (ExecutionEvidence, error)；仅当前有效 permit/epoch；网络在事务外 |
| OperationReconciler | Lookup(context.Context, AuthContext, LookupRequest) (LookupResult, error)；Inspect(context.Context, AuthContext, InspectRequest) (RemoteObservation, error)；只读、无私有循环/写库 |
| ports/queue.go：Queue | Publish(context.Context, WorkMessage) error；Consume(context.Context, WorkSubscription) (WorkDelivery, error)；Ack(context.Context, DeliveryToken) error；RetryDelivery(context.Context, DeliveryToken, time.Time) error |
| ports/workflow.go：StepHandler | Execute(context.Context, AuthContext, StepContext) (StepResult, error)；结果必须属于固定定义，L1 不接受 WaitSpec 或任意 YAML |
| Runner 读契约 | Get(context.Context, AuthContext, RunHandle) (RunState, error)；L1 仅测试实现，不获得提交权 |
| EventSchemaValidator | Validate(context.Context, string, string, json.RawMessage) error；两个string依次为注册subject/type与schema version，沿用主设计:3924，校验失败不能写/发/消费事件 |

OperationSpec 必填：type/key/integration、已注册 owner 及 revision、logical effect/generation、request schema/artifact/hash、policy/authorization、resource write key、confirmation profile/deadline；WorkflowID 可空。request hash 绑定规范化 payload、目标、策略/授权与确认条件；不含 Worker/Attempt。OperationReceipt 含 ID/state/version、获准展示的外部引用和证据；不返回任意目标 URL 或秘密。

新增的内部 Authorizer/UoW 实现约束由 NOW-04/05 承接：Authorizer 从认证成功的 principal 或已登记 workload 与持久化授权引用重新构造可信 AuthContext，检查当前对象/动作/成员/策略/grant；UoW 向回调提供同一 tx 的 Repository、OperationPlanner 与 Outbox Writer。具体 Repository 方法以本节事务行的输入/输出为实现合同，不开放通用 Save(model) 或全局 db。没有真实登录时，生产业务入口默认不注册；不能增加“传 Header 指定用户”的后门来演示 L1。

### 4.2 唯一事务边界

| 事务 | 输入/并发条件 | 原子提交结果与失败处理 |
|---|---|---|
| T1 计划效果 | 当前 AuthContext、固定对象/输入、原业务幂等键、request hash | 业务短检查点+Operation+typed Outbox 同 tx；ON CONFLICT 后下一条 SELECT 重新取快照并核对 hash；任何错误整体回滚 |
| T2 领取 prepared | 已授权 tenant/type、可用执行槽、到期、状态/epoch、资源租约 | CAS 到 sending、epoch/lease、execute Attempt 一起提交；提交成功后再复验 permit/授权并调用 Execute；确认未提交则不得执行，提交未知先重查原账本 |
| T3 记录调用结果 | 原 operation/attempt、expected epoch/version、目标/hash/证据 | 封存 Attempt、合法状态转移、证据、typed Outbox 同 tx；CAS=0 不发成功事件，迟到事实经独立授权路径追加 Observation |
| T4 过期 sending | recovery workload、已获准 tenant/type、到期 lease、有界批次 | 主设计:3746 的恢复 SQL；sending→unknown、epoch+1、旧 Attempt 封存为 lease_expired_effect_unknown、审计/Outbox 同 tx；失败整体回滚；不查远端 |
| T5 unknown 查证 | reconcile_read 身份、lookup/inspect 槽、原 key/hash、到期 | 先登记只读 Attempt；事务外查证；结果按 epoch/目标/确认谓词回写；无权限则 unknown+blocked_reason；不能自动释放资源 |
| T6 Watch/领域消费 | 已知目标、授权范围、观察版本、领域 expected version | 观察事实先持久化；领域 owner 在独立短 tx 以 operation_id+observation_version+handler_id 去重，更新自己的投影与 Outbox；可重放弥补两事务间崩溃 |
| T7 Relay/delivery | 未路由/到期事件、固定订阅者/路由版本 | 校验原 Schema/hash后同库幂等建 deliveries 并标 routed；消费持久化状态/下一步/no-op和receipt后 ACK；ACK 失败可重投但不重复业务 |
| T8 短步骤恢复 | tenant/workflow/step/generation、固定 definition、已持久化本地引用、版本/epoch | 只读本地 Job/Operation事实并 CAS 更新检查点；事件先到则读现状，提示丢失则同一有界扫描恢复；不私查 Provider、不重置 deadline |

READ COMMITTED 为普通短事务基线。23505 后整笔回滚，不在失败事务继续查询；40001/40P01 分类后复用原键重试整个短事务。lock_timeout、statement_timeout、ctx deadline 都有界。网络 COMMIT 回执断开：返回可解释的未知提交失败，重试核查原键，不创建新 generation。不用同 statement INSERT-CTE-SELECT 的空行当作新建许可。UoW 回调不得访问全局 db 或网络。

### 4.3 HTTP 契约草案（等待 Q3）

L1 不注册 /auth、/api/v1/reviews、通知测试、Operation 管理或任意业务写 API；主设计第 24 章是分阶段蓝图。没有前后端协作或新增多服务产品契约。本次 Go 端口供同一二进制/仓库内部协作；真实 OIDC 回调的“冻结”属于 G03，不代表实现。

拟仅在部署管理入口注册下列探测路由，尚未获准公开合同，待 Q3 确认；不修改现有 L2 OpenAPI/client/前端登记。

| 路径 | 请求 | 响应 | 业务/安全规则 |
|---|---|---|---|
| GET /healthz | 无 body、无 query | 200 application/json，status=ok | 只反映进程存活；固定静态正文，不给实例/依赖/版本或身份资料；仅管理网络可达 |
| GET /readyz | 无 body、无 query | 200 status=ready；未就绪 503 status=not_ready | 检查当前 core 迁移/checksum、批准 Registry、DB 有界 ping；不查未安装模块或第三方；失败不输出 DSN/SQL/secret |

若 Q3 选择不新增探测路由，保留 Gin server 生命周期及 bootstrap 必需 DB/Registry 校验，通过进程退出码和同仓 Harness 验证；两条路线均禁止 DB 不可用时假启动成功。管理接口之外的未知/禁用路由不注册。API-server 必须有 request body 上限、trusted proxies、超时、恢复、脱敏；不把 gin.Context 传出请求。

将来 L2 若出现前后端共同修改的接口，队长需在 NOW-08 开始前增加独立合同输出/校验阶段，冻结同一 OpenAPI commit；本任务不创建该阶段、不改原字段、不扩范围。

## 5. Operation/Watch/Handler 与 Schema 注册

### 5.1 机械状态表

| 场景 | 结果 | 严禁 |
|---|---|---|
| prepared 且授权/预算/资源槽有效 | T2 后 sending，事务外 Execute | 先写远端再记 sending |
| Execute 可信确认 | confirmed+证据；业务 owner 再判含义 | 将 confirmed 直接当 Job succeeded 或通知 delivered |
| 明确未执行且可重试 | 原键 prepared+有界退避，执行前重验授权 | 新 operation key 或新业务 generation |
| 超时/5xx/断连/租约到期 | unknown；原键 Lookup，已知主引用 Inspect | 当失败重发 |
| Lookup absent但分页不全/最终一致/旧请求可能迟到 | unknown、有限预算继续读取或人工阻塞 | 404/空页/短等待即允许写入 |
| 完整否定证据且 LateEffectState=ruled_out，或已核验原生幂等仍有效 | 策略/授权允许才同键 prepared | 空字符串/bool 零值当 ruled_out |
| ambiguous/unsupported | 保留 unknown、冻结新写入并审计 | 挑选首个候选、伪造外部 ID |
| 永久拒绝且证实无效果 | failed | 预算耗尽即把未知改 failed |
| 过时/撤销且证实无效果 | superseded | unknown 因取消/期限改 superseded |
| 已确认效果后来过时 | 保持 confirmed，business_relevance=stale | 擦除已经发生的外部效果 |
| 旧 Worker 迟到 | 原 epoch 写回拒绝，授权后追加 Observation | 覆盖封存 Attempt；宣称 epoch 已停止远端请求 |

单写者资源在 unresolved unknown 期间不交给新 writer。取消/补偿是独立获准效果，不是数据库回滚；L1 不实现修复取消产品。授权过期/执行暂停不停止内部过期恢复；只读权限也失效时阻塞查证，而不是借恢复角色取得写权限。

### 5.2 固定注册清单

| 注册项 | L1 允许内容 | 禁止/验证 |
|---|---|---|
| Handler | externalop.recover_expired_sending.v1；prepared/unknown 领取与最小本地引用恢复；测试用固定有界步骤 | 未知 handler/version 启动失败；没有周期 goroutine/sleep/业务私有 polling |
| Operation | 内核支持已注册 type 的执行/查证；生产只装 scope 已验证项；L1 不启用 scm.review.summary/message.send/runner.submit 的真实适配 | 用同一机械 Harness 的 SCM/Runner/message 测试参数验证公平性；测试类型不得进正式 Registry |
| Operation descriptor | type、owner kind、request/receipt/observation Schema及hash、确认 profile、Execute/Lookup/Inspect、原生幂等期限/一致性/迟到证据、退避/预算、write/reconcile_read角色、分区/Watch策略 | descriptor 缺字段/未验收 Provider不启用；不接受运行时 URL/命令/任意 JSON payload |
| Schema | external_operation.observed.v1、本地 run.reference.updated.v1、固定工作提示 envelope；文件由受审注册表引用并内嵌 | type/version/hash 一一匹配；未知/冲突版本隔离，不运行远程 $ref |
| Watch | 已知 Job+purpose+tenant/integration/授权范围/generation；active/paused/completed | 未知提交不得增加 Watch；同目的同范围共享，不同权限不能合并 |
| 固定定义 | L1 测试 binary 内固定定义与短检查点；真实 review.pipeline.v1 留 NOW-07 | 无 YAML 解析、Wait 表、用户脚本/远程 Go 插件；原 definition/hash 固定至终态 |

Schema envelope 必有 event_id、tenant_id、aggregate_id/version、event_type、schema_version、correlation/causation、occurred_at。payload 使用注册 Go struct（含 operation_id/state_version/observation_version 或本地 ref_kind/id/generation 等必要事实），序列化前校验；不提供 map[string]any 事件构造函数。

写入前、Relay 前、消费前三次核验；严格 UUID/时间/枚举/必填/大小/hash、重复 JSON key、未知字段/类型/版本负例。quarantine 保留原始字节引用、失败阶段/原因与审计，不接受被 JSONB 解析后已丢失的重复键作为合法输入。已落库不合法历史事件保留调查证据，不无限重投。领域消费再次鉴权/查对象；Schema 不授予执行权限。旧版本仅按原 Schema 解释，新增 upcaster 必须明确版本、保留原文和测试。

### 5.3 等待评审模板

每个 Handler 登记“等待事实、唯一领域 owner、远端对象是否已知、唯一调度时钟、相关键/generation、提前/重复/乱序策略、超时含义、停止观察条件、当前权限、预算”。L1 正例是无 Wait 表的 Job 引用恢复；L3 同意、PR 合并、停止请求三个例子只作规范 fixture，不提前实现业务链路。

反例必须拒绝：未知 PR 创建同时新建 Watch 并重发；message.read 直接批准；cancel accepted 直接 stopped；每个 Wait 创建重复 PR Watch；Handler 自写周期轮询；恢复器网络查询 Provider；Worker 主循环散落业务路由。静态反例证明边界命中，不能替代 L3 真实故障测试。

## 6. 公平分区、连接预算与恢复

使用有限分区轮转：operation_type + action_class(execute/read)，其内按获准 tenant/Integration 轮转。先保留 SCM/Runner 就绪槽，再借用闲置额度；不能 SELECT 所有到期 LIMIT N 后按类型过滤。先获得可运行槽才领取 lease，不提前占有整批等待外部 HTTP。

| 实验初值（来自主设计:5624、:3299，不是批准阈值） | 控制 |
|---|---|
| execute 总上限4；read总上限2 | execute/read 独立预算 |
| SCM weight4/max2/reserved1；Runner weight4/max2/reserved1 | 不被消息耗尽保留槽 |
| message weight1/max1 | 故障/429只作用本 Integration/type |
| PG 轮询1秒+jitter，领取上限20 | 使用第25章具体初值；第20.9节批100只是通用建议，真实值须由PG负责人锁定 |
| sending recovery 每轮独立 DB batch10 | 即使全部远端槽耗尽仍自动推进；运行暂停时也执行 |
| 公平负载：持续1万条失败消息 + SCM/Runner持续就绪 + 慢回包 | 每类P95/最大就绪等待、跨tenant最大等待、实际领取/槽、恢复延迟均归档 |

API/Worker 分开连接预算；Worker 短控制事务预留连接容量，不在网络等待中占 DB tx。池总连接上限、各SQL超时、恢复扫描上界/P95/最长等待目前未定，签署前不能通过公平性门禁。不额外做多进程 Coordinator 或全局锁；优先索引/批次/退避/配额修正，持续超批准阈值才提拆分 ADR。

HTTP 客户端按 Provider/Integration 的 base URL/TLS/代理/凭据边界复用，有界缓存；每请求注入当前身份，不共用可变 Authorization header。后台 auth/grant 每次复验，SET LOCAL 仅限事务，归还连接后不能遗留租户。RLS 档案由 Q2 决定，关闭 RLS 时同样执行 FK/Repository/对象权限隔离。

恢复路径：启动核验 core 安装与注册 → 不接新写直至DB可用 → 扫描未路由 Outbox、prepared/unknown/expired sending、本地等待引用 → 保留原键/版本/期限 → 只在当前权限与证据允许时执行。历史备份恢复先禁外部写，核对不确定效果；不能清空账本重放。停机停止领取，完成或取消有界本地调用，保留未确认 sending 供下次恢复；不删除执行历史与制品。

## 7. 逐任务实施与验收命令

以下均为后续实施命令合同，不是当前已存在/已运行的工具。真实版本、DSN和秘密由执行环境提供，不在命令写死。测试名称同时固定为实施输出，执行前必须校验 -list 非空以及预期用例数量；仅 exit=0 或 “no tests to run” 不能算通过。架构全图静态门禁必须全量，业务测试默认仅新增/修改目标；本次没有运行任何产品测试。

### NOW-01：冻结范围及职责（L0）

**文件：** release-scope.yaml、l0-evidence-index.yaml；本方案与 CONTEXT.md 作为依据。产品/技术签署人负责批准，队长落实具名 OCR/OIDC/飞书/后端/DevOps/测试/阈值角色。

**输入：** v1.7 hash、已确认 PRD、真实配置证据、Q1～Q3 结果。**输出：** 一个准确 revision/hash 的 scope，milestone=L1、core only、L2 候选范围单独记录、禁用项、依赖版本选择责任人、所需门禁与 owner；未验证候选不注册运行。

- [ ] 核对主设计与本稿 hash；差异时停止受影响冻结并交队长。
- [ ] 填具名职责、环境证据引用与缺口，拒绝通用占位/代理代签。
- [ ] 冻结一份 scope；将既有 L2 Page/build/D 登记作为未来引用保留，不改前端合同。
- [ ] 核对 AC-47/48 仍为原用例，AC-185/186 仅后续 scan，AC-187～206 仅后续复盘；标注 not_applicable 原因，不改编号含义。
- [ ] 取得 scope revision 的批准后提交文档；不启用产品能力。

验收命令（人工核验真实签署内容，hash 非签署）：
    git diff --check
    Get-FileHash release-scope.yaml -Algorithm SHA256
    Get-Content l0-evidence-index.yaml -Encoding UTF8

### NOW-02：Review 探针与 Scan 签署（L0）

**文件：** 只更新 l0-evidence-index.yaml 的 G01/G02 引用；原始输出放受控制品。**负责人：** OCR 探针执行人、产品/技术签署人。**依赖：** NOW-01。

**输入：** 批准二进制/hash/Runner镜像、样本base/head、规则/模型/身份与预算。**输出：** G01 成功+失败真实原件及解析映射；G02 具名 DEFERRED。

- [ ] 固定原生发行物与镜像，重新比对 hash 与 help；不复用 launcher 自动更新后的未知版本。
- [ ] 通过隔离受控环境执行成功样本，记录输入/配置加载路径/完整 argv 与退出状态。
- [ ] 执行经批准失败样本（无效输入/模型失败），保存失败原件并确认不会生成成功报告。
- [ ] 核验结构化输出、非交互终止、私有HOME/报告目录、出站与可信配置；不能将 agent audience 当非交互证明。
- [ ] 产品/技术对 scope revision 签署 DEFERRED，四项 scan 能力排除，复评条件明确；不运行 SCAN-P。
- [ ] 归档证据并提交 G01/G02 索引，失败/阻塞如实登记。

探针命令形状（变量必须在批准探针环境从清单绑定；禁止使用个人 HOME/凭证）：
    & $ApprovedOcrExe --version
    & $ApprovedOcrExe review --help
    Get-FileHash -LiteralPath $ApprovedOcrExe -Algorithm SHA256
    & $ApprovedOcrExe review --repo $FixturePath --from $BaseSHA --to $HeadSHA --format json --output $ResultPath --audience agent --color never

执行器须另设已批准的超时/输出/费用上限并保存 exit code；具体 model/rule/tools 参数只使用固定版本已验证配置。以上变量未绑定或模型/外发范围未批准即阻塞，不运行有费用的 Review。

### NOW-03：冻结实际 OIDC 与飞书契约（L0）

**文件：** l0-evidence-index.yaml 的 G03；同一 scope 的外部候选引用由 NOW-01 owner 串行汇总。**负责人：** OIDC 测试环境负责人、飞书管理员、测试负责人。**依赖：** NOW-01；可与 NOW-02 采集独立原件，但不能并行改同一 YAML。

**输入：** 实际实例/客户端/回调/测试身份/目标与凭据角色。**输出：** 版本绑定的兼容档案，不生成 L2 产品代码。

- [ ] 确认实际 Provider/版本、受信 issuer/端点、精确 callback、PKCE S256/state/nonce 与准入规则。
- [ ] 对已授权测试客户端核对正向回调及 state/nonce/issuer/audience/租户映射/撤权负向证据；不借通用占位绕过。
- [ ] 核对飞书自定义机器人目标、安全设置、发送响应语义、只读查证/回执是否支持；标明 ack_only/unsupported 的真实依据。
- [ ] 发送探针仅在管理员明确批准目标与消息后执行；不在本任务擅自发消息。
- [ ] 固化发送/查证/验证角色和Secret引用，确认不向Runner授予IdP token/SCM写权限。
- [ ] 汇总 G03 证据，环境缺失或协议不满足安全下限时阻塞，不自行替换 IdP。

验收为匹配实际 Provider 的原生命令/账号记录，必须由实际档案携带准确 argv/API 请求和脱敏响应；当前没有目标，不能编造一条可执行联调命令。通用索引核验：
    Get-Content l0-evidence-index.yaml -Encoding UTF8
    Get-FileHash release-scope.yaml -Algorithm SHA256

### NOW-04：边界、注册和等待门禁（先 L0 文档，后 L1 实现）

**文件：** 第3节 architecture/tools/ports/domain/bootstrap/Gin 基线、go.mod/go.sum；tests/architecture/{boundaries,fixtures}_test.go、tests/contract/{registry,schema,health}_test.go。**负责人：** 后端边界实施人、测试负责人；架构复核。**输入：** NOW-01/G04 签署及 G01～G03 结果。**输出：** core 编译边界和实际命中的正反门禁。

- [ ] 文档冻结 modules/表 owner、允许端口、等待模板、Schema/Handler 清单与PG计划。
- [ ] 执行 L0 退出审查：G01/G03/G04真实证据，G02有效DEFERRED；任何不完整停止，不先开发。
- [ ] 建单 module，module prefix 从实际项目仓库规范确定；锁定 Go/Gin/GORM/Driver/lint 版本，禁止复制 example.com/aidevops。
- [ ] 测试负责人建立合法与故意越界 fixture；先证明非法依赖能导致失败。
- [ ] 后端实现 depguard+go list -deps -test -json 图/AST 及 Registry/hash 拒绝逻辑；仅白名单核心启动。
- [ ] 验证 Q3 选定的 HTTP/生命周期契约，鉴权失败不能落入匿名业务路径。
- [ ] 检查本任务新增 fixture 后提交；规则/例外改动由架构复核。

验收：
    golangci-lint config verify
    golangci-lint run ./...
    go run ./tools/archcheck
    go run ./tools/check-architecture-fixtures
    go test ./tests/architecture -count=1
    go test ./tests/contract -run '^(TestRegistry|TestTypedSchema|TestCoreHealth)$' -count=1

反例覆盖 delivery→GORM、domain→Gin、application→adapters、review→intake、helper 传递绕过、alias/dot/blank import、generated/test/build tag 隐藏、平行业务目录。正例必须真实命中；不能因未匹配路径空跑通过。测试引用 review/intake 仅为隔离 fixture，不新增对应产品模块。

### NOW-05：真实 PostgreSQL 最小正确性（L1）

**文件：** core 三份迁移/manifest、gormpostgres UoW/Repos/sql、externalop 最小三类领取与身份复验；tests/integration/pg_gates_test.go、tests/security/auth_context_test.go、tests/fixtures/pg/。**负责人：** 后端数据库负责人、测试负责人；DevOps提供独立PG环境。**依赖：** 全部 L0 退出 + NOW-04。

**输入：** 锁定PG/GORM/Driver/正式迁移、非特权角色、Q2档案、至少双连接与屏障、冻结超时/阈值。**输出：** PG-G01～12报告与最小可恢复内核；不是完整业务交付。

- [ ] 用测试前置条件强制检查 PostgreSQL 产品/version、current_user 权限、正式迁移checksum、两独立backend PID；不满足直接失败。
- [ ] 将主设计 DDL 示例补齐成 core 正式关系/约束，不执行可选表；验证 sending 索引及非空租约。
- [ ] 按第8节逐项先构造能揭示错误的并发/崩溃路径，再实现SQL/事务/CAS，使目标测试通过。
- [ ] PG-G01 此时就实现实际 sending 自动恢复，不延期到完整Worker；真实PG+测试进程产生一次外部效果。
- [ ] 实测 23505/25P02、CAS行数、回滚、COMMIT未知、跨tenant/对象权限、零值更新与索引计划。
- [ ] 冻结报告与迁移 checksum；任何 required 测试 not_run/skip 均不得退出。

验收：
    go test ./tests/integration -list '^TestPGG(0[1-9]|1[0-2])$'
    go test ./tests/integration -run '^TestPGG(0[1-9]|1[0-2])$' -count=1 -timeout=15m
    go test ./tests/security -run '^TestAuthContext$' -count=1
    git diff --check

15分钟为测试命令超时上界，非吞吐/SLO；实际环境需在超时前给出失败诊断。PG 连接从受控测试环境注入，日志只存脱敏属性，不回显DSN。测试负责人归档预期12组用例及各子用例结果，命令输出为空不通过。

### NOW-06：单 Worker 统一操作与公平内核（L1）

**文件：** externalop coordinator/watch、workflows checkpoint/recovery、PG Queue/Relay、bootstrap/worker；tests/integration/{operation,queue,checkpoint,fairness,startup}_test.go、tests/contract/schema_test.go；同一 l0-evidence-index 延续引用 L1报告。**负责人：** 后端内核负责人、测试负责人，DevOps仅独立环境/流水线接线。**依赖：** NOW-05。

**输入：** PG已验证Repos、固定Registry、已批准预算/阈值、已授权workload。**输出：** 单Worker注册/调度/恢复一体化运行与负载报告。

- [ ] 接通同一Coordinator的三条领取、固定步骤、Outbox、最小Job Watch；不新增服务。
- [ ] 加入执行前权限/permit复验与unknown/迟到/旧epoch/断电恢复测试。
- [ ] 接通多订阅者delivery/receipt/ACK，重投不重建业务，禁用LISTEN仍能轮询推进。
- [ ] 固定10,000失败消息+持续SCM/Runner+慢响应负载，实测所有类型/tenant进展及槽/DB预算。
- [ ] 执行无NATS/Intake/Feed/Wait的启动和DB不可用失败测试；未知Schema/禁用模块不得运行。
- [ ] 仅复验被整合改动影响的PG-G与新增测试；全部退出项有真实结果后交付报告。

验收：
    go test ./tests/integration -run '^(TestOperationRecovery|TestQueueDelivery|TestCheckpointRecovery|TestFairness|TestCoreStartup)$' -count=1 -timeout=15m
    go test ./tests/contract -run '^(TestRegistry|TestTypedSchema|TestCoreHealth)$' -count=1
    go run ./tools/archcheck
    go run ./tools/check-architecture-fixtures

L1 最终退出需要第8节完整矩阵已有有效报告；“只跑新增/修改”不豁免从未执行的门禁。DevOps 的CI required-check接线待队长批准文件责任范围后执行，不能在此技术方案阶段修改部署或分支保护；后端与DevOps共享go.mod/迁移/bootstrap/scope时必须串行。

## 8. PG-G01～12 验收矩阵

所有行共同前置：真实锁定 PostgreSQL、实际 GORM Driver、正式 core 迁移、非 owner/非superuser/无BYPASSRLS的应用角色、至少两独立连接，READ COMMITTED 与各专用隔离场景明确；原始PG角色证据/版本/checksum/SQLSTATE/RowsAffected/屏障/前后数据/执行计划归档。测试服务缺失直接FAIL。

| 门禁 / 测试名称 | 实际故障/竞争 | 必须断言与证据 |
|---|---|---|
| PG-G01 / TestPGG01 | prepared、unknown、expired sending 双连接竞争/持锁跳过；测试Provider写入一次后杀掉确认前进程，无Queue/Webhook重启 | 自动sending→unknown→Lookup/Inspect；外部写次数1；原key/hash不变；旧epoch拒绝；sending索引/租约约束存在；另测调用前崩溃、暂停写仍恢复、恢复Outbox失败回滚、通知风暴下恢复预算 |
| PG-G02 / TestPGG02 | 同键同hash/异hash并发Plan/API幂等 | 只有一个新意图；同hash读原结果，异hash冲突；不出现新generation。capture专用表属L5不建 |
| PG-G03 / TestPGG03 | A插入未提交，B ON CONFLICT等待；A提交后B DO NOTHING | B下一条SELECT看见原记录；单statement空结果反例不得授权新建 |
| PG-G04 / TestPGG04 | 唯一冲突后继续查询的故意错误路径 | 实测23505→25P02；正常路径回滚后重开；无批准savepoint时不实现局部恢复 |
| PG-G05 / TestPGG05 | 两连接CAS同version/epoch及旧Worker迟到 | 仅一个RowsAffected=1；0不写成功Outbox；迟到仅追加受控Observation |
| PG-G06 / TestPGG06 | 更新false、0、NULL和允许清空字段；尝试Save/关联旁路 | 明确列完整持久化；关键状态不允许Save/自动关联/无条件Update |
| PG-G07 / TestPGG07 | 业务写完成后强制Outbox插入失败 | 业务/效果意图/Outbox整体回滚；tx内外部网络调用数0 |
| PG-G08 / TestPGG08 | 同资源跨tenant FK/owner关联、越权principal/对象/当前grant；连接复用 | 一律拒绝；Q2关闭RLS仍完整测试；开启时另验证RLS读/写/RETURNING及池复用，不能用owner绕过 |
| PG-G09 / TestPGG09 | 核心资源租约/活动操作槽竞争，新旧generation并行 | 一个有效writer；unknown未查清不能释放/改状态绕过；不新建L3修复表 |
| PG-G10 / TestPGG10 | lock/statement timeout、ctx取消、deadlock/serialization失败 | 有界失败与完整回滚、连接可再次使用；SQLSTATE分类后整tx同键重试 |
| PG-G11 / TestPGG11 | COMMIT已成功但客户端响应断开，重启/同请求重试 | 重查原幂等事实；资源/效果不新增；保留提交未知诊断；API不得假称受理成功 |
| PG-G12 / TestPGG12 | 正式迁移新装/向前/兼容应用回退；不装可选模块；代表性数据EXPLAIN | core独立启动；无Intake/Feed/NATS隐依赖；索引合理命中；小表Seq Scan不能被误判为失败；不靠enable_seqscan=off伪造计划 |

配套合同/故障矩阵：
- TestTypedSchema：正例及unknown版本、非法enum、缺字段、重复key、超限、hash不符在写/发/消费三个阶段拒绝或quarantine；旧版本解释不变。
- TestOperationRecovery：AC-111～118、120～121、123～124的core机械分支；L2批量/真实Provider尚不适用，不能记其真实验收通过。
- TestQueueDelivery：AC-130～132，多订阅者、乱序提交、ACK前崩溃、轮询无通知、lease回收、bounded retry/dead-letter。
- TestCheckpointRecovery：固定版本、稳定effect key、事件提前/重复/乱序、无Wait表的Job本地引用恢复、原deadline、未知handler拒绝。
- TestFairness：AC-150/175/176/182，真实PG+测试Provider固定负载，各分区P95/最大等待低于事先签署门槛，无跨tenant饥饿。
- TestCoreStartup：AC-157、core版PG-G12；DB不可达/错误角色/缺迁移/缺sending索引/无Registry必须FAIL；未装可选模块仍可启动。
- 架构正反例：AC-125～128、169、177、181。AC-151/161/163为L0范围证据核验。AC-183按RLS声明分支。AC-47/48、185/186、187～206保留原编号且本scope非适用。

不把测试Provider/伪时钟结果称为生产Provider一致性验证；PG-G01需要真实PG与进程崩溃，不能只写SQL模拟。后续L2复用同Harness参数化真实适配器，不复制第二套机械测试框架。

## 9. 六类技术风险快审

| 类别/风险 | 控制措施 | 验证与责任 |
|---|---|---|
| 数据库：示例DDL缺关系、ORM隐式更新、领取无索引、未知提交重建 | core显式迁移；同tenant FK/owner链接；参数化SQL/显式列/CAS；三条领取到期索引；短tx错误回滚；现有Repository模式只保留gormpostgres | PG-G01～12；EXPLAIN representative；后端DB负责人。当前无可审生产SQL，结论为设计控制项，不称SQL已通过 |
| 安全：伪AuthContext、服务账号扩大权限、秘密明文、SQL注入/XSS | 当前对象/动作/grant复验；独立write/read/recovery角色；Secret引用/受控制品；标识符白名单/值参数化；API固定JSON，外部正文不HTML直出；不建匿名L1业务入口 | TestAuthContext/PG-G08、跨Integration/Header/池复用、日志脱敏；OIDC/G03真实证据；安全审核人与测试负责人 |
| 性能：通知风暴/全表扫描/N+1/无限缓存 | SQL按type/action/tenant领取+索引；保留槽和DB预算；有界分页与批处理；Registry/Schema启动缓存；无N+1逐行远程授权或每行新连接；动态权限短期且执行前复验 | TestFairness、PG-G12、锁/池等待与query count；PG阈值负责人 |
| 并发：本地epoch误当远端fencing、重复Plan/ACK/迟到覆盖 | 原key/hash、资源单写者；unknown冻结；先sending后Execute；封存CAS、Observation追加；receipt幂等 | PG-G01/02/03/05/09/11、Queue/Recovery Harness；后端内核负责人 |
| 外部依赖：本机版本冒充已批准、OCR隐式更新/交互、机器人不可查证 | 不可变发行物/digest/原件；固定版本help；禁止POST自动重试；unsupported/ack_only明确；依赖版本/许可证/维护/漏洞在锁定时审核；不新增MQ/另一ORM | G01/G03、go.mod/go.sum与发行manifest、真实Provider契约；OCR/环境负责人 |
| 公共契约：蓝图全量实现、未冻结API、Schema静默改版 | L1不实现L2产品API；Q3确认探测合同；第23章原端口语义；新增Schema版本显式审查；后续前后端协作先独立合同冻结 | TestRegistry/TypedSchema/CoreHealth；当前无公共签名删除/重命名，无Breaking Change实施；架构/队长 |

风险结论：选定架构可按最小影响面实施，但必须补齐证据、RLS/探测合同决策及签署。无证据的“安全/性能已通过”结论不成立。架构 lint 不是运行授权保障；HA/扩容不解决 Provider 不可查证 unknown。

## 10. 当前决策前沿与交接

先前Q1～Q6产品决策全部沿用，不再询问是否扩大到L2。当前只关闭以下独立事项；本稿未实施任何选择。

| 问题 | 闭合选项与建议 | 阻断 |
|---|---|---|
| Q1 L0事实与具名负责人从何取得 | 1. 提供已批准的项目配置/证据位置及一名能协调OCR、OIDC、飞书、产品/技术签署、测试/PG阈值的具名负责人（建议）；2. 在本issue直接补充非秘密实际值及各角色名单；3. 其他（请说明） | G01～G04真实执行/签署；不要求把秘密贴评论 |
| Q2 L1是否声明RLS | 1. L1不声明RLS，强制当前AuthContext+对象权限+tenant FK+非特权角色完整隔离测试（建议，沿用可选RLS）；2. L1启用RLS并纳入真实角色/读写/RETURNING/池复用全分支；3. 其他（请说明） | G04档案及PG-G08/183实施判定 |
| Q3 Gin部署探测契约 | 1. 采用第4.3节仅管理入口GET /healthz和/readyz固定JSON合同（建议）；2. L1暂不新增路由，仅生命周期/启动失败与Harness（TestCoreHealth改验进程健康条件，不能空跑）；3. 其他（请说明） | Gin入口与测试合同；不涉及前端/L2产品API |

Q1取得事实来源后由执行者采集、核对并补齐准确值，不能让协调人替所有角色自动签署；环境平台、容量值、真实IdP映射若产生新取舍，再按依赖顺序确认。L0资料不齐不会抹去已有设计成果，但必须保持门禁blocked。

最终交接给队长：本稿+CONTEXT术语+基线hash；Q1～Q3结论；G01～G04缺口责任归属；六类风险和PG矩阵；NOW-01→06文件责任划分。收到回答后按需更新同一文档至冻结版，设计必需决策关闭才可把VTR-6设done。即使设计done，仍不代表L0/L1实现完成；真实L0退出审查必须先于任何L1产品开发。
