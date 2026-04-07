# mode=setup：环境检查

## S1：确认 env-setup 已完成

检查以下条件：
- `npx @byted-meego/cli@builder --version` 可正常执行
- `~/.lpm/auth.json` 存在且 `accessToken` 非空

若未满足 → 提示：`请先执行 /env-setup 完成环境准备`

## S2：获取 siteDomain

检查当前目录是否存在 `plugin.config.json`：

### 已有 plugin.config.json

从中读取 `siteDomain` 字段。若非空 → 直接使用。

### 无 plugin.config.json

向用户索要 Meego 页面 URL：

```
请提供一个 Meego 页面链接（如 https://meego.example.com/project/xxx），用于识别站点域名。
```

从 URL 中解析域名：
- `https://meego.example.com/project/xxx/...` → `https://meego.example.com`
- 提取规则：`new URL(url).origin`

解析后向用户确认：

```
识别到站点域名：https://meego.example.com
确认使用？
```

## 输出

```
✅ 环境就绪
✅ 站点域名：https://meego.example.com
```
