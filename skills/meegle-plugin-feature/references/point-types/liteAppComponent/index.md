# 轻应用组件（liteAppComponent）点位开发规范 — 索引

> **读法**：本 index 必读。读完后按用户需求判定命中哪几条维度，按命中条目 Read 对应文件/章节，**不要预加载全部**。
>
> **三源分工**（职责不重叠）：
> - **本 doc 系列** = 场景组合 / 流程 / 跨 API 协同 / 踩坑（主骨架，照抄流程）
> - **`@lark-project/js-sdk/dist/types/index.d.ts`** = TS 签名权威
> - **飞书项目知识 MCP** = 业务语义 / 枚举含义 / 参数取值空间
>
> **冲突优先级**：types > doc > MCP。签名不一致（SDK 升级）→ 以 types 为准，提醒用户 doc 可能过期。
>
> 点位类型 `liteAppComponent` 在 schema 里记作 `builder_comp`，用户口语里只说"轻应用 / 轻应用组件"——识别需求以口语为准。

---

## 核心概念（30 秒建立心智模型）

轻应用组件在管理员搭建时有两类属性 + 一个响应机制：

- **组件属性（properties）**：管理员在组件右侧配置表单填的入参，按 `prop_type` 分两大类
  - **普通类型**（text / number / select / ...）：填什么读什么，`getProps()` 直接给值
  - **数据源类型**（dataSource）：填的是"订阅哪个上游组件"，`getProps()` 只给订阅配置，真数据要走 `getDataSourceResult`；若声明时开了 `字段` toggle，工作项实例还会带上管理员选的字段数据（`item.fields[fieldKey]`）
- **输出属性（outputs）**：当前组件对外暴露的数据，别的组件（一方组件或同插件其他组件）的管理员可以把它作为自己组件的 dataSource 来订阅；代码里用 `notify(propKey, data)` 推
- **响应机制（watch）**：任一 prop 变化时触发回调，按 prop_type 分别处理（普通类型读新值；dataSource 类型在回调里重调 `getDataSourceResult` 取新数据）

**组件间联动原理**：上游组件通过 outputs + notify 推数据 → 下游组件用 dataSource 属性订阅 + watch 当触发时机 + `getDataSourceResult` 重取。同一插件内的多组件、跨插件的组件都可以通过这条链路联动。

---

## 三条操作维度（按需求独立判定，可组合）

本点位开发的动作归到三条维度，每条独立判断"用不用"。**由 AI 自判命中哪些，不要问用户。**

| 维度 | 做什么 | 关键 API | 要读哪里 |
|---|---|---|---|
| **读组件属性**（入参） | 读管理员在组件右侧表单配的值（普通 / dataSource 都算） | `getProps()` / `getDataSourceResult()` / `getPropFullConfig()` | `read-props.md` 按子章节挑 |
| **推输出属性**（联动） | 把数据暴露给下游/其他组件订阅 | 声明 outputs + `notify(propKey, data)` | `write-outputs.md` |
| **响应属性变化**（时机） | 任一属性变动时触发回调 | `watch(cb, watchKeys?)` | **`read-props.md` 的 §3 小节**（不是独立文件） |

⚠️ 三条维度对应 **2 个物理文件**：`read-props.md`（涵盖"读组件属性" + "响应属性变化"两条） + `write-outputs.md`（涵盖"推输出属性"）。不要把 "§3" 当成文件名去 Glob。

**判定流程**（每条独立回答 Y/N，AI 自判不问用户）：

1. 用户需求要读组件自己的属性？→ Y 命中 "读组件属性" → Read `read-props.md`
2. 用户需求要让数据暴露给别的组件用？→ Y 命中 "推输出属性" → Read `write-outputs.md`
3. 用户需求要随属性变动响应（上游数据变 / 管理员改配置后 UI 跟随）？→ Y 命中 "响应属性变化" → Read `read-props.md` §3 小节

判定结果可能是 1 条、2 条或 3 条都命中——按命中条数 Read 对应文件/章节，不要预加载全部。

---

## 需求 → 维度 的典型映射（帮 AI 把真实需求对号入座）

判定原则（第三列"响应属性变化"如何判）：
- **管理员的属性配置是静态（搭建完不改）且无上游推送** → 不命中（✗）
- **管理员会在运行期改配置需要 UI 实时跟随 / 上游 dataSource 会持续推新数据 / 筛选条件变要触发 notify** → 命中（✓）

| 用户真实需求 | 读组件属性 | 推输出属性 | 响应属性变化 |
|---|---|---|---|
| "展示一批工作项列表" | ✓（dataSource 类型） | ✗ | ✓（上游 dataSource 会变） |
| "展示管理员填的标题 + 基于它查工作项" | ✓（普通 text + dataSource 组合） | ✗ | ✓（管理员改标题后 UI 要跟随） |
| "做一个筛选器推给下游工作项视图" | ✓（普通类型读筛选条件） | ✓（notify 筛选结果 moql） | ✓（筛选条件变化要重新 notify） |
| "读管理员配的标题就展示" | ✓（普通 text） | ✗ | ✗（纯静态，一次读完） |
| "上游推一批工作项，我二次加工后推给下游" | ✓（dataSource） | ✓（notify 处理后的结果） | ✓ |
| "组件里放多个小配置项供管理员填，读完就展示" | ✓（多个普通类型） | ✗ | ✗（纯静态） |
| "组件里放多个小配置项，管理员改了要立即看到效果" | ✓（多个普通类型） | ✗ | ✓（管理员改配置 UI 要跟随） |

---

## 共享前置（所有维度都要遵守）

- **propKey 来源**：`getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)` / `watch(cb, propKey)` 里的 `propKey` **必须**是已声明的属性 key（声明规则见 [`../../config-plan.md`](../../config-plan.md) 的"声明边界"；代码侧查缺口见 [`../../code-plan.md`](../../code-plan.md) P0），读工程根 `point.config.local.json` 的 `liteAppComponent[].properties[].propKey` / `liteAppComponent[].outputs[].propKey` 确认；禁止编造
- **两套 key 别混**：propKey（开发者声明的属性 key）vs fieldKey（工作项字段 key，仅 dataSource 场景出现）。详见 `read-props.md §2.2`

## 附：非 API 值的溯源

| 值 | 合法来源 |
|---|---|
| spaceId / workObjectId / fieldKey | `lark-project` skill 查真实 或 用户粘贴 |
| plugin.config.json schema 字段 | CLI `schema` 命令输出 |

JSSDK API 签名 / 业务语义不在本表——见顶部三源分工。
