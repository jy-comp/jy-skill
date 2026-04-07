---
name: plugin-create
version: 1.0.0
description: |
  Meego 插件工程创建（内部 skill）：最小化创建插件工程骨架。
  当用户明确说"创建插件工程"、"初始化插件项目"时触发，或由 plugin-workflow 内部调用。
  若用户说"我需要一个 xxx 功能"，应由 plugin-workflow 统一编排，而非直接调用此 skill。
  前提：env-setup skill 已执行（CLI 可用、Token 已配置）。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder create --help"
---

# plugin-create Skill

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则等公共约定。**
**CRITICAL — 进入每个 mode 前，务必先用 Read 工具读取对应的 references 文档，禁止直接盲目执行。**

## 核心理念

**用户的出发点是功能需求，不是插件元信息。**

用户说"我需要一个在看板上显示燃尽图的功能"，而不是"帮我创建一个名叫工时统计助手、分类为效率工具的插件"。因此：
- 创建阶段只需要一个**工作名称**和 siteDomain
- 名称/描述/分类等展示信息在**发布前**由 AI 自动总结（plugin-polish skill）
- 让用户尽快进入点位配置和代码开发

## 核心流程

```
mode=setup   → 确认 env-setup 已完成 + 获取 siteDomain
mode=plan    → 从功能需求中提取工作名称 + 用户确认
mode=apply   → npx @byted-meego/cli@builder create（仅 name，无需描述/分类）
mode=verify  → 检查 plugin.config.json 完整性 + 工程目录就绪
mode=pipeline（默认）→ setup → plan → apply → verify
```

## 使用方式

```
/plugin-create                    # 端到端全流程
/plugin-create mode=plan          # 仅确认信息
/plugin-create mode=apply         # 执行创建
/plugin-create mode=verify        # 验证工程就绪
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
plugin-create → meego-point-config → plugin-code-gen → plugin-polish → plugin-publish
                                                        ↑ AI 总结已实现功能
                                                          生成名称/描述/分类
```

## 前置依赖

- env-setup skill 已执行（CLI 可用、Token 已配置）
