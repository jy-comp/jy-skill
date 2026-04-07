# mode=apply：同步配置 + 构建上传 + 版本发布

## A1：同步配置到后台（兜底）

> 若在点位配置阶段已执行 update，此步为兜底确认。

```bash
npx @byted-meego/cli@builder update --source-type=local
```

- 成功 → 继续 A2
- 失败 → 展示错误，终止整个发布流程

## A2：构建 + 上传（release）

```bash
npx @byted-meego/cli@builder release --source-type=local
```

**耗时操作**（约 30–120s），需展示构建进度。

### 成功标志

从 stdout 中提取产物版本号：

```
The latest version number is 1 / 2 / 3
```

正则提取：`/The latest version number is (\d+\.\d+\.\d+)/`

将提取到的版本号记为 `productVersion`，作为下一步 publish 的入参。

### 失败处理

**webpack 错误（自动修复最多 1 轮）**：

1. 捕获完整 stderr/stdout 中的 webpack 错误日志
2. AI 分析错误类型（常见：import 路径错误、缺少 module）
3. 修复代码文件，重新执行 `npx @byted-meego/cli@builder release`

若修复后成功 → 继续。若仍失败 → 终止并提示：

```
❌ 构建失败（已尝试自动修复）

webpack 错误：
[完整错误日志]

请在本地修复后手动执行：
  npx @byted-meego/cli@builder release
  npx @byted-meego/cli@builder publish --product-version <产物版本>
```

## A3：版本发布（publish）

release 成功后，AI 自动准备 publish 参数，**无需向用户逐个收集**：

| 参数 | 来源 | 说明 |
|------|------|------|
| `--version` | 自动 | 不传，默认在上一版本基础上 patch +1（如 `1.0.0` → `1.0.1`） |
| `--desc-zh` | AI 生成 | AI 基于当前代码改动（git diff / 文件变更）自动总结版本描述（中文） |
| `--product-version` | A2 输出 | 来自 release 输出的 productVersion |
| `--store` | 默认 | `no-publish`（默认不发布到商店） |
| `--upgrade` | 默认 | `manual`（默认手动升级） |

**AI 自动总结版本描述的方式**：
1. 检查 git diff 或已变更的源码文件
2. 总结功能改动要点，生成简洁的中文描述（如"新增看板点位，支持需求图谱展示"）

直接执行：

```bash
npx @byted-meego/cli@builder publish \
  --desc-zh "<AI 总结的版本描述>" \
  --product-version <A2 输出的 productVersion> \
  --store no-publish \
  --upgrade manual
```

> 如果用户主动指定了版本号，则加上 `--version <用户指定>`。

### 成功

`npx @byted-meego/cli@builder publish` 输出分享链接：

```
🍻🍻🍻 Publish successfully! Current version is 1.0.1.
Plugin share url: https://meego.example.com/openapp/plugin_share?appKey=PL_xxx
```

### 失败处理

| 错误类型 | 处理方式 |
|---------|---------|
| 版本号已存在 | 自动 patch +1 重试，或提示用户指定版本号 |
| 网络/权限错误 | 展示原始错误，告知用户可手动重试 |

## A4：输出

```
✅ 插件发布完成
   产物版本：1.0.1（release）
   发布版本：1.0.1（publish）
   分享链接：https://meego.example.com/openapp/plugin_share?appKey=PL_xxx
```
