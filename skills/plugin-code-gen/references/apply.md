# mode=apply：在模板基础上实现功能代码

> CLI `update` 已在各 entry 路径生成了模板代码。apply 阶段在模板基础上**填充用户真实需求的功能代码**。

## A1：为每个点位实现功能代码

对于 `resources` 数组中的每一项：

1. 读取 `entry` 路径下的**已有模板代码**
2. 根据 plan 阶段的实现方案，在模板代码基础上填充功能逻辑
3. 将修改后的代码写回 `entry` 路径

> **禁止修改 `plugin.config.json`**。`resources` 数组由 CLI `update` 命令生成并维护。

## A2：输出摘要

```
✅ 代码生成完成
   修改文件：
   • ./src/features/board_web_rj4v_m/index.tsx

   下一步：执行语法检查（mode=verify）
```
