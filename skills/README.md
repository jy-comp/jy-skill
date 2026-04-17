# Meegle Plugin Skills — 插件开发 Skill 集合

本目录包含 Meegle（飞书项目）插件开发的所有 Skill 文件。所有 skill 以 `meegle-plugin-` 为前缀，按用户意图分工。

## 路由分工

```
用户指令
  │
  ├─ 新插件端到端（"从零做一个 xxx 插件"/"新建一个插件"/"我需要一个 xxx 插件"）
  │     └─→ meegle-plugin-workflow    ── 新插件全流程（create → feature → polish → publish）
  │
  ├─ 存量插件功能迭代（"加个点位"/"改功能"/"生成代码"/"实现功能"）
  │     └─→ meegle-plugin-feature     ── Stage Config 点位配置 + Stage Code 代码生成，线性绑定
  │
  ├─ 存量插件单点操作
  │     ├─ "改名称/描述/分类"        → meegle-plugin-polish
  │     └─ "发布/上线"               → meegle-plugin-publish
  │
  ├─ 原子 CLI 操作（"启动调试"/"构建"/"同步配置"/"查看配置"）
  │     └─→ meegle-plugin-cli         ── 单条 CLI 命令 + 状态诊断
  │
  └─ 基础（不直接被用户触发）
        ├─ meegle-plugin-shared       ── 共享规则（认证/安全/识别/checkpoint）
        ├─ meegle-plugin-env-setup    ── 登录编排（通常由上层 skill 触发）
        └─ meegle-plugin-create       ── 单独创建空壳工程（通常由 workflow 触发）
```

### 分水岭

- **"插件"作为整体** → `meegle-plugin-workflow`（新插件从零全流程）
- **"功能/点位"作为局部** → `meegle-plugin-feature`（存量插件 config + code 绑定）
- **单一不可组合操作** → 对应单点 skill（polish / publish）
- **原子命令 / 诊断** → `meegle-plugin-cli`

两个顶层 skill（workflow 和 feature）各有**前置守卫**：
- `meegle-plugin-workflow` 检测到 `plugin.config.json` 已存在 → 引导到 `meegle-plugin-feature`
- `meegle-plugin-feature` 检测到 `plugin.config.json` 不存在 → 引导到 `meegle-plugin-workflow`

即使 AI 路由误命中，守卫会把用户接回正确路径。

## Skill 清单

| 层级 | Skill | 核心职责 | 触发场景 |
|------|-------|---------|---------|
| 顶层 | **`meegle-plugin-workflow`** | 新插件端到端（Phase 0 录音 → 1 搭建 → 2 功能 → 3 发布） | "从零做一个 xxx 插件" |
| 顶层 | **`meegle-plugin-feature`** | 存量插件功能迭代（Stage Config 点位配置 + Stage Code 代码生成） | "加个功能/点位"、"改功能"、"生成代码" |
| 单点 | `meegle-plugin-polish` | AI 总结生成插件名称/描述/分类 | "完善插件信息"、"改名称/描述" |
| 单点 | `meegle-plugin-publish` | 发布三步串行：update → release → publish（不可逆） | "发布"、"上线"、"release" |
| 原子 | `meegle-plugin-cli` | 单条 CLI 命令参考 + 插件状态诊断 + 问题排查 | "启动调试"、"构建"、"当前状态" |
| 基础 | `meegle-plugin-shared` | 共享规则（认证、安全、识别、checkpoint 协议） | 不直接触发，所有 skill 前置依赖 |
| 基础 | `meegle-plugin-env-setup` | CLI 登录编排（Device Code OAuth / Developer Token） | auth 失败时被上层触发，或"登录"、"初始化环境" |
| 基础 | `meegle-plugin-create` | 最小化创建插件工程骨架 | 通常被 workflow Phase 1 调用，独立触发较少 |

## 目录结构

```
skills/
├── meegle-plugin-workflow/          ← 顶层：新插件全流程编排
│   ├── SKILL.md
│   └── references/
│       ├── phase-0-context.md       （siteDomain + 原话录音）
│       ├── phase-1-scaffold.md      （env-setup + create）
│       ├── phase-2-feature.md       （委托 feature + 本地调试确认）
│       └── phase-3-release.md       （polish + 发布前确认 + publish）
├── meegle-plugin-feature/           ← 顶层：存量插件功能迭代
│   ├── SKILL.md
│   └── references/
│       ├── config-setup.md / config-plan.md / config-apply.md / config-verify.md  （Stage Config）
│       ├── code-setup.md   / code-plan.md   / code-apply.md   / code-verify.md    （Stage Code）
│       └── point-types/liteAppComponent/    （点位标准能力 doc）
├── meegle-plugin-polish/            ← 单点：信息完善
│   ├── SKILL.md
│   └── references/ (analyze / generate / confirm / apply)
├── meegle-plugin-publish/           ← 单点：构建发布
│   ├── SKILL.md
│   └── references/ (pre-check / apply / verify)
├── meegle-plugin-cli/               ← 原子命令 + 诊断
│   ├── SKILL.md                     （路由表 + A/B 速查表）
│   └── references/ (commands / diagnose)
├── meegle-plugin-shared/            ← 基础：公共规则
│   ├── SKILL.md
│   └── references/ (checkpoint / url-policy)
├── meegle-plugin-env-setup/         ← 基础：登录编排
│   ├── SKILL.md
│   └── references/setup.md
└── meegle-plugin-create/            ← 基础：创建骨架
    ├── SKILL.md
    └── references/ (setup / plan / apply / verify)
```

## 公共依赖

所有 skill 在执行前必须先 Read `meegle-plugin-shared/SKILL.md`，其中包含：
- 插件工程识别规则（`plugin.config.json` + `MII_` 前缀）
- CLI 可用性检查
- 统一认证（Device Code OAuth / Developer Token）
- 安全规则（禁止编造 URL、禁止输出密钥、全量提交约束、删除点位确认协议）
- Checkpoint 协议（workflow 进度追踪 + 断点续跑）
- 无源即停通则（AI 输出必须可溯源到合法信息源）

## 设计演进（v2）

v2 重构要点（对比 v1）：
- **合并**：`meego-point-config` + `plugin-code-gen` → `meegle-plugin-feature`（功能迭代 Stage Config + Stage Code 串行绑定）
- **重命名**：所有 skill 加 `meegle-plugin-` 前缀，统一命名空间
- **收窄**：`meegle-plugin-workflow` 限定在"新插件"场景，加入 `plugin.config.json` 存在性守卫
- **双顶层入口**：workflow（新插件）与 feature（存量迭代）各自线性、互相引导
- **不可逆护栏**：publish 之前必须用户显式确认，即使在 workflow 新插件自动流程中也不例外
