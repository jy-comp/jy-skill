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

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则（含禁止修改 `.lpm/` 目录、删除点位须确认、禁止编造 URL）、工具职责划分等公共约定。**
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

> **每种点位类型是彼此独立的顶层概念，不能将一种点位当作另一种点位的属性或子配置。**
> 例如：`customField`（拓展字段）是独立点位，不是 `liteAppComponent` 或 `component` 的属性；`control`（控件）也是独立点位，不是 `customField` 的一部分。
> config JSON 中每种类型各自一个顶层数组，互不嵌套。

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
| `point-schema.json` | plan 阶段生成（stdout 就是 JSON，不是 YAML），polish 阶段仍需参考，**发布完成后由 plugin-publish 删除** |
| `plugin.temp.local-remote.json` | 当前远端配置（local 格式），**plan 完成后必须删除** |
| `plugin.temp.local-{timestamp}.json` | 本次生成的配置，**apply 完成后必须删除** |

## 全量提交约束（CRITICAL — 防数据丢失）

`local-config set` 和 `update --source-type=local` 均为**全量替换**——提交什么就存什么，远端不做合并。**遗漏任何现有点位都会导致该点位被永久删除。**

**必须遵循的操作流程：**
1. 先 `local-config get --remote` 获取远端**完整配置**作为基础
2. 在完整配置基础上做**局部增删改**
3. 将修改后的**完整配置**提交（包含所有点位类型、所有点位实例）
4. 推送成功后执行 `npx @byted-meego/cli@builder update` 将远端配置同步回本地

> **删除点位须用户确认**：规则定义在 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md) 的"删除点位确认"条款，适用于所有阶段和调用路径。本 skill 在 apply 阶段通过 [`references/apply.md`](references/apply.md) 的 A0 步骤强制执行 diff 检查——即使 plan 阶段已确认过，也须再次检查（文件可能被手改）。

**严禁以下行为：**
- 只传变更部分（会导致未传的点位被删除）
- 直接修改 `plugin.config.json`（始终通过 CLI 命令同步）

### 增/改/删操作模式

假设远端当前配置为：
```json
{ "page": [{ "key": "board_abc123", ... }], "button": [{ "key": "button_x1", ... }, { "key": "button_x2", ... }] }
```

| 操作 | set 的 JSON 内容 | 说明 |
|------|-----------------|------|
| **新增** liteAppComponent | `{ "page": [原样保留], "button": [原样保留], "liteAppComponent": [新点位] }` | 必须带上 page 和 button，否则它们会被删除 |
| **修改** button_x1 的 name | `{ "page": [原样保留], "button": [{ "key": "button_x1", name已改, ... }, { "key": "button_x2", 原样 }] }` | 只改目标字段，其余字段和其他点位原样保留 |
| **删除** button_x2 | `{ "page": [原样保留], "button": [{ "key": "button_x1", 原样 }] }` | 从数组中移除目标项，其余原样保留 |
| **删除整个** button 类型 | `{ "page": [原样保留] }` | 不传 button 字段，远端 button 即被清除 |
