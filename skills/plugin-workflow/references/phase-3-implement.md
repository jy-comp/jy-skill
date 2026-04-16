# Phase 3：实现功能

生成代码并让用户本地调试确认。本阶段是**唯一需要用户深度参与**的环节。

> **Checkpoint**：本阶段每个子步骤执行前后都需要更新 `.lpm-cache/state.json`。

## 3.1 生成代码 → plugin-code-gen

**Checkpoint 写入**：`{ "phase": 3, "step": "3.1", "stepName": "plugin-code-gen", "nextCommand": "npx @byted-meego/cli@builder update", "nextStep": "3.1 拉取远端配置 + 生成模板" }`

**恢复检查**：若从 checkpoint 恢复：
- `step="3.1"` + `lastCommand` 含 `update` + `status="success"` → 模板已生成，跳到代码填充
- `step="3.1"` + `lastCommand` 含 `tsc` + `status="success"` → 代码已通过检查，跳到 3.2
- `step="3.1"` + `lastCommand` 含 `tsc` + `status="failed"` → 提示"上次 tsc 检查失败，正在重新修复"

```
调用 /plugin-code-gen mode=pipeline
```

AI 根据点位类型和用户的功能需求生成对应的 React 代码。code-gen 内部会：
1. 分析点位列表，规划代码骨架
2. 生成 `src/features/{type}/{key}/index.tsx`
3. 更新 `plugin.config.json` 的 resources
4. 执行 `tsc --noEmit` 语法检查（失败则自动修复，最多 2 轮）

code-gen 内部的 checkpoint 更新：
1. `update`（拉取远端配置） → `lastCommand="update"`, `nextCommand="tsc --noEmit"`
2. 代码填充完成 → `lastCommand="code-gen apply"`, `nextCommand="tsc --noEmit"`, `nextStep="3.1 语法检查"`
3. `tsc --noEmit` → `lastCommand="tsc --noEmit"`, `lastCommandStatus="success/failed"`

## 3.2 启动本地调试

**Checkpoint 写入**：`{ "step": "3.2", "stepName": "本地调试", "nextCommand": "npx @byted-meego/cli@builder start --auto", "nextStep": "3.2 启动调试服务器" }`

代码生成且 tsc 通过后，引导用户启动调试：

```
✅ 代码生成完成

现在启动本地调试看看效果：
  运行：npx @byted-meego/cli@builder start --auto

CLI 将自动打开浏览器调试页面，检查功能是否符合预期。
```

**等待用户反馈。** 这是整个流程中的关键交互点。

## 3.3 用户反馈处理

### 场景 A：功能 OK

```
用户："可以了" / "没问题" / "功能正常"
```

→ 进入 Phase 4

### 场景 B：需要调整

```
用户："图表颜色换成蓝色" / "按钮文案改成 xxx" / "加一个筛选功能"
```

AI 根据反馈修改代码 → 重新 tsc 检查 → 提示用户刷新页面查看。

**循环直到用户满意。**

### 场景 C：方向有误

```
用户："不是这个意思，我想要的是 xxx"
```

更新 checkpoint 的 `context.originalRequirement` 为用户新描述；如需变更点位类型回到 Phase 2.2 重新选型，否则直接在 Phase 3.1 按新原文重新生成代码。

### 场景 D：暂时搁置

```
用户："先这样，我改天再继续"
```

**Checkpoint 写入**：`{ "step": "3.3", "stepName": "用户调试中（暂停）", "lastCommand": "start --auto", "lastCommandStatus": "success", "nextStep": "3.3 等待用户反馈" }`

告知当前进度已保存，下次回来可以从断点继续：
- 代码在 `src/features/` 目录中
- 配置已推送远端
- 进度已记录到 `.lpm-cache/state.json`
- 随时可以 `/plugin-workflow phase=3` 继续调试，或 `/plugin-workflow phase=4` 直接发布

## 3.4 阶段完成标志

用户明确确认功能可以了 → 输出：

```
✅ 功能确认完成，准备发布

接下来我会：
1. 自动生成插件的正式名称和描述
2. 选择合适的分类
3. 构建并发布

开始吗？
```
