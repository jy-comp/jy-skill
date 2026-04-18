---
name: meegle-plugin-shared
version: 1.0.0
description: "Meegle 插件开发共享基础：插件工程识别、Device Code OAuth 认证、Token 管理、安全规则。所有 meegle-plugin-* skill 的公共前置依赖，不单独对用户触发。"
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# Meegle 插件开发共享规则

本技能包含 Meegle 插件开发的公共认证、环境检查和安全规则。所有其他 skill 在执行前必须先确认本 skill 中的前置条件已满足。

## 本 skill 的最少 Read 清单

- 本文件（SKILL.md）：所有 skill 的通用前置，含三条根原则、插件工程识别、认证、安全规则
- URL 详细策略 → 仅在涉及 URL 字段填写时 Read [`references/url-policy.md`](references/url-policy.md)
- Checkpoint 协议 → 仅在被 workflow 编排（`.lpm-cache/state.json` 存在）时 Read [`references/checkpoint.md`](references/checkpoint.md)
- 不要预加载两份 reference；按实际触发场景按需 Read

## 三条根原则（所有 CRITICAL 规则都派生自这里）

> **设计意图**：全文档的硬护栏统一收束在以下 3 条根原则。每条根原则在不同子域的具体落地标记为"CRITICAL · <根原则> · <域>"，便于 AI 一眼看出"它属于哪条根原则、为什么不可跳"。

### 根原则 1：无源即停（输出可溯源）
**核心**：AI 输出每一项有业务语义的内容，必须可追溯到合法信息源；找不到 → 立即停下问用户。
**派生应用**：
- 禁止编造 URL（本文件 · "禁止编造 URL"小节）
- Runtime URL 联动规则（references/url-policy.md · "变量声明 → Runtime URL"）
- 规则冲突兜底（references/url-policy.md · "规则冲突时的兜底"）
- 代码溯源四步协议（meegle-plugin-feature/SKILL.md · "代码溯源协议"）

### 根原则 2：数据完整性（写入前先拿基线，减少必先确认）
**核心**：任何对远端配置的写入都是全量覆盖，提交前必须以远端基线为参照、对比差异、对减少项让用户确认。
**派生应用**：
- 全量提交约束（本文件 · "全量提交约束"小节）
- 删除点位前置检查协议（本文件 · "删除点位前置检查协议"小节）
- 各 skill 落地：feature/references/config-apply.md A0、config-plan.md "删除操作"

### 根原则 3：真实数据动作显式确认
**核心**：任何会对远端产生真实影响的动作——不管能不能撤回——执行前必须向用户列出"将要做什么 + 影响哪些数据"，等用户显式同意才执行。可撤回不代表可擅自推进，先确认能防止远端被改成不期望的状态。

**四类必须先确认的动作**：
1. **创建插件**（在 Meegle 后台生成插件记录，pluginId 从此绑定）
2. **配置点位**（`local-config set` 等，把 plugin.config.json 全量推到后台）
3. **配置基本信息**（更新插件名称 / 简短描述 / 详细描述 / 分类等元信息）
4. **发布插件**（`publish`，发到 Meegle 市场，用户可见，最不可逆）

**派生应用**：
- 创建插件（meegle-plugin-create）
- local-config set 推送点位变更（meegle-plugin-feature/references/config-apply.md）
- 删除点位前置检查协议（本文件下方独立小节）
- 更新插件基本信息（meegle-plugin-polish）
- publish 发布到市场（meegle-plugin-publish/references/pre-check.md）

> **AI 阅读顺序**：见到任意 CRITICAL 标记，先识别它属于哪条根原则；如果识别不出，按"无源即停"默认处理（最保守动作 = 停下问用户）。

## 插件工程识别

**当 AI 进入一个目录时，应检查是否为 Meegle 插件工程。** 识别规则：

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
2. 用户的操作指令（"启动调试"、"发布"、"改名称"等）应路由到 `meegle-plugin-cli` skill 查找对应命令
3. 涉及多步编排的操作（"加个点位"、"做一个新功能"）应路由到对应的编排 skill

## 认证

Token 由 CLI 统一管理。CLI 遇 auth 问题会 stderr 打印完整登录指引（`Authentication required for ...` + 方式 A/B + 完整可执行命令）。

**AI 的应对**：
- **默认**：触发 `/meegle-plugin-env-setup` 接管登录编排（尤其方式 B 的后台 OAuth 流程，用户不切终端）。完成后重试刚才失败的命令。
- **简化**：若用户明确表示自己跑，或已在 meegle-plugin-env-setup 上下文中，**逐字转呈** CLI stderr 指引即可。

### 站点域名（siteDomain）

Token 按域名生效，确定 `siteDomain` 的优先级：
1. 当前目录 `plugin.config.json` 中的 `siteDomain` 字段
2. 询问用户选择：飞书项目 (project.feishu.cn) / Meegle (meegle.com) / 自定义域名

## 安全规则

- **`.lpm/` 目录由 CLI 内部管理**（`auth.json`、缓存等），不要用 Edit/Write 直接改，只能通过 CLI 命令间接操作。此限制仅限 `.lpm/` 目录内部。
- **`plugin.config.json` 由 CLI 维护**：点位信息通过 `local-config set` + `update --source-type=remote` 同步，`resources` 数组里的 `id` / `entry` 由 `update` 生成。不要用 Edit/Write 直接改这个文件——手工改 CLI 下次 update 会对不上。其他 skill 需要点位信息时读它、不写它。
- **禁止输出密钥**（accessToken、pluginSecret）到终端明文
- **真实数据动作前必须显式确认**（创建插件 / 配置点位 / 配置基本信息 / 发布 —— 见根原则 3）
- **全量提交约束**：见下方独立小节

### 全量提交约束（CRITICAL — 防数据丢失，本规则唯一权威源）

`local-config set` 和 `update --source-type=local` 均为**全量替换**——提交什么就存什么，远端不做合并。**遗漏任何现有点位都会导致该点位被永久删除。**

**操作流程（不可省略任何一步）**：
1. 先 `local-config get --remote` 获取远端完整配置作为基础
2. 在完整配置基础上做局部增删改
3. 将修改后的**完整配置**提交（包含所有现有点位类型、所有点位实例）
4. 推送成功后执行 `npx @byted-meego/cli@builder update` 将远端配置同步回本地

**不要做的事**：
- 不要只传变更部分（会导致未传的点位被删除）
- 不要直接 Edit/Write `plugin.config.json`（始终通过 CLI 命令同步）
- 不要在没有 `get --remote` 拿到基线的情况下直接构造 draft

**已知执行点**（各 skill 必须按此规则落地）：
- `meegle-plugin-feature/references/config-apply.md` — Stage Config 的 config.apply 步骤
- `meegle-plugin-cli` 中的"点位配置管理"原子命令
- `meegle-plugin-workflow` 走 Phase 2 时由被调方（feature）执行

### 删除点位前置检查协议（CRITICAL — 适用于所有阶段和所有调用路径）

执行 `local-config set` 之前，对比即将提交的 JSON 与远端配置（`local-config get --remote`）。如果提交的 JSON 相比远端**减少了任何点位**（整个类型缺失或某个 key 缺失），**立即暂停**，向用户列出被删除的点位清单（类型 + key + name），获得明确确认后才能执行 set。不要静默删除。

此检查是 mode-agnostic 的前置 gate——无论从 config.plan / config.apply / pipeline / meegle-plugin-workflow 哪条路径进入，都要走一次。以下 skill 的 plan/apply 是已知执行点，须各自落实：
- `meegle-plugin-feature/references/config-apply.md` A0 步骤
- `meegle-plugin-workflow` 新插件流程在走 Stage Config 时由被调方（meegle-plugin-feature）执行（编排层不重复检查）

### 无源即停（CRITICAL — 全局通则，统御所有"溯源协议"）

**适用范围**：URL、代码、API 调用、字段值、配置参数等所有 AI 输出场景。

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

**遇到无源时的统一动作（强制话术 + 强制停产出）**：

触发"无源即停"时 AI **MUST**：

1. **立即停止产出**：本轮不得再调任何产出工具（Write / Edit 写入点位 JSON、代码文件、CLI 命令等）。"停下来问用户" ≠ "嘴上停、同时继续写"。
2. **逐字输出下述话术**（允许填入具体内容、可选项 A/B/C 可剪裁为场景相关的版本，但"无法找到 … 依据 / 可选 / 禁止猜测 / 请告诉我选哪个" 四要素必须齐全）：

```
我无法在合法信息源中找到 "<具体内容>" 的依据。

可选：
A. 跳过此项，输出 TODO 占位（推荐——不写错的好过写错的）
B. 你提供真实值 / 示例 / 文档链接
C. 提供检索关键词，我重新查（可能找错了）

禁止我自行猜测——猜出来的内容会通过表层校验但运行时全废。
请告诉我选哪个。
```

3. **等待用户明确选择 A/B/C 之一** 才能继续；不得"选 A 同时继续产出其他项"的并发动作（同一批产出里任一项无源，整批阻塞）。

**输出标注溯源（机器可审计）**：
每项非平凡输出在邻近位置标注 source，便于 reviewer agent 和用户审计：
- 配置字段：`# source: 用户提供 @ 2026-04-14`
- 代码调用：`// source: js-sdk types/index.d.ts:Control.method`
- URL：`# source: 用户授权占位 @ 2026-04-14`

未标注 source 的内容 → reviewer 视为疑似编造，要求作者补来源或删除。

**已落地的具体协议**（均遵循此通则）：
- URL 溯源：`meegle-plugin-feature/references/config-plan.md` 的 "URL 溯源协议"
- 代码溯源：`meegle-plugin-feature/references/code-plan.md` 的 "代码溯源协议"
- 后续新场景（如自定义脚本生成、SQL 生成等）应继承此通则各自落实

### 禁止编造 URL（CRITICAL · 根原则 1 无源即停 · URL 域应用）

**绝对禁止编造任何 URL 或图标地址。** 编造的 mock URL 在运行时完全不可用，会导致插件功能失效。AI 永远不主动创造 URL 字符串——每个值必须来自"用户显式输入"或"用户授权占位"。

> **CLI 强制兜底**：`local-config set` / `update --source-type=local` 在提交前会自动扫 Runtime URL 占位符（example.com / your-server / `.test`/`.example`/`.invalid` / localhost / `<PLACEHOLDER:...>` 等），命中即 `exit 1`。AI 即使在生成压力下漏填也会被拦住——但这不代表可以省略 plan P3 的主动询问，validator 是**防线**，不是**替代**。

**URL 字段分两类**：
- **Runtime URL**（运行时会发起请求，如 `intercept.url` / `table_url.url` / `listen_event.url`）→ 必须真实可达 → **必须向用户询问**；用户暂无 → 占位符 `<PLACEHOLDER: ...>` + 发布前硬阻
- **Metadata URL**（如 `liteAppComponent.icon_url` / 插件 icon）→ 不填不影响功能 → 不阻塞

**涉及 URL 字段填写的 skill**（meegle-plugin-feature 的 config.plan/config.apply、meegle-plugin-publish 的 apply 阶段）**MUST Read** [`references/url-policy.md`](references/url-policy.md) 获取：
- 完整字段清单（Runtime vs Metadata 对照表）
- 变量声明 → 数据接口 URL 的联动规则（防止整列渲染失败）
- `<PLACEHOLDER>` 与 schema 冲突时的兜底流程
- 禁用的"看起来像真的"假地址黑名单

## Checkpoint（进度追踪）

**判断规则**：子 skill 执行前读项目根 `.lpm-cache/state.json`：
- **存在** → 处于 workflow 编排中，按 [`references/checkpoint.md`](references/checkpoint.md) 的协议在每个 CLI 命令前后更新 checkpoint
- **不存在** → 独立调用，不写 checkpoint

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| CLI 未安装 | 提示 `npm install -g @byted-meego/cli@builder` |
| Token 过期 | 自动尝试 refresh；失败则引导重新执行 `/meegle-plugin-env-setup` |
| 网络错误 | 展示原始错误，建议检查网络后重试 |
| `plugin.config.json` 缺失 | 引导先执行 `/meegle-plugin-workflow` 创建新插件（或 `/meegle-plugin-create` 单独创建空工程） |
