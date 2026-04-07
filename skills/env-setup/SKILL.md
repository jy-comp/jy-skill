---
name: env-setup
version: 1.0.0
description: |
  Meego 插件开发环境准备：检查 CLI 可用性，通过 Device Code 授权登录。
  当用户首次使用 npx @byted-meego/cli@builder、Token 过期、或需要重新授权时触发。
  也适用于用户说"初始化环境"��"登录"、"设置 token"等场景。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# 环境准备 Skill

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则等公共约定。**
**CRITICAL — 执行前务必先用 Read 工具读取 [`references/setup.md`](references/setup.md)，禁止直接盲目执行。**

## 核心流程

```
S1 → 检查 CLI 可用性（npx @byted-meego/cli@builder --version）
S2 → 检查授权 Token（~/.lpm/auth.json）
     ├─ Token 有效 → 完成
     └─ Token 缺失/过期 → Auth Guard（Device Code 授权）
```

## Auth Guard

所有需要授权的操作前必须先通过 Auth Guard。流程参考 `references/setup.md`。

支持两种授权方式（按域名独立存储，优先使用永久 Token）：
- **方式 A（推荐）**：永久 Developer Token — 通过 `npx @byted-meego/cli@builder login --site-domain <域名> --token <token>` 直接设置，永久有效，有它就不需要 OAuth 授权
- **方式 B**：Device Code OAuth — 通过 `npx @byted-meego/cli@builder login --site-domain <域名>` 启动浏览器授权，获取临时 Token，支持自动刷新

## 使用方式

```
/env-setup                    # 完整环境检查 + 授权
```

## 详细流程

读取 `references/setup.md`。

## Token 存储

Token 按域名存储在 `~/.lpm/auth.json`，详见 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md) 中的"双 Token 机制"章节。
