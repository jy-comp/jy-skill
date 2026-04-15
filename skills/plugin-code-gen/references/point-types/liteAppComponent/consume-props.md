# 场景 C：消费管理员配置的属性（非 dataSource / dataSet）

> 前置：已读 `index.md` 的三源分工 / 冲突优先级 / propKey 来源约束。

**触发**：组件要暴露给管理员可配置的参数（标题、每页条数、默认选中视图、筛选条件等），插件代码读这些配置决定行为/样式。

## 1. 定义属性

组件需要一个或多个 property，每项有 `propKey` + `propType`（声明路径见 plan.md P0）。**propType 只能从下表枚举取**（源：MCP `LiteAppPropTemplateType` 枚举）：

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
| `dataSource` / `*WithField` | 见 `consume-data.md` |

**禁止编造不在表里的 propType**（如 `color` / `richtext` / `image` **不存在**）。
**propType 发布后不可改**——设计阶段必须一次问清用户。

## 2. 读属性值

```ts
const props = await window.JSSDK.liteAppComponent.getProps();
// props = { instanceId, layout, [propKey]: value, ... }

const title      = props['titleKey'];    // text → string
const pageSize   = props['pageSizeKey']; // number → number
const view       = props['viewKey'];     // viewSelect → { viewId, isMultiProject }
```

- `getProps()` 是 **async**，返回 Promise
- 返回对象除开发者声明的 propKey 外，还含两个系统字段：
  - `instanceId`: string — 组件实例 ID
  - `layout`: `{ width, height, positionX, positionY }` — 布局信息
- 返回值**不是响应式**——只是那一刻的快照，后续变化需 `watch`（见步骤 4）

## 3. 读属性的完整配置（select / multiSelect 渲染 options 时）

`select` / `multiSelect` 属性 brief 只有 `title` + `propType`，要渲染候选下拉必须拿 `options`——用 `getPropFullConfig(propKey)`：

```ts
const config = await window.JSSDK.liteAppComponent.getPropFullConfig('myselect');
// config 结构（仅 select / multiSelect 多出 options 字段）：
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
- **只有 `select` / `multiSelect`** 的 fullConfig 多出 `options`；其他 propType 的 fullConfig = getConfig 的 brief（title + propType 两字段）
- **`options[].label` 是 i18n 对象**（`{raw?, zh?, en?, ja?}`）不是字符串——渲染时取对应语言字段，或 raw 兜底
- `getConfig(propKeys?)` 一次返回多个 propKey 的 brief；`getPropFullConfig(propKey)` 只能单个 propKey 的完整配置
- `getPreset(propKeys?)` 是**编辑态**专用（目前只有 layout preset），运行时调拿不到值

## 4. 订阅属性变化（UI 跟随 props 更新）

监听管理员修改的 propKey（文本、数字、选择等），同步更新组件 UI：

```ts
// 监听全部 props，任一变化都回调
const off = await window.JSSDK.liteAppComponent.watch((nextProps) => {
  setTitle(nextProps['titleKey']);
  setPageSize(nextProps['pageSizeKey']);
});

// 只关心特定 key（其他 key 变化不触发）
const offScoped = await window.JSSDK.liteAppComponent.watch(
  (nextProps) => setTitle(nextProps['titleKey']),
  ['titleKey', 'layout'],
);

off?.();   // 卸载时取消订阅
```

**要点**：
- `watch(cb, watchKeys?)` 返回 `Promise<() => void>`（取消订阅函数，React useEffect 里记得调）
- `watchKeys` 不传 = 监听全部可监听的 key；传了就只在指定 key 变化才触发 callback
- callback 入参永远是**全量 nextProps**（即便限定了 watchKeys，也是给你整个 props 对象，自己挑字段用）

**如果属性里有 dataSource 类型**：单纯的 watch 拿不到数据源背后的工作项列表，需要在 callback 里主动再调 `getDataSourceResult` —— 见 `consume-data.md` 步骤 5。

## 常见坑

- propType 发布后不可改，设计阶段定死
- `getProps()` 是快照不是响应式，响应 UI 变化必配 `watch`
- value 结构严格：`select` 是 option **id** 不是 label；`viewSelect` 是对象不是字符串 id
- **dataSource 属性的特殊**：dataSource 也是 propKey，但读**数据**用 `getDataSourceResult`（见 `consume-data.md`）；`getProps()` 返回的 dataSource 值是订阅配置元信息，不是订阅后的工作项列表

## 未覆盖的问题（查 types / 问用户）

MCP 文档未载明以下内容，遇到时查 types 或问用户：

- 插件首次 render 前能否调 `getProps()`？是否要等某个生命周期 hook？
- 管理员未配置某个 propKey 时返回 `undefined` 还是类型默认值？

> 注：propType 枚举、option 结构、声明字段格式等"声明端的 schema"问题不归本 skill —— 见 plan.md P0 边界，那是 meego-point-config 的事。
