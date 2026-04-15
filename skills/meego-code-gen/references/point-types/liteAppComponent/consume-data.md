# 场景 A：消费数据源（读订阅到的工作项列表）

> 前置：已读 `index.md` 的三源分工 / 冲突优先级 / propKey 来源约束。

**触发**：用户需要读某个工作项列表、展示/筛选/分组字段、订阅上游组件数据。

## 1. 定义数据源属性
组件需要一个 `dataSource` 类型的 property（`propKey` 自定）。管理员搭建时把它绑定到上游组件 `notify` 的 dataSet 输出（= 订阅）。

> 声明路径见 plan.md 的"点位配置的声明边界"段。本 skill 不直接声明。

## 2. 定位字段（系统字段 / 自定义字段两路）

工作项字段分两类，**先试 2a，匹配不上再走 2b**，可省去很多情况下的 spaceId/lark-project 调用。

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
| 排期 | `schedule` | `date_range`（**插件消费侧不暴露**，告知用户限制） |
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
    'items',                         // propKey：第 1 步声明的属性
    { pageNum: 1, pageSize: 50 },
  );

data.forEach(item => {
  const priority = item.fields['priority'];        // 内置字段
  const channel  = item.fields['field_f5ed1b'];    // 自建字段
});
```

**两套 key 别混**：

| 命名空间 | 例子 | 用在哪 |
|---|---|---|
| propKey | `items` / `displayMode` | `getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)` |
| fieldKey | `priority` / `field_f5ed1b` | `item.fields[fieldKey]` |

把 propKey 当 fieldKey 用（`item.fields['items']`）永远 undefined。

## 4. 提醒用户挂字段到表单（最常见踩坑）

写完代码 MUST 提醒用户：

> 选的字段 `XX (key: xxx)` 需要在飞书项目空间的**工作项类型表单配置**里挂上，并且调试工作项实际填了值——否则 `item.fields[xxx]` 会是 undefined。
> "字段定义存在" ≠ "表单在用"，未挂表单的字段不会出现在 fields 里。

undefined 排查顺序：fieldKey 拼写 → 字段是否挂到表单 → 工作项是否真填过值。
**禁止代码里静默兜底**，让 undefined 暴露出来给用户排查源头。

### 交付兜底（让用户人工确认字段在工作项类型里真实存在）

代码交付时，MUST 在结尾附加一句话给用户，方便他们排查问题：

> 如果调试时看不到数据 / 字段值是 undefined，除了上面"字段挂表单 / 真填值"之外，**还要先确认 `<fieldKey>` 这个字段在当前工作项类型里是否真实存在**——
> 请打开飞书项目空间，到对应**工作项类型的字段配置页**，按"对接标识"列搜一下 `<fieldKey>`：
> - **存在** → 回到上面的"挂表单 / 填值"两步排查
> - **不存在** → 告诉我，我用 lark-project 重新查这个工作项类型的字段定义，对一遍 fieldKey

用户反馈"看不到"时，AI 的第一反应不是改代码，而是先让用户确认 fieldKey 是否真在该工作项类型里——大概率是字段定位环节不对（用错了 fieldKey、或字段属于其他工作项类型），不是代码逻辑问题。

## 5. 订阅数据源变化

`watch(cb)` 回调入参是**全量 props**（dataSource 属性本身变化会给你新的 propValue），但**拿不到数据源背后的工作项列表**——背后数据变化要**主动再调** `getDataSourceResult`。标准骨架：

```ts
await window.JSSDK.liteAppComponent.watch(async (nextProps) => {
  // 判断是不是"数据源"属性发生变化（propKey 与步骤 1 声明的 dataSource 属性一致）
  if (Object.keys(nextProps).includes('items')) {
    const { data } = await window.JSSDK.liteAppComponent.getDataSourceResult(
      'items',
      { pageNum: 1, pageSize: 50 },
    );
    setData(data);
  }
});
```

**关键点**：
- 不能假设 watch 回调里就带着新的工作项数据——`nextProps['items']` 只是订阅**元信息**（比如指向哪个 dataSet 源），不是工作项数组本身
- 必须再调一次 `getDataSourceResult(propKey, ...)` 拿最新数据
- 判断 propKey 用 `Object.keys(nextProps).includes(propKey)` 而不是直接 `nextProps[propKey]`——属性值可能为 falsy，但"key 存在"说明这个属性被推送了新配置
