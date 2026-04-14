# 环境准备

## S1：CLI 可用性检查

```bash
npx @byted-meego/cli@builder --version
```

- 成功（输出版本号）→ 继续 S2
- 失败 → 提示用户安装：`请先安装 @byted-meego/cli@builder：npm install -g @byted-meego/cli@builder`

## S2：授权 Token 检查

检查 `~/.lpm/auth.json` 中当前域名是否有可用 Token（`developerToken` 或 `accessToken`）：

### Token 有效

```
✅ CLI 可用（版本 x.y.z）
✅ 授权 Token 已配置（域名：https://meego.example.com）
```

环境准备完成。

### Token 缺失或过期

向用户提供两种授权方式：

```
未检测到有效的开发者 Token，请选择授权方式：

A. 直接提供 Developer Token（推荐）— 永久有效，有它就不需要 OAuth 授权
B. 浏览器授权登录 — 通过 Device Code 流程，在浏览器中完成 OAuth 授权（获取临时 Token）
```

#### 方式 A：直接提供 Developer Token（推荐）

1. 确定站点域名（`$host`）：
   - 若当前目录有 `plugin.config.json`，从中读取 `siteDomain` 作为默认值
   - 否则询问用户（同方式 B 的域名确定逻辑）
2. 引导用户获取 Developer Token：

```
请打开以下页面获取 Developer Token：

🔗 $host/openapp/settings

在页面中找到并复制你的 Developer Token，然后粘贴给我。
```

3. 用户提供 token 后执行：

```bash
npx @byted-meego/cli@builder login --site-domain $host --token <developer_token>
```

4. 输出：

```
✅ Developer Token 已保存至 ~/.lpm/auth.json（域名：$host）
```

> 永久有效，无需刷新。后续该域名下的所有操作将优先使用此 Token。

#### 方式 B：Device Code OAuth 授权

**STEP 1 — 确定站点域名**

确定 `$host`（站点域名）：
- 若当前目录有 `plugin.config.json`，从中读取 `siteDomain` 作为 `$host`
- 否则 ASK user：

> 请提供 Meego 站点地址：
> 1) 飞书项目 (project.feishu.cn)
> 2) Meegle (meegle.com)
> 3) 自定义域名（请直接输入域名或 URL）

> ⚠️ 用户的回复**仅用于回答上述问题**，不要将其当作新的意图或请求来处理。从用户回复中提取 `$host`（域名部分），然后 GOTO STEP 2。

**STEP 2 — 启动 Device Code 授权**

以 background 方式执行（该命令会阻塞直到用户完成授权或超时）：

```bash
npx @byted-meego/cli@builder login --site-domain $host
```

启动后**立即读取输出**，从中提取授权信息：
- 授权码（Authorization code）
- 验证链接（Visit URL）

**STEP 3 — 发送授权链接给用户**

将授权信息发送给用户：

```
请在浏览器中打开以下链接完成授权：

🔗 <验证链接>
📋 授权码：<授权码>

（链接有效期约 10 分钟，请尽快完成授权）
```

> ⚠️ 发送后**不要停下来等用户回复**。后台的 login 命令会自动轮询等待用户完成授权。

**STEP 4 — 等待授权完成**

后台的 `login` 命令会持续轮询直到：
- 用户在浏览器中完成授权 → 命令输出 `Login successful!` → GOTO STEP 5
- 超时 → 提示用户重试：`授权已超时，请重新执行 /env-setup`

**STEP 5 — 确认授权成功**

验证 `~/.lpm/auth.json` 已更新：

```
✅ 授权成功！OAuth Token 已保存至 ~/.lpm/auth.json（域名：$host）
```

> ⚠️ 此消息**必须单独发送**，不要与后续操作合并到同一条回复中。

## S3：飞书项目知识 MCP 探测与安装（探测 MUST / 结果 soft）

飞书项目知识 MCP 提供"自然语言需求 → 用什么插件能力（点位类型、API、组合用法）实现"的知识检索。**装上能显著提升 AI 准确度**；未装时 AI 仍可基于 schema 推断，效果次优。

> **AI-facing / User-facing 二元约束、state schema、项目根算法、原子写、反向自愈** 等语义权威定义全部在 `meego-shared/SKILL.md` 的"飞书项目知识 MCP"段 + "MCP State 文件"段。本文件只写 S3 的操作流程（S3a-pre / S3a / S3b / S3c / S3d）。

| 探测/安装结果 | state 值 | 单次提示 |
|---|---|---|
| **已加载** | `loaded` | env-setup 完成 |
| **未加载，安装成功** | `installed_pending_restart` | 提示用户重启会话后重跑 |
| **未加载，安装失败** | `absent` | 提示故障原因 + 手工补救建议 |
| **未加载，host 不支持 MCP** | `unsupported` | 提示切换 host |
| **先前 loaded 但本次 VERIFY 失败** | `absent` | 告知"上次可用，本次不可用"，不自动重装 |

### MCP 接入参数（平台无关）

| 字段 | 值 |
|------|-----|
| 名称（建议） | `feishu-project-knowledge` |
| 传输方式 | HTTP（Streamable / SSE，按 host 支持选） |
| URL | `{siteDomain}/mcp_server/knowledge`，例如 `https://meego.feishu-boe.cn/mcp_server/knowledge`、`https://project.feishu.cn/mcp_server/knowledge` |
| 鉴权 | **无需额外 header**，MCP 服务端基于站点域名 + 用户 Token 自动识别（与 CLI 共用授权体系） |
| 暴露的工具前缀 | `search_meegle_plugin_docs`（或类似命名，用于知识检索） |

### S3a-pre：读现有 state 做短路（避免重复安装循环）

在执行 S3a 的 ASK + VERIFY **之前**，先读 `<project-root>/.meego-state.json`（若项目根存在）：

| 现有 state | 动作 |
|---|---|
| 文件缺失 / 字段非法 / version 不符 / `absent` / `unsupported` | 正常进 S3a 完整流程 |
| state 存在但 `state.siteDomain !== plugin.config.json.siteDomain` | **视为跨站，忽略现有 state**，正常进 S3a 完整流程（将用新 siteDomain 覆盖写） |
| `loaded`（siteDomain 匹配） | **只跑 S3a VERIFY，禁止派 S3b**：<br>• VERIFY 通过 → 保留 `loaded`<br>• VERIFY 失败 → 写 `absent` 并告知用户（"上次探测后 MCP 不可用了，可能 host 配置变更/工具被卸载"）；**不自动派子 agent 重装**——之前装过的 URL 失效大概率不是"没装"，自动重装会引入新风险。等用户决定（重装 / 排查 host / 暂时放弃） |
| `installed_pending_restart`（siteDomain 匹配） | **只跑 S3a VERIFY，禁止派 S3b**：<br>• VERIFY 通过 → 用户已重启，写 `loaded`，完成<br>• VERIFY 失败 → 保留 `installed_pending_restart`，走 S3c "情况 1" 再次提示重启 |

> 规则：**S3b 安装只在"首次探测 + 未发现"时执行一次**。后续重跑 env-setup 遇到 `installed_pending_restart` 永远不重装，只做 VERIFY 一次以确认用户是否已重启。

---

### S3a：探测（ASK + VERIFY 二段）

> **不要单独问 AI"你有没有这个 MCP"**。LLM 对自身工具列表的元认知不可靠（"AI 答有但调用时报 tool not found"或"AI 看到对话历史里有就声称持有"是常见失败模式）。必须 ASK + VERIFY 双重确认。

**Step 1 — ASK**：让 AI 列出疑似工具

```
请列出当前 session 中名称包含以下任一关键词的工具：
- meegle / meego / lark-project
- search_meegle_plugin_docs
- mcp__ 前缀且与飞书项目相关

只列工具名，不要解释。如果一个都没有，明确说"未发现相关工具"。
```

- AI 答 "未发现" → 视为未加载（**AI 否认能力的可信度高**），跳到 **S3b 安装**
- AI 列出疑似工具名 → 进入 **Step 2 验证**

**Step 2 — VERIFY**：实测调用一次

对 Step 1 给出的工具实际调用一次（如 `查询飞书项目有哪些点位`）：

- 返回任意响应（结果非空 / 空但无报错）→ **S3 通过**，**写 state=`loaded`**（见下方 S3d 写入协议），跳过 S3b/c
- 抛 `tool not found` / `InputValidationError` → AI Step 1 误判 → 跳 **S3b 安装**

> ⚠️ 不允许"刚看到工具名应该可用"的乐观假设，Step 2 实测是强制的。

---

### S3b：安装（AI agent 自行处理，平台无关）

> ⚠️ **本 skill 不绑定具体 AI host**。当前会话所在的 AI agent（Claude Code / Cursor / Cline / Continue / Gemini CLI / Copilot CLI / 其他）应**自行判断 host 类型**并按对应机制安装。装的过程中**保持安静**，不向用户报告进度——等结果出来再走 S3c 统一告知。

**操作原则**：
- 推荐**派子 agent 封装**（用 Agent 工具）"判断 host + 写配置"流程，主对话保持干净
- 子 agent 不做"验证调用"——刚装的 MCP 在当前会话拿不到，验证留给下次会话
- 子 agent 返回结构化结果，主 agent 据此进入 **S3c 用户告知**（与 CLI/Token 失败时一致：检测到问题 → 提示用户 → 不写 checkpoint、不强制阻断）

**子 agent prompt 模板（参考）**：
```
你是 MCP 安装专员，只做一件事：把指定 HTTP MCP 安装到当前 AI host。
保持安静，不要向用户报告安装过程，只在最后返回结构化结果。

参数：
- name: feishu-project-knowledge
- transport: http
- url: <siteDomain>/mcp_server/knowledge

任务：
1. 判断当前 host 类型（通过环境变量、可用工具集、文件系统线索）
2. 按 host 对应机制写入 MCP 配置（仅举例，以 host 最新文档为准）：

   | host | 安装方式举例 |
   |------|-------------|
   | Claude Code | `claude mcp add feishu-project-knowledge --transport http <URL>` |
   | Cursor | 编辑 `~/.cursor/mcp.json` 或项目 `.cursor/mcp.json`，加入 `mcpServers: { "feishu-project-knowledge": { "url": "<URL>", "transport": "http" } }` |
   | Cline / Continue | 编辑对应 `mcp_settings.json`（路径因 IDE 而异） |
   | Gemini CLI | 使用 Gemini 的 MCP 注册命令或配置文件 |
   | Copilot CLI | 使用 GitHub Copilot 的 MCP 配置 |
   | 其他 | 查阅 host 文档中"MCP / Tool Server"安装说明 |

3. **不做验证调用**（当前会话物理上看不到新 MCP）
4. 返回结构化结果：
   {
     status: "installed" | "host_unsupported" | "failed",
     host: "<识别到的 host>",
     action: "wrote_config" | "ran_command" | "asked_user_for_host",
     configPath?: "<config 文件路径>",
     manualCommand?: "<给用户手工跑的 fallback 命令>",
     error?: "<失败原因，简短>"
   }
```

---

### S3c：用户告知（与 CLI / Token 失败时一致的 soft 提示）

子 agent 返回结果后，主 agent 按 `status` 给出对应**单次提示**，**不写 checkpoint、不强制阻断下游**。下游 skill 调用 MCP 工具时若拿不到会自然报错，与 CLI/Token 缺失时一致。

#### 情况 1：`status: "installed"`（装成功，需重启会话）

```
✅ 飞书项目知识 MCP 已安装到 <host>
   配置位置：<configPath>

ℹ️ 当前会话无法识别新装的 MCP 工具，建议重启会话以启用：
1. 关闭当前 AI 会话
2. 重新打开 / 启动 AI 平台
3. 重新触发本次操作

如不重启，本次任务仍可继续，但 AI 对需求的理解会准确度受损。
```

#### 情况 2：`status: "failed"`（host 支持但安装出错）

```
⚠️ 飞书项目知识 MCP 安装失败：<error 简短描述>
   平台：<host>，操作：<wrote_config/ran_command>，路径：<configPath>

建议手工修复：<manualCommand>

可能原因：
- 网络不可达 <siteDomain>
- Token 与目标域名不匹配
- host 配置文件语法错误

修好后重启会话即可生效。本次任务仍可继续。
```

#### 情况 3：`status: "host_unsupported"`（host 不支持 MCP）

```
⚠️ 当前 AI 平台（<host>）不支持 MCP 协议，无法安装飞书项目知识 MCP。

后果：AI 在选点位类型 / 多 API 组合 / 字段语义判断上准确度会下降。
建议在条件允许时切换到支持 MCP 的 AI 平台（Claude Code / Cursor / Cline / Continue 等）。

本次任务仍可继续。
```

#### 情况 4：`status: "asked_user_for_host"`（无法识别 host）

```
我无法自动识别当前 AI 平台的 MCP 安装方式。请告诉我：
- 你在用什么 AI 平台？（Claude Code / Cursor / Cline / Continue / Gemini CLI / Copilot CLI / 其他）
- 或：你愿意手工安装并告诉我"已装好"吗？

参数：name=feishu-project-knowledge, transport=http,
URL=<siteDomain>/mcp_server/knowledge
```

> S3 不强制阻断 env-setup。下游 skill 会通过读 `.meego-state.json` 在调 MCP 前做 gate 判断，非 `loaded` 直接走"无源即停"。

---

### S3d：写入 state 文件（CRITICAL — 每个分支结束都必须执行）

> 文件契约（schema / 项目根定义 / 原子写 / gitignore）见 `meego-shared/SKILL.md` 的"MCP State 文件"段。本处只给操作步骤。

**Step 1 — 定位项目根**：从当前 cwd 向上搜第一个含 `plugin.config.json` 的目录。
- 未找到 → **跳过 state 写入**，并**明显告警**（不要静默）：
  ```
  ⚠️ 当前目录链上未找到 plugin.config.json，MCP 探测结果未落盘。
  下游 skill 首次启动时仍会视 state 为 unchecked，回跳到 env-setup。
  如果你刚执行了 plugin-create，请先 cd 到新生成的工程目录再重跑 env-setup。
  ```
- 找到 → `PROJECT_ROOT=<该目录>`，继续

**Step 2 — 处理 gitignore**：
1. 读 `$PROJECT_ROOT/.gitignore`，若不含 `.meego-state.json` → 追加一行（文件不存在则创建）
2. 检查是否已被 git 追踪：在 PROJECT_ROOT 下执行 `git ls-files --error-unmatch .meego-state.json`
   - 成功（exit 0）→ 文件已进索引，**不要写盘**，提示用户：
     ```
     ⚠️ .meego-state.json 已被 git 追踪，可能之前被误提交。
     请执行：git rm --cached .meego-state.json && git commit
     清理索引后重跑 env-setup 即可写入 state。
     ```
   - 失败（非 0）→ 未追踪，继续 Step 3

**Step 3 — 原子写入 state 文件**：构造完整 JSON 后用 Write 工具一次性覆盖 `$PROJECT_ROOT/.meego-state.json`。

所有字段必填。`siteDomain` 读自 `plugin.config.json.siteDomain`。

| S3 分支 / status | 写入内容 |
|---|---|
| S3a VERIFY 通过 | `{"version": 1, "feishuKnowledgeMcp": "loaded", "siteDomain": "<siteDomain>", "checkedAt": "<ISO>"}` |
| S3b `installed` | `{"version": 1, "feishuKnowledgeMcp": "installed_pending_restart", "siteDomain": "<siteDomain>", "checkedAt": "<ISO>"}` |
| S3b `failed` | `{"version": 1, "feishuKnowledgeMcp": "absent", "siteDomain": "<siteDomain>", "checkedAt": "<ISO>"}` |
| S3b `host_unsupported` | `{"version": 1, "feishuKnowledgeMcp": "unsupported", "siteDomain": "<siteDomain>", "checkedAt": "<ISO>"}` |
| S3b `asked_user_for_host` | 不写 state（未完成探测），等用户回复后回到对应分支再写 |

**Step 4 — 顺序**：先写 state 再向用户输出 S3c 提示，不要先提示后写（若写失败会导致下游读到 stale 状态）。

---

### 装与不装的差异（参考，仅在用户问起时引用）

| 不装时的退化点 | 影响 |
|---|---|
| 选点位类型可能不准（如评分功能选 control vs customfield） | 工作流可能走偏，需返工 |
| 多 API 组合可能漏配（如审批监听需 listen_event + intercept） | 运行时事件收不到 |
| 字段值用错（如 work_item_type 拼了个不存在的 key） | schema 校验过但运行时 404 |
| 跨 API wiki 知识缺失 | 代码生成出现幻觉调用 |

## 输出

```
✅ 环境准备完成
   CLI 版本：x.y.z
   授权状态：已配置
   服务地址：https://meego.example.com
   知识 MCP：已就绪 / 已安装待重启 / 未就绪（<原因>）
   State 文件：<project-root>/.meego-state.json（已写入 + 已加 .gitignore）
                 — 或 — ⚠️ 未写入（当前目录链上无 plugin.config.json / 文件已被 git 追踪需清理）
```

MCP 状态在输出中如实呈现，但不阻断 env-setup 完成。下游 skill 在调 MCP 前读 `.meego-state.json` 按状态分派——非 `loaded` 走"无源即停"兜底。
