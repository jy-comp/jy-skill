---
name: meego-point-config
version: 1.0.0
description: |
  Meego 插件点位配置管理（编排 skill）：添加/修改/删除点位配置。
  当用户在插件工程中说"加个看板点位"、"改一下拦截点位"、"删除某个点位"、"配置点位"时触发，或由 plugin-workflow 内部调用。
  若用户是从零开始说"我需要一个 xxx 功能"，应由 plugin-workflow 统一编排。
  前提：当前目录必须存在 plugin.config.json 且已安装 @byted-meego/cli@builder。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder local-config --help"
---

# Meego 点位配置管理 Skill

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则等公共约定。**
**CRITICAL — 进入每个 mode 前，务必先用 Read 工具读取对应的 references 文档，禁止直接盲目执行。**
**CRITICAL — 禁止修改 `.lpm/` 目录下的任何文件，该目录由 CLI 内部管理，只能通过 CLI 命令间接操作。**

## 核心流程

```
mode=setup   → 环境检查 + 前置验证
mode=plan    → 理解需求 + 交互补全 + 生成配置文件
mode=apply   → local-config set 校验 + update 推送远端
mode=verify  → local-config get --remote 验证远端数据
mode=pipeline（默认）→ setup → plan → apply → verify
```

## 点位类型速查

| 类型 | 必填字段 |
|------|---------|
| `board` | key, icon(JSON格式), name(max 15字符) |
| `view` | key, icon(预定义枚举), name(max 15字符), work_item_type |
| `dashboard` | key, name(max 15字符) |
| `config` | key |
| `control` | key, name(max 100字符), work_item_type |
| `button` | key, name(max 15字符), work_item_type |
| `intercept` | key, name(max 15字符), url, token, event_config(min 1条) |
| `listen_event` | url, token, event_config(min 1条)；key 可选 |
| `component` | key, component_type（以 schema 为准，目前支持轻应用） |
| `field_template` | key, i18n_info, subfield(min 1条), platform |
| `builder_comp` | key, icon_url, i18n_info, properties, platform |

**关键约束（来自表单分析）：**
- `board.icon`：JSON 格式，如 `{"color":"#B449C2","type":"work_object_icon_version"}`
- `view.icon`：预定义枚举 `icon-openapp_view_chart/flow/pie/pie2/line/graph/ecg`
- `mobile_block_style`：枚举 `not_display | small | medium | big`
- `field_template.subfield.field_type`：枚举 `multi-pure-text | tree-multi-select | number | multi-user | precise_date`
- `button.platform.web.mode`：枚举 `ui | script`

## 使用方式

| 参数 | 说明 |
|------|------|
| `mode=setup` | 环境检查 |
| `mode=plan` | 理解需求、交互补全、生成配置文件 |
| `mode=apply` | 校验 + 推送远端 |
| `mode=verify` | 拉取远端验证 |
| `mode=pipeline`（默认） | 端到端全流程 |

```
/meego-point-config                    # 端到端全流程
/meego-point-config mode=plan          # 仅生成配置文件
/meego-point-config mode=apply         # 校验 + 推送远端
/meego-point-config mode=verify        # 拉取远端验证
```

## 各模式详细流程

> 详细流程读取对应 references 文件：

- `mode=setup`  → 读取 `references/setup.md`
- `mode=plan`   → 读取 `references/plan.md`
- `mode=apply`  → 读取 `references/apply.md`
- `mode=verify` → 读取 `references/verify.md`

`mode=pipeline` 依次执行 setup → plan → apply → verify，每步失败则停止。

## 文件约定

| 文件 | 说明 |
|------|------|
| `point-schema.yaml` | plan 阶段生成，polish 阶段仍需参考，**发布完成后由 plugin-publish 删除** |
| `plugin.temp.local-remote.json` | 当前远端配置（local 格式），**plan 完成后必须删除** |
| `plugin.temp.local-{timestamp}.json` | 本次生成的配置，**apply 完成后必须删除** |

## 全量提交约束（CRITICAL — 防数据丢失）

`local-config set` 和 `update --source-type=local` 均为**全量替换**——提交什么就存什么，远端不做合并。**遗漏任何现有点位都会导致该点位被永久删除。**

**必须遵循的操作流程：**
1. 先 `local-config get --remote` 获取远端**完整配置**作为基础
2. 在完整配置基础上做**局部增删改**
3. 将修改后的**完整配置**提交（包含所有点位类型、所有点位实例）
4. 推送成功后执行 `npx @byted-meego/cli@builder update` 将远端配置同步回本地

**删除操作须用户确认（CRITICAL）：**
- 当 set 的 JSON 相比远端**减少了**点位（无论是删除某个点位实例还是整个类型），**必须先向用户列出即将被删除的点位清单，获得明确确认后才能执行 set**
- 禁止静默删除——即使用户只说"修改 X"，如果生成的 JSON 意外少了其他点位，也必须中止并提醒

**严禁以下行为：**
- 只传变更部分（会导致未传的点位被删除）
- 不经确认删除任何已有点位
- 直接修改 `plugin.config.json`（始终通过 CLI 命令同步）

### 增/改/删操作模式

假设远端当前配置为：
```json
{ "board": [{ "key": "board_abc123", ... }], "button": [{ "key": "button_x1", ... }, { "key": "button_x2", ... }] }
```

| 操作 | set 的 JSON 内容 | 说明 |
|------|-----------------|------|
| **新增** builder_comp | `{ "board": [原样保留], "button": [原样保留], "builder_comp": [新点位] }` | 必须带上 board 和 button，否则它们会被删除 |
| **修改** button_x1 的 name | `{ "board": [原样保留], "button": [{ "key": "button_x1", name已改, ... }, { "key": "button_x2", 原样 }] }` | 只改目标字段，其余字段和其他点位原样保留 |
| **删除** button_x2 | `{ "board": [原样保留], "button": [{ "key": "button_x1", 原样 }] }` | 从数组中移除目标项，其余原样保留 |
| **删除整个** button 类型 | `{ "board": [原样保留] }` | 不传 button 字段，远端 button 即被清除 |
