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

## P2：规划功能实现方案

结合用户的功能需求和已有的模板代码，规划具体的实现方案：

1. 分析用户需求需要哪些 UI 组件、数据交互、状态管理
2. 确认需要调用的 JSSDK API
3. 规划代码修改方案——在模板基础上需要新增/修改哪些部分

向用户确认实现方案后，进入 apply 阶段。
