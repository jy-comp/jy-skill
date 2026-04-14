# mode=plan：理解需求 + 交互补全 + 生成配置文件

## P1：获取 Schema 和当前完整配置

```bash
npx @byted-meego/cli@builder schema > point-schema.json
npx @byted-meego/cli@builder local-config get > plugin.temp.local-remote.json
```

- 若 `local-config get` 返回空，以 `{}` 作为基础
- **两个命令并行执行**，不要等第一个完成再跑第二个
- 这两个命令的 stdout 是纯 JSON（CLI v2.6.0+ 会自动避免输出 `[INFO]` 日志），若旧版 CLI 残留日志需 `2>/dev/null` 去噪后再使用
- 文件后缀使用 `.json`（内容就是 JSON）；勿再沿用 `.yaml` 扩展名

## P2：识别操作类型和点位类型

### P2.0：MCP 调用协议（前置 gate + 后置兜底）

调 MCP 工具（如 `search_meegle_plugin_docs`）的**前后**都按"无源即停"处置，**一段协议收口**：

**前置**：调用前 MUST 读 `<project-root>/.meego-state.json` 按状态分派——完整分派表、siteDomain 校验、反向自愈、强制阻塞语义，**全部**见 `meego-shared/SKILL.md` 的"MCP State 文件"段。本处不重复。

**后置**（MCP 返回后视为"无合法信息源"的情形）：
- `tool not found` / `InputValidationError`（MCP 断链） → 触发反向自愈（见 meego-shared），state 回写 `absent`
- 调用超时 / 401 / 403 → 同上
- 返回空结果 / 明显无关片段 / 覆盖不全 → 按"无源即停"停下问用户，state 保持 `loaded`（是结果问题不是工具问题）

**本 skill 专属的禁止降级动作**（通用"无源即停"在点位配置场景的落地）：
- ❌ 用 `point-schema.json` 的字段名 / 枚举值脑补业务含义（schema 只告字段形状，不告字段语义）
- ❌ 凭 AI 内置经验类比（"通常 work_item_type 的 key 大概就是 xxx"）
- ❌ 沉默继续 + 挑一个"看起来合理"的值

### P2.1：识别点位类型

结合以下信息综合判断：
- 用户的原始描述（添加/修改/删除）
- 飞书项目知识 MCP（查询 work_item_type 枚举值、event_type 编号等业务背景）
- `point-schema.json` 中的字段约束和枚举

**推断目标点位类型**（参考 SKILL.md 速查表），如有歧义直接询问。

### ⚠️ 点位类型是彼此独立的顶层概念（CRITICAL — 防混淆）

**每种点位类型在 config JSON 中是独立的顶层 key，绝对不能将一种点位类型作为另一种点位类型的子配置或属性。**

以下 11 种点位类型是**完全平级、互不隶属**的：

```
page | view | dashboard | config | control | button | intercept | listen_event | component | customField | liteAppComponent
```

**常见混淆（必须识别并纠正）：**

| 错误做法 | 正确做法 | 原因 |
|---------|---------|------|
| 把 `customField` 当成 `liteAppComponent` 的一个 property | `customField` 是独立点位，和 `liteAppComponent` 平级放在 config 顶层 | customField 有自己独立的 subfield、platform、table_layout，不是 liteAppComponent 的属性 |
| 把 `control` 当成 `customField` 的一部分 | `control` 和 `customField` 各自独立 | 两者虽然都在工作项表单中展示，但配置结构完全不同 |
| 用户说"添加拓展字段"却生成 liteAppComponent 的 properties | 应新增 `customField` 类型的点位 | "拓展字段"= customField 点位，不是 liteAppComponent 属性 |
| 用户说"添加拓展字段**组件**"却映射到 liteAppComponent | 应新增 `customField` 类型的点位 | 用户口语中的"组件"只是泛称，关键词是"拓展字段"→ customField |
| 用户说"添加控件"却塞进现有点位的配置里 | 应新增 `control` 类型的点位 | "控件"= control 点位，是独立的顶层配置 |

**术语→点位映射（CRITICAL — "组件"一词极易误导）：**

> schema 中 `liteAppComponent` 叫"轻应用组件"，`component` 叫"组件位"，但用户日常口语中"组件"可能指任何东西。
> **不要仅凭"组件"二字就映射到 liteAppComponent 或 component，必须看完整上下文中的核心名词。**

| 用户可能的说法 | 核心名词 | 正确点位类型 | 易错映射 |
|--------------|---------|------------|---------|
| "拓展字段"/"拓展字段组件"/"字段模板"/"自定义字段" | **拓展字段** | `customField` | ~~liteAppComponent~~ |
| "控件"/"控件组件" | **控件** | `control` | ~~liteAppComponent~~ |
| "轻应用组件"/"概览组件"/"构建器组件"/"拖拽组件" | **轻应用/概览/构建器** | `liteAppComponent` | — |
| "排期组件"/"节点排期" | **排期** | `component` | ~~liteAppComponent~~ |

**识别规则：先提取核心名词（拓展字段 / 控件 / 轻应用 / 排期），再映射点位类型。"组件"是修饰语，不是判断依据。**

**正确的 config 结构示例（多点位类型共存）：**
```json
{
  "component": [{ "key": "comp_xxx", ... }],
  "customField": [{ "key": "field_template_xxx", ... }],
  "control": [{ "key": "control_xxx", ... }],
  "liteAppComponent": [{ "key": "builder_comp_xxx", ... }]
}
```
每种类型各自一个顶层数组，互不嵌套。

## P3：交互补全缺失字段

**规则：一次只问一个问题**，语义化表达（例：不说"key 是什么"，而说"这个点位的唯一标识符打算用什么？建议格式如 my_board_point"）。

### 禁止编造 URL（CRITICAL）

**绝对禁止编造任何 URL、icon_url、token 值。完整字段清单和判定规则以 `meego-shared/SKILL.md` 的"禁止编造 URL"章节为准**（Runtime URL vs Metadata URL 的分类表），本节只补充 meego-point-config 在 P3 阶段的具体执行动作。

### Runtime URL 的统一询问流程（CRITICAL）

对本次计划生成的每个点位，按下表**从点位类型 + DSL 内容联合判断**要询问哪些 URL：

| 触发条件 | 要询问的 URL |
|---------|-------------|
| 点位类型是 `intercept` / `listen_event` | 顶层 `url`（始终问） |
| 点位类型是 `control` 且用户提到"新建页可见" / Webhook 提交 | `control.url`（仅此场景问） |
| 点位类型是 `control` 且 `table_cell.definitions.data` 非空 | `platform.web.table_url.url`（见下方专门规则） |
| 点位类型是 `customField` 且 `table_layout.definitions.data` 非空 | `platform.web.table_data_url`（同上） |
| 任意 DSL（table_cell / table_layout）里有节点用了 `onClick/onDoubleClick.action=httpRequest` | 对应 `params.url`（按用户业务语义问） |
| 任意 DSL 里有节点用了 `onClick.action=openLink` 且 URL 是字面量（非 `{{varName}}` 引用） | 对应 `params.url` |

> **token 字段不询问**：所有 `.token`（`intercept.token` / `listen_event.token` / `control.token` / `table_url.token` / `table_data_token`）由 CLI 在对应 URL 填写后自动生成 36 位 UUID，**AI 不要写 token，也不要问用户**。

**扫描方法**：AI **必须在生成 JSON 之前**先扫一遍 template 树，找出所有上述触发点，把 URL 询问合并成一轮对话问给用户（不要每个点位单独问一次）。

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

### URL 溯源协议（CRITICAL — 写入时强制，前置预防 ＞ 事后检测）

模式 lint（A0.5）只能逮"长得像假的"URL，**逮不到"长得像真的假"**（例：AI 拼一个 `https://meego-internal.byted.net/api/cb`，所有 lint 规则都过得去，但仍是编造）。
真正的防线是**写入时禁止 AI 自创 URL 值**。

#### 三条硬约束

1. **零自创原则**
   每个 URL 字段的值，必须能回答"这个字符串从哪来？"，且**只接受**以下两类来源：
   - (a) **用户显式输入** —— 用户在本对话中明确给出（粘贴的链接、回复"用 https://x.y.com" 等）
   - (b) **用户授权占位** —— 用户明确说"先占位发布前补"，写入 `<PLACEHOLDER: ...>` 形式
   - 任何**第三种情况都禁止**写入字符串，包括但不限于：
     * 编造看起来"像内网"的域名（`*.byted.net`、`*.bytedance.com` 等）
     * 复制 MCP 文档示例里的 URL（样例 ≠ 用户的接口）
     * 拼合用户提到过的部分片段（如用户提了"meego"，AI 自己拼成 `https://meego-prod.example.com`）
     * 套用其他点位的 URL（除非用户明说"和上面一样"）

2. **检测到无源即停**
   AI 在生成 JSON 前扫描所有 URL 字段，若任意字段**没有 (a) 或 (b) 来源**，**MUST 暂停**走 P3 询问流程问清楚，**禁止**先写一个看似合理的值"待 lint 处理"。

3. **输出标注溯源（机器可审计）**
   每个 URL 字段在 P6 输出摘要中必须**显式标注 source**，便于 reviewer agent 和用户审计：
   ```
   table_url.url: "https://prod.api.example.com/cb"   # source: 用户提供 @ 2026-04-14 14:32
   intercept.url: "<PLACEHOLDER: 拦截回调地址>"        # source: 用户授权占位 @ 2026-04-14 14:35
   ```
   未标注 source 的 URL 视为违反协议，reviewer 将判定为疑似编造。

#### 与下游防御的责任划分

| 层 | 防什么 | 局限 |
|---|---|---|
| **plan 协议（本节）**：写入时禁自创 | AI 不写不该写的（前置） | 依赖 AI 守纪律 |
| **apply A0.5 模式 lint** | 已知假特征命中即硬阻止（兜底，不允许占位） | 看不到"像真的假" |
| **reviewer agent 溯源审计** | URL 值 vs 对话历史比对 | 需要独立 subagent |
| **用户终审** | 涉及 url 的不可逆操作 | 最后一道闸 |

四层不重叠、不互相覆盖。本节是**最强一层**——堵住源头比事后检测更有效。

#### AI 的认知锚点

> "我没有用户的 ground truth，我永远不'善意默认'。
> 没有用户输入就停下来问，**不存在'先填一个看起来合理的，跑不通再说'的中间状态**。"

### 变量声明触发数据接口询问的细化规则（CRITICAL — 防调试时空白列）

**覆盖范围**：任何**共享 `PlatformWebForControl.table_cell.dslSchema`** 的 DSL（目前是 `control.table_cell` 和 `customField.table_layout`，未来若新增使用同一 dslSchema 的点位类型**自动覆盖**，不要把规则硬编码成只认 control）。

**判定算法**：
1. 取到该点位的 DSL 对象（归一化前后都行）
2. 判断以下任一条件为真即为"有数据变量"：
   - DSL 是完整对象形态 `{definitions, template}` 且 `definitions.data` 非空
   - DSL 是裸 template 形态但 template（递归地）里出现非 `$container` / `$i18n` / `$colorTokens` / `$fieldValue` 前缀的 `{{x}}` 引用 —— **这是非法状态**，应先引导用户补 definitions 再问 URL
   - DSL 是 JSON 字符串形态 → 解析一次后重复上述判断
3. 判定"有数据变量"的点位，配套的数据接口 URL 字段：
   - control → `platform.web.table_url.url`
   - customField → `platform.web.table_data_url`
4. 按"Runtime URL 统一询问流程"的话术询问

**为什么必须问**：
- 数据接口是**表格列每次渲染都会打**的，不是一次性事件
- template 里的 `{{varName}}` 值由 `work_item_datas[].work_item_data.{varName}` 填充
- URL 不可达或返回格式错 → 变量 undefined → 整列空白/报错；用户最常踩的调试坑
- 只过 schema 的假 URL（如 `example.com`）能 push 成功，但跑起来每一行都废

### 各类型的行为指引（schema 抓不到，必须 AI 主动遵守）

> 字段必填 / 长度 / 格式 / 枚举合法性由 `local-config set` 的 schema 校验兜底——不在此处重复。下表只列 **schema 过了但语义上仍会跑废** 的点，AI 写配置前必须按此执行。

| 类型 | 行为指引 |
|------|---------|
| control | URL 询问（`control.url` / `table_url.url` / 节点内 `httpRequest.params.url`）走"Runtime URL 统一询问流程"；`table_cell.definitions.data` 非空即视为"有数据变量"，触发数据接口 URL 必填联动 |
| intercept | `url` 必须向用户询问（schema 抓不到编造）；`event_config.work_item_type` 与 `event_type` 的数字编号 AI 易编错，必须来源于用户或 schema 枚举，禁止凭记忆填 |
| listen_event | `url` 必须向用户询问；`event_config` 同 intercept |
| component | `component_type` 只能从 schema 枚举取，禁止脑补"看起来像"的值（目前仅支持轻应用） |
| customField | URL 询问（`table_data_url` / 节点内 `httpRequest.params.url`）同 control；`subfield.name` 非必填时留空由 CLI 从 `i18n_info.zh.name` 取值，AI 不要自作主张另填 |
| liteAppComponent | **MUST 先调飞书项目知识 MCP**（关键词 `轻应用组件 LiteAppComponent 属性 getProps notify watch 输出属性`）拉完整指南再填 properties/outputs——类型选型、数据源（spaceId → 工作项类型 → 字段）、排期不支持等限制只在 MCP 里有、schema 抓不到；【过渡期 / 模拟 RAG 用，MCP 正式上线后删除此段】MCP 未命中则 fallback 读 `log/builder-comp-rag-doc.md`；发布后 property 类型不可改，定义阶段要一次问清 |

> 其他类型（page / view / dashboard / config / button）的必填字段全部可由 schema 校验抓住，AI 按 schema 报错提示修即可，plan 阶段不需要特殊行为指引。

### 对于删除操作

1. 从 `plugin.temp.local-remote.json` 中列出匹配的候选点位
2. 展示候选列表，让用户确认要删除哪个
3. **二次确认删除**，展示点位详情后请求确认

### 对于修改操作

1. 从 `plugin.temp.local-remote.json` 找到目标点位
2. 仅询问需要修改的字段
3. 其余字段保持原值

## P4：生成完整配置文件

**⚠️ CRITICAL：set 是全量替换，必须基于 `get --remote` 拉取的远端完整数据操作。遗漏任何现有点位都会导致被永久删除。**

基于 `plugin.temp.local-remote.json` 的全量配置：

### 添加操作

在对应类型数组中 append 新点位对象，**保留所有其他类型和其他点位不变**。

```
远端: { "page": [A], "button": [B1, B2] }
用户: 新增一个 liteAppComponent
结果: { "page": [A], "button": [B1, B2], "liteAppComponent": [新点位] }
```

对于 Single 类型（page/view/dashboard/config/intercept/listen_event/component），如果远端已有该类型的点位，**不能再添加**（maxItems=1）。应提示用户是要替换还是修改现有点位。

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

**⚠️ 如果只传 `{ "liteAppComponent": [新点位] }` 而不包含 page 和 button，远端的 page 和 button 将被清除！**

**删除确认规则：** 生成的完整 JSON 相比远端减少了任何点位时，**必须先向用户列出即将被删除的点位清单（key + name），获得明确确认后才能执行 set。** 禁止静默删除。

生成文件名：`plugin.temp.local-{YYYY-MM-DD_HHmmss}.json`

### 生成前自检

- [ ] 是否为**全量配置**（包含远端所有现有类型+本次变更，非仅变更部分）
- [ ] key 在同类型中唯一（不与现有点位冲突）
- [ ] Single 类型未超过 maxItems=1
- [ ] 必填字段均已填写
- [ ] 枚举值合法（mode、mobile_block_style、field_type、view.icon）
- [ ] name 长度在限制内
- [ ] **platform.web 字段以 schema 中对应 `PlatformWebFor{Type}` 定义为准，禁止跨点位类型混用**（如 `table_url` 属于 customField，不能写在 control 里；`mode`/`init_size` 属于 button，不能写在其他类型里）

## P5：清理临时文件（强制）

> **MUST — 此步骤不可跳过。** `plugin.temp.local-remote.json` 仅在 plan 阶段使用，生成配置文件后立即删除。
> `point-schema.json` 在后续 polish 阶段仍需参考，此处保留，发布完成后由 plugin-publish 统一清理。

```bash
rm -f plugin.temp.local-remote.json
```

## P6：输出

plan 阶段完成后输出：
- 生成的配置文件路径：`plugin.temp.local-{timestamp}.json`
- 变更摘要：添加/修改/删除了哪些点位（类型 + key）
