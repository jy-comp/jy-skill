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

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../meego-shared/SKILL.md`](../meego-shared/SKILL.md)，其中包含认证、安全规则（含禁止修改 `.lpm/` 目录）、工具职责划分等公共约定。**
**CRITICAL — 执行前务必先用 Read 工具读取 [`references/setup.md`](references/setup.md)，禁止直接盲目执行。**

## 核心流程

```
S1 → 检查 CLI 可用性（npx @byted-meego/cli@builder --version）
S2 → 检查授权 Token（~/.lpm/auth.json）
     ├─ Token 有效 → 继续 S3
     └─ Token 缺失/过期 → Auth Guard（Device Code 授权）→ 继续 S3
S3 → 飞书项目知识 MCP（与 CLI / Token 一致的 soft 校验）
     ├─ S3a 探测（ASK + VERIFY 二段）已加载 → 完成
     └─ 未加载 → S3b 派子 agent 安装
                  ├─ 装成功 → 提示用户重启会话后重跑（装好的 MCP 当前会话物理上看不到）
                  ├─ 装失败 → 提示故障原因 + 手工补救建议
                  └─ host 不支持 → 提示切换 host
```

> **校验原则**：与 env-setup 现有 CLI / Token 检查保持一致——**soft 校验**（检测 + 提示），不写 checkpoint、不强制阻断下游。MCP 缺失时 AI 仍可基于 schema 推断，效果次优但不死。
>
> ⚠️ **平台无关**：本 skill 不绑定 Claude Code，AI agent（Claude Code / Cursor / Cline / Continue / Gemini CLI / Copilot CLI / 其他）应自行判断 host 类型，按各自的 MCP 注册机制完成安装。详见 `references/setup.md`。

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
