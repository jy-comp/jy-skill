# mode=pre-check：TypeScript 语法预检 + 自动修复

> 与 plugin-code-gen 的 verify 阶段相同，若 code-gen 已执行过 verify 通过，可跳过此步骤。

## PC1：检查 plugin.config.json 的 resources 字段

读取沙盒中的 `plugin.config.json`，确认：
- `resources` 字段非空
- 至少有一个点位的 `entry` 字段指向存在的文件

```
GET /sandbox/sessions/:id/files?path=plugin.config.json
```

若 resources 为空 → 提示：`请先完成代码生成（plugin-code-gen skill）`

## PC1.5：Runtime URL 占位符硬阻止（由 CLI 强制）

**规则由 CLI 的 `validateRuntimeUrls` 强制**：配置推送到后台的唯一入口是 `local-config set` / `update --source-type=local`，两者都在提交前自动扫 Runtime URL 占位符（example.com / your-server / `.test` / `.example` / `.invalid` / localhost / `<PLACEHOLDER:...>` 等），命中即 `exit 1`。

**publish 命令本身不读点位配置**——它只包/发布已通过 set/update 推送过的配置。所以只要到达 publish 阶段，Runtime URL 已经通过 validator。这里作为概念节点保留，AI 无需再手动扫。

**若历史配置混入占位符（早于 validator 上线的遗留）**：用户需要在 meego-point-config 里替换为真实 URL，重新 `local-config set` 推送 → validator 拦假值 → 改对后通过 → 回到 publish。

## PC2：执行 tsc --noEmit

```bash
npx tsc --noEmit
```

约 10s，通过 SSE 展示输出。

### 成功（exit code 0）

直接进入 apply 阶段（`npx @byted-meego/cli@builder release`）。

### 失败 → 自动修复（最多 2 轮）

**修复流程**：

1. 解析 tsc 错误输出（格式：`文件:行号:列号 - error TSxxxx: 错误描述`）
2. AI 逐个分析错误，生成修复代码
3. 写入沙盒覆盖文件，重新 `tsc --noEmit`
4. 循环直至通过或达到上限

**修复上限**：2 轮。

### 2 轮后仍失败 → L3 降级

```
⚠️ TypeScript 预检失败，无法自动修复。

错误：[错误列表]

选项：
A. [下载代码 zip] — 本地修复后手动 npx @byted-meego/cli@builder release
B. [忽略并强制发布] — webpack 构建时可能再次报错，不推荐
```

**本地调试指引（随 zip 提供）**：
```bash
# 解压后
npm install
npx tsc --noEmit        # 查看语法错误
# 修复完成后
npx @byted-meego/cli@builder release
```
