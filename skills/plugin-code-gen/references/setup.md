# mode=setup：环境检查

## 检查项

### S1：沙盒 Session 就绪

从对话上下文中获取 `session_id`，调用：
```
GET /sandbox/sessions/:session_id/status
```
- 200 → 继续
- 404 → 报错：`沙盒会话已过期`，终止

### S2：点位配置已完成

读取沙盒中的 `point.config.local-remote.json`（或当前 apply 生成的配置），确认：
- 至少有 1 个点位类型的配置
- 每个点位的 `key` 字段均已填写

若点位配置为空 → 提示：`请先完成点位配置（meego-point-config skill）`

### S3：plugin.config.json 中 pluginId 已填写

```
GET /sandbox/sessions/:id/files?path=plugin.config.json
```

确认 `pluginId` 非空。若为空 → 提示先完成 `plugin-create` skill。

### S4：工程模板已拉取

在沙盒中执行：
```bash
ls src/ && ls node_modules/
```

两个目录均存在 → 继续
任一不存在 → 提示先执行 `plugin-create mode=apply`

## 输出

```
✅ 沙盒 Session 存活
✅ 点位配置就绪（N 个点位，类型：board/dashboard/...）
✅ pluginId 已填写
✅ 工程模板就绪（src/ + node_modules/ 存在）
```
