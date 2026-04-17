# mode=apply：最小化创建插件工程

## A1：执行 npx @byted-meego/cli@builder create

```bash
npx @byted-meego/cli@builder create \
  --site-domain <siteDomain> \
  --name "<工作名称>" \
  --force
```

> `--force` 跳过当前目录已是插件仓库时的确认提示，确保 AI 可非交互执行。
> 不传 `--description` / `--detail-description`，这些在 meegle-plugin-polish 阶段填充。

该命令会：
1. 调用 API 创建插件，获取 pluginId + pluginSecret
2. 自动执行 init 初始化工程（拉模板 + 写入 plugin.config.json + npm install）

### 成功

```
🍻🍻🍻 Create successfully! Plugin ID: PL_xxx. Local project directory: my-plugin.
```

### 失败���理

| 错误类型 | 处理方式 |
|---------|---------|
| API 超时 | 等待 5s 后重试一次 |
| 权限不足 | 告知��户检查 Token 是否有效，可重新执行 `/meegle-plugin-env-setup` |
| 名称已存在 | 提示用户���改��称，重新执行 A1 |
| npm install 超时 | 等待 30s 后重试一次（最多 2 次） |
| 未知错误 | 展示原始错误，终止并告知用户 |

## A2：输出

```
✅ 插件工程创建成功
   pluginId: PL_xxxxxxxxx
   工程目��：my-plugin/
   下一步：配置功能点位（/meegle-plugin-feature）
```
