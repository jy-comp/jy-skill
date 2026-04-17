---
name: meegle-plugin-create
version: 1.0.0
description: |
  Meegle 插件工程创建（原子 skill）：最小化创建插件工程骨架（plugin.config.json + 工程目录）。
  当用户明确只想创建空壳工程（"创建一个空的插件工程"、"初始化插件项目骨架"）时触发，或由 meegle-plugin-workflow 内部调用。
  若用户说"我需要一个 xxx 插件" / "从零做一个 xxx"，应由 meegle-plugin-workflow 统一编排（含 create + feature + polish + publish），而非直接调用此 skill。
  前提：meegle-plugin-env-setup 已执行（CLI 可用、Token 已配置）。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder create --help"
---

# meegle-plugin-create Skill

> **前置**：先 Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 获取共享规则；进入每个 mode 前 Read 对应的 `references/<mode>.md`。

## 本 skill 的最少 Read 清单

- 共享规则 → Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md)
- mode=setup → Read [`references/setup.md`](references/setup.md)
- mode=plan → Read [`references/plan.md`](references/plan.md)
- mode=apply → Read [`references/apply.md`](references/apply.md)
- mode=verify → Read [`references/verify.md`](references/verify.md)
- 不要预加载 4 个 mode reference；按当前 mode 按需 Read

## 核心理念

**用户的出发点是功能需求，不是插件元信息。**

用户说"我需要一个在看板上显示燃尽图的功能"，而不是"帮我创建一个名叫工时统计助手、分类为效率工具的插件"。因此：
- 创建阶段只需要一个**工作名称**和 siteDomain
- 名称/描述/分类等展示信息在**发布前**由 AI 自动总结（meegle-plugin-polish skill）
- 让用户尽快进入点位配置和代码开发

## 核心流程

```
mode=setup   → 确认 meegle-plugin-env-setup 已完成 + 获取 siteDomain
mode=plan    → 从功能需求中提取工作名称 + 用户确认
mode=apply   → npx @byted-meego/cli@builder create（仅 name，无需描述/分类）
mode=verify  → 检查 plugin.config.json 完整性 + 工程目录就绪
mode=pipeline（默认）→ setup → plan → apply → verify
```

## 使用方式

```
/meegle-plugin-create                    # 端到端全流程
/meegle-plugin-create mode=plan          # 仅确认信息
/meegle-plugin-create mode=apply         # 执行创建
/meegle-plugin-create mode=verify        # 验证工程就绪
```

## 各模式详细流程

- `mode=setup`  → 读取 `references/setup.md`
- `mode=plan`   → 读取 `references/plan.md`
- `mode=apply`  → 读取 `references/apply.md`
- `mode=verify` → 读取 `references/verify.md`

## 输出产物

| 产物 | 说明 |
|------|------|
| `plugin.config.json` | 含 pluginId、pluginSecret、siteDomain、name、resources: [] |
| `node_modules/` | 插件工程依赖，由 init 自动安装 |
| `src/` | 工程模板目录结构 |

## 后续流程

```
meegle-plugin-create → meegle-plugin-feature（Stage Config 点位配置 + Stage Code 代码）→ meegle-plugin-polish → meegle-plugin-publish
                                                                                  ↑ AI 总结已实现功能
                                                                                    生成名称/描述/分类
```

> 前置依赖链由 `meegle-plugin-workflow` 统一维护（meegle-plugin-env-setup → meegle-plugin-create → ...），独立调用时 AI 自行判断是否需要先跑 `meegle-plugin-env-setup`。
