---
name: plugin-code-gen
version: 1.0.0
description: |
  Meego 插件前端代码生成（内部 skill）：根据已配置的点位，AI 直接生成各点位对应的 React 代码文件，更新 plugin.config.json 的 resources 字段。
  当用户在已有插件工程中明确说"生成代码"、"帮我写插件代码"时触发，或由 plugin-workflow 内部调用。
  前提：meego-point-config skill 已执行完毕，点位配置已推送远端。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder start --help"
---

# plugin-code-gen Skill

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则等公共约定。**
**CRITICAL — 进入每个 mode 前，务必先用 Read 工具读取对应的 references 文档，禁止直接盲目执行。**

## 核心流程

```
mode=setup   → 检查点位配置已完成 + 沙盒工程目录就绪
mode=plan    → AI 分析点位列表，规划各点位的代码骨架
mode=apply   → AI 基于 entry 路径生成代码文件 → 写入对应路径
mode=verify  → tsc --noEmit 语法预检；如有错误最多自动修复 2 轮
mode=pipeline（默认）→ setup → plan → apply → verify
```

## 关键说明

**无 CLI，AI 直接生成**：本 skill 不调用任何 npx @byted-meego/cli@builder 命令来生成代码，完全由 AI 根据点位类型和 JSSDK 规范生成 TypeScript/React 代码。

## entry 驱动的代码生成（关键）

**代码生成必须基于 `plugin.config.json` 中已有的 `resources` entry 路径来创建文件。**

在 meego-point-config 的 apply 阶段执行 `npx @byted-meego/cli@builder update` 后，CLI 会自动在 `plugin.config.json` 的 `resources` 字段中生成各点位的 entry 信息，例如：

```json
{
  "resources": {
    "board": [
      {
        "key": "jira_board_main",
        "entry": "src/features/board/jira_board_main/index.tsx"
      }
    ]
  }
}
```

代码生成流程：
1. **读取** `plugin.config.json` 中 `resources` 各点位的 `entry` 路径
2. **按 entry 路径创建对应的代码文件**
3. **禁止自行拼接文件路径**，禁止修改 `plugin.config.json` 中已有的 entry 信息

entry 是 CLI 生成的唯一真实来源，代码生成只负责在 entry 指定的路径下创建代码文件。

## 代码骨架规范

详见 `references/plan.md` — 各点位类型的代码模板。

## 使用方式

```
/plugin-code-gen                    # 端到端全流程（推荐）
/plugin-code-gen mode=plan          # 仅规划代码骨架，不生成文件
/plugin-code-gen mode=apply         # 生成代码并写入沙盒
/plugin-code-gen mode=verify        # 执行 tsc 预检
```

## 各模式详细流程

- `mode=setup`  → 读取 `references/setup.md`
- `mode=plan`   → 读取 `references/plan.md`
- `mode=apply`  → 读取 `references/apply.md`
- `mode=verify` → 读取 `references/verify.md`

## 输出产物

| 产物 | 说明 |
|------|------|
| entry 路径对应的代码文件 | 基于 `plugin.config.json` resources 中的 entry 路径创建 |
| `project.zip`（按需）| 整个工程打包，供用户下载本地调试 |

## 代码质量保障分层策略

| 层级 | 方式 | 触发条件 |
|------|------|---------|
| L1 预检 | `tsc --noEmit` 语法检查（约 10s） | verify 阶段必须执行 |
| L2 自动修复 | AI 读取错误 → 生成 patch → 重试（最多 2 轮） | L1 发现错误时 |
| L3 降级兜底 | 提供 zip 下载 + 本地调试指引 | L2 仍失败时 |

## 本地调试引导

verify 通过后，引导用户启动本地开发服务器进行调试：

```bash
npx @byted-meego/cli@builder start --source_type local --auto
```

`--auto` 参数会自动根据 `getLocalConfig` 中的当前域名拼接调试 URL，并在浏览器中打开。

```
✅ 代码生成完成，语法检查通过

下一步：启动本地调试
  运行：npx @byted-meego/cli@builder start --source_type local --auto
  CLI 将自动打开浏览器调试页面
```
