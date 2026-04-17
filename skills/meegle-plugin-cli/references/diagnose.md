# Meegle 插件状态诊断 & 常见问题排查

> 本文件是 [`../SKILL.md`](../SKILL.md) 的诊断手册补充。SKILL.md 里 B 类速查表的"详见"列均指向本文。

## 项目状态诊断

当用户询问当前状态、首次进入插件目录、或从中断处恢复时，执行以下诊断：

### 诊断步骤

1. 读取 `plugin.config.json` → 提取 pluginId、siteDomain、resources
2. 读取 `.lpm-cache/state.json`（如存在）→ 获取上次执行进度
3. 检查 `src/features/` 目录 → 判断代码是否已生成
4. 检查 `node_modules/` → 判断依赖是否已安装

### 状态判定与建议

| 条件组合 | 状态 | 建议操作 |
|---------|------|---------|
| 无 `plugin.config.json` | 非插件目录 | → `/meegle-plugin-workflow` 创建新插件，或 `cd` 到已有插件目录 |
| 有 `plugin.config.json`，`resources` 为空 | 已创建，未配置点位 | → `/meegle-plugin-feature` 配点位 + 生成代码 |
| 有 `plugin.config.json`，`resources` 非空，无 `src/features/` | 已配置，未生成代码 | → `/meegle-plugin-feature stage=code` 生成代码 |
| 有 `src/features/` 但代码为模板 | 已生成模板，未实现功能 | → `/meegle-plugin-feature stage=code` 填充功能 |
| 有 `src/features/` 且代码已实现 | 可调试或发布 | `start --auto` 调试 或 → `/meegle-plugin-publish` 发布 |
| `.lpm-cache/state.json` 存在 | workflow 中断 | 展示上次进度，引导继续（详见 checkpoint 恢复） |

### 输出格式

```
📍 插件状态诊断：
   插件 ID：MII_xxxxxxxxxxxx
   站点：https://meego.feishu-boe.cn
   点位资源：3 个（board_web × 1, button_web × 1, control_web × 1）
   代码状态：已实现（src/features/ 下 3 个目录）
   上次进度：Phase {phase} — {stepName}（来自 .lpm-cache/state.json；字段定义见 meegle-plugin-workflow/SKILL.md 的"Checkpoint 文件格式"）
   最近命令：{lastCommand} → {lastCommandStatus}

   建议：运行 `npx @byted-meego/cli@builder start --auto` 继续调试
```

---

## 常见问题排查

### "调试报错了" / "start 起不来"

1. 检查 `node_modules/` 是否存在 → 不存在则 `npm install`
2. 检查端口占用 → `lsof -i :3000`，如占用则 kill 对应进程
3. 检查 `plugin.config.json` 的 `resources` 是否为空 → 空则需先配置点位
4. 检查 Token 是否过期 → `~/.lpm/auth.json` 对应域名的 Token
5. 尝试 `npx @byted-meego/cli@builder start --auto` 查看具体报错

### "tsc 报错" / "代码编译错误"

1. 执行 `npx tsc --noEmit` 查看完整错误列表
2. 常见原因：
   - JSSDK API 属性名拼写错误（如 `Schedule` → `schedule`）
   - FeatureContext 字段不存在（各点位 context 字段不同，参考 `@lark-project/js-sdk` 类型定义）
   - union type 直接解构不安全（如 `ButtonFeatureContext` 需先判断 scene）
   - Control 和 CustomField API 混用（如在控件代码中调用 `customField.getProps()`）
   - **排查方式**：传参和返回值以 `node_modules/@lark-project/js-sdk/dist/types/index.d.ts` 为准；查 API 能力可走飞书项目知识 MCP
3. 可使用 `/meegle-plugin-feature stage=code` 走 code.verify 让 AI 自动修复（最多 2 轮）

### "能回退版本吗" / "撤销发布"

CLI 不支持版本回退。处理方式：
- 需要在 Meegle 后台（`{siteDomain}/openapp/{pluginId}`）手动管理版本（`siteDomain` 和 `pluginId` 从 `./plugin.config.json` 读取）

### "改一下代码" / "调整 xxx 逻辑"

直接修改 `src/features/{resource_id}/` 下的代码文件即可，不需要任何 CLI 命令。
修改后：
- 如果 `start` 正在运行 → webpack 热更新自动生效
- 如果未运行 → `start --auto` 启动调试查看效果
- 修改完成后想发布 → `/meegle-plugin-publish`
