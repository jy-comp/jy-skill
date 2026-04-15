# Phase 0：确认站点上下文（最早必须完成的步骤）

> **关键规则**：Phase 0 **MUST 在 Phase 1 开始前执行完成**。在未拿到用户确认的 siteDomain 之前，**不得**进入任何后续 Phase、不得调任何 CLI 命令、不得默认填充"project.feishu.cn"或其它值。

## 0.1 为什么是 Phase 0

siteDomain 是**所有 CLI 操作的定位根**——Token 按域名分 key，插件 create 必须指定目标站点，schema/publish 都依赖它。如果在 Phase 1 聊完功能需求后才问，AI 容易：

- 直接用"默认值"（常见是 project.feishu.cn）——**但用户可能在 Meegle 或自建站点**
- 不声不响地从训练数据里挑一个看起来像的——**结果 token/插件全错域名**

所以在聊功能之前，**先把"在哪做"钉死**。

## 0.2 检查是否已有上下文（可能免问）

**优先级 1 — 当前目录已是插件工程**：
如果 cwd 下有 `plugin.config.json` 且 `siteDomain` 字段非空
→ 直接用该值，跳过 0.3 询问，记录到 checkpoint `context.siteDomain`。

**优先级 2 — checkpoint 已记录**：
如果 `.plugin-workflow-state.json` 的 `context.siteDomain` 存在且非空
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

拿到用户确认的 siteDomain 后：

```json
{
  "context": {
    "siteDomain": "https://project.feishu.cn"
  }
}
```

后续所有 Phase 从 `context.siteDomain` 取值，不重复询问。

## 0.5 产出

- 确定的 `siteDomain`（完整 URL,含 scheme）
- 写入 `.plugin-workflow-state.json` 的 `context.siteDomain`

→ 进入 Phase 1 理解需求。

---

**恢复语义**：如果 workflow 从 checkpoint 恢复且 `context.siteDomain` 已存在,视为 Phase 0 已完成,直接进入目标 Phase。
