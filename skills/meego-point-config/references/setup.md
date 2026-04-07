# mode=setup：环境检查

## 检查项

1. **plugin.config.json 存在性**
   - 在当前目录查找 `plugin.config.json`
   - 不存在 → 报错终止：`请先初始化项目（npx @byted-meego/cli@builder init）`

2. **CLI 可用性**
   ```bash
   npx @byted-meego/cli@builder --version
   ```
   - 失败 → 报错终止：`请先安装 @byted-meego/cli@builder`

3. **认证有效性**（可选但推荐）
   - 尝试 `npx @byted-meego/cli@builder local-config get` 确认凭证有效
   - 失败不终止，仅警告

## 输出

setup 通过后，打印环境摘要：
```
✅ plugin.config.json 存在
✅ @byted-meego/cli 可用（版本 x.y.z）
✅ 认证有效
```

任何必要检查失败则终止整个 pipeline。
