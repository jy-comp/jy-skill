---
name: meegle-plugin-publish
version: 1.0.0
description: |
  Meegle 插件完整发布流程（编排 skill）：同步配置到后台 + 构建上传产物 + 版本发布 + 输出分享链接。
  当用户在插件工程中说"发布插件"、"上线"、"部署"、"release"、"publish"时触发，或由 meegle-plugin-workflow 内部调用。
  前提：meegle-plugin-polish 已执行（插件名称/描述/分类已填充），代码文件已就绪，plugin.config.json resources 已更新。
  **不可逆动作**：会把插件发布到 Meegle 市场，执行前必须用户显式确认。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder publish --help"
---

# meegle-plugin-publish Skill

> **前置**：先 Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 获取共享规则；进入每个 mode 前 Read 对应的 `references/<mode>.md`。

## 本 skill 的最少 Read 清单

- 共享规则 → Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md)（含根原则 3 不可逆动作显式确认）
- mode=pre-check → Read [`references/pre-check.md`](references/pre-check.md)
- mode=apply → Read [`references/apply.md`](references/apply.md)
- mode=verify → Read [`references/verify.md`](references/verify.md)
- 不要预加载 3 个 mode reference；按当前 mode 按需 Read

## 核心流程

```
mode=pre-check → tsc --noEmit 语法预检；AI 自动修复（最多 2 轮）；失败则降级
mode=apply     → 三步串行：
                 A1: update 同步配置到后台（兜底）
                 A2: release 构建 + 上传（获取 productVersion）
                 A3: publish 版本发布（productVersion 作为入参）→ 输出分享链接
mode=verify    → 确认发布成功 + 验证分享链接
mode=pipeline（默认）→ pre-check → apply → verify
```

## 关键说明

**三步强一致性**：update → release → publish 必须串行执行，任一步失败则终止后续步骤。

**版本号传递**：
- `npx @byted-meego/cli@builder release` 成功后输出 `productVersion`（如 `1 / 2 / 3 / ...`）
- `productVersion` 作为 `npx @byted-meego/cli@builder publish --product-version` 的入参
- `--version`（发布版本号）：可选，不传则自动在上一个版本号基础上 patch +1（如 `1.0.0` → `1.0.1`）
- `--desc-zh`（版本描述）：AI 基于当前代码改动自动总结生成，无需用户提供

**分享链接**：`npx @byted-meego/cli@builder publish` 成功后输出插件分享 URL。

## 使用方式

```
/meegle-plugin-publish                    # 端到端全流程（推荐）
/meegle-plugin-publish mode=pre-check     # 仅语法预检，不构建
/meegle-plugin-publish mode=apply         # 执行同步 + 构建 + 发布
/meegle-plugin-publish mode=verify        # 确认发布状态
```

## 各模式详细流程

- `mode=pre-check` → 读取 `references/pre-check.md`
- `mode=apply`     → 读取 `references/apply.md`
- `mode=verify`    → 读取 `references/verify.md`

## 错误处理策略

| 错误场景 | 处理方式 |
|---------|---------|
| tsc 语法错误 | AI 分析错误，修复代码文件，重新 pre-check（最多 2 轮） |
| update 同步失败 | 展示错误，终止（不继续 release） |
| webpack import 路径错误 | 捕获 error 输出，AI 尝试自动修复 1 轮，重新 release |
| 构建后仍失败 | 展示错误详情 + 本地调试指引 |
| publish 失败 | 展示错误，告知用户可手动重试 `npx @byted-meego/cli@builder publish` |

## publish 命令参数

```bash
npx @byted-meego/cli@builder publish \
  --version <版本号>              # 可选，不传则默认在上一版本基础上 patch +1
  --desc-zh <版本描述>             # AI 自动基于改动总结生成
  --product-version <产物版本>     # 必填，来自 release 输出
  --store <publish|no-publish>    # 可选，是否发布到插件商店，默认 no-publish
  --upgrade <manual|all|limit>    # 可选，CLI 默认 manual；AI 一律不主动传，仅当用户明确要求"自动升级全部/按范围升级"时才传 all/limit
```

## 输出产物

| 产物 | 说明 |
|------|------|
| 产物版本号 | 从 `npx @byted-meego/cli@builder release` stdout 解析，如 `1 / 2 / 3 / ...` |
| 发布版本号 | 自动累加或用户指定的 `--version`，如 `1.0.1` |
| 分享链接 | `npx @byted-meego/cli@builder publish` 输出的插件分享 URL |
