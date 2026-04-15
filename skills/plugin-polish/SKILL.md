---
name: plugin-polish
version: 1.0.0
description: |
  Meego 插件信息完善（编排 skill）：AI 根据已实现的功能自动生成插件名称、短描述、详情描述，选择分类，更新到后台。
  当用户在插件工程中说"完善插件信息"、"改名称"、"改描述"、"改分类"、"更新插件描述"时触发，或由 plugin-workflow 内部调用。
  前提：plugin-code-gen skill 已执行，代码已就绪。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder update-description --help"
---

# plugin-polish Skill

> **前置**：先 Read [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md) 获取共享规则；进入每个 mode 前 Read 对应的 `references/<mode>.md`。

## 核心理念

**在功��实现之后填充基本信息，AI 能基于实际代码生成高质量的描述。**

此时 AI 已经知道：
- 插件配置了哪些点位（page/view/dashboard/button/...）
- 每个点位的名���和功能描述
- 代码实际实现了什么逻辑

基于这些信息生成的名称和描述比创建时"猜"的要准确得多。

## 核心流程

```
mode=analyze → 读取点位配置 + 代码，理解插件实际功能
mode=generate → AI 生成名称/短描述/详情描述 + 获取分类列表并推荐
mode=confirm → 展示生成结果，用户确认或调整
mode=apply   → 调用 CLI update-description 命令更新到后台
mode=pipeline（默认）→ analyze → generate → confirm → apply
```

## 使用方式

```
/plugin-polish                      # 端到端全流程（推荐）
/plugin-polish mode=analyze         # 仅分析功能
/plugin-polish mode=generate        # 生成描述信息
/plugin-polish mode=confirm         # 展示并确认
/plugin-polish mode=apply           # 提交到后台
```

## 各模式详细流程

- `mode=analyze`  → 读取 `references/analyze.md`
- `mode=generate` → 读取 `references/generate.md`
- `mode=confirm`  → 读取 `references/confirm.md`
- `mode=apply`    → 读取 `references/apply.md`

## 输入

| 来源 | 用途 |
|------|------|
| `plugin.config.json` | pluginId、siteDomain |
| `point-schema.yaml` 或点位配置 | 点位类型、名称、功能描述 |
| `src/` 代码文件 | 实际功能逻辑 |

## 输出

通过 `update-description` CLI ��令更新到后台：
- 插件名称（正式名称替换工作名称）
- 短描述（≤100字纯文本）
- 详情描述（多段纯文本，CLI 自动转富文本）
- 分类（从列表中选择）

> 前置依赖链由 `plugin-workflow` 统一维护（... → plugin-code-gen → plugin-polish → plugin-publish）。
