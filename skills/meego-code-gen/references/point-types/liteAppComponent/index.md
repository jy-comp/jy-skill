# 轻应用组件（liteAppComponent）点位开发规范 — 索引

> **读法**：本 index 必读。读完后按用户需求 Read 对应场景文件，**不要预加载全部场景**。
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

## 场景索引

按用户需求选 Read 一个或多个场景文件（相对本文件的路径）：

| 场景 | 触发关键词 | 文件 |
|---|---|---|
| **场景 A** 消费数据源 | 读工作项列表、展示/筛选/分组字段、订阅上游组件数据 | `consume-data.md` |
| **场景 B** 提供数据源 | 联动下游组件、按筛选推工作项列表给下游 | `provide-data.md` |
| **场景 C** 消费配置属性 | 读管理员配的标题/数字/选择等可配置参数 | `consume-props.md` |

**组合场景**（用户既要消费又要提供 / 既消费又要可配置）：把对应的多个场景文件都 Read 进来。

---

## 共享前置（所有场景都要遵守）

- **propKey 来源**：`getProps()[propKey]` / `getDataSourceResult(propKey, ...)` / `notify(propKey, ...)` 里的 `propKey` **必须**是已声明的属性 key（声明路径见 plan.md P0），读 `plugin.temp.local-remote.json` 的 `properties[].propKey` / `outputs[].propKey` 确认；禁止编造
- **两套 key 别混**：propKey（开发者声明的属性 key）vs fieldKey（工作项字段 key）。具体见 `consume-data.md` 步骤 3 的命名空间表

## 附：非 API 值的溯源

| 值 | 合法来源 |
|---|---|
| spaceId / workObjectId / fieldKey | `lark-project` skill 查真实 或 用户粘贴 |
| plugin.config.json schema 字段 | CLI `schema` 命令输出 |

JSSDK API 签名 / 业务语义不在本表——见顶部三源分工。
