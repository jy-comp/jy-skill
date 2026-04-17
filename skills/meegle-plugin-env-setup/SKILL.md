---
name: meegle-plugin-env-setup
version: 1.0.0
description: |
  Meegle 插件开发环境准备：检查 CLI 可用性，通过 Device Code 授权登录。
  当用户首次使用 npx @byted-meego/cli@builder、Token 过期、或需要重新授权时触发。
  也适用于用户说"初始化环境"、"登录"、"设置 token"等场景。
metadata:
  requires:
    bins: ["npx"]
  cliHelp: "npx @byted-meego/cli@builder --help"
---

# 环境准备 Skill

> **前置**：先 Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md) 获取共享规则；执行前 Read [`references/setup.md`](references/setup.md) 获取完整步骤。

## 本 skill 的最少 Read 清单

- 共享规则 → Read [`../meegle-plugin-shared/SKILL.md`](../meegle-plugin-shared/SKILL.md)
- 完整设置流程（含方式 A / 方式 B 后台 OAuth 编排）→ Read [`references/setup.md`](references/setup.md)

## 核心流程

```
S1 → 检查 CLI 可用性（npx @byted-meego/cli@builder --version）
S2 → 询问用户选择登录方式（A 或 B），执行对应流程
```

**meegle-plugin-env-setup 不读 `~/.lpm/auth.json`**——是否已登录由 CLI 在后续命令里实时判,这里专注"把登录执行完"。如果用户已经登录,重跑方式 A 会覆盖同一 token(无害),方式 B 会重开 OAuth(冗余但不破坏)。

## 触发场景

1. **用户主动**:说 "/meegle-plugin-env-setup"、"帮我登录"、"初始化环境" 等
2. **AI 升级触发**:其它 skill 执行命令时,CLI 报登录指引 → AI 按 meegle-plugin-shared 的"默认应对"触发 `/meegle-plugin-env-setup`,由此接管完整登录流程

## 登录方式

- **方式 A(推荐)**:永久 Developer Token — AI 引导用户从 `<siteDomain>/openapp/settings` 复制 token 粘贴,执行 `npx @byted-meego/cli@builder login --site-domain <域名> --token <token>`
- **方式 B**:Device Code OAuth — AI 后台启动 `npx @byted-meego/cli@builder login --site-domain <域名>`,解析 stdout 提取授权码 + 验证链接发给用户,等待命令自然退出

完整执行步骤(尤其方式 B 的后台编排)见 `references/setup.md`。

> 飞书项目知识 MCP 是可选能力,安装由独立引导文档承载,meegle-plugin-env-setup 不负责探测/安装。下游调 MCP 失败时按"无源即停"兜底(见 meegle-plugin-shared)。
