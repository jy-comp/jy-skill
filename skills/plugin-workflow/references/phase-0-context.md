# Phase 0：确认站点域名 + 留存用户原话

> **关键规则**：Phase 0 在 Phase 1 开始前执行，做两件事：
> 1. 钉死 `siteDomain`（硬前置，拿到之前不得调任何 CLI 命令，不得默认填充）
> 2. **逐字记录用户启动 workflow 时说的那句话**（纯记录，**零解析**——不追问、不推导点位类型、不判空指令）
>
> 功能需求的意图识别、术语消歧、点位匹配全部留给 Phase 2 的 meego-point-config，Phase 0 只负责"录音"。

## 0.1 为什么要先问 siteDomain

siteDomain 是**所有 CLI 操作的定位根**——Token 按域名分 key，插件 create 必须指定目标站点，schema/publish 都依赖它。如果到 Phase 1 真要执行 CLI 时才问，AI 容易：

- 直接用"默认值"（常见是 project.feishu.cn）——**但用户可能在 Meegle 或自建站点**
- 不声不响地从训练数据里挑一个看起来像的——**结果 token/插件全错域名**

所以在动 CLI 之前，先把"在哪做"钉死。

## 0.1.1 为什么要把用户原话也落盘

用户启动时说的那句话（"我想做一个看板点位"、"加个拦截器校验 XX"等）是 Phase 2 意图识别 / Phase 3 代码方案设计的**主输入**。但长链路 workflow 经过 CLI 输出、schema 切片、MCP 查询等多轮 tool call 后，Claude Code 可能对会话做自动压缩，原话措辞会失真——而意图识别对"拓展字段 / 字段扩展 / 自定义字段"这类措辞差异敏感。checkpoint 落盘一份逐字原话作为恢复锚点：压缩后失真、或几小时后从 checkpoint 恢复时，都能拿到准确原话。

## 0.2 检查是否已有上下文（可能免问）

**优先级 1 — 当前目录已是插件工程**：
如果 cwd 下有 `plugin.config.json` 且 `siteDomain` 字段非空
→ 直接用该值，跳过 0.3 询问，记录到 checkpoint `context.siteDomain`。

**优先级 2 — checkpoint 已记录**：
如果 `.lpm-cache/state.json` 的 `context.siteDomain` 存在且非空
→ 把该值作为**推荐项展示给用户并请求确认**：

```
检测到上次使用的站点：<siteDomain>
是否继续使用？(Y) 是 / (N) 换一个
```

- 用户回复 Y / 是 / 继续 → 沿用该值，跳过 0.3。
- 用户回复 N / 否 / 换一个 → 进入 0.3 重新询问。
- 不得在用户未回复前自动沿用。

**优先级 3 — 都没有**：进入 0.3 显式询问。

## 0.3 向用户显式询问（不得默认填充）

```
在开始之前，先确认一下你要把插件部署到哪个 Meego 站点：

  1) 飞书项目（project.feishu.cn）
  2) Meegle（meegle.com）
  3) 自定义域名（请直接贴站点 URL，如 https://meego.your-company.com）

请告诉我 1 / 2 / 3，或直接贴 URL。
```

**AI 硬约束**：
- ❌ **禁止**假设用户是 project.feishu.cn（即使概率最高）
- ❌ **禁止**从对话历史/训练数据"推断"用户的站点
- ❌ **禁止**填 `https://meego.example.com` 或任何占位符（CLI 的 `ensureRealSiteDomain` 会拒绝,但在 skill 层就该先拦住）
- ✅ **必须**等到用户明确回复后再继续

## 0.4 记录到 checkpoint

```json
{
  "context": {
    "siteDomain": "https://project.feishu.cn",
    "originalRequirement": "<用户启动 workflow 时说的那句话，逐字保留>"
  }
}
```

- `originalRequirement` 是纯文本逐字记录，**不做任何后处理**（不提取 mentionedPointType、不判空指令、不追问）
- 后续 Phase 2（meego-point-config P1 意图识别）和 Phase 3（plugin-code-gen plan 方案设计）从这里读原话

## 0.5 产出

- `context.siteDomain`（完整 URL,含 scheme）
- `context.originalRequirement`（用户原话，逐字）

→ 进入 Phase 1 搭建工程。

---

**恢复语义**：如果 workflow 从 checkpoint 恢复且 `siteDomain` 和 `originalRequirement` 都已存在，视为 Phase 0 已完成，直接进入目标 Phase。
