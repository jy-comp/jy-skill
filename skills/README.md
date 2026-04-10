# Plugin Skills — Meego 插件开发 Skill 集合

本目录包含 Meego 插件开发的所有 Skill 文件。

## Skill 分工

```
用户指令
  │
  ├─ 原子操作（"启动调试"/"构建"/"同步配置"/"查看配置"）
  │     └─→ meego-cli  ── 单条 CLI 命令直接执行
  │
  ├─ 编排操作（"发布"/"加点位"/"生成代码"/"改名称"）
  │     └─→ 对应编排 skill 直接处理
  │         ├── plugin-publish      发布流程（update → release → publish）
  │         ├── meego-point-config  点位配置（schema → plan → set → update → verify）
  │         ├── plugin-code-gen     代码生成（setup → plan → apply → tsc verify）
  │         ├── plugin-polish       信息完善（analyze → generate → confirm → apply）
  │         └── plugin-create       创建插件（plan → create → verify）
  │
  ├─ 端到端需求（"做一个 xxx 功能"）
  │     └─→ plugin-workflow  ── 串联所有编排 skill
  │
  └─ 诊断查询（"什么状态"/"调试报错"）
        └─→ meego-cli  ── 读取项目状态 / 排查引导
```

**分工原则**：单条 CLI 命令能完成的用 meego-cli，需要多步串行/交互确认的用编排 skill。

## Skill 清单

| 类型 | Skill | 核心职责 |
|------|-------|---------|
| 原子命令 | **`meego-cli`** | CLI 命令参考 + 状态诊断 + 问题排查 |
| 端到端 | **`plugin-workflow`** | 需求 → 创建 → 配置 → 代码 → 发布 |
| 基础 | `meego-shared` | 认证、安全规则、项目识别、checkpoint 协议 |
| 编排 | `env-setup` | 环境检查 + OAuth 授权 |
| 编排 | `plugin-create` | 最小化创建插件工程 |
| 编排 | `meego-point-config` | 点位配置全量增删改 |
| 编排 | `plugin-code-gen` | AI 生成代码 + tsc 检查 |
| 编排 | `plugin-polish` | AI 总结生成名称/描述/分类 |
| 编排 | `plugin-publish` | update → release → publish 三步串行 |

## 目录结构

```
skills/
├── meego-cli/                 ← 原子命令 + 诊断
│   └── SKILL.md
├── plugin-workflow/           ← 端到端编排
│   ├── SKILL.md
│   └── references/
├── meego-shared/              ← 公共基础
│   └── SKILL.md
├── env-setup/                 ← 编排：环境准备
├── plugin-create/             ← 编排：创建插件
├── meego-point-config/        ← 编排：点位配置
├── plugin-code-gen/           ← 编排：代码生成
├── plugin-polish/             ← 编排：信息完善
└── plugin-publish/            ← 编排：构建发布
```

## 公共依赖

所有 skill 在执行前必须先读取 `meego-shared/SKILL.md`，其中包含：
- 插件工程识别规则（plugin.config.json + MII_ 前缀）
- CLI 可用性检查
- 统一认证（双 Token 机制）
- 安全规则（禁止编造 URL、禁止输出密钥、全量提交约束）
- Checkpoint 协议（workflow 进度追踪）
