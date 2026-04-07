# mode=analyze：分析插件实际功能

## A1：读取插件配置

从 `plugin.config.json` 中获取：
- `pluginId`（app_key）
- `siteDomain`
- `resources` 列表（已注册的点位 resourceId）

## A2：读取点位配置

检查以下文件（按优先级）：
1. `point-schema.yaml` — 完整的点位配置
2. `point.config.local-remote.json` ��� 远端同步的配置

从中提取每个点位的：
- **类型**（board/view/dashboard/button/control/intercept/...）
- **名称**（i18n_info 中的 name）
- **描述**（i18n_info 中的 description）
- **适用的工作项类型**

## A3：快速浏览代码

扫描 `src/features/` 或 `src/` 下的入口文件（index.tsx），理解：
- 每个点位实际渲染了什么 UI
- 调用了哪些 JSSDK API
- 核心业务逻辑是什么

> 不需要逐行阅读，重点关注组件名、主要 import、render 函数中的核心结构。

## A4：输出功能摘要

内部生成一份结构化��功能摘要（不展示给用户），供 generate 阶段使用：

```
���能摘要：
- 点位 1：[类型] [名称] — [一句话说明做了什么]
- 点位 2：[类型] [名称] — [一句话说明做了什么]
- 核心能力：[���括插件整体解决的问题]
```
