# mode=apply：提交到后台

## A1：更新名称和描述（I18N）

> `siteDomain` 和 `appKey` 自动从 `plugin.config.json` 读取，无需手动传入。

```bash
npx @byted-meego/cli@builder update-description \
  --name "<插件名称>" \
  --short "<短描述>" \
  --detail-description "<详情描述>"
```

> `--detail-description` 是纯文本，CLI 内部会自动转为飞书富文本编辑器格式（doc + doc_html + doc_text）。

## A2：更新分类（CATEGORY）

```bash
npx @byted-meego/cli@builder update-description \
  --category-ids "<id1>,<id2>"
```

> 分类更新失败不阻塞流程，仅提示用户可在后台手动设置。

## A3：输出

```
✅ 插件信息已更新
   名称：[插件名称]
   短描述：[短描述]
   分类：[分类名称]
   下一步：发布插件（/plugin-publish）
```

### 失败处理

| 错误类型 | 处理方式 |
|---------|---------|
| 权限不足 | 提示检查 Token，可重新 `/env-setup` |
| 网络超时 | 等待 5s 后重试一次 |
| 分类更新失败 | 仅 warn，不阻塞 |
