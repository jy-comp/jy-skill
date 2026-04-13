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

### 禁止编造 URL（CRITICAL）

**绝对禁止编造任何 URL 或图标地址。** 编造的 mock URL（如 `https://example.com/webhook`）在实际使用中完全不可用，会导致插件功能失效。

各类 URL 字段的处理策略：

| 字段 | 所属点位 | 处理方式 |
|------|---------|---------|
| `intercept.url` | intercept | **必须向用户询问**（业务回调地址，无法猜测） |
| `listen_event.url` | listen_event | **必须向用户询问**（同上） |
| `control.table_url.url` / `customfield.table_data_url` | control / field_template | **必须向用户询问**（webhook 数据接口地址） |
| `template.onClick.params.url` (action=httpRequest) | table_cell DSL | **必须向用户询问**（业务回调地址） |
| `builder_comp.icon_url` | builder_comp | **不阻塞**，不填则 CLI 自动填充默认图标 |
| 插件图标 icon | update-description | **不阻塞**，不传则保持后台默认图标 |

**执行原则：**
- `url` 类字段（intercept、listen_event、table_url、httpRequest.params.url）：**必须暂停流程向用户询问**，这些是业务回调地址和密钥，无法猜测
- `icon_url` / 插件图标：**不阻塞流程**，不填即可，CLI/后台会自动使用默认图标
- 如果用户暂时没有 url，可以使用**不可能被误认为真实地址的占位符**（如 `"<PLACEHOLDER: 请替换为你的回调地址>"`）并在摘要中醒目标注待替换项
- **禁止使用** `https://example.com`、`https://your-server.com`、`https://PLACEHOLDER.invalid/...` 等看似真实但不可用的地址

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
