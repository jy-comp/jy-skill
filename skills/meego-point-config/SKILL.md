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

> **前置**：先 Read [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)

> 获取共享规则；进入每个 mode 前 Read 对应的 `references/<mode>.md`。

> **每个 mode 的执行流程、字段约束、URL 询问规则、schema 分片命令都只在 `references/<mode>.md` 里**——不读 reference 就没有 jq 切片命令、URL 触发表、P3 交互补全顺序，生成出来的配置会缺字段或被 `local-config set` 打回。凭记忆跳过 plan 是这个 skill 最常见的失败模式，进 mode 的第一个动作固定是 Read 对应 reference。

## 核心流程

```
mode=setup   → 环境检查 + 前置验证
mode=plan    → 理解需求 + 交互补全 + 生成配置文件
mode=apply   → local-config set 校验 + update 推送远端
mode=verify  → local-config get --remote 验证远端数据
mode=pipeline（默认）→ setup → plan → apply → verify
```

## 使用方式

| 参数 | 说明 |
|------|------|
| `mode=setup` | 环境检查 |
| `mode=plan` | 理解需求、匹配平台能力(主要需要匹配业务平台所提供的能力是否满足用户需求)、交互补全、生成配置文件 |
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

所有中间产物落在 `.lpm-cache/` 工作区，由 CLI 在命令成功时自动清理，AI 只读写、不负责 `rm`：

| 文件 | 写入者 | 消费者 | CLI 自动清理时机 |
|------|-------|-------|------------|
| `.lpm-cache/schema/point-schema.json` | `lpm schema` | plan 的 jq 切片；polish / code-gen 按需 Read（但在 code-gen 阶段已经被删，见下一行） | `publish` 成功 |
| `.lpm-cache/config/remote.json` | `lpm local-config get [--remote]` | plan 生成 draft 时的全量基线；apply A0 的 diff 检查 | `local-config set` 成功 |
| `.lpm-cache/config/draft-{timestamp}.json` | plan 阶段 AI 写 | `lpm local-config set --from` 读 | `local-config set` 成功 |
| `.lpm-cache/mcp/<slug>.md` | AI 在 MCP 命中后 Write | 后续同 slug 查询复用 | `publish` 成功 |

**下游消费者注意**：`set` 成功后 `remote.json` 即消失，Phase 3 及之后读点位配置请改读工程根目录的 **`point.config.local.json`**（CLI 在 `set` 成功时和 `update` 拉取时都会刷新这份文件，内容是同一份 local-format 点位配置）。

workflow 断点文件 `.lpm-cache/state.json` 由 plugin-workflow 维护，CLI 的 `publish` 也会保留它。若需整体清空（含 state），用 `npx @byted-meego/cli@builder workspace clean --include-state`。

## 全量提交约束（CRITICAL — 防数据丢失）

`local-config set` 和 `update --source-type=local` 均为**全量替换**——提交什么就存什么，远端不做合并。**遗漏任何现有点位都会导致该点位被永久删除。**

操作流程：
1. 先 `local-config get --remote` 获取远端完整配置作为基础
2. 在完整配置基础上做局部增删改
3. 将修改后的**完整配置**提交（包含所有点位类型、所有点位实例）
4. 推送成功后执行 `npx @byted-meego/cli@builder update` 将远端配置同步回本地

不要只传变更部分（会导致未传的点位被删除），也不要直接改 `plugin.config.json`（始终通过 CLI 命令同步）。

> **删除点位须用户确认**：规则定义在 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md) 的"删除点位确认"条款，适用于所有阶段和调用路径。本 skill 在 apply 阶段通过 [`references/apply.md`](references/apply.md) 的 A0 步骤强制执行 diff 检查——即使 plan 阶段已确认过，也须再次检查（文件可能被手改）。

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
