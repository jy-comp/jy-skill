---
name: meegle-plugin-workflow
version: 2.0.0
description: |
  Meegle 插件从零创建的完整编排（新插件全流程）：创建工程 → 配点位 + 生成代码 → 本地调试确认 → 完善信息 → 发布上线 → 输出分享链接。
  当用户说"从零做一个 xxx 插件"、"新建一个插件"、"我需要一个 xxx 插件"、"帮我做个 xxx 插件"、"飞书项目插件开发"、"Meegle 插件开发"等**整插件级别**的意图时触发。
  **仅适用于新插件场景**：入口守卫会检查当前目录是否存在 `plugin.config.json`——已存在则说明是已有插件工程，应引导用户切换到 meegle-plugin-feature（功能迭代）。
  存量插件加功能 → meegle-plugin-feature；只改插件元信息 → meegle-plugin-polish；只发布 → meegle-plugin-publish。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# meegle-plugin-workflow Skill（新插件全流程）

> **前置**：先 Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 获取共享规则；进入每个 Phase 前 Read 对应的 `references/phase-N-*.md`。

## 本 skill 的最少 Read 清单

- 共享规则 → Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md)（所有 skill 通用前置，含三条根原则）
- Phase 0 上下文采集 → Read [`references/phase-0-context.md`](references/phase-0-context.md)
- Phase 1 搭建工程 → Read [`references/phase-1-scaffold.md`](references/phase-1-scaffold.md)
- Phase 2 功能实现 → Read [`references/phase-2-feature.md`](references/phase-2-feature.md)
- Phase 3 发布上线 → Read [`references/phase-3-release.md`](references/phase-3-release.md)
- 不要预加载所有 phase reference；按当前进入的 Phase 按需 Read

## 入口守卫（MUST — 执行前必检）

```bash
test -f ./plugin.config.json && echo "EXIST" || echo "NOT_EXIST"
```

| 检查结果 | 处理 |
|---------|------|
| `NOT_EXIST` | ✅ 继续新插件全流程（Phase 0 → 3） |
| `EXIST` | ⚠️ 当前已是插件工程；告知用户"检测到当前目录已有插件工程（`plugin.config.json` 存在）。新插件全流程会与既有工程冲突。如果要在此插件上加功能请走 `/meegle-plugin-feature`；如果要重新创建请先切到空目录或确认删除后重试"，等用户决策，不得继续 |

守卫设计理由：本 skill 的 Phase 1 会执行 `create` 创建工程，在已有 `plugin.config.json` 的目录跑会破坏既有状态。无 guard 时 AI 路由若把"加功能"意图错误命中这里，会导致用户已有工作丢失。

## 核心理念

**用户不需要知道"插件开发"的分步概念。** 用户说"我要做一个在看板上显示燃尽图的插件"，AI 识别这是新插件意图后，**自动跑完从创建到发布的全部流程**，把用户推到"拿到分享链接 / 可以给别人用"的完成态，不在中途停下让用户分步操作。

用户全程只在三个不可逆决策点做确认：
1. **Phase 2**：AI 推荐的点位方案是否符合预期（基于 schema 分析 + 用户原始需求）
2. **Phase 2 尾**：本地预览调试后，功能是否符合要求（不 OK 会回到 Phase 2 调点位或改代码）
3. **Phase 3**：发布前二次确认（`publish` 不可逆）

## 完整流程（Phase 0 → 3，线性执行）

```
入口守卫（无 plugin.config.json）
   ↓
Phase 0 — 上下文采集
   钉死 siteDomain（CLI 硬前置）+ 逐字录音用户原话到 checkpoint
   ↓
Phase 1 — 搭建工程
   meegle-plugin-env-setup（按需）→ meegle-plugin-create
   产出：plugin.config.json + 可运行骨架
   ↓
Phase 2 — 功能实现（绑定两步 + 本地调试确认）
   meegle-plugin-feature（Stage Config 点位配置 → Stage Code 代码生成）
   → 本地调试：`npx @byted-meego/cli@builder start --auto`
   → 用户确认功能 OK（不 OK 则回 Phase 2 迭代）
   ↓
Phase 3 — 发布上线
   meegle-plugin-polish（AI 总结生成名称/描述/分类）
   → 向用户展示发布清单 → 等用户显式确认（不可逆护栏）
   → meegle-plugin-publish（update → release → publish）
   → 输出分享链接
```

**Phase 0 只录音，Phase 2 才理解**：`context.originalRequirement` 在 Phase 0 逐字记录到 checkpoint，Phase 2 的 config.plan / code.plan 从 checkpoint 读原话。编排层不对原话做任何解析、不推导默认值；各 Phase 的校验在自己真正需要的数据源（schema / point-type doc / MCP / types）到位时才做。

**Phase 2 内部委托**：config + code 的细节全在 meegle-plugin-feature 里，workflow 只负责触发和 checkpoint；meegle-plugin-feature 内部自成闭环（Stage Config → Stage Code），执行失败时回报给 workflow。

> **命名空间约定**：workflow 用 `Phase 0/1/2/3`，feature 用 `Stage Config / Stage Code`。两套命名故意不重叠，便于 checkpoint `step` 字段拼接（如 `2.config.apply` 表示 workflow Phase 2 委托给 feature 的 config.apply 子步骤）。

**Phase 3 发布护栏**：publish 是不可逆外部动作——即使在新插件全流程里，进入 publish 前也 MUST 向用户展示"即将发布到 Meegle 市场，发布后所有人可见"的确认提示，等用户显式说"确认发布"才执行。

## 各阶段详细流程

- Phase 0 → 读取 `references/phase-0-context.md`
- Phase 1 → 读取 `references/phase-1-scaffold.md`
- Phase 2 → 读取 `references/phase-2-feature.md`
- Phase 3 → 读取 `references/phase-3-release.md`

## 使用方式

```
/meegle-plugin-workflow                 # 新插件端到端全流程（推荐，默认跑到发布）
/meegle-plugin-workflow phase=1         # 仅 Phase 1（搭建工程）
/meegle-plugin-workflow phase=2         # 仅 Phase 2（功能实现 + 本地调试）
/meegle-plugin-workflow phase=3         # 仅 Phase 3（发布上线）
```

`phase=N` 选项主要用于断点续跑；默认不带参数走完整流程。

## 进度追踪（Checkpoint 机制）

每次执行 CLI 命令或关键步骤前后，**MUST** 更新项目根目录的 `.lpm-cache/state.json`，使中断后能精确恢复。

### 文件格式

```json
{
  "phase": 2,
  "step": "2.config.apply",
  "stepName": "meegle-plugin-feature / Stage Config / config.apply",
  "lastCommand": "npx @byted-meego/cli@builder local-config set ...",
  "lastCommandStatus": "success",
  "nextCommand": "npx @byted-meego/cli@builder update",
  "nextStep": "2.config.apply 推送远端后拉回",
  "timestamp": "2026-04-16T14:30:00Z",
  "context": {
    "pluginId": "MII_xxx",
    "siteDomain": "https://meego.feishu-boe.cn",
    "originalRequirement": "（Phase 0 录音的用户原话）"
  }
}
```

### 写入规则

| 时机 | 动作 |
|------|------|
| 执行 CLI 命令**前** | 写入 `nextCommand` + `nextStep`，`lastCommandStatus` 设为 `"running"` |
| 执行 CLI 命令**后** | 写入 `lastCommand` = 刚执行的命令，`lastCommandStatus` = `"success"` / `"failed"` |
| 进入新 Phase/Step | 更新 `phase` + `step` + `stepName` |
| Phase 3 发布成功 | 删除 `.lpm-cache/state.json`（流程已完成） |

### 恢复规则

每次 workflow 启动时：
1. 检查 `.lpm-cache/state.json` 是否存在
2. 存在时向用户展示恢复摘要：
   ```
   📍 上次执行到 Phase {phase} — {stepName}
      最后完成：`{lastCommand}`（{lastCommandStatus}）
      下一步：{nextStep} → `{nextCommand}`
   继续吗？
   ```
3. 用户确认后从断点继续；`lastCommandStatus` 为 `"failed"` 时提示是否重试

### 注意

- 此文件仅供 AI 读写，不影响 CLI 行为，不需要加入 `.gitignore`（插件目录本身不纳入 git）
- 被 workflow 调用的子 skill（meegle-plugin-feature、meegle-plugin-publish 等）在被 workflow 调用时同样需要更新此文件
- 独立调用子 skill 时（非 workflow 编排），不强制要求写 checkpoint

## 关键设计原则

1. **新插件推到底**：默认跑到发布完成、输出分享链接——用户进来时的心智是"我要一个能用的 xxx 插件"，不中途停下让用户手动进下一步
2. **不可逆动作显式确认**：Phase 3 publish 前必须用户显式同意，这是唯一不可省略的护栏
3. **延迟填充**：插件名称/描述/分类在发布前（Phase 3 polish）由 AI 基于实际功能总结生成，不在 Phase 1 创建时问用户
4. **快速到调试**：Phase 2 尾部强制引导本地调试，让用户在发布前通过真实预览确认功能
5. **容错兜底**：每个 Phase 失败不影响已完成的步骤，可从 `.lpm-cache/state.json` 断点继续
6. **精确恢复**：checkpoint 文件记录每一步 CLI 操作状态，中断后能告知用户确切进度
