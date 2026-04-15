# mode=apply：在模板基础上实现功能代码

> CLI `update` 已在各 entry 路径生成了模板代码。apply 阶段在模板基础上**填充用户真实需求的功能代码**。

## A1：为每个点位实现功能代码

对于 `resources` 数组中的每一项：

1. 读取 `entry` 路径下的**已有模板代码**
2. 根据 plan 阶段的实现方案，在模板代码基础上填充功能逻辑
3. 将修改后的代码写回 `entry` 路径

> **禁止修改 `plugin.config.json`**。`resources` 数组由 CLI `update` 命令生成并维护。

## A1.5：代码溯源审计（防 schema 脑补）

A1 写完后、A2 输出前，**MUST 派一个独立 reviewer subagent** 对所有改动文件做溯源审计，避免 AI 在写代码时悄悄从 schema 字段名脑补 SDK 调用。

### Reviewer 任务

派 subagent（`Agent` 工具，`subagent_type: "general-purpose"`，独立 context），prompt 如下：

```
你是代码溯源审计员。给你一组 .ts / .tsx 文件，扫描其中所有的 SDK 调用，
检查每个调用是否能追溯到合法信息源。

合法信息源（白名单）：
- @lark-project/js-sdk 包内类型定义（node_modules/@lark-project/js-sdk/dist/types/index.d.ts）
- 飞书项目知识 MCP 文档示例
- 用户在对话中明确提供的代码
- 当前工程已存在的代码

非法信息源（黑名单）：
- 从 plugin.config.json schema 字段名脑补
- AI 内置的"通用 SDK 经验"类比
- 没有 source 注释、看起来"应该有"的方法名

扫描以下模式的可疑调用：
- window.JSSDK.<module>.<method>(...)
- containerModal.<method>(...) / modal.<method>(...)
- 任何 import 自 @lark-project/js-sdk 的符号调用

对每个调用，验证：
1. 在 js-sdk types/index.d.ts 中能否找到精确签名？（执行 grep 验证）
2. 邻近是否有 // source: ... 注释？
3. 调用的方法名是否疑似从 schema 字段名脑补？（典型特征：方法名和 schema 字段重叠但 SDK 不存在）

输出结构化结果：
{
  audited_files: [...],
  suspicious_calls: [
    {
      file: "...",
      line: N,
      call: "window.JSSDK.xxx.yyy(...)",
      reason: "method 'yyy' not found in js-sdk types | no source comment | suspected schema-derived name",
      suggested_action: "ask user | remove | replace with <known API>"
    }
  ],
  pass: boolean  // suspicious_calls 为空才 true
}
```

### 主 agent 处理 reviewer 结果

- `pass: true` → 进入 A2 输出摘要
- `pass: false` → **暂停**，把 `suspicious_calls` 完整列表呈现给用户：

```
⚠️ 代码溯源审计发现以下疑似编造调用：

1. src/features/xxx/index.tsx:42
   call: window.JSSDK.control.getValue()
   reason: 'getValue' 在 @lark-project/js-sdk 类型定义中不存在
           疑似从 schema 字段名脑补
   建议: 改用 control.getCreateWorkItemFormItemValues 或询问真实 API 名

2. src/features/yyy/index.tsx:88
   call: window.JSSDK.customField.saveData(...)
   reason: 无 source 注释，'saveData' 在 types 中未找到
           真实 API 是 saveFieldValue
   建议: 替换为 saveFieldValue(...)

请告诉我每条怎么处理：
- 接受 reviewer 建议（修改/删除）
- 我提供真实 API（你贴文档/示例）
- 跳过这处功能（输出 TODO 占位）
```

> ⚠️ **禁止 orchestrator 自己"过滤" reviewer 报告**。所有 suspicious_calls 必须**原文呈现**，不允许"我帮你筛选了下，问题不大"这种 sycophancy。

### 何时可以跳过 A1.5？

仅在以下条件全满足时可跳过（plan 阶段已显式记录）：

- 本次 apply 没有产生任何新的 `window.JSSDK.*` / SDK 调用
- 仅修改 UI、样式、纯前端逻辑（无 SDK 接触）

否则一律执行 A1.5，**不允许"我自己看了一遍觉得没问题"代替 reviewer**。

## A2：输出摘要

```
✅ 代码生成完成
   修改文件：
   • ./src/features/board_web_rj4v_m/index.tsx

   下一步：执行语法检查（mode=verify）
```
