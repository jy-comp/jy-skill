# mode=plan：理解需求 + 交互补全 + 生成配置文件

## P1：获取 Schema 和当前完整配置

```bash
npx @byted-meego/cli@builder schema > point-schema.yaml
npx @byted-meego/cli@builder local-config get > plugin.temp.local-remote.json
```

- 若 `local-config get` 返回空，以 `{}` 作为基础
- **两个命令并行执行**，不要等第一个完成再跑第二个

## P2：识别操作类型和点位类型

结合以下信息综合判断：
- 用户的原始描述（添加/修改/删除）
- 飞书项目知识 MCP（查询 work_item_type 枚举值、event_type 编号等业务背景）
- `point-schema.yaml` 中的字段约束和枚举

**推断目标点位类型**（参考 SKILL.md 速查表），如有歧义直接询问。

### ⚠️ 点位类型是彼此独立的顶层概念（CRITICAL — 防混淆）

**每种点位类型在 config JSON 中是独立的顶层 key，绝对不能将一种点位类型作为另一种点位类型的子配置或属性。**

以下 11 种点位类型是**完全平级、互不隶属**的：

```
board | view | dashboard | config | control | button | intercept | listen_event | component | field_template | builder_comp
```

**常见混淆（必须识别并纠正）：**

| 错误做法 | 正确做法 | 原因 |
|---------|---------|------|
| 把 `field_template` 当成 `builder_comp` 的一个 property | `field_template` 是独立点位，和 `builder_comp` 平级放在 config 顶层 | field_template 有自己独立的 subfield、platform、table_layout，不是 builder_comp 的属性 |
| 把 `control` 当成 `field_template` 的一部分 | `control` 和 `field_template` 各自独立 | 两者虽然都在工作项表单中展示，但配置结构完全不同 |
| 用户说"添加拓展字段"却生成 builder_comp 的 properties | 应新增 `field_template` 类型的点位 | "拓展字段"= field_template 点位，不是 builder_comp 属性 |
| 用户说"添加拓展字段**组件**"却映射到 builder_comp | 应新增 `field_template` 类型的点位 | 用户口语中的"组件"只是泛称，关键词是"拓展字段"→ field_template |
| 用户说"添加控件"却塞进现有点位的配置里 | 应新增 `control` 类型的点位 | "控件"= control 点位，是独立的顶层配置 |

**术语→点位映射（CRITICAL — "组件"一词极易误导）：**

> schema 中 `builder_comp` 叫"轻应用组件"，`component` 叫"组件位"，但用户日常口语中"组件"可能指任何东西。
> **不要仅凭"组件"二字就映射到 builder_comp 或 component，必须看完整上下文中的核心名词。**

| 用户可能的说法 | 核心名词 | 正确点位类型 | 易错映射 |
|--------------|---------|------------|---------|
| "拓展字段"/"拓展字段组件"/"字段模板"/"自定义字段" | **拓展字段** | `field_template` | ~~builder_comp~~ |
| "控件"/"控件组件" | **控件** | `control` | ~~builder_comp~~ |
| "轻应用组件"/"概览组件"/"构建器组件"/"拖拽组件" | **轻应用/概览/构建器** | `builder_comp` | — |
| "排期组件"/"节点排期" | **排期** | `component` | ~~builder_comp~~ |

**识别规则：先提取核心名词（拓展字段 / 控件 / 轻应用 / 排期），再映射点位类型。"组件"是修饰语，不是判断依据。**

**正确的 config 结构示例（多点位类型共存）：**
```json
{
  "component": [{ "key": "comp_xxx", ... }],
  "field_template": [{ "key": "field_template_xxx", ... }],
  "control": [{ "key": "control_xxx", ... }],
  "builder_comp": [{ "key": "builder_comp_xxx", ... }]
}
```
每种类型各自一个顶层数组，互不嵌套。

## P3：交互补全缺失字段

**规则：一次只问一个问题**，语义化表达（例：不说"key 是什么"，而说"这个点位的唯一标识符打算用什么？建议格式如 my_board_point"）。

### 禁止编造 URL（CRITICAL）

**绝对禁止编造任何 URL、icon_url、token 值。** 编造的 mock URL（如 `https://example.com/webhook`、`https://your-server.com/callback`）在实际使用中完全不可用，会导致插件功能失效。

| 字段 | 所属点位 | 处理方式 |
|------|---------|---------|
| `url` | intercept, listen_event | **必须向用户询问**（业务回调地址，无法猜测） |
| `token` | intercept, listen_event | **必须向用户询问**（安全校验密钥） |
| `icon_url` | builder_comp | **不阻塞**，不填则 CLI 自动填充默认图标 |

**遇到 url/token 字段时**：暂停自动填充流程，明确向用户提问，例如：
> "intercept 点位需要一个回调 URL（服务端接收事件推送的地址）和验证 token，请提供。如果暂时没有，我可以先用占位符标记，发布前需要替换。"

如果用户表示暂时没有，使用**不可能被误认为真实地址的占位符**：`"<PLACEHOLDER: 请替换为你的回调地址>"` 并在输出摘要中**醒目标注待替换项**。

> `icon_url` 和插件图标不需要询问用户，不填即可，CLI/后台会自动使用默认图标。

### 各类型必须确认的字段

| 类型 | 必须确认（若未提供） |
|------|-------------------|
| board | key、name（≤15字符）、icon（JSON格式） |
| view | key、name（≤15字符）、icon（从预定义枚举选）、work_item_type |
| dashboard | key、name（≤15字符） |
| config | key |
| control | key、name（≤100字符）、work_item_type |
| button | key、name（≤15字符）、work_item_type、platform.web.mode（ui/script） |
| intercept | key、name（≤15字符）、**url（必须向用户询问）**、**token（必须向用户询问）**、event_config（至少1条，需确认 work_item_type 和 event_type 数字编号） |
| listen_event | **url（必须向用户询问）**、**token（必须向用户询问）**、event_config（同上） |
| component | key、component_type（以 schema 枚举为准，目前支持轻应用） |
| field_template | key、i18n_info.name（≤50字符）、i18n_info.description、subfield（至少1条：i18n_info必填如`{zh:{name:"人员",desc:""}}`、name非必填为空时取i18n_info中文name、field_type枚举、field_key）、platform.web.resource、platform.mobile.resource、platform.mobile.mobile_block_style |
| builder_comp | key、i18n_info（多语言）、properties（至少1条）、platform.web.resource、platform.web.layout.mode（0/1）；icon_url 可不填（CLI 自动填充默认图标） |

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
远端: { "board": [A], "button": [B1, B2] }
用户: 新增一个 builder_comp
结果: { "board": [A], "button": [B1, B2], "builder_comp": [新点位] }
```

对于 Single 类型（board/view/dashboard/config/intercept/listen_event/component），如果远端已有该类型的点位，**不能再添加**（maxItems=1）。应提示用户是要替换还是修改现有点位。

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
远端: { "board": [A], "button": [B1, B2] }
用户: 删除 B2
结果: { "board": [A], "button": [B1] }

用户: 删除整个 button 类型的所有点位
结果: { "board": [A] }
```

**⚠️ 如果只传 `{ "builder_comp": [新点位] }` 而不包含 board 和 button，远端的 board 和 button 将被清除！**

**删除确认规则：** 生成的完整 JSON 相比远端减少了任何点位时，**必须先向用户列出即将被删除的点位清单（key + name），获得明确确认后才能执行 set。** 禁止静默删除。

生成文件名：`plugin.temp.local-{YYYY-MM-DD_HHmmss}.json`

### 生成前自检

- [ ] 是否为**全量配置**（包含远端所有现有类型+本次变更，非仅变更部分）
- [ ] key 在同类型中唯一（不与现有点位冲突）
- [ ] Single 类型未超过 maxItems=1
- [ ] 必填字段均已填写
- [ ] 枚举值合法（mode、mobile_block_style、field_type、view.icon）
- [ ] name 长度在限制内
- [ ] **platform.web 字段以 schema 中对应 `PlatformWebFor{Type}` 定义为准，禁止跨点位类型混用**（如 `table_url` 属于 field_template，不能写在 control 里；`mode`/`init_size` 属于 button，不能写在其他类型里）

## P5：清理临时文件（强制）

> **MUST — 此步骤不可跳过。** `plugin.temp.local-remote.json` 仅在 plan 阶段使用，生成配置文件后立即删除。
> `point-schema.yaml` 在后续 polish 阶段仍需参考，此处保留，发布完成后由 plugin-publish 统一清理。

```bash
rm -f plugin.temp.local-remote.json
```

## P6：输出

plan 阶段完成后输出：
- 生成的配置文件路径：`plugin.temp.local-{timestamp}.json`
- 变更摘要：添加/修改/删除了哪些点位（类型 + key）
