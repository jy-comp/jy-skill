# mode=apply：同步配置 + 构建上传 + 版本发布

**Checkpoint 恢复检查**（仅在 `.lpm-cache/state.json` 存在时）：
- `lastCommand` 含 `update` + `"success"` → 跳过 A1，直接执行 A2
- `lastCommand` 含 `release` + `"success"` → 跳过 A1+A2，直接执行 A3（从 `context.productVersion` 获取版本号）
- `lastCommand` 含 `publish` + `"success"` → 全部完成，跳到 A4 输出
- 其他 → 从 A1 开始

## A1：同步配置到后台（兜底）

**Checkpoint**：执行前写入 `{ nextCommand: "update --source-type=local", nextStep: "A1 同步配置", lastCommandStatus: "running" }`

> 若在点位配置阶段已执行 update，此步为兜底确认。

```bash
npx @byted-meego/cli@builder update --source-type=local 
```

- 成功 → **Checkpoint**：`{ lastCommand: "update --source-type=local", lastCommandStatus: "success", nextCommand: "release", nextStep: "A2 构建上传" }` → 继续 A2
- 失败 → **Checkpoint**：`{ lastCommand: "update --source-type=local", lastCommandStatus: "failed" }` → 展示错误，终止

## A2：构建 + 上传（release）

**Checkpoint**：执行前写入 `{ nextCommand: "release", nextStep: "A2 构建上传", lastCommandStatus: "running" }`

```bash
npx @byted-meego/cli@builder release
```

**耗时操作**（约 30–120s），需展示构建进度。

### 成功标志

从 stdout 中提取产物版本号：

```
Product version: 1
(Use this as --product-version when running publish)
```

正则提取：`/Product version: (\S+)/`

> 注意：产物版本号是后端返回的内部版本（如 `1`、`2`、`3`），不是语义化版本号。直接作为 `--product-version` 传给 publish 即可。

将提取到的版本号记为 `productVersion`，作为下一步 publish 的入参。

**Checkpoint**（release 成功后，**关键**）：`{ lastCommand: "release", lastCommandStatus: "success", nextCommand: "publish ...", nextStep: "A3 版本发布", context: { ..., productVersion: "<提取到的版本号>" } }`

> **必须将 `productVersion` 保存到 checkpoint 的 `context` 中**，否则中断恢复后无法跳过 release 直接 publish。

### 失败处理

**Checkpoint**：`{ lastCommand: "release", lastCommandStatus: "failed" }`

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

**Checkpoint**：执行前写入 `{ nextCommand: "publish --product-version <X>", nextStep: "A3 版本发布", lastCommandStatus: "running" }`

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

**Checkpoint**：`{ lastCommand: "publish", lastCommandStatus: "success", nextStep: "A4 输出" }`

`npx @byted-meego/cli@builder publish` 输出分享链接：

```
🍻🍻🍻 Publish successfully! Current version is 1.0.1.
Plugin share url: https://meego.example.com/openapp/plugin_share?appKey=PL_xxx
```

### 失败处理

**Checkpoint**：`{ lastCommand: "publish", lastCommandStatus: "failed" }`

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

> `publish` 成功时 CLI 会自动清理整个 `.lpm-cache/`（仅保留 `state.json` 供 workflow 断点恢复）；发布失败或用户中断则不清理，保留缓存便于续跑。AI 不需要手动 `rm`。
