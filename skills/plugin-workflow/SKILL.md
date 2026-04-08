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

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则等公共约定。**
**CRITICAL — 进入每个 Phase 前，务必先用 Read 工具读取对应的 references 文档，禁止直接盲目执行。**
**CRITICAL — 禁止修改 `.lpm/` 目录下的任何文件，该目录由 CLI 内部管理，只能通过 CLI 命令间接操作。**

## 核心理念

**用户不需要知道"插件开发"的概念。** 用户只需要说"我想要一个 xxx 功能"，AI 判断这个功能需要自定义插件来承载，然后自动完成从创建到发布的全部流程。

用户全程只需要在关键节点做确认：
1. 确认功能理解是否正确
2. 确认 AI 推荐的点位方案（基于 schema 分析）
3. 本地预览调试后确认功能是否 OK
4. 确认发布

## 完整流程

```
┌─────────────────────────────────────────────────────┐
│  用户："我需要一个在详情页展示关联需求图谱的功能"        │
└──────────────────────┬──────────────────────────────┘
                       ▼
              ┌── Phase 1：理解需求 ──┐
              │  分析功能需求          │
              │  → 用户确认理解 ←      │  ← 只确认功能，不涉及点位
              └────────┬──────────────┘
                       ▼
              ┌── Phase 2：搭建工程 ──┐
              │  env-setup（如需要）   │
              │  plugin-create         │
              │  拉取 schema           │
              │  分析推荐点位          │
              │  → 用户确认点位方案 ←  │  ← 基于 schema 的专业推荐
              │  meego-point-config    │
              └────────┬──────────────┘
                       ▼
              ┌── Phase 3：实现功能 ──┐
              │  plugin-code-gen       │
              │  本地调试预览          │
              │  → 用户确认功能 OK ←   │  ← 关键交互点
              └────────┬──────────────┘
                       ▼
              ┌── Phase 4：发布上线 ──┐
              │  plugin-polish         │
              │  plugin-publish        │
              └───────────────────────┘
```

## 各阶段详细流程

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

## 关键设计原则

1. **最小打断**：只在必须用户决策时才暂停，其余 AI 自动处理
2. **延迟填充**：插件名称/描述/分类在发布前由 AI 总结生成，不在创建时问用户
3. **快速到调试**：尽快让用户看到实际效果，通过调试预览来验证需求理解是否正确
4. **容错兜底**：每个阶段失败不影响已完成的步骤，可从断点继续
