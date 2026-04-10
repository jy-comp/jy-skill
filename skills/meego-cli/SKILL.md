---
name: meego-cli
version: 1.2.0
description: |
  Meego 插件 CLI 原子化命令参考与诊断工具（npx @byted-meego/cli@builder）。
  当用户在插件工程目录（存在 plugin.config.json 且含 MII_ 开头的 pluginId）下需要执行单条 CLI 命令时触发。
  覆盖原子化操作：启动调试、本地预览、跑起来看看、构建、打包、同步配置、拉取远端、推送配置、
  查看配置、查看远端配置、生成 schema、查看分类、登录、认证。
  覆盖诊断查询：当前状态、进度到哪了、这个插件是干什么的、调试报错、start 起不来、
  tsc 报错、代码报错、能回退吗、改代码。
  注意：多步编排操作（发布、点位配置、代码生成等）由对应的编排 skill 直接处理，不经过本 skill。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# Meego 插件 CLI 原子化命令参考 & 诊断工具

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则、插件工程识别等公共约定。**
**CRITICAL — 禁止修改 `.lpm/` 目录下的任何文件，该目录由 CLI 内部管理，只能通过 CLI 命令间接操作。**

## 项目上下文感知

使用本 skill 前，**MUST** 先确认当前处于插件工程目录：

1. **检查**：读取当前目录的 `plugin.config.json`
2. **验证**：确认含有 `pluginId`（以 `MII_` 开头）
3. **提取上下文**：
   - `siteDomain` — CLI 命令自动读取，无需手动传参
   - `pluginId` — 插件标识
   - `resources` — 已配置的点位资源列表（`id` 前缀反映点位类型，如 `board_web_`、`button_web_`）

如果 `plugin.config.json` 不存在或无 `pluginId`，则当前不是插件目录，需要先 `create` 或 `cd` 到插件目录。

---

## 本 skill 的职责范围

**本 skill 只处理两类场景：原子化 CLI 命令 和 诊断查询。**

多步编排操作（发布、点位配置、代码生成、信息完善等）由对应的编排 skill 直接处理：
- "发布"/"上线" → `plugin-publish`
- "加个点位"/"改点位" → `meego-point-config`
- "生成代码"/"写代码" → `plugin-code-gen`
- "改名称"/"改描述" → `plugin-polish`
- "做一个 xxx 功能" → `plugin-workflow`

### A. 原子化 CLI 命令（本 skill 直接执行）

每条命令独立完成一个操作，无需多步编排：

| 用户可能的说法 | CLI 命令 | 详见 |
|--------------|---------|------|
| "启动调试" / "本地预览" / "跑起来看看" / "dev" | `start --auto` | [启动调试](#启动调试) |
| "构建" / "打包" / "build" | `build` | [构建](#构建) |
| "同步配置" / "拉取远端" / "拉取最新" | `update` | [同步配置](#同步配置) |
| "推送配置" / "推到远端" | `update --source-type=local` | [同步配置](#同步配置) |
| "查看点位配置" / "当前配置" | `local-config get` | [点位配置](#点位配置管理) |
| "查看远端配置" | `local-config get --remote` | [点位配置](#点位配置管理) |
| "生成 schema" / "看看有哪些点位类型" | `schema > point-schema.yaml` | [生成 Schema](#生成-schema) |
| "查看分类" / "有哪些分类" | `list-categories` | [分类列表](#分类列表) |
| "登录" / "认证" / "设置 token" | `login` | [登录认证](#登录认证) |

### B. 诊断查询（本 skill 读取项目状态回答）

用户不是要执行操作，而是要了解当前状态或排查问题：

| 用户可能的说法 | 处理方式 | 详见 |
|--------------|---------|------|
| "当前什么状态" / "进度到哪了" / "之前做到哪了" | 读取 checkpoint + config + src/ 综合判断 | [项目状态诊断](#项目状态诊断) |
| "这个插件是干什么的" / "有哪些点位" | 读取 resources 列表，按 id 前缀分析点位类型 | [项目状态诊断](#项目状态诊断) |
| "调试报错了" / "start 起不来" / "白屏" | 常见调试问题排查 | [常见问题排查](#常见问题排查) |
| "tsc 报错" / "代码编译错误" / "类型错误" | 引导执行 `tsc --noEmit` 或使用 `/plugin-code-gen mode=verify` | [常见问题排查](#常见问题排查) |
| "能回退版本吗" / "撤销发布" / "版本回滚" | CLI 不支持版本回退，引导后台手动操作 | [常见问题排查](#常见问题排查) |
| "改一下代码" / "按钮逻辑改成 xxx" | 直接修改 `src/features/` 下的代码文件，不需要 CLI | [常见问题排查](#常见问题排查) |

---

## 项目状态诊断

当用户询问当前状态、首次进入插件目录、或从中断处恢复时，执行以下诊断：

### 诊断步骤

1. 读取 `plugin.config.json` → 提取 pluginId、siteDomain、resources
2. 读取 `.plugin-workflow-state.json`（如存在）→ 获取上次执行进度
3. 检查 `src/features/` 目录 → 判断代码是否已生成
4. 检查 `node_modules/` → 判断依赖是否已安装

### 状态判定与建议

| 条件组合 | 状态 | 建议操作 |
|---------|------|---------|
| 无 `plugin.config.json` | 非插件目录 | `create` 创建插件或 `cd` 到插件目录 |
| 有 config，`resources` 为空 | 已创建，未配置点位 | → `/meego-point-config` 配置点位 |
| 有 config，`resources` 非空，无 `src/features/` | 已配置，未生成代码 | → `/plugin-code-gen` 生成代码 |
| 有 `src/features/` 但代码为模板 | 已生成模板，未实现功能 | → `/plugin-code-gen mode=apply` 填充功能 |
| 有 `src/features/` 且代码已实现 | 可调试或发布 | `start --auto` 调试 或 → `/plugin-publish` 发布 |
| `.plugin-workflow-state.json` 存在 | workflow 中断 | 展示上次进度，引导继续（详见 checkpoint 恢复） |

### 输出格式

```
📍 插件状态诊断：
   插件 ID：MII_xxxxxxxxxxxx
   站点：https://meego.feishu-boe.cn
   点位资源：3 个（board_web × 1, button_web × 1, control_web × 1）
   代码状态：已实现（src/features/ 下 3 个目录）
   上次进度：Phase 3 — 本地调试中（来自 checkpoint）
   
   建议：运行 `npx @byted-meego/cli@builder start --auto` 继续调试
```

---

## 常见问题排查

### "调试报错了" / "start 起不来"

1. 检查 `node_modules/` 是否存在 → 不存在则 `npm install`
2. 检查端口占用 → `lsof -i :3000`，如占用则 kill 对应进程
3. 检查 `plugin.config.json` 的 `resources` 是否为空 → 空则需先配置点位
4. 检查 Token 是否过期 → `~/.lpm/auth.json` 对应域名的 Token
5. 尝试 `npx @byted-meego/cli@builder start --auto` 查看具体报错

### "tsc 报错" / "代码编译错误"

1. 执行 `npx tsc --noEmit` 查看完整错误列表
2. 常见原因：
   - JSSDK API 属性名拼写错误（如 `Schedule` → `schedule`）
   - FeatureContext 字段不存在（各点位 context 字段不同，参考 `@lark-project/js-sdk` 类型定义）
   - union type 直接解构不安全（如 `ButtonFeatureContext` 需先判断 scene）
3. 可使用 `/plugin-code-gen mode=verify` 让 AI 自动修复（最多 2 轮）

### "能回退版本吗" / "撤销发布"

CLI 不支持版本回退。处理方式：
- 需要在 Meego 后台（`{siteDomain}/openapp/{pluginId}`）手动管理版本

### "改一下代码" / "调整 xxx 逻辑"

直接修改 `src/features/{resource_id}/` 下的代码文件即可，不需要任何 CLI 命令。
修改后：
- 如果 `start` 正在运行 → webpack 热更新自动生效
- 如果未运行 → `start --auto` 启动调试查看效果
- 修改完成后想发布 → `/plugin-publish`

## CLI 基础

**命令前缀**：所有命令均通过以下方式调用：

```bash
npx @byted-meego/cli@builder <command> [options]
```

**工程识别**：大部分命令需要在插件工程目录下执行（存在 `plugin.config.json`）。以下命令例外，可在任意目录执行：
- `login` — 全局认证
- `create` — 创建新插件工程

**配置来源**：`siteDomain`、`pluginId` 等信息自动从 `plugin.config.json` 读取，无需手动传入。

---

## 命令详解

### 登录认证

```bash
# 方式 A：浏览器 OAuth 授权（交互式）
npx @byted-meego/cli@builder login --site-domain <域名>

# 方式 B：直接设置永久 Developer Token（推荐）
npx @byted-meego/cli@builder login --site-domain <域名> --token <developer_token>
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--site-domain` | 是 | Meego 站点域名，含协议（如 `https://meego.feishu-boe.cn`） |
| `--token` | 否 | 永久 Developer Token，有则跳过 OAuth |

### 创建插件

```bash
npx @byted-meego/cli@builder create --site-domain <域名> --name "<名称>" [--force]
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--site-domain` | 是 | 站点域名 |
| `--name` | 是 | 插件名称 |
| `--description` | 否 | 短描述 |
| `--force` | 否 | 已在插件目录中时跳过确认 |

**产出**：在当前目录创建插件工程，生成 `plugin.config.json` + `src/` + `node_modules/`。

### 启动调试

```bash
npx @byted-meego/cli@builder start --auto
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--auto` | 否 | 自动打开浏览器调试页面（推荐） |
| `--source-type` | 否 | `local`（用本地配置）或 `remote`（用远端配置，默认） |

**行为**：启动 webpack-dev-server + 热更新，`--auto` 时自动拼接调试 URL 并在浏览器中打开。

### 同步配置

```bash
# 远端 → 本地（默认，拉取远端配置 + 生成代码模板）
npx @byted-meego/cli@builder update

# 本地 → 远端（推送本地点位配置到远端）
npx @byted-meego/cli@builder update --source-type=local
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--source-type` | 否 | `local`（推送到远端）或默认不传（从远端拉取） |

**行为**：
- 不传 `--source-type`：从远端拉取最新点位定义，生成/更新代码模板，更新 `plugin.config.json` 的 resources
- `--source-type=local`：将本地 `local-config.json` 的点位配置推送到远端

### 构建

```bash
npx @byted-meego/cli@builder build [--zip]
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--zip` | 否 | 构建后打包为 zip |
| `--source-type` | 否 | `local` 或 `remote` |

**产出**：`build/` 目录。

### 构建发布

**完整发布是两步串行操作**，不可跳步：

```bash
# 第 1 步：构建 + 上传产物
npx @byted-meego/cli@builder release
# 输出：Product version: <productVersion>

# 第 2 步：版本发布（必须使用第 1 步输出的 productVersion）
npx @byted-meego/cli@builder publish \
  --product-version <productVersion> \
  --desc-zh "<版本描述>"
```

**release 参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `--source-type` | 否 | `local` 或 `remote` |

**publish 参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `--product-version` | **是** | 来自 `release` 输出的产物版本号 |
| `--desc-zh` | 是 | 版本描述（中文） |
| `--version` | 否 | 语义化版本号，不传则自动 patch +1 |
| `--store` | 否 | `publish`（发布到商店）/ `no-publish`（默认，仅内部） |
| `--upgrade` | 否 | `manual`（默认）/ `all`（自动升级所有安装）/ `limit` |

**关键**：`--product-version` 必须从 `release` 的 stdout 中提取（正则：`/Product version: (\S+)/`），不可编造。

### 修改基本信息

```bash
npx @byted-meego/cli@builder update-description \
  --name "<插件名称>" \
  --short "<短描述>" \
  --detail-description "<详情描述>" \
  --category-ids "<id1>,<id2>"
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--name` | 否 | 插件名称 |
| `--short` | 否 | 短描述（纯文本） |
| `--detail-description` | 否 | 详情描述（纯文本，CLI 自动转富文本） |
| `--category-ids` | 否 | 分类 ID 列表，逗号分隔（从 `list-categories` 获取） |
| `--icon` | 否 | 图标 URL |

> 所有参数均为可选，只传需要修改的字段即可。

### 点位配置管理

```bash
# 查看当前本地配置
npx @byted-meego/cli@builder local-config get

# 查看远端配置
npx @byted-meego/cli@builder local-config get --remote

# 设置点位配置（全量替换，JSON 字符串）
npx @byted-meego/cli@builder local-config set --config '<完整 JSON>'
```

**全量替换语义**：`local-config set` 是**全量覆盖**，必须传完整配置（所有点位），不能只传变更部分。

**典型流程**：
1. `local-config get` → 获取当前完整配置
2. 在完整配置基础上增删改
3. `local-config set --config '<修改后的完整 JSON>'` → 校验 + 写入本地
4. `update --source-type=local` → 推送到远端
5. `update` → 拉取远端配置同步回本地

### 生成 Schema

```bash
npx @byted-meego/cli@builder schema > point-schema.yaml
```

生成点位配置的 JSON Schema，用于参考各点位类型的字段定义和约束。

### 分类列表

```bash
npx @byted-meego/cli@builder list-categories [--type 0]
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--type` | 否 | `0`（插件，默认）/ `1`（AI 助手） |

输出 JSON 数组，每项含 `id` 和 `name`，用于 `update-description --category-ids`。

---

## 命令依赖关系

```
login ─────────────────────────────────────────── 全局认证（一次性）
  │
  ▼
create ────────────────────────────────────────── 创建插件（一次性）
  │
  ▼
schema ──→ local-config set ──→ update ────────── 配置点位（可随时）
  │                                │
  │                                ▼
  │                            start --auto ───── 本地调试
  │
  ▼
update-description ────────────────────────────── 修改基本信息（可随时）
  │
  ▼
release ──→ publish ───────────────────────────── 构建发布
```

## 与编排 skill 的分工

```
用户指令
  │
  ├─ 原子操作（"启动调试"/"构建"/"同步"/"查看配置"）
  │     └─→ meego-cli（本 skill）→ 直接执行单条 CLI 命令
  │
  ├─ 编排操作（"发布"/"加点位"/"生成代码"/"改名称"）
  │     └─→ 对应编排 skill 直接处理（plugin-publish / meego-point-config / ...）
  │
  ├─ 端到端需求（"做一个 xxx 功能"）
  │     └─→ plugin-workflow 编排全流程
  │
  └─ 诊断查询（"什么状态"/"调试报错"）
        └─→ meego-cli（本 skill）→ 读取状态 / 排查引导
```

**分工原则**：单条 CLI 命令能完成的用 meego-cli，需要多步串行/交互确认的用编排 skill。
