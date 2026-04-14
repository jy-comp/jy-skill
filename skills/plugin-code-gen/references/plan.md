# mode=plan：分析需求 + 规划功能实现

> `npx @byted-meego/cli@builder update` 已在 setup 阶段生成了各点位的模板代码。
> plan 阶段的目标是**规划用户真实需求的功能实现方案**，而非生成模板。

## P1：读取已有模板代码

从 `plugin.config.json` 的 `resources` 数组读取各点位的 entry 路径，然后**读取 entry 对应的模板代码文件**，了解 CLI 已生成的代码结构：

```json
{
  "resources": [
    { "id": "board_web_rj4v_m", "entry": "./src/features/board_web_rj4v_m/index.tsx" }
  ]
}
```

读取每个 entry 文件的内容，确认模板的导出方式、初始化逻辑和可用的 JSSDK API。

## P2：查询 JSSDK API 类型定义（CRITICAL — 防幻觉）

**禁止凭记忆编写 JSSDK 调用代码。** 必须先查询目标点位的 API 名称和 FeatureContext 字段。

工具选型遵循 [`../../meego-shared/SKILL.md`](../../meego-shared/SKILL.md) 的"工具职责划分"（飞书项目知识 MCP / CLI schema<!-- TODO(lark-project): 调试完成后补回：/ lark-project skill 三分法 --> 分法），在其之上补充本 skill 的一条细化原则：

> **飞书知识查能力，包定义查签名。** 飞书项目知识 MCP 适合回答"这个点位能做什么、API 怎么用、有哪些场景"；`node_modules/@lark-project/js-sdk/dist/types/index.d.ts` 是方法签名/参数/返回值的唯一权威，两者冲突时以包内类型定义为准。

### 查询流程

**P2.0 前置 — MCP 调用协议**

调 MCP（查能力）前后均按 `meego-shared/SKILL.md` 的"MCP State 文件"段处置（前置 gate + 后置反向自愈 + 强制阻塞话术）。本处不复制分派表。

**本 skill 专属后置降级约束**：若 state gate 通过后 MCP 仍无法给出有效代码示例（空结果 / 不相关片段），MUST 按本文 P3 的"代码无源即停"模板停下问用户（跳过某 API 占 TODO / 用户提供示例代码 / 换关键词），**禁止**仅凭 AI 经验脑补 `window.JSSDK.xxx` 调用。

**若 gate 通过，继续下方：**

1. **查能力**：用飞书项目知识 MCP 检索目标点位可用的 JSSDK API、场景、注意事项
2. **查签名**：读取 `node_modules/@lark-project/js-sdk/dist/types/index.d.ts`，以精确类型为准写代码，不可凭 MCP 示例推断参数结构
<!-- TODO(lark-project): 调试完成后启用
3. **查实例数据**：若代码中需要真实的工作项/视图/空间 ID 或字段值，通过 `lark-project` skill 查询，禁止从对话上下文或 schema 脑补
-->
3. **禁止**：跳过上述步骤直接写 `window.JSSDK.xxx` 调用

### 代码溯源协议（CRITICAL — 禁止从 schema 脑补代码）

与 URL 溯源协议同源思想：**AI 写的每行业务代码都必须能追溯到合法信息源**。这是为了堵住一个真实的失败模式——当 MCP 查询和 SDK 类型定义都没给出明确可用的实现时，AI 退化到**从 schema 字段名猜测 API 调用**，生成完全不可用的代码。

#### 合法的代码信息源（白名单）

| 来源 | 可信度 | 用途 |
|---|---|---|
| **飞书项目知识 MCP** 返回的明确示例代码 | ✅ 高 | 完整片段可直接借鉴 |
| **`@lark-project/js-sdk` 包内类型定义** | ✅ 高（最权威） | 方法签名、参数/返回值类型 |
| **用户在对话中明确写出的代码片段** | ✅ 高 | 用户贴的样例 |
| **当前工程已存在的代码（src/）** | ✅ 中 | 业务约定、命名风格 |

#### 非法的代码信息源（黑名单 — 严禁）

| 来源 | 为什么禁止 |
|---|---|
| **`plugin.config.json` schema 字段名脑补** | schema 是配置约束，不是 SDK 接口；字段名长得像 API 但语义完全不同 |
| **AI 内置的"常见 SDK 经验"类比** | 飞书项目 SDK 不是通用模式，类比常常错（如把 `getProps` 当 `getValue`）|
| **MCP 返回模糊片段后的"补全推断"** | 没拿到完整签名就拼凑 = 编造 |
| **从 TypeScript 类型反向推断"应该有"的方法** | 类型定义里没有的方法就是不存在 |
| **猜测的命名约定**（如 `getX` / `setX` 配对推断） | SDK 不一定遵循这套，常出现单向 API |

#### 检测到无源代码即停

AI 在生成任何 `window.JSSDK.xxx`、`containerModal.xxx`、`modal.xxx` 等 SDK 调用前，**MUST** 自检：

1. **能否在 MCP 返回结果中找到完整匹配的示例？**
2. **能否在 `js-sdk/dist/types/index.d.ts` 找到精确签名？**

两个问题都答 NO → **暂停、不写代码、立刻向用户求助**：

```
我无法在飞书项目知识 MCP 和 @lark-project/js-sdk 类型定义中找到
"<具体 API / 功能>" 的实现方式。

我可以选择：
A. 跳过这个功能，输出 TODO 注释占位（推荐——不写错的好过写错的）
B. 你提供示例代码或文档链接，我按你的指引实现
C. 你帮我确认目标 API 名（我可能找错了关键词）

禁止我自行猜测——猜错的代码会通过 ts 编译但运行时全废。
请告诉我选哪个，或直接给我线索。
```

#### 输出标注溯源（机器可审计）

与 URL 溯源类似，每段非平凡 SDK 调用代码必须在邻近注释中标注 source：

```ts
// source: js-sdk types/index.d.ts:Control.getCreateWorkItemFormItemValues
const values = await window.JSSDK.control.getCreateWorkItemFormItemValues([
  { key: 'name', type: 'text' },
]);

// source: 飞书项目知识 MCP / 控件场景示例
window.JSSDK.modal.open({ entry: 'detail_modal', width: 800 });

// source: 用户对话提供 @ 2026-04-14
const customFn = mySpecialHandler();
```

未标注 source 的 SDK 调用 → reviewer agent 视为疑似编造，要求作者补来源或删除。

#### 与下游防御的责任划分

| 层 | 防什么 |
|---|---|
| **plan 协议（本节）** | 写代码前禁止用 schema 脑补；找不到信息源必须停下问 |
| **代码 ts 编译** | 类型不存在直接报错（兜底）|
| **reviewer agent** | 扫描代码里的 `window.JSSDK.xxx` 调用，对照 js-sdk types 验证存在性 + 检查 source 注释 |
| **运行时** | 真实调用时崩溃（最后兜底，但代价高）|

**核心原则**：和 URL 一样——**没有合法信息源就停下来问用户，不存在"先写一个看起来合理的，跑不通再改"的中间状态**。

### 点位专属 MCP 检索线索（避免凭记忆组合 API）

不同点位有独立的属性体系和 API 组合姿势（易错但无法靠类型定义看出来），P2 阶段识别到下表场景时 **MUST** 用对应关键词调 `飞书项目知识 MCP` 拉取完整指南，再进入 P3：

| 场景识别线索 | 建议 MCP 检索关键词 | 重点覆盖 |
|---|---|---|
| 用户提到「轻应用」「轻应用组件」「搭建组件」「拖拽组件」，或 plugin.config.json 的 resources 里 entry 文件位于 `src/features/builder_comp_*/`、或点位类型为 `liteAppComponent` | `轻应用组件 LiteAppComponent 属性 getProps notify watch getPropFullConfig 输出属性` | 组件属性/输出属性/系统属性三组概念；10+8 种 propType 与值结构；API 决策树；数据源 spaceId→工作项类型→字段 查询流程；排期不支持；发布后类型不可改等限制 |

> 如果检索结果不足以覆盖场景，走 plan.md 的"检测到无源代码即停"流程向用户求助，**禁止**凭记忆/类比脑补属性类型和 API 组合。
>
> 已知易错点（E2E 评测发现）：
> - `customComponent.schedule` 全小写，不是 `Schedule`
> - `ConfigurationFeatureContext` 只有 `spaceId`，没有 `pluginId`
> - `InterceptFeatureContext` 的事件类型字段是 `eventType`，不是 `interceptEvent`
> - `ControlFeatureContext` 没有 `scene` 字段
> - `ButtonFeatureContext` 是 union type（4 种），不能直接解构，需先判断 scene 后 narrow
> - `Control` 没有 `getProps()` / `saveFieldValue()`（那是 `CustomField` 的方法）
> - `CustomField` 没有 `getCreateWorkItemFormItemValues()`（那是 `Control` 的方法）
> - `Modal.open()` 的 `entry` 参数是 resource ID（非文件路径），弹窗内用 `containerModal` 而非 `modal`

## P3：规划功能实现方案

结合用户的功能需求和已有的模板代码，规划具体的实现方案：

1. 分析用户需求需要哪些 UI 组件、数据交互、状态管理
2. 基于 P2 查询结果确认需要调用的 JSSDK API 和 FeatureContext 字段
3. 规划代码修改方案——在模板基础上需要新增/修改哪些部分

向用户确认实现方案后，进入 apply 阶段。
