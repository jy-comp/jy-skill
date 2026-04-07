---
name: meego-point-config
version: 1.0.0
description: |
  Meego 插件点位配置管理（内部 skill）：添加/修改/删除点位配置。
  当用户在已有插件工程中明确说"加个看板点位"、"改一下拦截点位"、"删除某个点位"、"配置点位"时触发，或由 plugin-workflow 内部调用。
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
| `point-schema.yaml` | plan 阶段生成，**verify 后必须删除（强制）** |
| `point.config.local-remote.json` | 当前远端配置（local 格式），plan 阶段生成，**verify 后必须删除（强制）** |
| `point.config.local-{timestamp}.json` | 本次生成的配置，**apply 后必须删除（强制）** |

> **临时文件清理是强制步骤，不可跳过。** 每个临时文件在其最后消费步骤完成后必须立即删除。

## 全量提交约束

`local-config set` 和 `update --source-type=local` 均为**全量操作**。必须：
1. 先 `local-config get` 获取完整配置
2. 在完整配置基础上做局部增删改
3. 将修改后的完整配置提交
4. 推送成功后执行 `npx @byted-meego/cli@builder update` 将远端配置同步回本地

禁止只传变更部分。**禁止直接修改 `plugin.config.json`**，始终通过 CLI 命令同步。
