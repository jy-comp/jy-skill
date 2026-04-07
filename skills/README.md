# Plugin Skills — Meego 插件开发 Skill 集合

本目录包含 Meego 插件开发的所有 Skill 文件，供开发者在本地完成插件的开发、调试和发布。

## 推荐入口

**用户只需说"我需要一个 xxx 功能"���使用 `plugin-workflow` 自动串联所有步骤。**

```
/plugin-workflow    ← 端到端全流程，面向用户的顶层入口
```

## 完整执行流程

```
plugin-workflow（顶层编排）
│
├── Phase 1：理解需求
│   └── 分析功能 → 推导点位类型 → 用户确认
│
├── Phase 2：搭建工程
│   ├── env-setup           → 环境准备（如需要）
│   ├── plugin-create       → 最小化创建插件工程
│   └── meego-point-config  → 配置功能点位
│
├── Phase 3：实现功能
│   ├── plugin-code-gen     → AI 生成代码
│   └── 本地调试            → 用户确认功能 OK
│
└── Phase 4：发布上线
    ├── plugin-polish       → AI 总结生成名称/描述/分类
    └── plugin-publish      → 构建 + 发布 + 输出分享链接
```

## Skill 清单

| Skill | 核心职责 | 主要命令 |
|-------|---------|---------|
| **`plugin-workflow`** | **端到端编排：需求 → 创建 → 开发 → 发布** | 串联以下所有 skill |
| `meego-shared` | 公共基础：认证、安全规则、错误处理 | 所有 skill 的前置依赖 |
| `env-setup` | 环境检查 + Device Code OAuth 授权 | `npx @byted-meego/cli@builder --version` / `login --site-domain` |
| `plugin-create` | 最小化创建插件工程骨架 | `npx @byted-meego/cli@builder create` |
| `meego-point-config` | 点位配置全量增删改 | `local-config set` + `update --source-type=local` |
| `plugin-code-gen` | AI 生成各点位代码 + 引导本地调试 | AI 直接生成 + `npx @byted-meego/cli@builder start --source_type local --auto` |
| `plugin-polish` | 发布前 AI 总结生成名称/描述/分类 | `npx @byted-meego/cli@builder update-description` + `list-categories` |
| `plugin-publish` | 同步配置 + 构建上传 + 版本发布 | `update` + `release` + `publish` |

## 目录结构

```
skills/
├── README.md                          # 本文件
├── meego-shared/                      # 公共基础（认证、安全规则）
│   └── SKILL.md
├── schema/                            # 点位 schema 定义
├── plugin-workflow/                   # 顶层编排（入口）
│   ├── SKILL.md
│   └── references/
│       ├── phase-1-understand.md
│       ├── phase-2-scaffold.md
│       ├── phase-3-implement.md
│       └── phase-4-release.md
├── env-setup/                         # 环境准备
│   ├── SKILL.md
│   └── references/
│       └── setup.md
├── plugin-create/                     # 插件创建（最小化）
│   ├── SKILL.md
│   └── references/
│       ├── setup.md
│       ├── plan.md
│       ├── apply.md
│       └── verify.md
├── meego-point-config/                # 点位配置管理
│   ├── SKILL.md
│   └── references/
│       ├── setup.md
│       ├── plan.md
│       ├── apply.md
│       └── verify.md
├── plugin-code-gen/                   # 代码生成 + 调试引导
│   ├── SKILL.md
│   └── references/
│       ├── setup.md
│       ├── plan.md
│       ├── apply.md
│       └── verify.md
├── plugin-polish/                     # 发布前信息完善
│   ├── SKILL.md
│   └── references/
│       ├── analyze.md
│       ├── generate.md
│       ├── confirm.md
│       └── apply.md
└── plugin-publish/                    # 完整发布流程
    ├── SKILL.md
    └── references/
        ├── pre-check.md
        ├── apply.md
        └── verify.md
```

## 公共依赖

所有 skill 在执行前必须先读取 `meego-shared/SKILL.md`，其中包含：
- CLI 可用性检查
- 统一认证（一次授权，全功能可用）
- 安全规则（禁止输出密钥、写操作需确认、全量提交约束）
- 错误处理指引

## 单独使用各 skill

各 skill 也可独立调用，适用于只需执行某一步的场景：

```
/env-setup                  # 仅环境准备
/plugin-create              # 仅创建工程
/meego-point-config         # 仅配置点位
/plugin-code-gen            # 仅生成代码
/plugin-polish              # 仅完善信息
/plugin-publish             # 仅发��
```
