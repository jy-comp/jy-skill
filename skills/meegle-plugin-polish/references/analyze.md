# mode=analyze：分析插件实际功能

## A1：读取插件配置

从 `plugin.config.json` 中获取：
- `pluginId`（app_key）
- `siteDomain`
- `resources` 列表（已注册的点位 resourceId）

## A2：读取点位配置

从 `plugin.config.json` 的 `extensions` 字段中提取每个点位的：
- **类型**（page/view/dashboard/button/control/intercept/...）
- **名称**（i18n_info 中的 name）
- **描述**（i18n_info 中的 description）
- **适用的工作项类型**

> **注意：** 不要依赖 `.lpm-cache/config/remote.json`——它在 `local-config set` 成功时已被 CLI 删除。点位名称 / 描述 / 适用工作项类型的权威源是工程根目录的 `point.config.local.json`（由 `local-config set` 和 `update` 维护）；`plugin.config.json` 里只有 `resources` 映射表，不含点位的业务字段。

## A3：快速浏览代码

扫描 `src/features/` 或 `src/` 下的入口文件（index.tsx），理解：
- 每个点位实际渲染了什么 UI
- 调用了哪些 JSSDK API
- 核心业务逻辑是什么

> 不需要逐行阅读，重点关注组件名、主要 import、render 函数中的核心结构。

## A4：输出功能摘要

内部生成一份结构化的功能摘要（不展示给用户），供 generate 阶段使用：

```
功能摘要：
- 点位 1：[类型] [名称] — [一句话说明做了什么]
- 点位 2：[类型] [名称] — [一句话说明做了什么]
- 核心能力：[概括插件整体解决的问题]
```
