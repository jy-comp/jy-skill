# Phase 4：发布上线

完善插件信息并发布。用户只需最终确认。

> **Checkpoint**：本阶段每个子步骤执行前后都需要更新 `.lpm-cache/state.json`。

## 4.1 完善信息 → plugin-polish

**Checkpoint 写入**：`{ "phase": 4, "step": "4.1", "stepName": "plugin-polish", "nextCommand": "npx @byted-meego/cli@builder update-description ...", "nextStep": "4.1 完善插件名称/描述/分类" }`

**恢复检查**：若从 checkpoint 恢复：
- `step="4.1"` + `lastCommand` 含 `update-description` + `status="success"` → 描述已更新，跳到 4.2

```
调用 /plugin-polish mode=pipeline
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

**Checkpoint 更新**：`update-description` 成功后 → `lastCommand="update-description"`, `nextStep="4.2"`

## 4.2 发布 → plugin-publish

**Checkpoint 写入**：`{ "step": "4.2", "stepName": "plugin-publish", "nextCommand": "npx @byted-meego/cli@builder update", "nextStep": "4.2.1 同步配置" }`

**恢复检查**：若从 checkpoint 恢复，根据 `lastCommand` 精确判断：
- `lastCommand` 含 `update` + `status="success"` → 跳过 update，直接 release
- `lastCommand` 含 `release` + `status="success"` → 跳过 release，直接 publish（从 `context.productVersion` 获取版本号）
- `lastCommand` 含 `publish` + `status="success"` → 已发布，跳到 4.3

```
调用 /plugin-publish mode=pipeline
```

**AI 自动处理，无需用户逐项输入：**
- **版本号**：不传，默认在上一版本基础上 patch +1
- **版本描述**：AI 基于当前代码改动自动总结生成

发布流程（串行），每步更新 checkpoint：
1. `update` — 同步配置 → checkpoint: `lastCommand="update"`, `nextCommand="release"`, `nextStep="4.2.2 构建上传"`
2. `release` — 构建 + 上传 → checkpoint: `lastCommand="release"`, `nextCommand="publish"`, `nextStep="4.2.3 版本发布"`, `context.productVersion=<从输出解析>`
3. `publish` — 版本发布 → checkpoint: `lastCommand="publish"`, `lastCommandStatus="success"`

> **关键**：`release` 成功后必须将 `productVersion` 写入 checkpoint 的 `context`，这样中断恢复时无需重新 release。

## 4.3 完成

**发布成功后删除 `.lpm-cache/state.json`**（流程已完结，无需保留 checkpoint）。

```
🎉 插件发布成功！

  插件名称：[正式名称]
  版本：[自动累加的版本号]
  分享链接：[URL]

用户可以通过分享链接安装使用此插件。
```
