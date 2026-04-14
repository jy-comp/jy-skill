---
name: meego-shared
version: 1.0.0
description: "Meego 插件开发共享基础：CLI 可用性检查、Device Code OAuth 认证、Token 管理、安全规则。所有 meego-* / plugin-* skill 的公共前置依赖。"
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# Meego 共享规则

本技能包含 Meego 插件开发的公共认证、环境检查和安全规则。所有其他 skill 在执行前必须先确认本 skill 中的前置条件已满足。

## 插件工程识别

**当 AI 进入一个目录时，应检查是否为 Meego 插件工程。** 识别规则：

### 判定条件

当前目录（或用户指定的工作目录）存在 `plugin.config.json`，且文件包含以下必要字段：

```json
{
  "siteDomain": "https://meego.feishu-boe.cn",   // 站点域名（必有）
  "pluginId": "MII_xxxxxxxxxxxx",                 // 插件 ID，MII_ 前缀（必有）
  "pluginSecret": "...",                           // 加密后的密钥（必有）
  "resources": [...]                               // 资源列表（可能为空数组）
}
```

**判定为插件工程的充要条件**：`plugin.config.json` 存在 **且** 包含 `pluginId` 字段（以 `MII_` 开头）。

> 注：v1 框架的旧插件使用 `manifest.json`，当前 skill 体系仅适用于 v2 框架（`plugin.config.json`）。

### 从配置中可获取的上下文

| 字段 | 含义 | 用途 |
|------|------|------|
| `siteDomain` | 站点域名 | CLI 命令自动读取，无需手动传 `--site-domain` |
| `pluginId` | 插件唯一标识 | 所有 API 操作的目标 |
| `pluginSecret` | 加密密钥 | CLI 内部使用，**禁止明文输出** |
| `resources` | 资源列表 | 每项含 `id`（点位资源标识）和 `entry`（代码入口路径） |
| `resources[].id` | 资源 ID | 格式 `{点位类型}_{平台}_{后缀}`，从前缀可判断点位类型（如 `board_web_xxx`、`button_web_xxx`） |
| `source_type` | 配置来源 | `remote`（默认）或 `local` |

### 识别后的行为

一旦确认当前目录为插件工程：
1. 所有 CLI 命令均可直接执行（`siteDomain`、`pluginId` 等自动从配置读取）
2. 用户的操作指令（"启动调试"、"发布"、"改名称"等）应路由到 `meego-cli` skill 查找对应命令
3. 涉及多步编排的操作（"加个点位"、"做一个新功能"）应路由到对应的编排 skill

## CLI 可用性检查

```bash
npx @byted-meego/cli@builder --version
```

- 成功（输出版本号）→ CLI 可用
- 失败 → 提示用户安装：`npm install -g @byted-meego/cli@builder`

## 飞书项目知识 MCP（与 CLI / Token 一致的 soft 校验）

env-setup 第三段：飞书项目知识 MCP 提供"自然语言需求 → 用什么插件能力实现"的知识检索。装上 AI 准确度显著提升；未装时 AI 仍可基于 schema 推断，效果次优。

| 接入参数 | 值 |
|---------|-----|
| 名称（建议） | `feishu-project-knowledge` |
| 传输方式 | HTTP |
| URL | `{siteDomain}/mcp_server/knowledge`（与 CLI 同域） |
| 鉴权 | 无需额外 header，复用 CLI 的 Token |

> ⚠️ **安装由当前 AI agent 自行完成**：本 skill 不绑定具体 host（Claude Code / Cursor / Cline / Gemini CLI / Copilot CLI / 其他）。AI agent 应判断自身 host 类型，派子 agent 按对应机制安装；不要直接把"装 MCP"丢回给用户。
>
> **校验风格**：与 env-setup 现有 CLI / Token 检查保持一致——soft 校验（检测 + 提示，不写 checkpoint、不强制阻断）。装成功后会提示用户重启会话以激活，但不阻断本次任务；下游 skill 调 MCP 工具时若拿不到会自然报错。

详细分支见 `env-setup/references/setup.md` 的 S3a/b/c。

## 工具职责划分（CRITICAL — 三类知识源，不得串用）

插件开发过程中涉及三类查询场景，各有专属工具，不得混用：

| 场景 | 专属工具 | 典型问题 |
|------|---------|---------|
| **插件能力 / 功能知识**（点位能做什么、SDK 有哪些 API、某个属性如何使用、轻应用组件体系、排期能力限制等概念性知识） | **飞书项目知识 MCP**（`feishu-project-knowledge`） | "轻应用组件支持哪些 propType？"、"customField 怎么拿到行数据？"、"intercept 回调里能拿到什么字段？" |
| **工作项实例 / 视图实例 / 空间实例等真实业务数据**（调用 SDK 或生成代码时需要引用的真实 ID、字段、状态等） | **`lark-project` skill** | "这个空间下有哪些工作项类型？"、"某个工作项实例的字段值是什么？"、"视图 id 是多少？" |
| **点位配置字段形状 / 枚举值匹配**（仅限生成 `point.config.local.json` 时查字段类型、枚举、必填性） | **CLI schema**（`local-config schema` / `meego-cli/schema/developer-plugin.yaml`） | "liteAppComponent 有哪些必填字段？"、"component_type 的枚举值是什么？" |

### 不得串用

- **禁止**用 CLI schema 回答功能性问题（schema 只告诉你"字段叫什么"，不告诉你"这个字段控制什么行为"）→ 走飞书项目知识 MCP
- **禁止**用飞书项目知识 MCP 查实例数据（MCP 是知识库，不是业务数据库）→ 走 lark-project skill
- **禁止**凭 schema 枚举 / 字段名脑补 SDK 行为或运行时数据 → 任何"这个属性应该是这样用"的推断都必须先查飞书项目知识 MCP

### 场景对照（常见混淆）

- 写点位配置：CLI schema 匹配字段 + 飞书项目知识 MCP 查每个字段的业务含义
- 写 SDK 调用代码：飞书项目知识 MCP 查 API 签名和用法 + lark-project skill 拿实例数据（如果代码里要填真实 work_item_id / space_id / view_id）
- 用户问"这个插件里 xxx 功能怎么做"：飞书项目知识 MCP
- 用户问"我这个空间/工作项里有什么"：lark-project skill

> 此分工是"无源即停"通则在工具选择层面的落地：每次查询前先确认"我要的是知识、实例数据、还是字段形状"，再选对应工具。选错工具拿回的答案本质上也是编造。

## 认证

Meego 插件开发采用统一认证，完成一次授权即可使用所有功能。无细分权限，Token 有效即可执行全部操作。

### 双 Token 机制

支持两种 Token，**按域名独立存储**，优先使用永久 Token：

| Token 类型 | 来源 | 有效期 | 优先级 |
|-----------|------|--------|-------|
| **Developer Token**（永久） | `login --token <token>` 手动设置 | 永久有效 | **高** — 有它就不需要 OAuth 授权 |
| **OAuth Token**（临时） | `login` 浏览器授权获取 | 临时，支持自动刷新 | 低 — 仅在无 Developer Token 时使用 |

### Token 存储

Token 按域名存储在 `~/.lpm/auth.json`，格式：

```json
{
  "https://project.feishu.cn": {
    "developerToken": "永久有效的token",
    "accessToken": "OAuth临时token",
    "accessTokenExpiresAt": 1234567890,
    "refreshToken": "...",
    "refreshTokenExpiresAt": 1234567890,
    "clientId": "<client_id>",
    "tokenId": "<token_id>"
  },
  "https://meegle.com": {
    "developerToken": "另一个域名的token"
  }
}
```

- 不同域名的 Token 互不影响
- `developerToken` 永久有效，无需刷新
- `clientId` + `refreshToken` 用于 OAuth Token 的自动刷新
- API 请求通过 `Authorization: Bearer <token>` 消费（优先取 `developerToken`）

### Token 检查

检查 `~/.lpm/auth.json` 中当前域名是否有可用 Token：
- 有 `developerToken` → 直接使用，认证通过
- 有 `accessToken` → 检查是否过期，过期则自动刷新
- 都没有 → 需执行认证流程（见下方）

### 认证流程

支持两种方式：

**方式 A（推荐）：直接设置永久 Developer Token**

引导用户前往 `<站点域名>/openapp/settings` 页面复制 Developer Token，然后执行：

```bash
npx @byted-meego/cli@builder login --site-domain <域名> --token <developer_token>
```

将永久 Token 写入对应域名的 `developerToken` 槽。有此 Token 后无需再走 OAuth 授权。

**方式 B：Device Code OAuth 浏览器授权**

```bash
# 以 background 方式执行（阻塞直到用户完成授权或超时）
npx @byted-meego/cli@builder login --site-domain <域名>
```

启动后立即读取输出，提取授权码和验证链接发送给用户。后台命令会自动轮询等待授权完成。获取的 OAuth Token 为临时 Token，支持自动刷新。

> 完整的认证步骤详见 [`../env-setup/references/setup.md`](../env-setup/references/setup.md)。

### 站点域名（siteDomain）

Token 按域名生效，确定 `siteDomain` 的优先级：
1. 当前目录 `plugin.config.json` 中的 `siteDomain` 字段
2. 询问用户选择：飞书项目 (project.feishu.cn) / Meegle (meegle.com) / 自定义域名

## 安全规则

- **禁止修改 `.lpm/` 目录下的任何文件**：`.lpm/` 目录由 CLI 内部管理（如 `auth.json`、缓存等），禁止通过 Edit/Write 工具或任何方式直接修改其中的文件，只能通过 CLI 命令间接操作
- **禁止输出密钥**（accessToken、pluginSecret）到终端明文
- **写入/删除操作前必须确认用户意图**（如发布、删除点位等不可逆操作）
- **全量提交约束**：`local-config set` 和 `update --source-type=local` 均为全量操作，禁止只传变更部分
- **删除点位确认（CRITICAL — 适用于所有阶段和所有调用路径）**：在执行 `local-config set` 之前，**MUST** 对比即将提交的 JSON 与远端配置（`local-config get --remote`）。如果提交的 JSON 相比远端**减少了任何点位**（整个类型缺失或某个 key 缺失），**必须立即暂停，向用户列出即将被删除的点位清单（类型 + key + name），获得明确确认后才能执行 set。禁止静默删除。** 此规则无论从 plan、apply、pipeline 还是 plugin-workflow 任何路径进入均须执行，不可跳过。

### 无源即停（CRITICAL — 全局通则，统御所有"溯源协议"）

**适用范围**：URL、token、代码、API 调用、字段值、配置参数等所有 AI 输出场景。

**核心原则**：AI 输出的每一项**有业务语义的内容**都必须能追溯到合法信息源。找不到合法信息源时，**MUST 立即停下来询问用户**，禁止"先填一个看起来合理的，跑不通再说"的中间状态。

**合法信息源（白名单，因场景而异）**：
- 用户在对话中显式输入 / 粘贴
- 用户明确授权的占位（`<PLACEHOLDER: ...>`）
- 权威工具的精确返回（飞书项目知识 MCP 文档示例、`@lark-project/js-sdk` 类型定义、CLI 命令的实测输出等）
- 当前工程已存在的代码 / 配置中已落地的事实

**非法信息源（黑名单，永不接受）**：
- AI 内置经验类比（"通常这种 SDK 都有 getX/setX"、"看起来像内网域名"等）
- 从相邻概念名称脑补（如从 schema 字段名推断 SDK 方法名、从 URL 路径片段编造完整 URL）
- MCP 返回模糊片段后的"补全推断"
- 反向"应该有"推断（"既然有 read 应该有 write"）
- 命名/格式约定猜测（"通常都用驼峰命名所以这里也是"）

**遇到无源时的统一动作**：

```
我无法在合法信息源中找到 "<具体内容>" 的依据。

可选：
A. 跳过此项，输出 TODO 占位（推荐——不写错的好过写错的）
B. 你提供真实值 / 示例 / 文档链接
C. 提供检索关键词，我重新查（可能找错了）

禁止我自行猜测——猜出来的内容会通过表层校验但运行时全废。
请告诉我选哪个。
```

**输出标注溯源（机器可审计）**：
每项非平凡输出在邻近位置标注 source，便于 reviewer agent 和用户审计：
- 配置字段：`# source: 用户提供 @ 2026-04-14`
- 代码调用：`// source: js-sdk types/index.d.ts:Control.method`
- URL：`# source: 用户授权占位 @ 2026-04-14`

未标注 source 的内容 → reviewer 视为疑似编造，要求作者补来源或删除。

**已落地的具体协议**（均遵循此通则）：
- URL 溯源：`meego-point-config/references/plan.md` 的 "URL 溯源协议"
- 代码溯源：`plugin-code-gen/references/plan.md` 的 "代码溯源协议"
- 后续新场景（如自定义脚本生成、SQL 生成等）应继承此通则各自落实

**核心信念（写进 AI 的认知锚点）**：
> "我没有用户的 ground truth，我永远不'善意默认'。
> 没有合法信息源就停下来问用户，**不存在'先写一个看起来合理的，跑不通再改'的中间状态**。"

### 禁止编造 URL（CRITICAL — "无源即停"通则的 URL 落地）

**绝对禁止编造任何 URL 或图标地址。** 编造的 mock URL（如 `https://example.com/webhook`）在实际使用中完全不可用，会导致插件功能失效。

> **本规则继承自上方"无源即停"全局通则**。模式 lint（apply 阶段的 A0.5 黑名单）只能识别"长得像假的"URL（`example.com` / `placeholder` 等），**完全识别不出"长得像真的假"**（如 AI 编造的 `https://meego-internal.byted.net/cb`）。
> 真正的防线是**写入时的溯源协议**——AI 永远不主动创造 URL 字符串，每个值必须能追溯到"用户显式输入"或"用户授权占位"。详细执行流程见各点位 skill 的 plan 阶段（如 `meego-point-config/references/plan.md` 的 "URL 溯源协议"章节）。

### 判断原则：Runtime URL vs Metadata URL

把 schema 里所有 URL 字段按"是否会被运行时打"分两类：

- **Runtime URL（运行时会发起请求）**：必须真实可达。假地址一旦上线就触发功能失败 —— 表现为表格列渲染空白、按钮点击报错、事件收不到回调。**必须向用户询问**。
- **Metadata URL（仅作为元信息展示）**：不填影响美观但不影响功能，**不阻塞**。

### 完整的 URL 字段清单（以 `meego-cli/schema/developer-plugin.yaml` 为准）

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

### 变量声明 → Runtime URL 的联动规则（CRITICAL）

这是最容易漏的一条：任何形如"顶层 `type` + 内含 `{{varName}}`"的 DSL（control.table_cell / customField.table_layout，以及未来任何共用 dslSchema 的新点位）只要 `definitions.data` 非空，就**必须**伴随真实的数据接口 URL（`table_url.url` / `table_data_url`）。

- **为什么**：DSL 里 `{{varName}}` 的值由数据接口响应填充（`work_item_datas[].work_item_data.{varName}`）。URL 不可达 → 变量 undefined → **整列渲染失败**。
- **判断方法**：AI 不能只凭点位类型判断，而要**扫 DSL 本身**——只要发现 template 里有非 `$container / $i18n / $colorTokens / $fieldValue` 前缀的 `{{x}}` 引用，该点位就属于"有数据变量"类别。
- **与裸 template 的关系**：裸 template 天然没有 `definitions.data`（结构决定的），不触发此规则；一旦改成完整对象形态并声明任何 `data.xxx`，立即触发。

### 执行原则

- **Runtime URL**：**必须暂停流程向用户询问**；用户暂时没有 → 占位符 `"<PLACEHOLDER: 请替换为你的<具体字段名>>"` 并在摘要醒目标注，发布前硬阻止
- **Metadata URL**：**不阻塞流程**，不填即可
- **禁止使用** `https://example.com`、`https://your-server.com`、`https://PLACEHOLDER.invalid/...`、`https://localhost/...`、`.test` / `.example` 结尾等"看起来真实但不可用"的地址——包括 RFC 保留 TLD

### 规则冲突时的兜底（CRITICAL — 不可擅自绕过）

**当 `<PLACEHOLDER: ...>` 字面量与 schema/CLI 校验冲突时**（例如 schema 的 `UrlHttp` 要求 `^https?://`，导致纯文本占位符被 `local-config set` 拒绝），**AI MUST 立即暂停并向用户上报冲突**，不得自作主张采取任何"看似合理"的折中（包括但不限于：使用 RFC 保留 TLD `.invalid`/`.test`/`.example`、伪造看起来不真实的域名、把 PLACEHOLDER 塞进 URL path/query 里拼成合法 URL）。

**正确处理流程：**
1. 停止当前 apply/set 流程
2. 向用户明确说明：
   - 哪个字段
   - 为什么占位符不能通过校验（引用具体 schema 错误）
   - 两个选项：(a) 提供真实 URL（推荐）；(b) 明确授权某种 placeholder 写法
3. 等待用户回复后再继续

**为什么这条规则存在：** 任何被 AI 发明出来的"像 URL 的字符串"都可能被用户当成"已经填好了"漏掉替换，上线后静默失败；且自创占位符破坏评测的独立性（AI 用巧思替代了"向用户要真信息"的硬规则）。**"绝对禁止编造"是绝对的，遇到阻力时默认动作是问用户，不是发明绕过方案。**

## Checkpoint（进度追踪）

当子 skill 被 `plugin-workflow` 调用时（即项目根目录存在 `.plugin-workflow-state.json`），**每个 CLI 命令执行前后必须更新 checkpoint 文件**。

### 判断规则

1. 读取项目根目录 `.plugin-workflow-state.json`
2. **存在** → 当前处于 workflow 编排中，每步 CLI 前后更新此文件
3. **不存在** → 子 skill 被独立调用，不需要写 checkpoint

### 更新协议

```
CLI 命令执行前：
  → 写入 nextCommand、nextStep，lastCommandStatus 设为 "running"

CLI 命令执行后（成功）：
  → lastCommand = 刚执行的命令
  → lastCommandStatus = "success"
  → nextCommand/nextStep 更新为下一步

CLI 命令执行后（失败）：
  → lastCommand = 刚执行的命令
  → lastCommandStatus = "failed"
  → 保留 nextCommand/nextStep 不变（重试时使用）
```

### 恢复语义

当 workflow 从 checkpoint 恢复并调用子 skill 时，子 skill 应检查 `lastCommand` + `lastCommandStatus`：
- 上一步 `"success"` → 跳过该步，执行下一步
- 上一步 `"failed"` → 重试该步
- 上一步 `"running"` → 未知状态，重新执行该步

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| CLI 未安装 | 提示 `npm install -g @byted-meego/cli@builder` |
| Token 过期 | 自动尝试 refresh；失败则引导重新执行 `/env-setup` |
| 网络错误 | 展示原始错误，建议检查网络后重试 |
| `plugin.config.json` 缺失 | 引导先执行 `/plugin-create` 创建工程 |
