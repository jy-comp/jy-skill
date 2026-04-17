---
name: meegle-plugin-cli
version: 2.0.0
description: |
  Meegle 插件开发 CLI 原子化命令参考与诊断工具（npx @byted-meego/cli@builder）。
  **本 CLI 仅服务于插件工程本身的脚手架/配置/构建/调试/发布，不操作 Meegle 产品业务数据（工作项、视图、仪表盘、需求、缺陷等）**——后者请使用对应的业务工具或 skill，不要派发到本 skill。
  当用户在插件工程目录（存在 plugin.config.json 且含 MII_ 开头的 pluginId）下需要执行单条 CLI 命令时触发。
  覆盖原子化操作：启动调试、本地预览、跑起来看看、构建、打包、同步配置、拉取远端、推送配置、
  查看配置、查看远端配置、生成 schema、查看分类、登录、认证。
  覆盖诊断查询:当前状态、进度到哪了、这个插件是干什么的、调试报错、start 起不来、
  tsc 报错、代码报错、能回退吗、改代码。
  注意:多步编排操作（发布、点位配置、代码生成等）由对应的编排 skill 直接处理，不经过本 skill。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# Meegle 插件开发 CLI 原子化命令参考 & 诊断工具

> **前置**：先 Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 获取共享规则（认证、安全、工具职责、插件工程识别、三条根原则等）。

## 本 skill 的最少 Read 清单

- 命令具体参数 / 行为 → Read [`references/commands.md`](references/commands.md)
- 状态诊断 / 故障排查 → Read [`references/diagnose.md`](references/diagnose.md)
- 不要预加载两个 references；按用户实际需要按需 Read

## 项目上下文感知

使用本 skill 前，**MUST** 按 [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 的"插件工程识别"规则确认当前处于插件工程目录（存在 `plugin.config.json` 且含 `MII_` 前缀的 `pluginId`）。识别通过后，`siteDomain` / `pluginId` / `resources` 由 CLI 自动读取，无需手动传参。

不在插件目录时，引导用户先 `create` 或 `cd` 到插件目录。

---

## 本 skill 的职责范围

**只处理两类场景：原子化 CLI 命令 和 诊断查询。**

多步编排操作由对应的编排 skill 直接处理：
- "发布"/"上线" → `meegle-plugin-publish`
- "加个点位"/"改点位"/"生成代码"/"实现功能" → `meegle-plugin-feature`（存量插件）
- "改名称"/"改描述" → `meegle-plugin-polish`
- "从零做一个 xxx 插件"/"新建插件" → `meegle-plugin-workflow`（新插件全流程）

### A. 原子化 CLI 命令（本 skill 直接执行）

每条命令独立完成一个操作，无需多步编排。**详细参数见 [`references/commands.md`](references/commands.md)**：

| 用户可能的说法 | CLI 命令 | 详见 |
|--------------|---------|------|
| "启动调试" / "本地预览" / "跑起来看看" / "dev" | `start --auto` | [启动调试](references/commands.md#启动调试) |
| "构建" / "打包" / "build" | `build` | [构建](references/commands.md#构建) |
| "同步配置" / "拉取远端" / "拉取最新" | `update` | [同步配置](references/commands.md#同步配置) |
| "推送配置" / "推到远端" | `update --source-type=local` | [同步配置](references/commands.md#同步配置) |
| "查看点位配置" / "当前配置" | `local-config get` | [点位配置管理](references/commands.md#点位配置管理) |
| "查看远端配置" | `local-config get --remote` | [点位配置管理](references/commands.md#点位配置管理) |
| "生成 schema" / "看看有哪些点位类型" | `schema` | [生成 Schema](references/commands.md#生成-schema) |
| "查看分类" / "有哪些分类" | `list-categories` | [分类列表](references/commands.md#分类列表) |
| "登录" / "认证" / "设置 token" | `login` | [登录认证](references/commands.md#登录认证) |

### B. 诊断查询（本 skill 读取项目状态回答）

用户不是要执行操作，而是要了解当前状态或排查问题。**详细诊断流程见 [`references/diagnose.md`](references/diagnose.md)**：

| 用户可能的说法 | 处理方式 | 详见 |
|--------------|---------|------|
| "当前什么状态" / "进度到哪了" / "之前做到哪了" | 读取 checkpoint + config + src/ 综合判断 | [项目状态诊断](references/diagnose.md#项目状态诊断) |
| "这个插件是干什么的" / "有哪些点位" | 读取 resources 列表，按 id 前缀分析点位类型 | [项目状态诊断](references/diagnose.md#项目状态诊断) |
| "调试报错了" / "start 起不来" / "白屏" | 常见调试问题排查 | [常见问题排查](references/diagnose.md#常见问题排查) |
| "tsc 报错" / "代码编译错误" / "类型错误" | 引导执行 `tsc --noEmit` 或使用 `/meegle-plugin-feature stage=code` | [常见问题排查](references/diagnose.md#常见问题排查) |
| "能回退版本吗" / "撤销发布" / "版本回滚" | CLI 不支持版本回退，引导后台手动操作 | [常见问题排查](references/diagnose.md#常见问题排查) |
| "改一下代码" / "按钮逻辑改成 xxx" | 直接修改 `src/features/` 下的代码文件，不需要 CLI | [常见问题排查](references/diagnose.md#常见问题排查) |

---

## 与编排 skill 的分工（路由总览）

| 用户指令类型 | 示例 | 由谁处理 |
|------------|------|--------|
| 原子操作 | "启动调试" / "构建" / "同步" / "查看配置" | meegle-plugin-cli（本 skill）直接执行单条 CLI |
| 存量插件功能迭代 | "加点位" / "生成代码" / "实现功能" / "改功能逻辑" | meegle-plugin-feature（Stage Config 配置 + Stage Code 代码合并） |
| 发布 / 信息完善 | "发布" / "改名称" | meegle-plugin-publish / meegle-plugin-polish |
| 新插件端到端 | "从零做一个 xxx 插件" / "新建插件" | meegle-plugin-workflow（新插件全流程） |
| 诊断查询 | "什么状态" / "调试报错" | meegle-plugin-cli（本 skill）读取状态 / 排查引导 |

**分工原则**：单条 CLI 命令能完成的用 meegle-plugin-cli，需要多步串行/交互确认的用编排 skill；编排 skill 按"新插件 vs 存量插件 vs 单点操作"分工。
