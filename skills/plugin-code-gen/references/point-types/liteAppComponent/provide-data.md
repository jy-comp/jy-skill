# 场景 B：提供被消费的数据源（推工作项列表给下游组件）

> 前置：已读 `index.md`；字段定位部分需先读 `consume-data.md` 步骤 2（2a/2b 两路）。

**触发**：用户需要联动下游组件按自己的筛选展示。

## 1. 定义输出属性
组件需要一个 `dataSet` 类型的 output（声明路径见 plan.md）。`notify(propKey, ...)` 的 `propKey` **就是这里声明的输出属性 key**——读工程根 `point.config.local.json` 的 `liteAppComponent[].outputs[].propKey` 确认，禁止编造。
⚠️ 输出属性**没有字段类型**——想让下游拿字段数据，只能推包含筛选条件的 moql，下游自己查。

## 2. 定位字段（复用 consume-data.md 步骤 2）
按 `consume-data.md` 步骤 2 的 **2a / 2b 两路**定位字段 → 拿到 `workObjectId` + 筛选用的 `fieldKey`。moql 的 `FROM` 和 `WHERE` 两个值都要真实。

## 3. 写代码推 moql

```ts
const { spaceId } = await window.JSSDK.liteAppComponent.getContext();

await window.JSSDK.liteAppComponent.notify('filteredItems', {
  moql: `SELECT \`work_item_id\` FROM \`${spaceId}\`.\`story\` WHERE \`field_f5ed1b\` IN ('用户反馈','技术支持反馈')`,
});
```

**moql 硬约束（错一条就跑不起来）**：
- MUST 以 `` SELECT `work_item_id` `` 开头，不能 select 其他列 / 不能 `SELECT *`
- spaceId / workObjectId / fieldKey 用**反引号**
- where 值用**单引号**
- 排期字段不能作为 where 条件

**为什么必须 moql 不能推数组**：下游"工作项视图"等一方组件的订阅协议就是 moql，推数组不认。

## 4. 联调验证
- 管理员在下游"数据源"下拉里看到**你的插件 > 输出属性名** → outputs 声明正确
- 下游订阅后能看到 notify 对应的工作项列表 → moql 写对
- 下游空白 / 报错 → 99% 是 moql 错（select 列、字段名、引号）
