---
name: meegle-plugin-feature
version: 1.0.0
description: |
  Meegle 插件功能迭代（存量插件）：在已有插件工程上完成点位配置变更（增/改/删）和对应的 React 代码生成，Stage Config + Stage Code 串行绑定、线性执行到本地可调试。
  当用户**在已有插件工程目录内**说"加个看板点位"、"改一下点位"、"删除点位"、"配置点位"、"生成代码"、"重新生成代码"、"改这个点位的展示逻辑"、"实现 xxx 功能"、"加个 xxx 功能到看板/详情页/按钮"、"给插件加一个 xxx 能力"、"改功能逻辑"时触发。
  前置要求：当前目录存在 `plugin.config.json`（判别"是否为插件工程"）。若不存在 → 不执行本 skill，引导到 meegle-plugin-workflow（新插件全流程）。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# Meegle 插件功能迭代 Skill

> **前置**：先 Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 获取共享规则；进入每个 step 前 Read 对应的 `references/<step>.md`。

## 本 skill 的最少 Read 清单

- 共享规则 → Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md)（含三条根原则）
- Stage Config 子步骤 → 按当前 step 取 [`references/config-{setup,plan,apply,verify}.md`](references/)
- Stage Code 子步骤 → 按当前 step 取 [`references/code-{setup,plan,apply,verify}.md`](references/)
- 点位标准能力 doc → 仅 `liteAppComponent` 提供，进 `references/point-types/liteAppComponent/index.md` 看分场景索引；其他点位类型当前无 doc，走 code-plan.md Step 4 兜底
- URL 详细策略 → 仅在涉及 URL 字段填写时 Read [`../meegle-plugin-shared/references/url-policy.md`](../meegle-plugin-shared/references/url-policy.md)
- **不要预加载 8 个子步骤 reference + 4 个 point-type doc**；只读当前 step 和当前点位类型对应的那 1-2 份

### 按场景细化最小读取（识别意图后再决定读哪些）

| 场景 | 必读 | 不读 |
|---|---|---|
| **stage=config**（只改点位声明） | `config-{setup,plan,apply,verify}.md` 按当前子步 | 不读任何 `code-*.md` |
| **stage=code**（只改代码，点位配置已在远端） | `code-{setup,plan,apply,verify}.md` 按当前子步 + 点位 doc 按 index 命中 | 不读任何 `config-*.md`；点位配置直接从 `point.config.local.json` 用 jq 切片 |
| **端到端** | Stage Config 子步按需 → 进 Stage Code 时才读 `code-*` 和点位 doc | — |
| **liteAppComponent 相关任务** | 进 Stage Code 时加读 `point-types/liteAppComponent/index.md`，按维度判定再 Read `read-props.md` 或 `write-outputs.md` | 其他维度对应的 doc |
| **其他点位类型** | 无专属 doc，走 `code-plan.md` Step 4 兜底 | `point-types/` 目录整体跳过 |

## 前置守卫（入口必须先检查）

执行任何后续步骤前，先验证当前工作目录是否为 Meegle 插件工程：

```bash
test -f ./plugin.config.json && echo "OK" || echo "NOT_PLUGIN_DIR"
```

| 检查结果 | 处理 |
|---------|------|
| 存在 `plugin.config.json` | ✅ 继续本 skill 后续流程 |
| 不存在 | ❌ 停止；告知用户"当前目录不是 Meegle 插件工程，新插件请走 meegle-plugin-workflow（会从零创建 → 配点位 → 写代码 → 本地调试 → 完善信息 → 发布）"，让用户决定是否切换 skill |

守卫设计理由：本 skill 是"在已有插件上做功能迭代"，不负责创建插件工程。AI 路由层若把"我想做一个 xxx 插件"这种新建意图命中到本 skill，守卫会把用户接回正确路径。

## 核心流程（线性，Stage Config + Stage Code 绑定）

**功能迭代 = 点位配置变更 + 代码实现**。两件事在本 skill 内绑定执行，不可拆开单独跑（单跑点位不产生可运行代码，单跑代码没有配置依赖就是孤立文件）。

> **命名约定**：本 skill 内部用 `Stage Config` / `Stage Code` 描述两个串行阶段（与 workflow 的 `Phase 0/1/2/3` 命名空间不重合，避免 checkpoint 写串）。

```
前置守卫（plugin.config.json 存在）
   ↓
Stage Config — 点位配置
   config.setup → config.plan → config.apply → config.verify
   ↓（Stage Config 成功推送远端后才能进 Stage Code）
Stage Code — 代码实现
   code.setup → code.plan → code.apply → code.verify
   ↓
本地就绪：引导用户 `npx @byted-meego/cli@builder start --auto` 启动本地调试
```

### 为什么必须线性串行

- **Stage Config 的 apply 是全量替换**：`local-config set` 和 `update --source-type=local` 一次提交全部点位数据，遗漏任何现有点位 = 永久删除
- **Stage Code 的 plan 必须走完代码溯源四步**：点位标准能力 doc → `@lark-project/js-sdk` 类型定义 → 用户提供代码 → 飞书项目知识 MCP。凭经验拼的 SDK 调用能过 tsc 但运行时会跑废
- **Stage Config 不 verify 就进 Stage Code** = 基于未确认的配置去生成代码，配置一旦回滚代码就是废的

## 使用方式

```
/meegle-plugin-feature                    # 端到端（推荐，Stage Config + Stage Code 全走）
/meegle-plugin-feature stage=config       # 仅点位配置（存量迭代中只改 schema）
/meegle-plugin-feature stage=code         # 仅代码生成（需 Stage Config 已完成，适合重新生成）
```

- `stage=config`：只跑 Stage Config（config.setup → config.plan → config.apply → config.verify），不进入 Stage Code
- `stage=code`：只跑 Stage Code（code.setup → code.plan → code.apply → code.verify），前提是点位配置已在远端就位
- 不带参数：完整串行跑完 Stage Config + Stage Code

## 各阶段详细流程（必读 references）

> 进入每个子步骤前 MUST Read 对应 reference 文件——SKILL.md 没有复述执行细节、jq 切片命令、URL 触发表、P3 交互顺序、代码溯源协议这些只在 references 里，凭记忆跳过是这个 skill 最常见的失败模式。

**Stage Config — 点位配置**：

| 子步骤 | 职责 | Reference |
|--------|------|-----------|
| config.setup | 环境检查 + 前置验证 | `references/config-setup.md` |
| config.plan | 理解需求 + 匹配平台能力 + 交互补全 + 生成配置文件 | `references/config-plan.md` |
| config.apply | `local-config set` 校验 + `update` 推送远端 | `references/config-apply.md` |
| config.verify | `local-config get --remote` 验证远端数据 | `references/config-verify.md` |

**Stage Code — 代码实现**：

| 子步骤 | 职责 | Reference |
|--------|------|-----------|
| code.setup | 从远端拉取配置 + 确认 entry 已生成（含模板代码） | `references/code-setup.md` |
| code.plan | 读点位标准能力 doc，设计功能可行性方案 | `references/code-plan.md`（+ `references/point-types/<点位类型>/`） |
| code.apply | 按 plan 方案实现功能代码 | `references/code-apply.md` |
| code.verify | `tsc --noEmit` 语法预检；失败时最多自动修复 2 轮 | `references/code-verify.md` |

## 文件约定

所有中间产物落在 `.lpm-cache/` 工作区，由 CLI 在命令成功时自动清理，AI 只读写、不负责 `rm`：

| 文件 | 写入者 | 消费者 | CLI 自动清理时机 |
|------|-------|-------|------------|
| `.lpm-cache/schema/point-schema.json` | `lpm schema` | config.plan 的 jq 切片；code.plan / meegle-plugin-polish 按需 Read（在 code-gen 阶段被清理，见下一行） | `publish` 成功 |
| `.lpm-cache/config/remote.json` | `lpm local-config get [--remote]` | config.plan 生成 draft 的全量基线；config.apply A0 的 diff 检查 | `local-config set` 成功 |
| `.lpm-cache/config/draft-{timestamp}.json` | config.plan 阶段 AI 写 | `lpm local-config set --from` 读 | `local-config set` 成功 |
| `.lpm-cache/mcp/<slug>.md` | AI 在 MCP 命中后 Write | 后续同 slug 查询复用 | `publish` 成功 |

**下游消费者注意**：`set` 成功后 `remote.json` 即消失，Stage Code 及之后读点位配置请改读工程根目录的 **`point.config.local.json`**（CLI 在 `set` 成功时和 `update` 拉取时都会刷新这份文件，内容是同一份 local-format 点位配置）。

Workflow 断点文件 `.lpm-cache/state.json` 由 meegle-plugin-workflow 维护，CLI 的 `publish` 也会保留它。若需整体清空（含 state），用 `npx @byted-meego/cli@builder workspace clean --include-state`。

## 全量提交约束 + 删除点位确认

> **规则定义在 [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 的"全量提交约束"和"删除点位前置检查协议"两节**——本文档只描述本 skill 的执行点和操作模式，不复述规则。
>
> **本 skill 的执行点**：Stage Config 的 config.apply 通过 [`references/config-apply.md`](references/config-apply.md) 的 A0（diff 检查）和 A1（local-config set 全量提交）落地这两条规则。即使 config.plan 阶段已确认过删除，A0 仍要再走一次（draft 可能被手改）。

### 增/改/删操作模式（本 skill 的实施手册）

假设远端当前配置为：
```json
{ "page": [{ "key": "board_abc123", ... }], "button": [{ "key": "button_x1", ... }, { "key": "button_x2", ... }] }
```

| 操作 | set 的 JSON 内容 | 说明 |
|------|-----------------|------|
| **新增** liteAppComponent | `{ "page": [原样保留], "button": [原样保留], "liteAppComponent": [新点位] }` | 必须带上 page 和 button，否则它们会被删除 |
| **修改** button_x1 的 name | `{ "page": [原样保留], "button": [{ "key": "button_x1", name已改, ... }, { "key": "button_x2", 原样 }] }` | 只改目标字段，其余字段和其他点位原样保留 |
| **删除** button_x2 | `{ "page": [原样保留], "button": [{ "key": "button_x1", 原样 }] }` | 从数组中移除目标项，其余原样保留 |
| **删除整个** button 类型 | `{ "page": [原样保留] }` | 不传 button 字段，远端 button 即被清除 |

## 代码溯源协议（CRITICAL · 根原则 1 无源即停 · 代码域应用 · Stage Code 阶段）

这个 skill 生成的代码**强依赖业务平台提供业务上下文及每个点位的特定 API**，凭经验拼接的代码能过 tsc 但运行时会跑废。所以进 code.apply 写代码前，code.plan 阶段要先走完四步溯源，顺序固定：

1. **点位标准能力 doc**（`references/point-types/<点位类型>/`）—— 每个点位的场景化标准能力，是最贴近用户需求的参考资料
2. **`@lark-project/js-sdk` 类型定义** —— 单纯只描述每个 API 的校准签名 / 参数 / 返回值
3. **用户提供的代码** —— 真实业务上下文
4. **飞书项目知识 MCP** —— 前三源没讲清业务语义（枚举含义、取值空间）时兜底查

任何一行 SDK 调用都要能指回上面某个源；找不到源就停下问用户，不要拿 MCP 片段硬拼。plan 完成后会产出代码溯源审计报告，apply 会脚本化校验，详见 [`references/code-plan.md`](references/code-plan.md) 的"代码溯源协议"章节。

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

## 输出产物

| 产物 | 说明 |
|------|------|
| 远端点位配置 | Stage Config 成功后，远端点位已更新到目标状态 |
| `point.config.local.json` | `update` 同步后刷新的本地镜像，供 Stage Code 消费 |
| entry 路径对应的代码文件 | Stage Code 在 CLI 生成的模板基础上填充功能代码 |
| `project.zip`（按需） | 整个工程打包，供用户下载本地调试 |

## 代码质量保障分层策略（Stage Code — code.verify）

| 层级 | 方式 | 触发条件 |
|------|------|---------|
| L1 预检 | `tsc --noEmit` 语法检查（约 10s） | code.verify 阶段执行 |
| L2 自动修复 | AI 读取错误 → 生成 patch → 重试（最多 2 轮） | L1 发现错误时 |
| L3 降级兜底 | 提供 zip 下载 + 本地调试指引；保留 `.lpm-cache/state.json` 和 `.lpm/`（这是流程断点，用户从本地修复后要靠它续跑，清了就回不来） | L2 仍失败时 |

## 本地调试引导（完成态交接）

Stage Code 的 code.verify 通过后，引导用户启动本地开发服务器：

```bash
npx @byted-meego/cli@builder start --auto
```

`--auto` 参数会自动根据 `getLocalConfig` 中的当前域名拼接调试 URL，并在浏览器中打开。

```
✅ 点位配置已推送远端 + 代码生成完成 + tsc 通过

下一步：启动本地调试
  运行：npx @byted-meego/cli@builder start --auto
  CLI 将自动打开浏览器调试页面

调试完成后可以：
  • 继续迭代：再次调用本 skill 调整点位或重新生成代码
  • 完善信息：/meegle-plugin-polish（准备发布前补名称/描述/分类）
  • 发布上线：/meegle-plugin-publish（发布到 Meegle 市场，不可逆，须用户显式确认）
```

## 关键设计原则

1. **两阶段绑定**：Stage Config 和 Stage Code 在本 skill 内是同一个决策单元，不允许 AI 在 Stage Config 完成后擅自停下不进 Stage Code（除非用户显式 `stage=config`）
2. **全量替换防护**：Stage Config 的 apply 前永远先 `get --remote` 拿基线，在基线上做局部改动
3. **代码溯源优先**：Stage Code 的 plan 必须走完四步溯源再写代码，找不到源就停
4. **停在本地就绪**：本 skill 收束在"本地可调试"，polish/publish 交给独立 skill，由用户显式触发
5. **断点续跑**：`.lpm-cache/state.json` 记录执行进度，中断后从精确断点恢复
