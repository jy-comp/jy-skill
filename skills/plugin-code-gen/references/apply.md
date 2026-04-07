# mode=apply：生成代码 + 写入沙盒 + 更新配置

## A1：为每个点位生成代码文件

根据 plan 阶段的文件清单，按顺序生成每个点位的代码：

对于每个点位 `{type}/{key}`：

1. AI 根据点位类型的骨架模板生成完整 `index.tsx`
2. 根据用户对插件功能的描述，填充业务逻辑
3. 写入沙盒：

```
POST /sandbox/sessions/:id/files
Body: {
  "path": "src/features/{type}/{key}/index.tsx",
  "content": "<生成的 TypeScript 代码>"
}
```

**文件路径规则**：
- `type` = 点位类型（board/dashboard/button/control/config/intercept/...）
- `key` = 点位配置中的 `key` 字段（即 resourceId）
- 文件名固定为 `index.tsx`

## A2：更新 plugin.config.json 的 resources 字段

读取当前 `plugin.config.json`，更新 `resources` 字段，为每个点位添加 entry 映射：

```json
{
  "pluginId": "PL_xxx",
  "pluginSecret": "...",
  "siteDomain": "...",
  "resources": {
    "board": [
      {
        "key": "jira_board_main",
        "entry": "src/features/board/jira_board_main/index.tsx"
      }
    ],
    "dashboard": [
      {
        "key": "detail_tab",
        "entry": "src/features/dashboard/detail_tab/index.tsx"
      }
    ]
  }
}
```

写入沙盒：
```
POST /sandbox/sessions/:id/files
Body: { "path": "plugin.config.json", "content": "<更新后的 JSON>" }
```

## A3：打包工程 zip（可选，按需生成）

若用户请求下载代码，或后续 release 失败需要降级，在沙盒内打包：

```bash
# 通过 exec 调用
cd /workspace/{sessionId} && zip -r project.zip . --exclude "node_modules/*" --exclude ".git/*"
```

打包完成后，前端通过以下链接下载：
```
GET /sandbox/sessions/:id/files?path=project.zip
```

在对话中输出 `code_download` Artifact 卡片，提供下载链接。

## A4：输出摘要

```
✅ 代码生成完成
   生成文件：
   • src/features/board/jira_board_main/index.tsx
   • src/features/dashboard/detail_tab/index.tsx
   plugin.config.json resources 已更新

   [下载代码 zip] — 可在本地查看和调试
   下一步：执行语法检查（mode=verify）
```
