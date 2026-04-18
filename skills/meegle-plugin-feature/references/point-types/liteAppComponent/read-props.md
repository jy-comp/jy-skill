# 读组件属性（管理员在组件右侧表单配的入参）

> 前置：已读 `index.md` 的三源分工 / 冲突优先级 / propKey 来源约束。

## ToC（按维度判定结果跳读，不要全读）

- **§0** 四个 API 的职责分工（所有情况都先过一眼，定位 API 用法）
- **§1** 普通类型属性（text / number / select / ...）—— 命中"读组件属性"且 propType 是普通类型
- **§2** 数据源类型属性（dataSource）—— 命中"读组件属性"且 propType 是 dataSource
  - §2.1 `字段` toggle 决定能读到什么
  - §2.2 API 返回形态 + 字段读取位置 + propKey/fieldKey 区分
  - §2.3 fieldKey 定位（2a 速查 / 2b 查真实）
  - §2.4 交付兜底（让用户确认字段在工作项类型真实存在）
- **§3** 响应属性变化（watch）—— 命中"响应属性变化"维度读本节（含普通 + dataSource 两种处理）
- **§常见坑 / 未覆盖** —— 写完代码扫一眼防低级错

"组件属性"是管理员在组件右侧表单配的入参（图形化配置表单）。按 `prop_type` 分两大类，API 用法完全不同：

- **普通类型**（text / number / select / dateWithTime / workItemTypeSelect / ...）：管理员填什么，`getProps()` 直接给什么值
- **数据源类型**（dataSource）：管理员填的是"订阅哪个上游组件 / 按什么条件筛"，`getProps()` 只给这份订阅配置（不含数据），真数据必须调 `getDataSourceResult(propKey, pagination)` 才能拿到

## 0. 四个 API 的职责分工（先定位再写代码）

| API | 签名 | 返回 / 作用 |
|---|---|---|
| `getProps()` | 读组件当前 props 快照 | 返回 `{ instanceId, layout, [propKey]: value, ... }`。**普通类型**的 value = 值本身；**dataSource 类型**的 value = 订阅配置（不含数据） |
| `getPropFullConfig(propKey)` | 读某个 propKey 的完整配置 | 只用来读 `select` / `multiSelect` 属性的 `options` 候选；其他 propType 返回 = brief |
| `getDataSourceResult(propKey, pagination)` | 获取 dataSource 类型属性的真数据 | **dataSource 类型唯一的数据读取入口**；返回 `{ data, total, hasMore }`，`data` 是工作项实例数组 |
| `watch(callback, watchKeys?)` | 监听 props 对象值变化 | 回调入参是全量 `nextProps`（结构同 `getProps`）；**当变更触发时机用**，dataSource 类型要在回调内再调 `getDataSourceResult` 取新数据 |

**关键差异**：
- 拿普通类型值 → `getProps()[propKey]`
- 拿 dataSource 订阅配置 → `getProps()[propKey]`（静态）
- 拿 dataSource 真数据 → `getDataSourceResult(propKey, ...)`（唯一入口）
- 响应任一属性变化 → `watch(cb, watchKeys?)`，按类型在回调里重取

**常见误用**：
- 用 `getProps()` 想读 dataSource 的工作项列表 → 拿到的是订阅配置，不是数据
- 以为 watch 回调直接给 dataSource 数据 → 回调只给 props 对象，数据仍需在回调内调 `getDataSourceResult`
- 用 `getDataSourceResult` 当响应式刷新 → 它只是一次性读取，需配 `watch` 当触发时机

---

## 1. 普通类型属性（text / number / select / ...）

### 1.1 propType 枚举（源：MCP `LiteAppPropTemplateType`）

**propType 只能从下表取，不在表里就是不存在，禁止编造**（如 `color` / `richtext` / `image` **不存在**）：

| propType | 管理员配的值结构 |
|---|---|
| `text` | `string` |
| `number` | `number` |
| `boolean` | `boolean` |
| `select` | `string`（option id，非 label） |
| `multiSelect` | `string[]`（option id 数组） |
| `dateWithTime` | `number`（ms 时间戳） |
| `dateRange` | `{ start: number, end: number }`（时间戳） |
| `workItemTypeSelect` | `{ spaceId, workObjectId, fieldList? }` |
| `workItemInstance` | `{ spaceId, workObjectId, workItemId, fieldList? }` |
| `viewSelect` | `{ viewId: string, isMultiProject: boolean }` |
| `dataSource` | 见 §2 |

**propType 发布后不可改**——设计阶段必须一次问清用户。

### 1.2 读值

```ts
const props = await window.JSSDK.liteAppComponent.getProps();
// props = { instanceId, layout, [propKey]: value, ... }

const title    = props['titleKey'];     // text → string
const pageSize = props['pageSizeKey'];   // number → number
const view     = props['viewKey'];       // viewSelect → { viewId, isMultiProject }
```

- `getProps()` 是 **async**，返回 Promise
- 返回对象除开发者声明的 propKey 外，还含两个系统字段：
  - `instanceId`: string — 组件实例 ID
  - `layout`: `{ width, height, positionX, positionY }` — 布局信息
- 返回值**不是响应式**——跟随变化配 `watch`（见 §3）

### 1.3 select / multiSelect 拿 options 候选

`select` / `multiSelect` 属性的 brief 只有 `title` + `propType`，要渲染下拉候选必须拿 `options` —— 用 `getPropFullConfig(propKey)`：

```ts
const config = await window.JSSDK.liteAppComponent.getPropFullConfig('myselect');
// {
//   title: { raw?, zh?, en?, ja? },      // i18n 对象
//   propType: 'select' | 'multiSelect',
//   options: [
//     { value: 'b64kpxnbw', label: { raw: '程序员', zh: '', en: '', ja: '' } },
//     ...
//   ]
// }
```

关键点：
- **只有 `select` / `multiSelect`** 的 fullConfig 多出 `options`；其他 propType 的 fullConfig = brief（title + propType 两字段）
- **`options[].label` 是 i18n 对象**（`{raw?, zh?, en?, ja?}`）不是字符串——渲染时取对应语言字段，或 raw 兜底
- `getConfig(propKeys?)` 一次返回多个 propKey 的 brief；`getPropFullConfig(propKey)` 只能单个 propKey 的完整配置
- `getPreset(propKeys?)` 是**编辑态**专用（目前只有 layout preset），运行时调拿不到值

---

## 2. 数据源类型属性（dataSource）—— 最易错的类型

### 2.1 属性声明里的 `字段` toggle 决定能读到什么

在组件属性配置面板声明 dataSource 属性时，会看到一个 `字段` toggle（声明归 Stage Config，见 `config-plan.md`）。**toggle 状态决定 `getDataSourceResult` 返回的工作项实例里有没有字段数据**：

| `字段` toggle | getDataSourceResult 返回的 item | 能否读 `item.fields[fieldKey]` |
|---|---|---|
| **关** | 工作项基础信息（name / id 等） | 不能，`item.fields` 里不包含管理员选的字段，全 undefined |
| **开** | 基础信息 + 管理员在属性面板选中的字段 | 能，`item.fields[fieldKey]` 返回字段值 |

**动作**：
- 要显示 / 筛选 / 分组工作项字段 → **必须在 Stage Config 声明时开 `字段` toggle**
- 只需要工作项 ID / 名称 → 可以不开

> **术语对应**：UI 上的 `字段` toggle 在 `point.config.local.json` 的 dataSource property 里对应 `withField: true` 字段。用 jq 校验配置缺口时按这个字段名查（见 code-plan.md P0 "能力开关未启用"缺口模式）。

toggle 没开但代码想读字段 → 走 code-plan.md P0 的"能力开关未启用"缺口模式，回 Stage Config 补声明。

### 2.2 API 返回形态 + 字段读取位置

```ts
const { data, total, hasMore } =
  await window.JSSDK.liteAppComponent.getDataSourceResult(
    'items',                           // propKey：你声明的 dataSource 属性 key
    { pageNum: 1, pageSize: 50 },
  );

// data = 工作项实例数组
// 每个工作项实例的字段值在 item.fields[fieldKey]：
//   - fieldKey 来自上游数据源绑定的工作项类型定义（见 §2.3），不是你声明的 propKey
//   - `字段` toggle 开 → item.fields[fieldKey] 是字段值
//   - `字段` toggle 关 → item.fields[fieldKey] = undefined
data.forEach(item => {
  const priority = item.fields['priority'];      // 系统字段
  const channel  = item.fields['field_f5ed1b'];  // 自建字段（hash key）
});
```

**两套 key 归属完全不同，严格区分**：

| 命名空间 | 归属 | 例子 | 用在哪 |
|---|---|---|---|
| **propKey** | 你自己声明的组件属性 key（远端 properties/outputs） | `items` / `displayMode` | `getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)` |
| **fieldKey** | 上游推来的工作项实例的字段 key（工作项类型定义） | `priority` / `field_f5ed1b` | 只能用在 `item.fields[fieldKey]` |

**严格禁止**：
- `item.fields['items']`（把 propKey 当 fieldKey 用）→ 永远 undefined
- `getProps()['priority']`（把 fieldKey 当 propKey 读）→ 拿不到工作项字段值
- `getProps()[propKey].fields[...]`（以为 getProps 嵌套工作项数据）→ getProps 不返回工作项数据

### 2.3 fieldKey 定位（先试 2a 速查，匹配不上走 2b 查真实）

先知道**上游数据源绑的是哪个工作项类型**（问用户或从已有配置读），再定位该工作项类型上的 fieldKey。

#### 2a-1 系统字段 fieldKey 速查（用户口语 → 直接用）

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

#### 2a-2 字段类型 → value 结构（全集，解释 `item.fields[fieldKey]` 形态）

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

**未覆盖**：dataSource 字段若是 `select` / `multi-select` 等枚举类型，**如何拿到字段本身配置的 option 候选**（如 priority 的 P0/P1/P2 label）目前 doc 未覆盖——2a/2b 都拿不到（2b 走 lark-project 给字段清单，不含字段内部 options；`getPropFullConfig` 只能读**自己声明**的属性配置，不能读订阅进来的工作项字段）。遇到需要渲染选项候选时走"无源即停"问用户。

#### 2b 自定义字段 / 2a 匹配不上 / 用户说"都不对"

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

#### 2a vs 2b 怎么选（防 AI 瞎选 2a）

**强制走 2b 的场景**：
- 用户口语**没命中** 2a-1 速查表（业务自定义字段、未列入的其他系统字段）
- 用户描述有歧义（如"创建时间"可能是 `start_time` / `archiving_date` / 自建字段）

**能走 2a 的场景**：用户口语直接命中 2a-1 速查表某行 → 用对应 fieldKey；按 field type 查 2a-2 解释 value 结构。

### 2.4 交付兜底（让用户人工确认字段在工作项类型里真实存在）

代码交付时，MUST 在结尾附加一句话给用户：

> 如果调试时看不到数据 / 字段值是 undefined，除了"字段挂表单 / 真填值"之外，**还要先确认 `<fieldKey>` 这个字段在当前工作项类型里是否真实存在**——
> 请打开飞书项目空间，到对应**工作项类型的字段配置页**，按"对接标识"列搜一下 `<fieldKey>`：
> - **存在** → 回到"挂表单 / 填值"两步排查
> - **不存在** → 告诉我，我用 lark-project 重新查这个工作项类型的字段定义，对一遍 fieldKey

用户反馈"看不到"时，AI 的第一反应不是改代码，而是先让用户确认 fieldKey 是否真在该工作项类型里——大概率是字段定位环节不对（用错了 fieldKey、或字段属于其他工作项类型），不是代码逻辑问题。

---

## 3. 响应属性变化（watch）—— 所有 prop_type 的统一入口

管理员改属性配置（切换 dataSource 绑定、改字段选择、改普通参数等）后，组件默认不会自动刷新。`watch` 是**所有属性变化的统一响应入口**，按 prop_type 在回调里分别处理：

### 3.1 watch 的定位

> ⚠️ **watch 回调不携带 dataSource 的真数据**：回调入参 `nextProps` 结构同 `getProps`——对普通类型 prop，`nextProps[propKey]` 是新值；对 dataSource 类型 prop，`nextProps[propKey]` 仍是订阅配置（不是工作项数据）。
>
> 把 `watch` 当成"上游/配置有变化，该重新取数了"的**触发时机**。普通类型回调里直接读新值；dataSource 类型必须在回调内重调 `getDataSourceResult` 才能拿到新数据。

```ts
const off = await window.JSSDK.liteAppComponent.watch(async (nextProps) => {
  // 普通类型：nextProps[propKey] 就是最新值
  setTitle(nextProps['titleKey']);
  setPageSize(nextProps['pageSizeKey']);

  // dataSource 类型：nextProps['items'] 是订阅配置（不是数据）
  // 必须在回调内再调 getDataSourceResult 拉新数据
  const { data } = await window.JSSDK.liteAppComponent.getDataSourceResult(
    'items',
    { pageNum: 1, pageSize: 50 },
  );
  setItems(data);
});

off?.();   // 卸载时取消订阅（React useEffect cleanup）
```

### 3.2 要点

- `watch(cb, watchKeys?)` 返回 `Promise<() => void>`（取消订阅函数）
- `watchKeys` 参数形态：不传（监听全部）/ 单 propKey string（`'titleKey'`）/ propKey 数组（`['titleKey', 'items']`）
- 回调入参永远是**全量 nextProps**（即便限定了 watchKeys，也给整个 props 对象，自己挑字段用）
- **回调给的是 props 对象（同 getProps 结构）**：
  - 普通属性 → `nextProps[propKey]` 是新配置值
  - dataSource 属性 → `nextProps[propKey]` 是订阅配置（不是数据），数据走 `getDataSourceResult`
- 防抖：上游频繁推送时回调会高频触发，必要时在回调内节流再调 `getDataSourceResult`
- 卸载时一定调 `off()`，否则内存泄漏 + 重复刷新

### 3.3 典型生命周期

1. **mount** → 按属性类型初次取值
   - 普通类型：`const props = await getProps()` 一次性读
   - dataSource 类型：`getDataSourceResult(propKey, { pageNum: 1 })` 拉首屏
2. **mount** → `watch(cb, watchKeys?)` 注册变更触发订阅
3. **属性变化** → watch 回调触发
   - 普通类型：从 `nextProps[propKey]` 取新值
   - dataSource 类型：在回调内再调 `getDataSourceResult` 拉新数据
4. **翻页 / 用户主动 refresh** → 直接调 `getDataSourceResult`
5. **unmount** → `off()` 取消订阅

---

## 常见坑

- propType 发布后不可改，设计阶段定死
- `getProps()` 是快照不响应式，响应 UI 变化必配 `watch`
- 普通类型的 value 结构严格：`select` 是 option **id** 不是 label；`viewSelect` 是对象不是字符串
- **dataSource 属性三套 API 不要混**（见 §0 API 表）：
  - `getProps()` → 订阅配置（静态元信息）
  - `getDataSourceResult(propKey, ...)` → 真数据（唯一入口）
  - `watch(cb, propKey)` → 变更触发时机，不给数据
- `item.fields[fieldKey]` 的 fieldKey **来自上游工作项类型定义**，不是你声明的 propKey
- `字段` toggle 关的情况下想读字段 → 全 undefined；先回 Stage Config 开 toggle 再重新 update

## 未覆盖的问题（查 types / 问用户）

- 插件首次 render 前能否调 `getProps()`？是否要等某个生命周期 hook？
- 管理员未配置某个 propKey 时返回 `undefined` 还是类型默认值？

> 注：propType 枚举、option 结构、`字段` toggle 等"声明端的 schema"问题不归 Stage Code —— 见 [`../../code-plan.md`](../../code-plan.md) P0 边界，那是本 skill 的 Stage Config 的事。
