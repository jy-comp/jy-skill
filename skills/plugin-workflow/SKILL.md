---
name: plugin-workflow
version: 1.0.0
description: |
  端到端插件开发流程编排：用户描述功能需求 → 自动创建插件 → 配置点位 → 生成代码 → 本地调试 → 完善信息 → 发布。
  当用户说"我需要一个 xxx 功能"、"帮我做一个 xxx"、"我想在看板/详情页/按钮上加 xxx"等场景时触发。
  这是一个面向用户的顶层 skill，内部串联 env-setup / plugin-create / meego-point-config / plugin-code-gen / plugin-polish / plugin-publish。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# plugin-workflow Skill

> **前置**：先 Read [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md) 获取共享规则；进入每个 Phase 前 Read 对应的 `references/phase-N-*.md`。

## 核心理念

**用户不需要知道"插件开发"的概念。** 用户只需要说"我想要一个 xxx 功能"，AI 判断这个功能需要自定义插件来承载，然后自动完成从创建到发布的全部流程。

用户全程只需要在关键节点做确认：
1. 确认功能理解是否正确
2. 确认 AI 推荐的点位方案（基于 schema 分析）
3. 本地预览调试后确认功能是否 OK
4. 确认发布

## 完整流程

1. **Phase 0 — 确认站点上下文** → 询问用户 siteDomain（MUST，禁止默认）
2. **Phase 1 — 理解需求** → 确认功能（不涉及点位）
3. **Phase 2 — 搭建工程** → env-setup（按需）/ plugin-create / schema / 点位方案确认 / meego-point-config
4. **Phase 3 — 实现功能** → plugin-code-gen / 本地调试 / 功能确认
5. **Phase 4 — 发布上线** → plugin-polish / plugin-publish

## 各阶段详细流程

- Phase 0 → 读取 `references/phase-0-context.md`
- Phase 1 → 读取 `references/phase-1-understand.md`
- Phase 2 → 读取 `references/phase-2-scaffold.md`
- Phase 3 → 读取 `references/phase-3-implement.md`
- Phase 4 → 读取 `references/phase-4-release.md`

## 使用方式

```
/plugin-workflow                        # 端到端全流程（推荐）
/plugin-workflow phase=1                # 仅需求分析
/plugin-workflow phase=2                # 仅搭建工程
/plugin-workflow phase=3                # 仅实现功能
/plugin-workflow phase=4                # 仅发布上线
```

## 进度追踪（Checkpoint 机制）

每次执行 CLI 命令或关键步骤前后，**MUST** 更新项目根目录的 `.plugin-workflow-state.json`，使中断后能精确恢复。

### 文件格式

```json
{
  "phase": 2,
  "step": "2.4",
  "stepName": "meego-point-config",
  "lastCommand": "npx @byted-meego/cli@builder local-config set ...",
  "lastCommandStatus": "success",
  "nextCommand": "npx @byted-meego/cli@builder update",
  "nextStep": "2.4 推送远端",
  "timestamp": "2026-04-10T14:30:00Z",
  "context": {
    "pluginId": "MII_xxx",
    "siteDomain": "https://meego.feishu-boe.cn"
  }
}
```

### 写入规则

| 时机 | 动作 |
|------|------|
| 执行 CLI 命令**前** | 写入 `nextCommand` + `nextStep`，`lastCommandStatus` 设为 `"running"` |
| 执行 CLI 命令**后** | 写入 `lastCommand` = 刚执行的命令，`lastCommandStatus` = `"success"` / `"failed"` |
| 进入新 Phase/Step | 更新 `phase` + `step` + `stepName` |
| Phase 4 发布成功 | 删除 `.plugin-workflow-state.json`（流程已完成） |

### 恢复规则

每次 workflow 启动时：
1. 检查 `.plugin-workflow-state.json` 是否存在
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
- 各子 skill（meego-point-config、plugin-publish 等）在被 workflow 调用时同样需要更新此文件
- 独立调用子 skill 时（非 workflow 编排），不强制要求写 checkpoint

## 关键设计原则

1. **最小打断**：只在必须用户决策时才暂停，其余 AI 自动处理
2. **延迟填充**：插件名称/描述/分类在发布前由 AI 总结生成，不在创建时问用户
3. **快速到调试**：尽快让用户看到实际效果，通过调试预览来验证需求理解是否正确
4. **容错兜底**：每个阶段失败不影响已完成的步骤，可从断点继续
5. **精确恢复**：通过 checkpoint 文件记录每一步 CLI 操作状态，中断后能告知用户确切进度
