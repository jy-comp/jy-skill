# mode=plan：分析需求 + 规划功能实现

> `npx @byted-meego/cli@builder update` 已在 setup 阶段生成了各点位的模板代码。
> plan 阶段的目标是**规划用户真实需求的功能实现方案**，而非生成模板。

## P0：点位配置的声明边界（CRITICAL — 本 skill 不管声明）

本 skill **只写代码**，不管 properties / outputs / 点位字段的**声明**——那是 `meego-point-config` skill 的职责。

- Phase 2.4 已跑 meego-point-config 后，远端点位配置就绪，本地快照：`plugin.temp.local-remote.json`（缺就跑 `npx @byted-meego/cli@builder local-config get > plugin.temp.local-remote.json`）
- 代码里的 `propKey`（用在 `getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)` 等位置）**必须是快照里已声明的 propKey**，不是 AI 自创
- **发现缺 propKey 或 propType 不合适** → 停下告诉用户"需要新增/修改属性 X，请先调 `/meego-point-config` 完成声明后再回来"。**禁止**自己编辑 `plugin.config.json` / **禁止**自己跑 `local-config set`

## P1：读取已有模板代码

从 `plugin.config.json` 的 `resources` 数组读取各点位的 entry 路径，Read 每个 entry 文件，了解 CLI 已生成的代码结构、导出方式、初始化逻辑。

## P2：查能力（按点位开发的真实动线）

### 2.1 定位点位类型

从 `plugin.config.json` 的 `resources[].type` 读出要开发的点位类型（如 `liteAppComponent` / `control` / `customField` 等），作为后续 2.2/2.3 的锚点。

### 2.2 Read 点位专属 doc（场景入口）

doc 是本点位的**流程/场景/跨 API 协同**指南，首先 Read 对应那一份：

| 点位类型 | doc 路径 |
|---|---|
| `liteAppComponent` | `references/point-types/liteAppComponent/` 目录：先 Read `index.md`，按用户需求 Read `consume-data.md` / `provide-data.md` / `consume-props.md` 中的一份或多份；**禁止预加载全部场景** |
| 其他点位 | 待补（当前版本暂无场景指南，跳到 2.4 兜底） |

### 2.3 三源分工（职责不重叠）

| 源 | 权威范围 | 用法 |
|---|---|---|
| **点位 doc** | 场景组合 / 流程 / 跨 API 协同 / 踩坑 | 主骨架，照抄流程步骤，不拆散重组 |
| **`@lark-project/js-sdk/dist/types/index.d.ts`** | TS 签名（存在性 / 参数 / 返回值） | 写代码前 MUST 校准 |
| **飞书项目知识 MCP** | 业务语义 / 枚举含义 / 参数取值空间 | doc 没讲清业务时查；调不通走"无源即停" |

**冲突优先级**：types > doc > MCP。signature 不一致（SDK 升级）→ 以 types 为准并提醒用户 doc 可能过期。

**禁止**：types 里没写的方法"应该有"反推、MCP 模糊片段补全、凭经验写 `window.JSSDK.xxx`。

### 2.4 doc 缺失的兜底

本点位无专属 doc 时，只靠 2.3 的 MCP + types 两源开发。若两源也拿不到可用线索 → 按 meego-shared "无源即停"停下问用户，`<具体内容>` 填 API 名或场景关键词。**禁止**凭经验直接写 `window.JSSDK.xxx`。

## P3：代码反模式清单（本 skill 专属黑名单）

禁止以下退化推断：

- ❌ 用 `plugin.config.json` 的 schema 字段名脑补 SDK 方法（schema 是配置约束，不是 SDK 接口）
- ❌ 从 TS 类型"应该有"反推（类型里没写的方法 = 不存在）
- ❌ MCP 模糊片段补全 / 命名约定猜测（如 `getX` 配 `setX`）
- ❌ 凭内置经验直接写 `window.JSSDK.xxx.xxx()`

每段 SDK 调用必须在邻近注释标 source（reviewer 可审计）：

```ts
// source: js-sdk types/index.d.ts:Control.getCreateWorkItemFormItemValues
const values = await window.JSSDK.control.getCreateWorkItemFormItemValues([...]);
```

来源取值：`js-sdk types/<路径:符号>` / `飞书项目知识 MCP / <场景>` / `point-types/<type>.md` / `用户对话 @ <日期>`。

## P4：已知易错点（E2E 评测发现，写代码前必扫一遍）

- `customComponent.schedule` 全小写，不是 `Schedule`
- `ConfigurationFeatureContext` 只有 `spaceId`，没有 `pluginId`
- `InterceptFeatureContext` 的事件类型字段是 `eventType`，不是 `interceptEvent`
- `ControlFeatureContext` 没有 `scene` 字段
- `ButtonFeatureContext` 是 union type（4 种），不能直接解构，需先判断 scene 后 narrow
- `Control` 没有 `getProps()` / `saveFieldValue()`（那是 `CustomField` 的方法）
- `CustomField` 没有 `getCreateWorkItemFormItemValues()`（那是 `Control` 的方法）
- `Modal.open()` 的 `entry` 参数是 resource ID（非文件路径），弹窗内用 `containerModal` 而非 `modal`

## P5：规划功能实现方案

1. 分析用户需求涉及的 UI 组件、数据交互、状态管理
2. 基于三源查询结果确认要用的 JSSDK API 和 FeatureContext 字段
3. 规划代码修改方案——在模板基础上需要新增/修改哪些部分

向用户确认实现方案后，进入 apply 阶段。
