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
1. 确认 AI 推荐的点位方案（基于 schema 分析）
2. 本地预览调试后确认功能是否 OK
3. 确认发布

## 完整流程

1. **Phase 0 — 确认站点域名 + 留存原话** → 钉死 `siteDomain`（CLI 硬前置）+ 逐字记录用户启动时说的那句话到 checkpoint（纯录音，零解析）
2. **Phase 1 — 搭建工程** → env-setup（按需）/ plugin-create，产出可运行的插件骨架
3. **Phase 2 — 点位配置** → 完整把用户意图交给 meego-point-config：意图识别 / 术语消歧 / 点位匹配 / schema 切片 / 用户确认 / URL 询问 / 推送配置
4. **Phase 3 — 实现功能** → plugin-code-gen / 本地调试 / 功能确认
5. **Phase 4 — 发布上线** → plugin-polish / plugin-publish

**Phase 0 只录音，Phase 2 / 3 才理解**：`context.originalRequirement` 在 Phase 0 逐字记录到 checkpoint，后续 Phase 2（meego-point-config 意图识别）和 Phase 3（plugin-code-gen 方案设计）从 checkpoint 读原话。编排层不对原话做任何解析、不推导默认值；各 Phase 的校验在自己真正需要的数据源（schema / point-type doc / MCP / types）到位时才做。

## 各阶段详细流程

- Phase 0 → 读取 `references/phase-0-context.md`
- Phase 1 → 读取 `references/phase-1-scaffold.md`
- Phase 2 → 读取 `references/phase-2-point-config.md`
- Phase 3 → 读取 `references/phase-3-implement.md`
- Phase 4 → 读取 `references/phase-4-release.md`

## 使用方式

```
/plugin-workflow                        # 端到端全流程（推荐）
/plugin-workflow phase=1                # 仅搭建工程
/plugin-workflow phase=2                # 仅点位配置
/plugin-workflow phase=3                # 仅实现功能
/plugin-workflow phase=4                # 仅发布上线
```

## 进度追踪（Checkpoint 机制）

每次执行 CLI 命令或关键步骤前后，**MUST** 更新项目根目录的 `.lpm-cache/state.json`，使中断后能精确恢复。

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
| Phase 4 发布成功 | 删除 `.lpm-cache/state.json`（流程已完成） |

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
- 各子 skill（meego-point-config、plugin-publish 等）在被 workflow 调用时同样需要更新此文件
- 独立调用子 skill 时（非 workflow 编排），不强制要求写 checkpoint

## 关键设计原则

1. **最小打断**：只在必须用户决策时才暂停，其余 AI 自动处理
2. **延迟填充**：插件名称/描述/分类在发布前由 AI 总结生成，不在创建时问用户
3. **快速到调试**：尽快让用户看到实际效果，通过调试预览来验证需求理解是否正确
4. **容错兜底**：每个阶段失败不影响已完成的步骤，可从断点继续
5. **精确恢复**：通过 checkpoint 文件记录每一步 CLI 操作状态，中断后能告知用户确切进度
