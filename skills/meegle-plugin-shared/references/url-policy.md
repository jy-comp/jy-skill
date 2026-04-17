# URL 策略（"无源即停"的 URL 落地）

> **前置**：本段是 meegle-plugin-shared 主文件"禁止编造 URL"的完整规则。涉及 URL 字段填写的 skill（meegle-plugin-feature / meegle-plugin-publish）触发时按需 Read 此文件。

## 核心原则

**绝对禁止编造任何 URL 或图标地址。** 编造的 mock URL（如 `https://example.com/webhook`）在实际使用中完全不可用，会导致插件功能失效。

**继承自"无源即停"全局通则**。模式 lint（apply 阶段的 A0.5 黑名单）只能识别"长得像假的"URL（`example.com` / `placeholder` 等），**完全识别不出"长得像真的假"**（如 AI 编造的 `https://meego-internal.byted.net/cb`）。真正的防线是**写入时的溯源协议**——AI 永远不主动创造 URL 字符串，每个值必须能追溯到"用户显式输入"或"用户授权占位"。

## 判断原则：Runtime URL vs Metadata URL

把 schema 里所有 URL 字段按"是否会被运行时打"分两类：

- **Runtime URL（运行时会发起请求）**：必须真实可达。假地址一旦上线就触发功能失败 —— 表现为表格列渲染空白、按钮点击报错、事件收不到回调。**必须向用户询问**。
- **Metadata URL（仅作为元信息展示）**：不填影响美观但不影响功能，**不阻塞**。

## 完整的 URL 字段清单

| 字段 | 所属点位 | 类别 | 触发条件 / 处理方式 |
|------|---------|------|-------------------|
| `intercept.url` | intercept | Runtime | 始终必问：接收拦截事件的回调地址 |
| `listen_event.url` | listen_event | Runtime | 始终必问：接收监听事件的回调地址 |
| `control.url` | control（顶层） | Runtime | 仅当"新建页可见"开启时必填：新建提交时的 Webhook |
| `control.platform.web.table_url.url` | control | Runtime | **当 `table_cell.definitions.data` 非空时必问**（展示态数据接口，每次展开列都会打） |
| `customField.platform.web.table_data_url` | customField | Runtime | **当 `table_layout.definitions.data` 非空时必问**（同上） |
| `template.onClick.params.url` / `onDoubleClick.params.url`（`action=httpRequest`） | 任意 table_cell / table_layout DSL | Runtime | 用户主动声明了 httpRequest action 时必问（点击触发业务请求） |
| `template.onClick.params.url`（`action=openLink`） | 同上 DSL | Runtime | 字面量 URL 时必问；若用 `{{varName}}` 引用则不填（由数据接口返回） |
| `liteAppComponent.icon_url` | liteAppComponent | Metadata | **不阻塞**：不填 CLI 自动填充默认图标 |
| 插件图标 icon | update-description | Metadata | **不阻塞**：不传则保持后台默认图标 |

> ⚠️ **token 字段一律不询问用户**：`intercept.token` / `listen_event.token` / `control.token` / `table_url.token` / `table_data_token` 均由 CLI 在对应 URL 填写后自动生成 36 位 UUID，**AI 不要主动填 token，也不要向用户索要**（即使 schema 标了 required，CLI 也会兜底）。

## 变量声明 → Runtime URL 的联动规则（CRITICAL · 根原则 1 无源即停 · URL 联动场景）

这是最容易漏的一条：任何形如"顶层 `type` + 内含 `{{varName}}`"的 DSL（control.table_cell / customField.table_layout，以及未来任何共用 dslSchema 的新点位）只要 `definitions.data` 非空，就**必须**伴随真实的数据接口 URL（`table_url.url` / `table_data_url`）。

- **为什么**：DSL 里 `{{varName}}` 的值由数据接口响应填充（`work_item_datas[].work_item_data.{varName}`）。URL 不可达 → 变量 undefined → **整列渲染失败**。
- **判断方法**：AI 不能只凭点位类型判断，而要**扫 DSL 本身**——只要发现 template 里有非 `$container / $i18n / $colorTokens / $fieldValue` 前缀的 `{{x}}` 引用，该点位就属于"有数据变量"类别。
- **与裸 template 的关系**：裸 template 天然没有 `definitions.data`（结构决定的），不触发此规则；一旦改成完整对象形态并声明任何 `data.xxx`，立即触发。

## 执行原则

- **Runtime URL**：**必须暂停流程向用户询问**；用户暂时没有 → 占位符 `"<PLACEHOLDER: 请替换为你的<具体字段名>>"` 并在摘要醒目标注，发布前硬阻止
- **Metadata URL**：**不阻塞流程**，不填即可
- **禁止使用** `https://example.com`、`https://your-server.com`、`https://PLACEHOLDER.invalid/...`、`https://localhost/...`、`.test` / `.example` 结尾等"看起来真实但不可用"的地址——包括 RFC 保留 TLD

## 规则冲突时的兜底（CRITICAL · 根原则 1 无源即停 · 冲突时的默认动作）

**当 `<PLACEHOLDER: ...>` 字面量与 schema/CLI 校验冲突时**（例如 schema 的 `UrlHttp` 要求 `^https?://`，导致纯文本占位符被 `local-config set` 拒绝），**AI MUST 立即暂停并向用户上报冲突**，不得自作主张采取任何"看似合理"的折中（包括但不限于：使用 RFC 保留 TLD `.invalid`/`.test`/`.example`、伪造看起来不真实的域名、把 PLACEHOLDER 塞进 URL path/query 里拼成合法 URL）。

**正确处理流程：**
1. 停止当前 apply/set 流程
2. 向用户明确说明：
   - 哪个字段
   - 为什么占位符不能通过校验（引用具体 schema 错误）
   - 两个选项：(a) 提供真实 URL（推荐）；(b) 明确授权某种 placeholder 写法
3. 等待用户回复后再继续

**为什么这条规则存在：** 任何被 AI 发明出来的"像 URL 的字符串"都可能被用户当成"已经填好了"漏掉替换，上线后静默失败；且自创占位符破坏评测的独立性（AI 用巧思替代了"向用户要真信息"的硬规则）。**"绝对禁止编造"是绝对的，遇到阻力时默认动作是问用户，不是发明绕过方案。**

## 落地执行点

- URL 溯源（写入前）：`meegle-plugin-feature/references/config-plan.md` 的 "URL 溯源协议" 章节
- URL lint（写入后）：`meegle-plugin-feature/references/config-apply.md` 的 A0.5 + `meegle-plugin-publish/references/pre-check.md` 的 PC1.5
