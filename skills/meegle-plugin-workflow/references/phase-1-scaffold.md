# Phase 1：搭建工程

将 Phase 0 采集到的 `context.originalRequirement` 转化为可运行的插件工程骨架。本阶段串联环境准备和创建工程两步，**不涉及点位配置**（延后到 Phase 2）。

> **Checkpoint**：本阶段每个子步骤执行前后都需要更新 `.lpm-cache/state.json`，详见 SKILL.md「进度追踪」章节。

## 1.1 环境准备 → meegle-plugin-env-setup

检查开发环境是否就绪：

**Checkpoint 写入**：`{ "phase": 1, "step": "1.1", "stepName": "meegle-plugin-env-setup", "nextCommand": "npx @byted-meego/cli@builder login ...", "nextStep": "1.1 环境准备" }`

```
调用 /meegle-plugin-env-setup
```

**判断逻辑（两步预检，meegle-plugin-workflow 作为 orchestrator 专享）：**

1. **CLI 可用性**：`npx @byted-meego/cli@builder --version`
   - 失败 → 提示安装 `npm install -g @byted-meego/cli@builder`，**终止 workflow**
   - 成功 → 进入第 2 步

2. **Auth probe**（使用 `context.siteDomain`）：
   ```bash
   npx @byted-meego/cli@builder check --option auth --site-domain <siteDomain>
   ```
   - `exit 0` → 已登录 → **跳过 meegle-plugin-env-setup**，直接进入 1.2 meegle-plugin-create
   - `exit 1` → stderr 已含完整登录指引 → **触发 `/meegle-plugin-env-setup`** 执行登录编排（方式 A token 粘贴 / 方式 B Device Code 后台驱动），完成后继续 1.2

> **为什么 meegle-plugin-workflow 做预检而其它 skill 不做**：meegle-plugin-workflow 是多步编排的 orchestrator，中断在链条中间（如 meegle-plugin-create 报 auth 错）体验差。其它 skill（meegle-plugin-feature、meegle-plugin-publish 等）独立调用时规则简单即可，依赖 CLI 内置的 preflight 兜底，不需要每个都加一次 probe。
>
> **为什么用 probe 而不是读 auth.json**：skill 不直接耦合 CLI 的 token 存储格式。probe 命令是 CLI 对外暴露的稳定接口，auth.json 的内部结构可能演进。

## 1.2 创建插件 → meegle-plugin-create

从 `context.originalRequirement` 派生工作名称，最小化创建：

**Checkpoint 写入**：`{ "phase": 1, "step": "1.2", "stepName": "meegle-plugin-create", "nextCommand": "npx @byted-meego/cli@builder create ...", "nextStep": "1.2 创建插件" }`

```
调用 /meegle-plugin-create mode=pipeline
```

**AI 自动完成，无需用户输入：**
- 从 `context.originalRequirement` 中提取工作名称（如"关联需求图谱" → 工作名称"需求图谱"）
- 执行 `npx @byted-meego/cli@builder create --site-domain <siteDomain> --name "<工作名称>" --force`
- 等待工程初始化完成

**产出：** `plugin.config.json` + 工程目录

**meegle-plugin-create 成功后 MUST 切 cwd 到新工程目录**，再继续 Phase 2（后续 CLI 命令、Read `plugin.config.json` 等都依赖 cwd 指向新工程）。

**Checkpoint 更新**：create 成功后将 `context.pluginId`、`context.projectRoot`（新工程绝对路径）写入 checkpoint。

## 1.3 阶段输出

```
✅ 插件工程骨架搭建完成
   pluginId: MII_xxxxxxxxx
   下一步：正在分析可用点位...
```

**自动进入 Phase 2，无需用户确认。**
