# mode=plan：分析需求 + 规划功能实现

> `npx @byted-meego/cli@builder update` 已在 setup 阶段生成了各点位的模板代码。
> plan 阶段的目标是**规划用户真实需求的功能实现方案**，而非生成模板。

## P1：读取已有模板代码

从 `plugin.config.json` 的 `resources` 数组读取各点位的 entry 路径，然后**读取 entry 对应的模板代码文件**，了解 CLI 已生成的代码结构：

```json
{
  "resources": [
    { "id": "board_web_rj4v_m", "entry": "./src/features/board_web_rj4v_m/index.tsx" }
  ]
}
```

读取每个 entry 文件的内容，确认模板的导出方式、初始化逻辑和可用的 JSSDK API。

## P2：查询 JSSDK API 类型定义（CRITICAL — 防幻觉）

**禁止凭记忆编写 JSSDK 调用代码。** 必须先查询目标点位的 API 名称和 FeatureContext 字段。

### 两个信息源及其定位

| 信息源 | 适合查什么 | 权威性 |
|--------|-----------|--------|
| **飞书项目知识 MCP** | 有哪些 API 可用、使用场景、业务概念、最佳实践、示例片段 | 概念丰富、场景完整，适合**发现能力**（"这个点位能做什么"） |
| **`node_modules/@lark-project/js-sdk/dist/types/index.d.ts`** | 方法签名、参数类型、返回值类型、泛型约束、union type 结构 | **类型定义的唯一权威**，传参和返回值以此为准 |

> **原则：飞书知识查能力，包定义查签名。** 两者冲突时以 `@lark-project/js-sdk` 包内类型为准。

### 查询流程

1. **查能力（飞书知识 MCP）**：通过飞书项目知识 MCP 查询目标点位有哪些可用的 JSSDK API、使用场景和注意事项
   - 如果当前环境未配置飞书项目知识 MCP，**提示用户前往 `{siteDomain}/openapp/mcp/docs` 获取 MCP 配置 URL 并完成配置**（`siteDomain` 从 `./plugin.config.json` 的 `siteDomain` 字段读取）
2. **查签名（js-sdk 包）**：读取 `node_modules/@lark-project/js-sdk/dist/types/index.d.ts`，确认方法的**精确参数类型和返回值类型**
   - 必须以包内类型定义为准编写代码，不可仅凭 MCP 返回的示例推断参数结构
3. **禁止**：跳过上述两步直接编写 `window.JSSDK.xxx` 调用代码

> 已知易错点（E2E 评测发现）：
> - `customComponent.schedule` 全小写，不是 `Schedule`
> - `ConfigurationFeatureContext` 只有 `spaceId`，没有 `pluginId`
> - `InterceptFeatureContext` 的事件类型字段是 `eventType`，不是 `interceptEvent`
> - `ControlFeatureContext` 没有 `scene` 字段
> - `ButtonFeatureContext` 是 union type（4 种），不能直接解构，需先判断 scene 后 narrow
> - `Control` 没有 `getProps()` / `saveFieldValue()`（那是 `CustomField` 的方法）
> - `CustomField` 没有 `getCreateWorkItemFormItemValues()`（那是 `Control` 的方法）
> - `Modal.open()` 的 `entry` 参数是 resource ID（非文件路径），弹窗内用 `containerModal` 而非 `modal`

## P3：规划功能实现方案

结合用户的功能需求和已有的模板代码，规划具体的实现方案：

1. 分析用户需求需要哪些 UI 组件、数据交互、状态管理
2. 基于 P2 查询结果确认需要调用的 JSSDK API 和 FeatureContext 字段
3. 规划代码修改方案——在模板基础上需要新增/修改哪些部分

向用户确认实现方案后，进入 apply 阶段。
