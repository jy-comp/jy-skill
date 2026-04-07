# Phase 2：搭建工程

将 Phase 1 确认的需求转化为可运行的插件工程骨架。本阶段串联环境准备、创建工程、拉取 schema 分析点位、配置点位四个步骤。

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

## 2.3 拉取 Schema + 分析推荐点位

**在配置点位之前，必须先拉取 schema 并基于 schema 分析适合的点位类型。**
用户在 Phase 1 只确认了功能需求，对点位没有概念，不应让用户自己选择点位。

### 2.3.1 拉取 Schema

```bash
npx @byted-meego/cli@builder schema > point-schema.yaml
```

### 2.3.2 分析并推荐点位

**完全基于 `point-schema.yaml` 的实际内容进行分析。** 不要依赖任何硬编码的点位类型假设——schema 是唯一真实来源，点位类型和字段约束可能随版本变化。

AI 需要：
1. 解析 schema 中所有可用的点位类型及其描述
2. 结合用户的功能需求，匹配最合适的点位类型
3. 一个功能可能需要多个点位配合（如展示 + 配置）

### 2.3.3 向用户确认点位方案

拉取 schema 并分析后，向用户展示推荐的点位方案：

```
基于你的需求，我分析了可用的插件点位类型，推荐以下方案：

🧩 需要的点位：
   - [点位类型1]：[用途说明]
   - [点位类型2]：[用途说明]（如有多个）

📝 说明：[简要解释为什么选择这些点位]

方案可以吗？如果需要调整可以告诉我。
```

**用户确认点位方案后再进入点位配置。**

## 2.4 配置点位 → meego-point-config

将确认的点位类型写入配置并推送到后台：

```
调用 /meego-point-config mode=pipeline
```

**AI 根据确认的点位方案自动填充点位配置：**
- key、name、description 从功能描述中派生
- icon 使用合理默认值
- work_item_type 根据上下文推导（默认 `_all`）
- 其他必填字段按点位类型填充默认值

**关键：AI 应尽可能自主完成配置，仅在信息确实不足时才问用户。**

## 2.5 阶段输出

```
✅ 插件工程搭建完成
   pluginId: PL_xxxxxxxxx
   点位：[board] 需求图谱看板 / [dashboard] 关联需求标签页
   下一步：正在为你生成代码...
```

**自动进入 Phase 3，无需用户确认。**
