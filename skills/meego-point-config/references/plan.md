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

## P3：交互补全缺失字段

**规则：一次只问一个问题**，语义化表达（例：不说"key 是什么"，而说"这个点位的唯一标识符打算用什么？建议格式如 my_board_point"）。

### 各类型必须确认的字段

| 类型 | 必须确认（若未提供） |
|------|-------------------|
| board | key、name（≤15字符）、icon（JSON格式） |
| view | key、name（≤15字符）、icon（从预定义枚举选）、work_item_type |
| dashboard | key、name（≤15字符） |
| config | key |
| control | key、name（≤100字符）、work_item_type |
| button | key、name（≤15字符）、work_item_type、platform.web.mode（ui/script） |
| intercept | key、name（≤15字符）、url、token、event_config（至少1条，需确认 work_item_type 和 event_type 数字编号） |
| listen_event | url、token、event_config（同上） |
| component | key、component_type（以 schema 枚举为准，目前支持轻应用） |
| field_template | key、i18n_info.name（≤50字符）、i18n_info.description、subfield（至少1条：name/field_type枚举/field_key）、platform.web.resource、platform.mobile.resource、platform.mobile.mobile_block_style |
| builder_comp | key、icon_url、i18n_info（多语言）、properties（至少1条）、platform.web.resource、platform.web.layout.mode（0/1） |

### 对于删除操作

1. 从 `plugin.temp.local-remote.json` 中列出匹配的候选点位
2. 展示候选列表，让用户确认要删除哪个
3. **二次确认删除**，展示点位详情后请求确认

### 对于修改操作

1. 从 `plugin.temp.local-remote.json` 找到目标点位
2. 仅询问需要修改的字段
3. 其余字段保持原值

## P4：生成完整配置文件

基于 `plugin.temp.local-remote.json` 的全量配置：
- **添加**：在对应类型数组中 append 新点位对象
- **修改**：找到匹配 key 的点位，合并更新字段
- **删除**：从对应类型数组中移除匹配 key 的点位

生成文件名：`plugin.temp.local-{YYYY-MM-DD_HHmmss}.json`

### 生成前自检

- [ ] key 在同类型中唯一（不与现有点位冲突）
- [ ] 必填字段均已填写
- [ ] 枚举值合法（mode、mobile_block_style、field_type、view.icon）
- [ ] name 长度在限制内
- [ ] 是否为全量配置（非仅变更部分）

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
