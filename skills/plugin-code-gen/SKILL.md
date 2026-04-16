---
name: plugin-code-gen
version: 1.0.0
description: |
  Meego 插件前端代码生成（编排 skill）：根据已配置的点位，AI 生成各点位对应的 React 代码文件。
  当用户在插件工程中说"生成代码"、"帮我写插件代码"、"实现功能"时触发，或由 plugin-workflow 内部调用。
  前提：meego-point-config skill 已执行完毕，点位配置已推送远端。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder start --help"
---

# plugin-code-gen Skill

> **前置**：先 Read [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md) 获取共享规则。

这个 skill 生成的代码**强依赖业务平台提供业务上下文及每个点位的特定API**，凭经验拼接的代码能过 tsc 但运行时会跑废。

所以进 apply 写代码前，plan 阶段要先走完四步溯源，顺序固定：

1. **点位标准能力 doc**（`references/point-types/<点位类型>/`）—— 每个点位的场景化标准能力，是最贴近用户需求的参考资料
2. **`@lark-project/js-sdk` 类型定义** —— 单纯只描述每个API的校准签名 / 参数 / 返回值
3. **用户提供的代码** —— 真实业务上下文
4. **飞书项目知识 MCP** —— 前三源没讲清业务语义（枚举含义、取值空间）时兜底查

任何一行 SDK 调用都要能指回上面某个源；找不到源就停下问用户，不要拿 MCP 片段硬拼。<!-- TODO(lark-project): 调试完成后补回：工作项/视图等实例数据必须通过 `lark-project` skill 查询 -->plan 完成后会产出代码溯源审计报告，apply 会脚本化校验，详见 [`references/plan.md`](references/plan.md) 的"代码溯源协议"章节。

## 核心流程

```
mode=setup   → 从远端拉取配置 + 确认 entry 已生成（含模板代码）
mode=plan    → 读取平台提供的每个点位的标准能力，在这基础之上来完成用户功能的可行性方案设计
mode=apply   → 按plan的方案设计，实现用户功能的功能代码
mode=verify  → tsc --noEmit 语法预检；如有错误最多自动修复 2 轮
mode=pipeline（默认）→ setup → plan → apply → verify
```

## 关键说明

- **模板代码由 CLI 生成**：setup 阶段的 `npx @byted-meego/cli@builder update` 会在各 entry 路径下生成模板代码
- **AI 负责填充功能**：在 CLI 生成的模板代码基础上，根据用户需求实现真实功能

## resources 结构（关键）

`plugin.config.json` 的 `resources` 是一个**扁平数组**，每项包含 `id` 和 `entry`：

```json
{
  "resources": [
    {
      "id": "board_web_rj4v_m",
      "entry": "./src/features/board_web_rj4v_m/index.tsx"
    },
    {
      "id": "dashboard_web_xk9j2p",
      "entry": "./src/features/dashboard_web_xk9j2p/index.tsx"
    }
  ]
}
```

- `id`：由 CLI 生成的唯一标识，格式为 `{点位类型}_{平台}_{随机后缀}`，可从 id 前缀判断点位类型
- `entry`：代码文件路径，格式为 `./src/features/{id}/index.tsx`，CLI update 后该路径下已有模板代码

`id` 和 `entry` 由 CLI 维护——直接读 `plugin.config.json` 里的值，不要自己拼路径、也不要改已有的 `id`/`entry`（CLI 后续 update 会对不上）。

## 使用方式

```
/plugin-code-gen                    # 端到端全流程（推荐）
/plugin-code-gen mode=plan          # 匹配平台能力，结合用户需求，设计功能实现方案
/plugin-code-gen mode=apply         # 基于plan的方案设计，执行功能代码的生成
/plugin-code-gen mode=verify        # 执行 tsc 预检
```

## 各模式详细流程

每个 mode 的具体步骤、溯源检查点、point-types 专属 doc 清单都在对应 reference 里——这些是 SKILL.md 没有复述的执行细节，不读 reference 就没有足够信息写出能运行的代码。

- `mode=setup`  → `references/setup.md`
- `mode=plan`   → `references/plan.md`（及其指向的 `references/point-types/<点位类型>/` 下相关 doc）
- `mode=apply`  → `references/apply.md`
- `mode=verify` → `references/verify.md`

## 输出产物

| 产物 | 说明 |
|------|------|
| entry 路径对应的代码文件 | 在 CLI 生成的模板基础上填充功能代码 |
| `project.zip`（按需）| 整个工程打包，供用户下载本地调试 |

## 代码质量保障分层策略

| 层级 | 方式 | 触发条件 |
|------|------|---------|
| L1 预检 | `tsc --noEmit` 语法检查（约 10s） | verify 阶段执行 |
| L2 自动修复 | AI 读取错误 → 生成 patch → 重试（最多 2 轮） | L1 发现错误时 |
| L3 降级兜底 | 提供 zip 下载 + 本地调试指引；保留 `.lpm-cache/state.json` 和 `.lpm/`（这是流程断点，用户从本地修复后要靠它续跑，清了就回不来） | L2 仍失败时 |

## 本地调试引导

verify 通过后，引导用户启动本地开发服务器进行调试：

```bash
npx @byted-meego/cli@builder start --auto
```

`--auto` 参数会自动根据 `getLocalConfig` 中的当前域名拼接调试 URL，并在浏览器中打开。

```
✅ 代码生成完成，语法检查通过

下一步：启动本地调试
  运行：npx @byted-meego/cli@builder start --auto
  CLI 将自动打开浏览器调试页面
```
