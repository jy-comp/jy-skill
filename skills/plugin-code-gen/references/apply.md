# mode=apply：生成代码 + 写入沙盒 + 更新配置

## A1：为每个点位生成代码文件

根据 plan 阶段的文件清单，按顺序生成每个点位的代码。

**文件路径来源**：直接使用 `plugin.config.json` 中 `resources` 各点位的 `entry` 路径，**禁止自行拼接路径**。

对于每个点位：

1. 从 `plugin.config.json` 的 `resources` 中读取该点位的 `entry` 路径
2. AI 根据点位类型的骨架模板生成完整代码文件
3. 根据用户对插件功能的描述，填充业务逻辑
4. 将代码写入 `entry` 指定的路径

> **禁止修改 `plugin.config.json`**。entry 路径和 resources 结构由 CLI `update` 命令生成并维护，代码生成只负责在对应路径下创建代码文件。

## A2：打包工程 zip（可选，按需生成）

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

## A3：输出摘要

```
✅ 代码生成完成
   生成文件：
   • src/features/board/jira_board_main/index.tsx
   • src/features/dashboard/detail_tab/index.tsx

   [下载代码 zip] — 可在本地查看和调试
   下一步：执行语法检查（mode=verify）
```
