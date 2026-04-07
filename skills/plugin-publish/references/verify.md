# mode=verify：确认发布成功

## V1：确认 release 成功

- `npx @byted-meego/cli@builder release` exit code 为 0
- 产物版本号已成功提取（格式 `X.Y.Z`）

若未能解析版本号：
1. 读取完整 stdout 日志，手动查找版本信息
2. 若仍找不到，询问用户确认版本号

## V2：确认 publish 成功

- `npx @byted-meego/cli@builder publish` exit code 为 0
- 输出中包含分享链接（格式 `https://.../openapp/plugin_share?appKey=...`）

## V3：输出

```
✅ 插件发布验证通过
   产物版本：1.0.1
   发布版本：1.0.0
   分享链接：https://meego.example.com/openapp/plugin_share?appKey=PL_xxx

插件已发布完成，用户可通过分享链接安装使用。
```
