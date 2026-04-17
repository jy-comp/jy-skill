# Meegle Plugin Skills

Meegle（飞书项目）插件开发的 Skill 集合。所有 skill 以 `meegle-plugin-` 为统一前缀，放在 [`skills/`](./skills/) 目录下。

## 入口总览

| 用户意图 | Skill |
|---------|------|
| 从零做一个新插件（"我要一个 xxx 插件"） | [`meegle-plugin-workflow`](./skills/meegle-plugin-workflow/SKILL.md) |
| 存量插件加/改功能（"加个点位"、"实现 xxx 功能"） | [`meegle-plugin-feature`](./skills/meegle-plugin-feature/SKILL.md) |
| 改插件名称/描述/分类 | [`meegle-plugin-polish`](./skills/meegle-plugin-polish/SKILL.md) |
| 发布上线 | [`meegle-plugin-publish`](./skills/meegle-plugin-publish/SKILL.md) |
| 原子 CLI 操作 + 状态诊断 | [`meegle-plugin-cli`](./skills/meegle-plugin-cli/SKILL.md) |
| 登录 / 环境准备 | [`meegle-plugin-env-setup`](./skills/meegle-plugin-env-setup/SKILL.md) |
| 创建空壳工程（通常由 workflow 内部调用） | [`meegle-plugin-create`](./skills/meegle-plugin-create/SKILL.md) |
| 共享规则（所有 skill 前置依赖） | [`meegle-plugin-shared`](./skills/meegle-plugin-shared/SKILL.md) |

## 详细说明

完整的路由分工、skill 清单、目录结构、设计原则见 [`skills/README.md`](./skills/README.md)。
