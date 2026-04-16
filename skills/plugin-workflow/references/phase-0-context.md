# Phase 0：采集上下文（最早必须完成的步骤）

> **关键规则**：Phase 0 **MUST 在 Phase 1 开始前执行完成**，采集两项上下文：**站点域名 + 用户原始需求**。在未拿到两者之前，**不得**进入任何后续 Phase、不得调任何 CLI 命令、不得默认填充"project.feishu.cn"或其它值。

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

## 0.4 采集原始需求（最小闸口）

用户的原始描述即本次 workflow 的**唯一真实来源**，后续 Phase 2 点位匹配、Phase 3 代码生成都从此读取，不依赖对话记忆。

**判定规则**（宽松）：只要用户原话里包含一个动词 + 对象即可（"加个按钮"、"做一个看板"都算）。完全无功能信息的空指令（如"帮我做个插件"）才触发追问：

```
你想让这个插件解决什么具体问题？一句话说清楚即可。
```

**禁止**：展开功能确认模板、主动推导点位类型、询问字段/事件/propKey 等配置细节——这些都是后续 Phase 的工作。

用户原话里若**主动提到**点位类型（如"轻应用组件"、"按钮点位"）或具体字段/事件字符串，**原文记录**即可，本阶段不校验。

## 0.5 记录到 checkpoint

```json
{
  "context": {
    "siteDomain": "https://project.feishu.cn",
    "originalRequirement": "<用户原话,逐字保留>",
    "mentionedPointType": "<用户主动提到的点位 or null>"
  }
}
```

后续所有 Phase 从 checkpoint 取值，不重复询问、不再让用户确认已说过的内容。

## 0.6 产出

- `context.siteDomain`（完整 URL,含 scheme）
- `context.originalRequirement`（用户原话）
- `context.mentionedPointType`（可选）

→ 进入 Phase 1 搭建工程。

---

**恢复语义**：如果 workflow 从 checkpoint 恢复且上述字段已存在,视为 Phase 0 已完成,直接进入目标 Phase。
