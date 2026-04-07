# mode=verify：验证工程初始化完成

## V1：检查 plugin.config.json 完整性

```
GET /sandbox/sessions/:id/files?path=plugin.config.json
```

验证以下字段均已填充（非空）：
- `pluginId` — 非空字符串
- `pluginSecret` — 非空（加密格式）
- `siteDomain` — 合法 URL
- `name` — 非空字符串

### 失败

pluginId 为空 → 提示 `npx @byted-meego/cli@builder create 可能未成功写入凭证`，回到 apply A2 重新执行。

## V2：检查工程目录结构

在沙盒中执行：
```bash
ls node_modules && ls src
```

- `node_modules/` 存在 → npm install 成功 ✅
- `src/` 存在 → 模板拉取成功 ✅

任一不存在 → 提示重新执行 `cli init`（A2）。

## V3：输出

```
✅ plugin-create 验证通过
   pluginId: PL_xxxxxxxxx
   工程目录：/workspace/{session_id}/
   下一步：配置点位（meego-point-config skill）
```
