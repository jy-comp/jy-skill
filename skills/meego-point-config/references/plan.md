# mode=plan：理解需求 + 交互补全 + 生成配置文件

## 点位类型速查

| 类型 | 一句话用途 |
|------|---------|
| `page` | 顶部导航扩展页 |
| `view` | 工作项列表自定义视图 |
| `dashboard` | 独立仪表盘页 |
| `config` | 插件级配置入口 |
| `control` | 工作项详情页的自定义控件（表单字段/展示块） |
| `button` | 工作项详情页的自定义按钮 |
| `intercept` | 状态流转拦截器（同步返回放行/拒绝） |
| `listen_event` | 工作项事件异步监听（创建/更新/状态变更等） |
| `component` | 通用组件位（目前支持轻应用） |
| `customField` | 扩展字段（列表/表单展示 + 数据接口） |
| `liteAppComponent` | 轻应用组件（搭建器拖拽组件） |

> **必填字段 / 长度 / 枚举合法性以 `npx @byted-meego/cli@builder schema` 输出为准**，由 `local-config set` 强制校验，本表不再镜像——避免 schema 迭代后漂移。schema 抓不到的行为指引（URL 询问、MCP 强制查询、派生字段规则等）见 `references/plan.md` 的"各类型的行为指引"节。


## P1：识别操作类型和点位类型

结合以下信息综合判断：
- 用户的原始描述（添加/修改/删除）
- 飞书项目知识 MCP（查询 work_item_type 枚举值、event_type 编号等业务背景）
- `.lpm-cache/schema/point-schema.json` 中的字段约束和枚举

**若由 plugin-workflow 调用**：会收到 `context.originalRequirement`（用户原话，作为主输入）和可选的 `context.mentionedPointType`（用户主动提到的点位类型）。`mentionedPointType` 有值时直接按该类型走（只需用 schema 切片验证其存在），不再做推荐；为 null 时按常规流程做意图识别和术语消歧后向用户确认点位方案。无论哪条路径，key/name/description 等字段都在 P3 交互补全，不靠上游"推导默认值"。

**MCP 缓存协议（防 context 爆炸）**：返回后立即 Write 到 `.lpm-cache/mcp/<slug>.md`；回复只给路径 + ≤100 字摘要，禁止原文回流；同一查询先 Read 缓存，7 天以上视为过期重拉。CLI 在 create/init 时自动把 `.lpm-cache/` 写入 `.gitignore`，并在 `local-config set` / `update` / `publish` 成功后按子目录清理，AI 只负责写入、不负责清理。

## P2：获取 Schema 和当前完整配置

```bash
npx @byted-meego/cli@builder schema
npx @byted-meego/cli@builder local-config get --remote
```

- 两条命令都是**文件优先**：CLI 把结果写到 `.lpm-cache/schema/point-schema.json` 和 `.lpm-cache/config/remote.json`，stdout 只回一行相对路径
- **两个命令并行执行**，不要等第一个完成再跑第二个
- 若 `local-config get` 对应的远端为空，`remote.json` 内容是 `{}`，作为后续增改的基础
- 后续所有消费（jq 切片、写 draft、diff 对比）都走文件，**不要 Read 整个 JSON 到 context**

### 按点位分片读取 schema

`.lpm-cache/schema/point-schema.json` 约 2400+ 行（~30K tokens）。用 jq 切片按需取用，不要 Read 整文件。

```bash
# 本次要配置的点位类型的完整 definition（把 LiteAppComponentPoint 换成实际 definition 名）
jq '.definitions.LiteAppComponentPoint // .integrate_point_schema.definitions.LiteAppComponentPoint' .lpm-cache/schema/point-schema.json

# 自动解析点位类型对应的 definition 名（liteAppComponent → LiteAppComponentPoint）
jq -r '(.properties.liteAppComponent["$ref"] // .integrate_point_schema.properties.liteAppComponent["$ref"]) | sub("#/.*definitions/"; "")' .lpm-cache/schema/point-schema.json

# 子结构 definition（如嵌套引用的 PlatformWebForControl / WorkItemTypeForButton）
jq '.definitions.PlatformWebForControl // .integrate_point_schema.definitions.PlatformWebForControl' .lpm-cache/schema/point-schema.json
```

**只取当前要操作的点位类型** + 它引用的子 definition，其他点位类型的 definition 不要取。生成配置后，`local-config set` 会做完整 schema 校验，不需要 AI 先通读全 schema。

### 按需读取 remote.json（对称 schema 分片）

`.lpm-cache/config/remote.json` 是远端全量配置，点位数多时 JSON 可能很大。用 jq 切片按需取，不要 Read 整文件。

```bash
# 类型 + key 列表（做 diff / 删除确认 的最小视野）
jq 'to_entries | map({type: .key, keys: [.value[].key]})' .lpm-cache/config/remote.json

# 读某个类型下的某个点位（本次要改 / 要比对的那个）
jq '.liteAppComponent[] | select(.key=="lite_app_xxx")' .lpm-cache/config/remote.json

# 读整个类型下所有点位（append 新点位、需要保留同类原样时）
jq '.liteAppComponent // []' .lpm-cache/config/remote.json

# 所有 key 扁平集合（仅用于删除确认的集合比较）
jq '[.[][].key]' .lpm-cache/config/remote.json
```

写 draft 时也按类型 jq 拼装，不要把 remote.json 整块 Read 进 AI 上下文再拼。

### 术语→点位类型映射（"组件"一词歧义）

schema 中 `liteAppComponent` 叫"轻应用组件"、`component` 叫"组件位"，但用户日常口语里"组件"可以指任何东西。识别规则：**先从用户描述里抽核心名词（拓展字段 / 控件 / 轻应用 / 排期），再映射点位类型。"组件"二字是修饰语，不是判断依据。**

| 用户可能的说法 | 核心名词 | 正确点位类型 | 易错映射 |
|--------------|---------|------------|---------|
| "拓展字段"/"拓展字段组件"/"字段模板"/"自定义字段" | **拓展字段** | `customField` | ~~liteAppComponent~~ |
| "控件"/"控件组件" | **控件** | `control` | ~~liteAppComponent~~ |
| "轻应用组件"/"概览组件"/"构建器组件"/"拖拽组件" | **轻应用/概览/构建器** | `liteAppComponent` | — |
| "排期组件"/"节点排期" | **排期** | `component` | ~~liteAppComponent~~ |

## P3：交互补全缺失字段

**规则：一次只问一个问题**，语义化表达（例：不说"key 是什么"，而说"这个点位的唯一标识符打算用什么？建议格式如 my_board_point"）。

### 禁止编造 URL

完整判定规则（含 Runtime / Metadata 分类表与所有 URL 字段清单）见 [`meego-shared/references/url-policy.md`](../../meego-shared/references/url-policy.md)。本节补充 P3 阶段的具体执行动作。

### Runtime URL 的统一询问流程

对本次计划生成的每个点位，按下表**从点位类型 + DSL 内容联合判断**要询问哪些 URL：

| 触发条件 | 要询问的 URL |
|---------|-------------|
| 点位类型是 `intercept` / `listen_event` | 顶层 `url`（始终问） |
| 点位类型是 `control` 且用户提到"新建页可见" / Webhook 提交 | `control.url`（仅此场景问） |
| 点位类型是 `control` 且 `table_cell.definitions.data` 非空 | `platform.web.table_url.url`（见下方专门规则） |
| 点位类型是 `customField` 且 `table_layout.definitions.data` 非空 | `platform.web.table_data_url`（同上） |
| 任意 DSL（table_cell / table_layout）里有节点用了 `onClick/onDoubleClick.action=httpRequest` | 对应 `params.url`（按用户业务语义问） |
| 任意 DSL 里有节点用了 `onClick.action=openLink` 且 URL 是字面量（非 `{{varName}}` 引用） | 对应 `params.url` |

**扫描方法**：生成 JSON 前先扫一遍 template 树，找出所有上述触发点，把 URL 询问合并成一轮对话问给用户（不要每个点位单独问一次）。

**询问话术模板**：
> "本次计划里有 N 个 URL 字段需要你提供真实地址，否则上线后对应功能会失败：
> 1. `intercept` 点位的回调 URL
> 2. 点位 `control_xxx` 的表格列数据接口（`table_cell` 里声明了变量 `varName1, varName2`，列渲染需要）
> 3. 点位 `control_yyy` 里 popover 的 httpRequest URL
> ...
> 对每一项请告诉我：真实地址 / 先占位（发布前补）/ 改方案去掉这个 URL 需求。"

**三路分支**：

- **(A) 真实 URL** → 直接写入 URL 字段；**所有配套 token 字段留空，由 CLI 自动生成 36 位 UUID**
- **(B) 先占位** → 写入 `"<PLACEHOLDER: 请替换为你的<字段语义>>"`；若 schema 拒绝纯文本占位符，按 `meego-shared` 的"规则冲突兜底"向用户上报再决策；P6 摘要用 🔴 醒目标注，plugin-publish 的 pre-check 会硬阻止
- **(C) 改方案** → 如用户不想提供 URL，引导改造点位（例：声明变量的 DSL 改成裸 template / 把 httpRequest 换成 openLink / 放弃 intercept 这个点位）


### 对于修改操作

1. jq 定位目标点位（不要 Read 整文件）：`jq '.<type>[] | select(.key=="<key>")' .lpm-cache/config/remote.json`
2. 仅询问需要修改的字段
3. 其余字段保持原值

## P4：生成完整配置文件

> set 是全量替换（详见 [`../SKILL.md`](../SKILL.md#全量提交约束critical--防数据丢失)）——基于 `.lpm-cache/config/remote.json` 的远端全量配置操作，而不是只传变更部分。

### 添加操作

在对应类型数组中 append 新点位对象，**保留所有其他类型和其他点位不变**。

```
远端: { "page": [A], "button": [B1, B2] }
用户: 新增一个 liteAppComponent
结果: { "page": [A], "button": [B1, B2], "liteAppComponent": [新点位] }
```

对于 Single 类型（page/view/dashboard/config/intercept/listen_event/component），如果远端已有该类型的点位，不能再添加（maxItems=1，schema 会拒绝）。应提示用户是要替换还是修改现有点位。

### 修改操作

找到匹配 key 的点位，合并更新字段，**其余字段和其他点位原样保留**。

```
远端: { "button": [{ "key": "button_x1", "name": "旧名" }, { "key": "button_x2", ... }] }
用户: 把 button_x1 的名字改成"新名"
结果: { "button": [{ "key": "button_x1", "name": "新名" }, { "key": "button_x2", 原样 }] }
```

### 删除操作（CRITICAL — 须用户确认）

从对应类型数组中移除匹配 key 的点位，**其余原样保留**。

```
远端: { "page": [A], "button": [B1, B2] }
用户: 删除 B2
结果: { "page": [A], "button": [B1] }

用户: 删除整个 button 类型的所有点位
结果: { "page": [A] }
```

（如果只传 `{ "liteAppComponent": [新点位] }` 而不包含 page 和 button，远端的 page 和 button 会被清除。）

**删除确认规则**：生成的完整 JSON 相比远端减少了任何点位时，先向用户列出被删除的点位清单（key + name）并获得确认，再执行 set——不要静默删除。

生成文件名：`.lpm-cache/config/draft-{YYYY-MM-DD_HHmmss}.json`

### 生成前自检

- [ ] 是否为**全量配置**（包含远端所有现有类型+本次变更，非仅变更部分）
- [ ] key 在同类型中唯一（不与现有点位冲突）
- [ ] Single 类型未超过 maxItems=1
- [ ] 必填字段均已填写
- [ ] 枚举值合法（mode、mobile_block_style、field_type、view.icon）
- [ ] name 长度在限制内
- [ ] platform.web 字段按当前点位类型对应的 `PlatformWebFor{Type}` 取（例：`table_url` 是 customField 专有，`mode`/`init_size` 是 button 专有，不跨类型混用）

## P5：输出

plan 阶段完成后输出：
- 生成的配置文件路径：`.lpm-cache/config/draft-{timestamp}.json`
- 变更摘要：添加/修改/删除了哪些点位（类型 + key）

> 中间产物（`.lpm-cache/config/remote.json` / `draft-*.json` / `schema/point-schema.json` / `mcp/*.md`）由 CLI 在 `local-config set` / `update` / `publish` 成功后自动清理，skill 不负责清理。
