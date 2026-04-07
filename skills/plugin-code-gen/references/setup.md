# mode=setup：环境检查

## 检查项

### S1：沙盒 Session 就绪

从对话上下文中获取 `session_id`，调用：
```
GET /sandbox/sessions/:session_id/status
```
- 200 → 继续
- 404 → 报错：`沙盒会话已过期`，终止

### S2：点位配置已推送远端
> **关键前置条件**：代码生成必须基于已推送到远端的点位配置，不能仅依赖本地文件。
从远端拉取当前配置并确认：

```bash
npx @byted-meego/cli@builder update
```
并确认：
- 至少有 1 个点位类型的 entry 配置
- 每个点位的 `id` 和 `entry` 路径均已填写

## 输出

```
✅ 沙盒 Session 存活
✅ 点位配置就绪（N 个点位，类型：board/dashboard/...）
```
