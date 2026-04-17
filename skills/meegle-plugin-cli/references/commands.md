# Meegle 插件开发 CLI 命令详解

> 本文件是 [`../SKILL.md`](../SKILL.md) 的命令字典补充。SKILL.md 里 A 类速查表的"详见"列均指向本文。

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
| `--site-domain` | 是 | Meegle 站点域名，含协议（如 `https://meego.feishu-boe.cn`） |
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
| `--store` | 否 | `publish`（发布到商店）/ `no-publish`(默认，仅内部) |
| `--upgrade` | 否 | CLI 默认 `manual`；AI 不主动传，仅当用户明确要求 `all`（自动升级所有安装）/ `limit`（按范围）时才显式传 |

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
# 写入远端配置到 .lpm-cache/config/remote.json
npx @byted-meego/cli@builder local-config get --remote

# 推送 draft（全量替换语义）。成功后 CLI 会：
#   1) 删除该 draft 和 .lpm-cache/config/remote.json（已消费的基线）
#   2) 把新的完整配置写入工程根 point.config.local.json（下游 code-gen / polish 读这里）
npx @byted-meego/cli@builder local-config set --from .lpm-cache/config/draft-<timestamp>.json
```

- `get` / `schema` 的 stdout 只回一行相对路径，JSON 落在 `.lpm-cache/` 里；AI 通过 `jq` / Read 按需取片段，不要把整份 JSON 带进对话 context
- `set` 只接受 `--from <path>`（相对或绝对），不再接受 inline JSON

> **CRITICAL — 全量提交 + 删除确认**：`local-config set` 是全量覆盖语义，且任何"减少点位"的提交都需用户确认。完整规则（含操作流程、不要做的事、各 skill 执行点）见 [`../../meegle-plugin-shared/SKILL.md`](../../meegle-plugin-shared/SKILL.md) 的"全量提交约束"和"删除点位前置检查协议"两节，本文不复述。

### 生成 Schema

```bash
npx @byted-meego/cli@builder schema
```

CLI 把 JSON Schema 写入 `.lpm-cache/schema/point-schema.json` 并在 stdout 回显路径。用 `jq` 按需切片，**不要 Read 整个文件**（约 30K tokens）。

### 清理工作区（兜底）

```bash
npx @byted-meego/cli@builder workspace clean --scope=all          # 清全部（保留 state.json）
npx @byted-meego/cli@builder workspace clean --scope=mcp          # 只清 MCP 缓存
npx @byted-meego/cli@builder workspace clean --include-state      # 连 workflow checkpoint 一起清
```

`.lpm-cache/` 的生命周期由 CLI 自动维护——`local-config set` / `update` / `publish` 成功时会按子目录清理。仅在中断后需要强制重置时才手动调用 `workspace clean`。

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

典型顺序（前置依赖 → 后置操作）：

1. `login` — 全局认证（一次性）
2. `create` — 创建插件工程（一次性）
3. `schema` → `local-config set` → `update` — 配置点位（可随时）
4. `start --auto` — 本地调试（依赖 step 3 的 `update`）
5. `update-description` — 修改基本信息（可随时）
6. `release` → `publish` — 构建发布（两步串行，不可跳）
