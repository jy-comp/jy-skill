---
name: meego-shared
version: 1.0.0
description: "Meego 插件开发共享基础：CLI 可用性检查、Device Code OAuth 认证、Token 管理、安全规则。所有 meego-* / plugin-* skill 的公共前置依赖。"
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# Meego 共享规则

本技能包含 Meego 插件开发的公共认证、环境检查和安全规则。所有其他 skill 在执行前必须先确认本 skill 中的前置条件已满足。

## CLI 可用性检查

```bash
npx @byted-meego/cli@builder --version
```

- 成功（输出版本号）→ CLI 可用
- 失败 → 提示用户安装：`npm install -g @byted-meego/cli@builder`

## 认证

Meego 插件开发采用统一认证，完成一次授权即可使用所有功能。无细分权限，Token 有效即可执行全部操作。

### 双 Token 机制

支持两种 Token，**按域名独立存储**，优先使用永久 Token：

| Token 类型 | 来源 | 有效期 | 优先级 |
|-----------|------|--------|-------|
| **Developer Token**（永久） | `login --token <token>` 手动设置 | 永久有效 | **高** — 有它就不需要 OAuth 授权 |
| **OAuth Token**（临时） | `login` 浏览器授权获取 | 临时，支持自动刷新 | 低 — 仅在无 Developer Token 时使用 |

### Token 存储

Token 按域名存储在 `~/.lpm/auth.json`，格式：

```json
{
  "https://project.feishu.cn": {
    "developerToken": "永久有效的token",
    "accessToken": "OAuth临时token",
    "accessTokenExpiresAt": 1234567890,
    "refreshToken": "...",
    "refreshTokenExpiresAt": 1234567890,
    "clientId": "<client_id>",
    "tokenId": "<token_id>"
  },
  "https://meegle.com": {
    "developerToken": "另一个域名的token"
  }
}
```

- 不同域名的 Token 互不影响
- `developerToken` 永久有效，无需刷新
- `clientId` + `refreshToken` 用于 OAuth Token 的自动刷新
- API 请求通过 `Authorization: Bearer <token>` 消费（优先取 `developerToken`）

### Token 检查

检查 `~/.lpm/auth.json` 中当前域名是否有可用 Token：
- 有 `developerToken` → 直接使用，认证通过
- 有 `accessToken` → 检查是否过期，过期则自动刷新
- 都没有 → 需执行认证流程（见下方）

### 认证流程

支持两种方式：

**方式 A（推荐）：直接设置永久 Developer Token**

引导用户前往 `<站点域名>/openapp/settings` 页面复制 Developer Token，然后执行：

```bash
npx @byted-meego/cli@builder login --site-domain <域名> --token <developer_token>
```

将永久 Token 写入对应域名的 `developerToken` 槽。有此 Token 后无需再走 OAuth 授权。

**方式 B：Device Code OAuth 浏览器授权**

```bash
# 以 background 方式执行（阻塞直到用户完成授权或超时）
npx @byted-meego/cli@builder login --site-domain <域名>
```

启动后立即读取输出，提取授权码和验证链接发送给用户。后台命令会自动轮询等待授权完成。获取的 OAuth Token 为临时 Token，支持自动刷新。

> 完整的认证步骤详见 [`../env-setup/references/setup.md`](../env-setup/references/setup.md)。

### 站点域名（siteDomain）

Token 按域名生效，确定 `siteDomain` 的优先级：
1. 当前目录 `plugin.config.json` 中的 `siteDomain` 字段
2. 询问用户选择：飞书项目 (project.feishu.cn) / Meegle (meegle.com) / 自定义域名

## 安全规则

- **禁止修改 `.lpm/` 目录下的任何文件**：`.lpm/` 目录由 CLI 内部管理（如 `auth.json`、缓存等），禁止通过 Edit/Write 工具或任何方式直接修改其中的文件，只能通过 CLI 命令间接操作
- **禁止输出密钥**（accessToken、pluginSecret）到终端明文
- **写入/删除操作前必须确认用户意图**（如发布、删除点位等不可逆操作）
- **全量提交约束**：`local-config set` 和 `update --source-type=local` 均为全量操作，禁止只传变更部分

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| CLI 未安装 | 提示 `npm install -g @byted-meego/cli@builder` |
| Token 过期 | 自动尝试 refresh；失败则引导重新执行 `/env-setup` |
| 网络错误 | 展示原始错误，建议检查网络后重试 |
| `plugin.config.json` 缺失 | 引导先执行 `/plugin-create` 创建工程 |
