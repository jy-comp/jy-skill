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

**Step 1 — 点位标准能力 doc（开发起点）**

点位 doc 不是可选参考材料，是这个点位上**已知踩坑的场景集合**和**业务侧强约束的落地说明**——哪些场景必须靠特定 API 组合才能跑通、哪些 return 形态看 types 签名推不出来、哪些 propKey/fieldKey 归属容易混、哪些订阅要配合 watch 当触发时机才能响应。这些都是真实踩出来并沉淀到 doc 的，跳过 doc 直接走代码/MCP 路线，基本就是在重走别人踩过的坑：tsc 能过、运行时必炸。

所以 **P2 的顺序是固定的，不能调**：

Step 1（doc）→ Step 2（types）→ Step 3（MCP）→ 进 P3 写代码

**从 Step 1 开始的第一个动作**：按点位找 doc 入口 Read `index.md`，**由 AI 自判**（不要问用户）命中哪几条操作维度，产出一份「维度判定表」作为"AI 真的读了 doc" + "后面围绕哪些维度展开"的锚点。

**Phase 1 判定**（读 index.md 前就能写出来，不带 §章节号 / 不带 propType 细节）：

```
本次需求维度判定（liteAppComponent）
────────────────
用户需求（AI 复述）：<一句话>

[ ] 读组件属性：用户需求是否涉及"读组件自己被管理员配的任一值"？
[ ] 推输出属性：用户需求是否涉及"把数据暴露给下游/其他组件订阅"？
[ ] 响应属性变化：用户需求是否涉及"上游数据或管理员配置变了要实时响应"？
```

判定规则（按行为目的判，不依赖字面关键词）：
- 要读组件自己的属性（任一 prop_type）→ 命中"读组件属性"
- 要把数据暴露给下游/其他组件订阅 → 命中"推输出属性"
- 要随属性变动（上游推送 / 管理员改配置）响应 → 命中"响应属性变化"

一个需求常常命中多条（如"展示上游工作项列表" = 读组件属性[dataSource] + 响应属性变化）。维度→文件映射以及典型需求→维度的映射见 `index.md` 的"三条操作维度"和"需求→维度 的典型映射"两节。按命中条目 Read 对应文件/章节，不要预加载全部。

**Phase 2 补全**（Read 完对应 doc 章节后回填）：

**Phase 2 只补全 Phase 1 命中 `[x]` 的维度；未命中的维度直接省略，不占位、不留 `[ ]`。**

```
（仅示意：命中 3 条全勾时的样子）
[✓] 读组件属性
    · propType 具体值：<从 read-props.md §1.1 表里挑，不在表里不存在>
    · dataSource 是否要读字段数据（`字段` toggle 状态）：<是 / 否 / N/A>
    · 引用章节：read-props.md §<实际读到的章节号>
[✓] 推输出属性
    · 引用章节：write-outputs.md §<章节号>
[✓] 响应属性变化
    · 引用章节：read-props.md §3
```

**部分命中示例**（仅命中"读组件属性"，另外两条未命中）：

```
[✓] 读组件属性
    · propType 具体值：text
    · 引用章节：read-props.md §1
```

贴完 Phase 2 才能进 Step 2。读 doc 过程中判断：用户想要的功能是不是这个点位标准能力覆盖的范围？代码能写但需要点位配置侧的开关/声明支持（典型如消费数据源字段要开 `字段` toggle），回流到 P0 的缺口模式处理，不要假设"配置默认开着"。

| 点位类型 | doc 路径 |
|---|---|
| `liteAppComponent` | `references/point-types/liteAppComponent/` 目录：先 Read `index.md`，按判定结果 Read `read-props.md` / `write-outputs.md` 中需要的章节；注意"响应属性变化"维度对应 `read-props.md §3`（同文件的小节，不是独立文件），三维度只有两个物理文件 |
| 其他点位 | 当前版本暂无，跳到 Step 4 兜底 |

**Step 2 — 按 doc 指向查 `@lark-project/js-sdk/dist/types/index.d.ts`**

types 是 API 签名（存在性 / 参数 / 返回值）的唯一权威。types 里没写的方法 = 不存在——不要从"既然有 read 应该有 write"反推，也不要从 `plugin.config.json` 的 schema 字段名推 SDK 方法（schema 是配置约束，不是 SDK 接口）。

**Step 3 — 仅当 doc + types 没覆盖业务语义时查飞书项目知识 MCP**

MCP 只负责业务语义：枚举含义、参数取值空间、业务概念解释。**API 签名走 types、API 组合走 doc、propKey/fieldKey 归属走 doc**——MCP 对这三类问题只会返回碎片，拼起来像对其实没对。

**MCP 缓存协议**：返回后立即 Write 到 `.lpm-cache/mcp/<slug>.md`，回复只给路径 + ≤100 字摘要（原文回流会炸 context）；同一查询先 Read 缓存，7 天以上视为过期重拉。CLI 在 create/init 时自动把 `.lpm-cache/` 写入 `.gitignore`，并在 `local-config set` / `update` / `publish` 成功后按子目录清理，AI 只负责写入、不负责清理。

**Step 4 — doc 缺失的兜底**

当点位无专属 doc 时，只靠 types + MCP 两源开发。两源都拿不到线索 → 走 meegle-plugin-shared 的"无源即停"停下问用户，不要凭经验直接写 `window.JSSDK.xxx`——这类猜测能过 tsc 但运行时全废。

### 2.3 冲突裁决（两源矛盾时用）

**优先级**：types > doc > MCP。signature 不一致（SDK 升级）→ 以 types 为准并提醒用户 doc 可能过期。

跳过 doc 直接拼 API 是最常见的失败模式，踩过的坑：**把 propKey 当 fieldKey 用 / `getDataSourceResult` 忘了配 watch 刷新 / `notify` 推数组而非 moql**——这些 types 签名完全合法、tsc 检查全过、运行时全挂。

## P3：源标注（reviewer 可审计）

每段 SDK 调用在邻近注释标 source：

```ts
// source: point-types/liteAppComponent/read-props.md §2.2 —— getDataSourceResult 返回 { data, total, hasMore }，字段在 item.fields[fieldKey]
const { data } = await window.JSSDK.liteAppComponent.getDataSourceResult('items', { pageNum: 1, pageSize: 50 });

// source: js-sdk types/index.d.ts:Control.getCreateWorkItemFormItemValues
const values = await window.JSSDK.control.getCreateWorkItemFormItemValues([...]);
```

来源取值：`point-types/<type>/<file>.md §<章节>` / `js-sdk types/<路径:符号>` / `飞书项目知识 MCP / <场景>` / `用户对话 @ <日期>`。

**当调用涉及以下两种情况时，source 必须指向点位 doc 的具体 § 章节**——types 路径不接受，因为 types 不约束 return 嵌套形态也不讲组合顺序：

- **解构 return 形态**：例如 `const { data, total } = await getDataSourceResult(...)` 里对 `data` 的字段/形态有依赖
- **多个 API 有先后组合顺序**：例如 `watch(cb)` 的回调内再调 `getDataSourceResult`，或 `getProps` + `watch` 的先后依赖

若对应调用找不到 doc § 章节当 source → 说明 Step 1 的场景匹配没落地/doc 没读到位，回到 P2 Step 1 补上场景匹配 + Read 对应章节再写代码。

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
