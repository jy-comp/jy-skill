# Phase 2：功能实现（点位配置 + 代码生成 + 本地调试确认）

Phase 2 把"**功能可运行**"作为交付目标，内部委托给 `meegle-plugin-feature`（Stage Config 点位配置 + Stage Code 代码生成），完成后引导用户本地调试确认。

> **Checkpoint**：本阶段每个子步骤执行前后都需要更新 `.lpm-cache/state.json`，详见 SKILL.md「进度追踪」章节。

## 2.1 调用 meegle-plugin-feature（Stage Config + Stage Code）

把控制权直接交给 meegle-plugin-feature。**编排层不预判、不拉 schema、不填默认值、不替用户做选型**——点位识别（意图理解 / 术语消歧 / mentionedPointType 提取）、schema 切片、用户交互确认、URL 询问、MCP 业务语义查询、代码溯源、tsc 预检全部在 meegle-plugin-feature 的 references 里完成。编排层这里插手哪一项，那一项的护栏就会被绕过。

**Checkpoint 写入**：`{ "phase": 2, "step": "2.1", "stepName": "meegle-plugin-feature", "nextStep": "2.1 调用 meegle-plugin-feature pipeline" }`

**恢复检查**：若从 checkpoint 恢复，读 `.lpm-cache/state.json` 里 meegle-plugin-feature 自己维护的子状态（step 可能是 `2.config.plan` / `2.config.apply` / `2.code.plan` / `2.code.apply` / `2.code.verify` 等）——由 meegle-plugin-feature 决定从哪一步继续，编排层不替它解读 `local-config set` / `update` / `tsc` 的成功与否。

meegle-plugin-feature 的 config.plan 从 `.lpm-cache/state.json` 的 `context.originalRequirement` 读取用户原话做意图识别（Phase 0 逐字记录的版本，不受会话压缩影响），编排层不预消化、不传 hint 参数。

```
调用 /meegle-plugin-feature
```

meegle-plugin-feature 内部依次跑完：
1. **Stage Config（点位配置）**：config.setup → config.plan → config.apply → config.verify（点位 schema 推送远端 + 拉回本地）
2. **Stage Code（代码实现）**：code.setup → code.plan → code.apply → code.verify（tsc 通过）

完成后回写 checkpoint: `{ "step": "2.1", "stepName": "meegle-plugin-feature", "status": "success" }`，附带点位清单和 resources entry 列表。

## 2.2 启动本地调试

**Checkpoint 写入**：`{ "step": "2.2", "stepName": "本地调试", "nextCommand": "npx @byted-meego/cli@builder start --auto", "nextStep": "2.2 启动调试服务器" }`

meegle-plugin-feature 交付完成后，引导用户本地调试：

```
✅ 点位配置已推送远端 + 代码生成完成 + tsc 通过

现在启动本地调试看看效果：
  运行：npx @byted-meego/cli@builder start --auto

CLI 将自动打开浏览器调试页面，检查功能是否符合预期。
```

**等待用户反馈。** 这是整个流程中的关键交互点——是把用户推到 Phase 3 发布的唯一闸门。

## 2.3 用户反馈处理

### 场景 A：功能 OK

```
用户："可以了" / "没问题" / "功能正常"
```

→ 进入 Phase 3 发布

### 场景 B：需要调整

```
用户："图表颜色换成蓝色" / "按钮文案改成 xxx" / "加一个筛选功能"
```

判断调整范围：
- **仅代码层面改动**（样式 / 文案 / 逻辑调整）→ 回到 meegle-plugin-feature 的 Stage Code（`/meegle-plugin-feature stage=code`），code.plan → code.apply → code.verify
- **涉及点位配置变更**（加属性 / 改字段 / 新增点位类型）→ 回到 meegle-plugin-feature 完整流程（`/meegle-plugin-feature`），Stage Config 改配置，Stage Code 跟进代码

修改完成后提示用户刷新浏览器查看（webpack 热更新自动生效）。

**循环直到用户满意。**

### 场景 C：方向有误

```
用户："不是这个意思，我想要的是 xxx"
```

更新 checkpoint 的 `context.originalRequirement` 为用户新描述；调 `/meegle-plugin-feature` 完整重跑（Stage Config 重新识别点位、Stage Code 重新生成代码）。

### 场景 D：暂时搁置

```
用户："先这样，我改天再继续"
```

**Checkpoint 写入**：`{ "step": "2.3", "stepName": "用户调试中（暂停）", "lastCommand": "start --auto", "lastCommandStatus": "success", "nextStep": "2.3 等待用户反馈" }`

告知当前进度已保存，下次回来可以从断点继续：
- 代码在 `src/features/` 目录中
- 配置已推送远端
- 进度已记录到 `.lpm-cache/state.json`
- 随时可以 `/meegle-plugin-workflow phase=2` 继续调试，或 `/meegle-plugin-workflow phase=3` 直接发布

## 2.4 阶段完成标志

用户明确确认功能可以了 → 输出：

```
✅ 功能确认完成，准备发布

接下来我会：
1. 自动生成插件的正式名称 / 描述 / 分类（meegle-plugin-polish）
2. 向你展示发布清单，你确认后才实际发布（meegle-plugin-publish 不可逆）
3. 发布完成后输出分享链接

开始吗？
```

等待用户回复"开始" / "好" / "确认" 后进入 Phase 3。
