# 场景 A：消费数据源（读订阅到的工作项列表）

> 前置：已读 `index.md` 的三源分工 / 冲突优先级 / propKey 来源约束。

**触发**：用户需要读某个工作项列表、展示/筛选/分组字段、订阅上游组件数据。

> ## ⚠️ API 边界声明（进入本场景前必读）
>
> **三个 API 的职责分工（dataSource 场景下）**：
>
> | API | 签名 | 作用 |
> |---|---|---|
> | `getProps()` | 读取当前组件的入参（props 对象） | 一次性读 props 快照；dataSource 类型的 prop 值是订阅配置，不是工作项数据 |
> | `watch(callback, watchKeys)` | 监听 `getProps` 返回的 props 对象值变化 | 回调入参是变化后的 props 对象（同 `getProps` 结构）；**用作变更触发时机**，不携带数据源数据 |
> | `getDataSourceResult(propKey, pagination)` | 获取数据源类型实例属性值 | **唯一的数据源读取入口**：mount 首屏、watch 回调内重刷、翻页、用户主动 refresh |
>
> **关键差异**：
> - **拿 props / 配置元信息 → `getProps`**（静态快照）
> - **响应 props 变化 → `watch`**（回调给变化后的 props 对象，**不给数据源数据**；当触发时机用）
> - **拿数据源数据 → `getDataSourceResult`**（唯一入口；mount 初始化 / watch 回调内重刷 / 翻页）
>
> **常见误用**：
> - 用 `getProps()` 想读工作项列表 → 拿到的是 dataSource 的订阅配置，不是数据
> - 以为 watch 回调直接给工作项数据 → 回调只给 props 对象（同 `getProps` 结构），数据仍需在回调内调 `getDataSourceResult` 拉取
> - 用 `getDataSourceResult` 当响应式刷新 → 它只是一次性读取，需配 `watch` 当触发时机才能随上游变化刷新

## 1. 定义数据源属性
组件需要一个 `dataSource` 类型的 property（`propKey` 自定）。管理员搭建时把它绑定到上游组件 `notify` 的 dataSet 输出（= 订阅）。

> 声明规则归 Stage Config，详见 [`../../config-plan.md`](../../config-plan.md)；Stage Code 只检查声明是否齐全、不自己改声明（见 [`../../code-plan.md`](../../code-plan.md) P0）。

## 2. 定位要消费的工作项字段

> ⚠️ **语义边界（必读，常见混淆源）**：
> - 这里定位的"字段"是**上游推给你的工作项实例**上的 `fieldKey`（即 `item.fields[fieldKey]`），是在你在步骤 1 声明的 `dataSource` 实例列表中的字段
> - **字段完全来源于上游数据源对应的工作项类型**——插件侧决定不了工作项上有哪些字段，只能基于该工作项类型实际存在的字段去取值

判断路径：先知道**上游数据源绑的是哪个工作项类型**（问用户或从已有配置读），再定位要消费该工作项类型上的哪些字段。两类路径 **先试 2a 速查，匹配不上再走 2b 查真实**：

### 2a-1 系统字段 fieldKey 速查（用户口语 → 直接用）

工作项的内置系统字段，fieldKey 跨空间稳定。**只有命中下表才能直接用 fieldKey 不查空间**。value 结构按"字段类型"列查 2a-2：

| 用户口语 | fieldKey | 字段类型 |
|---|---|---|
| 名称 | `name` | `text` |
| 描述 | `description` | `multi-text` |
| 需求文档 | `wiki` | `link` |
| 创建者 | `owner` | `user`（单人） |
| 提出时间 | `start_time` | `date` |
| 完成日期 | `finish_time` | `date` |
| 关注人 | `watchers` | `multi-user` |
| 业务线 | `business` | 级联单选 |
| 优先级 | `priority` | `select` |
| 当前负责人 | `current_status_operator` | `multi-user` |
| 归档日期 | `archiving_date` | `date` |
| 工作项类型 | `work_item_type_key` | `select` |
| 所属空间 | `owned_project` | `text` |
| 排期 | `schedule` | `date_range` |
| 工作项 id | `work_item_id` | `number` |

**不在上表的 fieldKey 必须走 2b**——业务自定义字段（`field_xxxxxx` hash）+ 其他系统字段（如终止 `aborted` 等，全集查 MCP `字段与属性解析格式`）。

### 2a-2 字段类型 → value 结构（全集，解释 `item.fields[fieldKey]` 形态）

拿到任意 fieldKey 后，按它的字段类型解释 value。**类型不在表里就是不存在，禁止脑补**：

| type | 中文 | TS 类型 | value 结构 |
|---|---|---|---|
| `text` | 单行文本 | `string` | `"示例单行文本"` |
| `multi-text` | 多行文本 / 富文本 | `string` | HTML 字符串 |
| `number` | 数字 | `number` | `123.45` |
| `bool` | 布尔 | `boolean` | `true` / `false` |
| `date` | 日期/时间 | `number` | 毫秒时间戳 |
| `date_range`（`schedule`） | 日期范围 | `number[]` | `[开始时间戳, 结束时间戳]` |
| `select` | 单选菜单 | `string` | option key/value |
| `multi-select` | 多选菜单 | `string[]` | option key/value 数组 |
| `tree-select` | 树形单选 | `string` | 节点 key/value |
| `tree-multi-select` | 树形多选 | `string[]` | 节点 key/value 数组 |
| `radio` | 单选框 | `string` | option key/value |
| `checkbox_group` | 复选框组 | `string[]` | option key/value 数组 |
| `priority` | 优先级 | `number` / `string` | `1` 或 `"P0"` |
| `user` | 人员 | `string[]` | 用户标识数组 |
| `multi-user` | 多人 | `string[]` | 用户标识数组 |
| `role` | 角色 | `string[]` | 角色下用户列表 |
| `role_owners` | 角色负责人 | `object[]` | `[{role, owners: string[]}]` |
| `link` | 链接 | `string` | URL |
| `file` / `multi-file` | 附件 | `object[]` | `[{name, token, ...}]` |
| `link_cloud_doc` | 关联云文档 | `string` | 云文档链接/标识 |
| `compound_field` | 复合字段 | `object` | 子字段 kv |
| `workitem_related_select` | 关联工作项（单选） | `string` | 工作项 uuid/id |
| `workitem_related_multi_select` | 关联工作项（多选） | `string[]` | uuid/id 数组 |
| `vote-option` / `vote-boolean` | 投票 | `object` | `{value, voted}` |
| `signal` / `multi-signal` | 信号灯 | `string` / `object[]` | 状态 key 或对象数组 |

> **2a-1 vs 2a-2 不要混**：2a-1 是"中文 → fieldKey"映射；2a-2 是"类型 → value 结构"。**禁止用 2a-2 的 type 名当 fieldKey** —— `item.fields['text']` / `item.fields['select']` 永远 undefined。

**未覆盖**：消费数据源字段若是 `select` / `multi-select` 等枚举类型，**如何拿到字段本身配置的 option 候选**（如 priority 的 P0/P1/P2 label）目前 doc 未覆盖——2a/2b 都拿不到（2b 走 lark-project 给字段清单，不含字段内部 options；场景 C 的 `getPropFullConfig` 只能读**自己声明**的属性配置，不能读订阅进来的工作项字段）。遇到需要渲染选项候选时走"无源即停"问用户。

### 2b 自定义字段 / 2a 匹配不上 / 用户说"都不对"

走完整流程：
1. 问用户 spaceId（话术："告诉我要调试的飞书项目空间 ID（project_key），在空间内双击空间名即可复制"，禁止编造/禁止用 demo 值）
2. 用 `lark-project` skill 拉工作项类型列表（展示 workObjectId + 中文名让用户选）
3. 用 `lark-project` skill 查该工作项类型的字段列表，按业务需求列候选必须带 **label + key**：

```
候选字段：
- 优先级       (key: priority)          ← 系统字段
- 反馈渠道     (key: field_f5ed1b)      ← 自建字段，hash key
```

让用户点选 → 拿到确认的 `fieldKey`。

- 自定义字段 fieldKey 形态永远是 `field_xxxxxx`（hash），**空间级，不可跨空间硬编码**
- 禁止凭业务语义猜 `fieldKey`

### 2a vs 2b 怎么选（防 AI 瞎选 2a）

**强制走 2b 的场景**：
- 用户口语**没命中** 2a-1 速查表（业务自定义字段、未列入的其他系统字段）
- 用户描述有歧义（如"创建时间"可能是 `start_time` / `archiving_date` / 自建字段）

**能走 2a 的场景**：用户口语直接命中 2a-1 速查表某行 → 用对应 fieldKey；按 field type 查 2a-2 解释 value 结构。

## 3. 写代码消费

```ts
const { data, total, hasMore } =
  await window.JSSDK.liteAppComponent.getDataSourceResult(
    'items',                         // propKey：步骤 1 声明的 dataSource 属性
    { pageNum: 1, pageSize: 50 },
  );

// data = 上游推来的工作项实例数组
// item.fields[fieldKey] = 该工作项实例上的字段值；字段来自上游绑定的工作项类型定义，
// 不是你声明的组件属性。fieldKey 不能从 getProps() 获取。
data.forEach(item => {
  const priority = item.fields['priority'];        // 系统字段，即上面 2a-1 速查表的 priority fieldKey
  const channel  = item.fields['field_f5ed1b'];    // 自建字段，即上面 2a-1 速查表的 field_f5ed1b fieldKey
});
```

**两套 key 别混（归属完全不同）**：

| 命名空间 | 归属 | 例子 | 用在哪 |
|---|---|---|---|
| **propKey** | **你自己声明的组件属性** key（在远端 properties/outputs 里） | `items` / `displayMode` | `getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)` |
| **fieldKey** | **上游推来的工作项实例**的字段 key（由工作项类型定义） | `priority` / `field_f5ed1b` | 只能用在 `item.fields[fieldKey]` |

**严格禁止**：
- `item.fields['items']`（把 propKey 当 fieldKey 用）→ 永远 undefined
- `getProps()['priority']`（把 fieldKey 当 propKey 读）→ 拿不到工作项字段值，只会拿到同名组件属性（通常 undefined）
- `getProps()[propKey].fields[...]`（以为 getProps 里嵌套工作项数据）→ getProps 不返回工作项数据

## 5. 订阅数据源更新（响应式刷新）

只在 mount 时调一次 `getDataSourceResult` → 上游数据/配置变化（管理员改筛选、改绑定、调整消费字段等）后组件不会自动刷新。配 `watch` 作为**变更触发时机**——

> ⚠️ **watch 回调不给数据源数据**：`watch` 只监听组件属性（props）层面的变化，回调入参 `newProps` 结构**同 `getProps`**，是配置元信息快照。dataSource 类型的属性变化时，回调里 `newProps[dataSourceKey]` 依然是订阅配置元信息（指向哪个上游），**不是工作项数据**。
>
> 工作项数据的唯一来源永远是 `getDataSourceResult`。把 `watch` 当成"上游有变化，该重新取数了"的触发信号，在回调里重新调 `getDataSourceResult` 才能拿到新数据。

```ts
// watch 作为触发时机：回调触发 → 在回调内重新调 getDataSourceResult 拉新数据
const off = await window.JSSDK.liteAppComponent.watch(
  async (newProps) => {
    // newProps 结构同 getProps，是配置元信息快照，不含数据源数据
    // 回调本身的价值是"时机"：告诉你上游配置/订阅有变化，该重新取数了
    const { data } = await window.JSSDK.liteAppComponent.getDataSourceResult(
      'items',
      { pageNum: 1, pageSize: 50 },
    );
    setItems(data);
  },
  'items',   // dataSource propKey；也支持数组形式 ['items', 'titleKey']
);

off?.();   // 卸载时取消订阅（React useEffect cleanup）
```

**要点**：
- **watch 回调 = 时机，不是数据**：数据永远从 `getDataSourceResult` 拿，mount 初始化、watch 回调、翻页都走这一条路
- `watch` 第二参数支持**单个 propKey string** 或 **propKey 数组**两种形态
- 防抖：上游频繁推送时回调会高频触发，必要时在回调内加节流再调 `getDataSourceResult`
- 卸载时一定调 `off()`，否则会内存泄漏 + 重复刷新

**典型生命周期**：
1. **mount** → `getDataSourceResult(propKey, { pageNum: 1 })` 主动拉首屏
2. **mount** → `watch(cb, propKey)` 注册变更触发订阅
3. **上游变化** → watch 回调触发 → **在回调内再调 `getDataSourceResult`** 拉新数据
4. **翻页 / 用户主动 refresh** → 直接调 `getDataSourceResult`
5. **unmount** → `off()` 取消订阅

### 交付兜底（让用户人工确认字段在工作项类型里真实存在）

代码交付时，MUST 在结尾附加一句话给用户，方便他们排查问题：

> 如果调试时看不到数据 / 字段值是 undefined，除了上面"字段挂表单 / 真填值"之外，**还要先确认 `<fieldKey>` 这个字段在当前工作项类型里是否真实存在**——
> 请打开飞书项目空间，到对应**工作项类型的字段配置页**，按"对接标识"列搜一下 `<fieldKey>`：
> - **存在** → 回到上面的"挂表单 / 填值"两步排查
> - **不存在** → 告诉我，我用 lark-project 重新查这个工作项类型的字段定义，对一遍 fieldKey

用户反馈"看不到"时，AI 的第一反应不是改代码，而是先让用户确认 fieldKey 是否真在该工作项类型里——大概率是字段定位环节不对（用错了 fieldKey、或字段属于其他工作项类型），不是代码逻辑问题。

