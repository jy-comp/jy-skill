# mode=plan：分析需求 + 规划功能实现

> `npx @byted-meego/cli@builder update` 已在 setup 阶段生成了各点位的模板代码。
> plan 阶段的目标是**规划用户真实需求的功能实现方案**，而非生成模板。

## P0：点位配置的声明边界（CRITICAL · Stage Config / Stage Code 职责边界 · 防越权写声明）

Stage Code **只写代码方案**，properties / outputs / 点位字段 / 能力开关的**声明**归 Stage Config 管。

plan 阶段本质是"**理解用户意图 → 匹配点位能力能否实现**"——读 doc 时判断当前点位提供的能力能不能覆盖用户要的功能，这个过程常常会**反向发现上游配置缺口**：代码能写出来，但没有对应的点位配置支持时运行不起来。发现缺口的第一时间停下，回到本 skill 的 Stage Config 补齐配置再回到 Stage Code 继续，不要自己跑 `local-config set`，也不要直接改 `plugin.config.json`（见 [`../../meegle-plugin-shared/SKILL.md`](../../meegle-plugin-shared/SKILL.md#安全规则)）。

**前置数据**：Stage Config 已跑完 config.apply 并经 `update --source-type=remote` 同步，**点位配置的权威快照是工程根目录的 `point.config.local.json`**（CLI 在 `local-config set` 成功时写入，`update` 拉取时刷新；`.lpm-cache/config/remote.json` 在 `set` 成功时已被 CLI 删除，不要再找）。若 `point.config.local.json` 缺失或过期，重新跑 `npx @byted-meego/cli@builder update` 同步一次即可。

**常见缺口模式**（见一个就停下、回到 Stage Config 补配置再回来，不要自己补）：

| 缺口类型 | 具体表现 | 典型场景 |
|---|---|---|
| 属性未声明 | 代码里用到的 `propKey`（`getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)`）在 `point.config.local.json` 里找不到 | 轻应用组件需要消费/提供某个配置，但 `properties` 数组里没这项 |
| 属性类型不对 | propKey 在，但 `prop_type` 与代码用法不匹配（如想当 `dataSource` 消费，实际声明成 `text`） | 属性类型选错，API 调用无法生效 |
| 能力开关未启用 | 快照里某个属性的功能开关没开（如 `dataSource` 属性缺 `withField: true`，导致无法按字段级消费数据源） | liteAppComponent 想消费数据源里的具体字段，代码走 `withField` 但配置没开 |
| 订阅/事件未声明 | 代码想监听某类事件，但 `listen_event` 里没声明对应 `event_type` | 拦截器 / 事件监听型点位代码走通了但不会被触发 |
| 子字段未声明 | customField 代码用到某个 subfield，快照里没有 | 字段模板缺 subfield 声明 |

判断流程：**不要 Read 整个 `point.config.local.json` 进上下文**（点位多时 JSON 很大）。用 jq 按要判断的东西按需取：

```bash
# 某点位某 propKey 的完整声明（propKey 缺失 / prop_type 不对 / 能力开关未开 都能一眼看出来）
jq --arg t "liteAppComponent" --arg k "<点位key>" --arg p "<propKey>" \
  '.[$t][] | select(.key==$k) | .properties[] | select(.prop_key==$p)' \
  point.config.local.json

# 某点位声明了哪些 propKey（想确认 prop 是否存在时）
jq --arg t "liteAppComponent" --arg k "<点位key>" \
  '.[$t][] | select(.key==$k) | .properties | map(.prop_key)' \
  point.config.local.json

# 某类型下某点位的整张声明（仅在需要完整视图时）
jq --arg t "liteAppComponent" --arg k "<点位key>" \
  '.[$t][] | select(.key==$k)' point.config.local.json
```

**代码里的 `propKey` 必须是 `point.config.local.json` 里已声明的，不是 AI 自创。**

## P1：读取要实现代码的 entry 文件

用 jq 从 `plugin.config.json` 取 entry 路径（不要 Read 整份 `plugin.config.json`）：

```bash
jq -r '.resources[] | "\(.id)\t\(.entry)"' plugin.config.json
```

然后 Read 每个 entry 文件（单文件体量小，Read 整份 OK），了解 CLI 已生成的代码结构、导出方式、初始化逻辑。

## P2：查能力（按点位开发的真实动线）

### 2.1 定位点位类型

用 jq 从 `plugin.config.json` 取本次要开发的点位类型：

```bash
jq -r '.resources[].type' plugin.config.json | sort -u
```

作为后续 2.2 的锚点。

### 2.2 查询顺序

doc → types → MCP → 无源即停。**doc 是主源，types 和 MCP 是定义层补充**：doc 同时回答"**场景能力 + 用哪些 API + 怎么组合 API**"三件事；types 只讲 API 签名，MCP 讲产品语义 + API 定义说明——这两者帮你把 doc 里看到的业务名词和 API 用法对号入座，不帮你选型、不讲组合顺序。

**Step 1 — 点位标准能力 doc（有就先读完再进下一步）**

doc 是业务侧给出的本点位**标准能力说明**：讲这个点位能做什么场景、要调哪些 API、API 之间的组合顺序、propKey vs fieldKey 归属、订阅协议。读 doc 同时要判断：用户想要的功能**是不是这个点位标准能力覆盖的范围**？如果代码能写但需要点位配置侧的某个开关/声明支持（典型如消费数据源字段要 `withField`），回流到 P0 的缺口模式处理，不要假设"配置默认开着"。

| 点位类型 | doc 路径 |
|---|---|
| `liteAppComponent` | `references/point-types/liteAppComponent/` 目录：先 Read `index.md`，按用户需求 Read `consume-data.md` / `provide-data.md` / `consume-props.md` 中的一份或多份（预加载全部场景浪费 context，按需读） |
| 其他点位 | 当前版本暂无，跳到 Step 4 兜底 |

**Step 2 — 按 doc 指向查 `@lark-project/js-sdk/dist/types/index.d.ts`**

types 是 API 签名（存在性 / 参数 / 返回值）的唯一权威。types 里没写的方法 = 不存在——不要从"既然有 read 应该有 write"反推，也不要从 `plugin.config.json` 的 schema 字段名推 SDK 方法（schema 是配置约束，不是 SDK 接口）。

**Step 3 — 仅当 doc + types 没覆盖业务语义时查飞书项目知识 MCP**

MCP 只负责业务语义：枚举含义、参数取值空间、业务概念解释。**API 签名走 types、API 组合走 doc、propKey/fieldKey 归属走 doc**——MCP 对这三类问题只会返回碎片，拼起来像对其实没对。

**MCP 缓存协议**：返回后立即 Write 到 `.lpm-cache/mcp/<slug>.md`，回复只给路径 + ≤100 字摘要（原文回流会炸 context）；同一查询先 Read 缓存，7 天以上视为过期重拉。CLI 在 create/init 时自动把 `.lpm-cache/` 写入 `.gitignore`，并在 `local-config set` / `update` / `publish` 成功后按子目录清理，AI 只负责写入、不负责清理。

**Step 4 — doc 缺失的兜底**

本点位无专属 doc 时，只靠 types + MCP 两源开发。两源都拿不到线索 → 走 meegle-plugin-shared 的"无源即停"停下问用户，不要凭经验直接写 `window.JSSDK.xxx`——这类猜测能过 tsc 但运行时全废。

### 2.3 冲突裁决（两源矛盾时用）

**优先级**：types > doc > MCP。signature 不一致（SDK 升级）→ 以 types 为准并提醒用户 doc 可能过期。

跳过 doc 直接拼 API 是最常见的失败模式，踩过的坑：**把 propKey 当 fieldKey 用 / `getDataSourceResult` 忘了配 watch 刷新 / `notify` 推数组而非 moql**——这些 types 签名完全合法、tsc 检查全过、运行时全挂。

## P3：源标注（reviewer 可审计）

每段 SDK 调用在邻近注释标 source：

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

**读取用户原始需求**：被 meegle-plugin-workflow 编排时从 `.lpm-cache/state.json` 的 `context.originalRequirement` 读（Phase 0 逐字录音，不受会话压缩影响）；独立调用时从对话上下文读。

结合原始需求 + P2 查到的点位能力，做方案设计：

1. 分析用户需求涉及的 UI 组件、数据交互、状态管理
2. 基于三源查询结果确认要用的 JSSDK API 和 FeatureContext 字段
3. 对照 P0 缺口模式检查是否有配置缺口（若有 → 回到 Stage Config 补齐）
4. 规划代码修改方案——在模板基础上需要新增/修改哪些部分

向用户确认实现方案后，进入 apply 阶段。
