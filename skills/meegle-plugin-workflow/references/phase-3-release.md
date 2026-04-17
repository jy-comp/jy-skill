# Phase 3：发布上线

完善插件元信息并发布到 Meegle 市场。用户在 publish 不可逆动作前必须显式确认。

> **Checkpoint**：本阶段每个子步骤执行前后都需要更新 `.lpm-cache/state.json`。

## 3.1 完善信息 → meegle-plugin-polish

**Checkpoint 写入**：`{ "phase": 3, "step": "3.1", "stepName": "meegle-plugin-polish", "nextCommand": "npx @byted-meego/cli@builder update-description ...", "nextStep": "3.1 完善插件名称/描述/分类" }`

**恢复检查**：若从 checkpoint 恢复：
- `step="3.1"` + `lastCommand` 含 `update-description` + `status="success"` → 描述已更新，跳到 3.2

```
调用 /meegle-plugin-polish mode=pipeline
```

AI 基于已实现的代码和点位配置自动：
1. **分析**：读取点位配置 + 扫描代码，理解插件实际功能
2. **生成**：
   - 正式插件名称（替换工作名称）
   - 短描述（≤100字）
   - 详情描述（功能介绍 + 使用方式 + 适用场景）
3. **选择分类**：拉取分类列表，推荐最匹配的 1-3 个
4. **确认**：展示给用户确认，支持逐项修改
5. **提交**：调用 `update-description` 更新到后台

**Checkpoint 更新**：`update-description` 成功后 → `lastCommand="update-description"`, `nextStep="3.2"`

## 3.2 发布前二次确认（不可逆护栏 — 不可跳过）

polish 完成后，**MUST** 向用户展示发布清单并等待显式确认，才能进入 3.3 publish。

```
即将发布插件到 Meegle 市场：

  插件名称：<正式名称>
  短描述：<短描述>
  分类：<选中的分类>
  点位：<点位清单，来自 resources>
  站点：<siteDomain>

⚠️ 发布是不可逆动作：发布后所有人可见，当前 CLI 不支持版本回退（只能在后台手动管理）。

确认发布？请回复"确认发布" / "取消" 或任何修改请求（如"名称改成 xxx"）。
```

**AI 硬约束**：
- ❌ **禁止**在用户未明确回复"确认发布" / "OK发布" / "好，发布吧"等肯定意图前执行 publish
- ❌ **禁止**把"开始 Phase 3"当作发布确认——Phase 3 开头的"开始吗"是 polish 的前置，不是 publish 的前置
- ✅ **必须**接受用户"改名称"/"改分类"等修改请求，回到 3.1 更新后再回到 3.2 重新确认
- ✅ **必须**接受用户"取消"，终止 workflow（保留 checkpoint 供后续续跑）

只有用户明确同意发布后，才进入 3.3。

## 3.3 发布 → meegle-plugin-publish

**Checkpoint 写入**：`{ "step": "3.3", "stepName": "meegle-plugin-publish", "nextCommand": "npx @byted-meego/cli@builder update", "nextStep": "3.3.1 同步配置" }`

**恢复检查**：若从 checkpoint 恢复，根据 `lastCommand` 精确判断：
- `lastCommand` 含 `update` + `status="success"` → 跳过 update，直接 release
- `lastCommand` 含 `release` + `status="success"` → 跳过 release，直接 publish（从 `context.productVersion` 获取版本号）
- `lastCommand` 含 `publish` + `status="success"` → 已发布，跳到 3.4

```
调用 /meegle-plugin-publish mode=pipeline
```

**AI 自动处理，无需用户逐项输入：**
- **版本号**：不传，默认在上一版本基础上 patch +1
- **版本描述**：AI 基于当前代码改动自动总结生成

发布流程（串行），每步更新 checkpoint：
1. `update` — 同步配置 → checkpoint: `lastCommand="update"`, `nextCommand="release"`, `nextStep="3.3.2 构建上传"`
2. `release` — 构建 + 上传 → checkpoint: `lastCommand="release"`, `nextCommand="publish"`, `nextStep="3.3.3 版本发布"`, `context.productVersion=<从输出解析>`
3. `publish` — 版本发布 → checkpoint: `lastCommand="publish"`, `lastCommandStatus="success"`

> **关键**：`release` 成功后必须将 `productVersion` 写入 checkpoint 的 `context`，这样中断恢复时无需重新 release。

## 3.4 完成

**发布成功后删除 `.lpm-cache/state.json`**（流程已完结，无需保留 checkpoint）。

```
🎉 插件发布成功！

  插件名称：<正式名称>
  版本：<自动累加的版本号>
  分享链接：<URL>

用户可以通过分享链接安装使用此插件。
```
