# Phase 2：搭建工程

将 Phase 1 确认的需求转化为可运行的插件工程骨架。本阶段串联三个内部 skill。

## 2.1 环境准备 → env-setup

检查开发环境是否就绪：

```
调用 /env-setup
```

**判断逻辑：**
- `npx @byted-meego/cli@builder --version` 正常 + `~/.lpm/auth.json` Token 有效 → 跳过
- 否则 → 执行 env-setup 完整流程

> 如果 env-setup 过程中获取了 siteDomain（从 URL 解析或 plugin.config.json），记录下来供后续使用。

## 2.2 创建插件 → plugin-create

用 Phase 1 中的功能描述生成一个工作名称，最小化创建：

```
调用 /plugin-create mode=pipeline
```

**AI 自动完成，无需用户输入：**
- 从功能描述中提取工作名称（如"关联需求图谱" → 工作名称"需求图谱"）
- 执行 `npx @byted-meego/cli@builder create --site-domain <siteDomain> --name "<工作名称>" --force`
- 等待工程初始化完成

**产出：** `plugin.config.json` + 工程目录

## 2.3 配置点位 → meego-point-config

将 Phase 1 推导的点位类型写入配置并推送到后台：

```
调用 /meego-point-config mode=pipeline
```

**AI 根据 Phase 1 的需求分析自动填充点位配置：**
- key、name、description 从功能描述中派生
- icon 使用合理默认值
- work_item_type 根据上下文推导（默认 `_all`）
- 其他必填字段按点位类型填充默认值

**关键：AI 应尽可能自主完成配置，仅在信息确实不足时才问用户。**

## 2.4 阶段输出

```
✅ 插件工程搭建完成
   pluginId: PL_xxxxxxxxx
   点位：[board] 需求图谱看板 / [dashboard] 关联需求标签页
   下一步：正在为你生成代码...
```

**自动进入 Phase 3，无需用户确认。**
