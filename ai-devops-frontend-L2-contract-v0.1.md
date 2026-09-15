# AI DevOps L2 前端最小合同

| 属性 | 内容 |
| --- | --- |
| 版本 | FE-L2-0.1；浏览器协议标识 `web-l2-v1` |
| 配套主设计 | `ai-devops-platform-design-v1.5.md`，第 24.4/24.7/26.6 节 |
| 目的 | NOW-08 开始界面集成前的最小前后端合同，不是 FE-1.1 全量设计 |
| 当前状态 | 待实施团队确认并录入 OpenAPI/Schema；示例不代表后端已经实现 |
| 范围权威 | 主文档第 3、19.8、31 章；仅使用 NOW-01～08，不创建新路线或里程碑 |
| 历史关系 | FE-1.0 保留为视觉/组件/安全蓝图；其中全量页面与 durable Feed 默认不适用于 L2 |

## 1. 唯一首发页面白名单

| 页面/路由 | 允许内容与动作 | 不包含 |
| --- | --- | --- |
| 登录 `/login` | 受控组织入口、一个已验收 OIDC、SSO 失败/返回流程 | 枚举其他租户、邮箱自动合并身份 |
| 工作台 `/` | 当前授权范围的 Review/外部操作摘要与错误待办 | 需要新聚合服务的全平台仪表盘；未安装模块卡片 |
| 项目/仓库 `/projects`、`/repositories` | 已登记 GitHub 关系、最小配置状态与范围；数据不可得时明确缺失 | 服务部署/观测目录、多 SCM 接入向导 |
| Review `/reviews`、`/reviews/:id` | 列表/详情、固定 SHA、覆盖、结论、摘要操作；有权时重新审查、申请重试已有回写、取消尚可取消的流程 | 行级评审编辑器、修复 Agent、自动批准/合并 |
| 操作 `/operations`、`/operations/:id` | 本次已启用类型的账本/Attempt/证据，获权只读对账、受审重试申请 | 任意 Provider HTTP 执行器、批量“全部重发” |
| 通知/基础设置 | 一个企业通道的状态/投递；当前身份、GitHub/IdP/通知配置状态及 scope 信息 | 五平台消息市场、完整策略服务/任意 YAML 编辑器 |

页面按“已交付功能 × 安装/启用 × 当前用户权限”装配。服务器的已交付 Provider 目录只返回当前 release-scope 中通过契约的实际项；L2 Agent 列表为空，通常无需请求它。Qcoder、Multica、Codex、Cursor、其他通知/SCM 等未在本次范围内的条目，**不以灰色卡片、disabled 按钮、可点击“即将上线”或隐藏下拉选项出现**。

扫描、Issue 修复、审批、Incident、Intake/扩展、工作流编辑器、Feed/HA 管理均不注册首发路由，不预加载代码/调用其接口。已交付但暂时故障的 GitHub/首个通知通道可以显示真实 disabled 原因；不要把未实现与暂时不可用混为一类。直接访问未交付路由统一显示“本发行版不提供此功能”，不查询该模块的数据表。

## 2. 启动与 capabilities

启动顺序：完成必要 SSO → `GET /api/v1/me` → `GET /api/v1/capabilities` → 构造授权路由 → 读取当前页面 REST 快照 → 依照 realtime 档案开启 SSE 或轮询。所有调用经同源 Gin/BFF，会话/Token 不复制到源平台、扩展或 localStorage。

下面是 L2 固定字段合同的一个已配置样例；UUID、revision 和通道由服务器根据当前主体/已登记发行物返回，不允许前端自行声明 ready：

```json
{
  "contract_version": "web-l2-v1",
  "release": {
    "profile": "postgres-minimal",
    "milestone": "L2",
    "scope_revision": "L2-scope-revision-1"
  },
  "modules": {
    "review": {"installed": true, "enabled": true},
    "notification": {"installed": true, "enabled": true},
    "scan": {"installed": false, "enabled": false},
    "repair": {"installed": false, "enabled": false},
    "incident": {"installed": false, "enabled": false},
    "intake": {"installed": false, "enabled": false},
    "feed": {"installed": false, "enabled": false}
  },
  "review": {"summary": true, "inline": false},
  "providers": {"scm": ["github"], "notifications": ["feishu"], "agents": []},
  "realtime": {
    "mode": "db_snapshot",
    "sse_enabled": true,
    "resume_supported": false,
    "protocol_version": "snapshot-sse-v1",
    "poll_interval_ms": 3000
  }
}
```

实际通知选钉钉时只替换为已验收的 `dingtalk_bot`，不能同时注册第二个通道来扩充首发。`providers` 是当前可见的已交付产品种类，不替代 Integration/对象权限；后台对每次 GET/POST 再鉴权。

关键字段缺失、协议未知或组合矛盾（如 L2 却 `resume_supported=true`）时，不猜测权限或自动开启模块；显示版本兼容错误，停用变更动作。只有已确认安全兼容的 REST 读取可以继续，不能在协议错误时照旧执行旧客户端动作。能力响应不能包含可执行脚本、任意目标 URL 或凭据。

所有缓存/未完成请求绑定 `origin + principal + tenant + scope_revision`。退出、换租户、scope 改变或撤权后关闭 SSE、终止待取请求、清除敏感缓存，再重新获取 me/capabilities；旧上下文的迟到响应不可写回新页面。`DEFERRED` 只可在获权 scope 管理摘要显示为“扫描已排除/延后”，不生成扫描产品导航。

## 3. db_snapshot 的 REST/SSE 合同

`GET /api/v1/events/stream` 使用同源会话。订阅仅允许当前有权页面的固定资源/列表范围，资源枚举和条数由后端白名单/上限控制，不接受 SQL、表名、其他租户或任意远端 URL。L2 无持久 Feed，不发 `id:`，前端不保存/发送 Last-Event-ID，也不尝试重放遗漏的中间动画。

| SSE event | 最小 data | 前端动作 |
| --- | --- | --- |
| `snapshot_required` | `protocol_version`、`reason` | 合并刷新当前页面有权 REST 快照；首次连接、断线、API 重启、过滤变化均执行 |
| `resource_invalidated` | 固定 `resource_kind`、`resource_id`；可有版本提示 | 将对应缓存标过时后 GET；事件本身不是新状态的权威 DTO |
| `scope_changed` | 固定 `reason`，不含无权资源 | 清理敏感缓存，重取 me/capabilities；必要时重新登录 |
| 注释心跳 | 无业务正文 | 仅维持连接；不能作为任务活跃/完成证据 |

示意帧（没有 `id:`；这是浏览器协议，不复用领域事件 Schema）：

```text
event: snapshot_required
data: {"protocol_version":"snapshot-sse-v1","reason":"connected"}

```

REST 返回当前对象版本与所需 ETag/动作可用性，前端按注册 DTO 无损读取版本字段；只接受相同授权上下文中的当前/更高版本。删除或撤权时清除旧正文，不因缓存命中继续显示。`401` 结束会话；`403/404` 清除对应对象；不能用旧 SSE 数据绕过 GET 失败。

SSE 关闭/不可达时，每个可见页面按服务器建议间隔有界刷新当前列表页/详情，并合并重复 GET，最多两个读取请求并发；页面隐藏暂停普通轮询，重新可见立即取快照。SSE 正常时也保留低频快照兜底，不把事件丢失当作任务未变。使用指数退避/jitter 和 Retry-After 防止断线风暴；不轮询整个数据库或全部历史页。

断线提示为“连接中断，状态可能过时”，**不是“任务已停止”**。L2 不宣称多 API HA/游标恢复；能力将来切到 durable_feed 时须重新协商，当前客户端不理解该协议则安全回退已兼容 REST，而不是猜解游标。

## 4. L2 动作与副作用展示

| 用户动作 | 现有主方案 API | 表现要求 |
| --- | --- | --- |
| 查看 Review | `GET /api/v1/reviews`、`GET /api/v1/reviews/{id}` | 显示 reviewed/excluded/failed 覆盖，不仅显示绿色成功 |
| 重新审查 | `POST /api/v1/reviews/{id}/reruns` | 明确新 Run/模型费用；只在当前权限和版本允许时提交 |
| 重试已有回写 | `POST /api/v1/reviews/{id}/retry-publication` | 仅使用已有结果，底层统一 Operation；unknown 不直接 Create |
| 只读查证 | `POST /api/v1/external-operations/{id}/reconcile` | “已安排查证”，不显示“已重发/发布成功” |
| 申请安全重试 | `POST /api/v1/external-operations/{id}/retry-requests` | 后端当前授权/否定证据门禁；按钮禁用不是唯一保护 |
| 取消流程 | `POST /api/v1/workflows/{id}/cancel` | 只表示请求取消；旧 sending/unknown 的事实继续展示 |

POST 只提交固定 DTO；写操作按主合同带稳定 Idempotency-Key，适用的对象修改带 If-Match/期望版本与 CSRF。重复点击、网络回执未知仅复用原键和相同请求；修改内容必须新用户意图，不为自动重试换键。`409` 重取状态并提示冲突，不自动覆盖；过期 key 停止自动重试。没有单独批准动作进入 L2，不能提前显示 L3 的审批/修复按钮。

L2 摘要展示为“PR 讨论区摘要（固定 SHA）”，不是 GitHub 原生 Review 审批；一次 ReviewRun 的全部 Finding 只关联一项 `scm.review.summary`。Operation `confirmed` 只表明本次外部效果被确认，不代表 OCR 无问题、用户已读或生产已恢复；`unknown` + 取消/过期仍显示“外部效果未确认”，不可变成无风险灰色完成。

## 5. NOW-08 的确认清单（不是新里程碑）

前后端共同确认 `web-l2-v1` 必填字段、未知版本处理、资源 DTO/ETag、三类 SSE event、路由白名单以及 L2 实际 Provider。缺少字段/接口就先补同一合同并实施，不能将 Mock 结果作为验收。FE-1.0 其余 D 项在相应后续切片再处理。

复用主设计既有 AC-07、23、65、112、152、153、160、178、180 等适用用例：L2 无 Scan/Wait/Intake/Feed/Agent 依赖、无 pending 可点项、断线/API 重启重新取快照、相同 tenant 跨项目撤权清缓存、响应未知不重复摘要、旧备份的会话作废。新增协议 fixture 作为这些 AC 的子用例，不新增一组全球首发必测编号。

**检查边界：** 本合同只是可签署的设计，不代表双方已签署，也未执行 React 页面、真实 SSO、浏览器 SSE、数据库或 GitHub/通知联调。主文档仍是范围/语义权威，本文仅补足 NOW-08 的最小前端接口。
