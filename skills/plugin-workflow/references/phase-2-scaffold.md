# Phase 2：搭建工程

将 Phase 1 确认的需求转化为可运行的插件工程骨架。本阶段串联环境准备、创建工程、拉取 schema 分析点位、配置点位四个步骤。

> **Checkpoint**：本阶段每个子步骤执行前后都需要更新 `.plugin-workflow-state.json`，详见 SKILL.md「进度追踪」章节。

## 2.1 环境准备 → env-setup

检查开发环境是否就绪：

**Checkpoint 写入**：`{ "phase": 2, "step": "2.1", "stepName": "env-setup", "nextCommand": "npx @byted-meego/cli@builder login ...", "nextStep": "2.1 环境准备" }`

```
调用 /env-setup
```

**判断逻辑：**
- `npx @byted-meego/cli@builder --version` 正常 + `~/.lpm/auth.json` Token 有效 → 跳过
- 否则 → 执行 env-setup 完整流程

> 如果 env-setup 过程中获取了 siteDomain（从 URL 解析或 plugin.config.json），记录下来供后续使用。

## 2.2 创建插件 → plugin-create

用 Phase 1 中的功能描述生成一个工作名称，最小化创建：

**Checkpoint 写入**：`{ "phase": 2, "step": "2.2", "stepName": "plugin-create", "nextCommand": "npx @byted-meego/cli@builder create ...", "nextStep": "2.2 创建插件" }`

```
调用 /plugin-create mode=pipeline
```

**AI 自动完成，无需用户输入：**
- 从功能描述中提取工作名称（如"关联需求图谱" → 工作名称"需求图谱"）
- 执行 `npx @byted-meego/cli@builder create --site-domain <siteDomain> --name "<工作名称>" --force`
- 等待工程初始化完成

**产出：** `plugin.config.json` + 工程目录

**Checkpoint 更新**：create 成功后将 `context.pluginId` 和 `context.siteDomain` 写入 checkpoint。

## 2.3 拉取 Schema + 确认点位

### 2.3.1 拉取 Schema

**Checkpoint 写入**：`{ "step": "2.3", "stepName": "schema + 点位确认", "nextCommand": "npx @byted-meego/cli@builder schema > point-schema.yaml", "nextStep": "2.3.1 拉取 Schema" }`

```bash
npx @byted-meego/cli@builder schema > point-schema.yaml
```

### 2.3.2 确定点位

分两种情况处理，**二者互斥**：

#### 情况 A：用户已在 Phase 1 中指定了点位类型

**直接使用用户指定的点位，不要另行推荐替代方案。** 只需在 schema 中验证该点位确实存在即可。

- 验证通过 → 直接进入 2.4 配置点位，**跳过 2.3.3 确认环节**
- 验证失败（schema 中无此点位） → 告知用户该点位不存在，列出 schema 中可用的点位供选择

#### 情况 B：用户未指定点位类型

**完全基于 `point-schema.yaml` 的实际内容进行分析推荐。** 不要依赖任何硬编码的点位类型假设——schema 是唯一真实来源，点位类型和字段约束可能随版本变化。

**防幻觉约束：** 向用户展示的点位名称和描述**必须直接引用 schema 中的原文**，禁止自行编造点位描述（如不能把 board 称为"导航位"，除非 schema 中确实这样描述）。

AI 需要：
1. 解析 schema 中所有可用的点位类型及其描述
2. 结合用户的功能需求，匹配最合适的点位类型
3. 一个功能可能需要**多个独立点位**配合（如展示 + 配置）

> **⚠️ 每种点位类型是独立的顶层概念，不能嵌套。** 例如用户需要"轻应用 + 拓展字段"时，应推荐 `component` 和 `field_template` 两个独立点位，而不是把 field_template 作为 component 或 builder_comp 的属性。同理，"控件"是独立的 `control` 点位，不是某个点位的子配置。

→ 进入 2.3.3 向用户确认。

### 2.3.3 向用户确认点位方案（仅情况 B）

```
基于你的需求和 schema 中的点位定义，推荐以下方案：

🧩 需要的点位：
   - [点位类型1]：[schema 中的原文描述]
   - [点位类型2]：[schema 中的原文描述]（如有多个）

📝 说明：[简要解释为什么选择这些点位]

方案可以吗？如果需要调整可以告诉我。
```

**用户确认点位方案后再进入点位配置。**

## 2.4 配置点位 → meego-point-config

将确认的点位类型写入配置并推送到后台：

**Checkpoint 写入**：`{ "step": "2.4", "stepName": "meego-point-config", "nextCommand": "npx @byted-meego/cli@builder local-config set ...", "nextStep": "2.4 配置点位" }`

**恢复检查**：若从 checkpoint 恢复，根据 `lastCommand` 判断：
- `lastCommand` 含 `local-config set` + `lastCommandStatus` = `"success"` → 跳过 set，直接执行 `update`
- `lastCommand` 含 `update` + `lastCommandStatus` = `"success"` → 点位已配置，跳过整个 2.4

```
调用 /meego-point-config mode=pipeline
```

**AI 根据确认的点位方案自动填充点位配置：**
- key、name、description 从功能描述中派生
- icon 使用合理默认值
- work_item_type 根据上下文推导（默认 `_all`）
- 其他必填字段按点位类型填充默认值

**关键：AI 应尽可能自主完成配置，仅在信息确实不足时才问用户。**

meego-point-config 内部的 CLI 命令执行顺序和 checkpoint 更新：
1. `local-config set <config>` → 成功后更新 checkpoint: `lastCommand="local-config set"`, `nextCommand="update"`
2. `npx @byted-meego/cli@builder update` → 成功后更新 checkpoint: `lastCommand="update"`, `nextStep="2.5"`

## 2.5 阶段输出

```
✅ 插件工程搭建完成
   pluginId: PL_xxxxxxxxx
   点位：[board] 需求图谱看板 / [dashboard] 关联需求标签页
   下一步：正在为你生成代码...
```

**自动进入 Phase 3，无需用户确认。**
