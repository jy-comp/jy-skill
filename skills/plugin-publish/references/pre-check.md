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

## PC1.5：Runtime URL 占位符硬阻止（CRITICAL — 上线前不可跳过）

发布前对 `point.config.local.json` 做最后一道 **Runtime URL** 真实性扫描，完全复用 `meego-point-config/references/apply.md` 的 **A0.5 扫描规则**（扫描范围、可疑模式、触发条件同步维护，以 A0.5 为准）——涵盖：

- 顶层：`intercept.url` / `listen_event.url` / `control.url`
- 数据接口：`control.table_url.url`（当 table_cell 声明了变量）/ `customField.table_data_url`（当 table_layout 声明了变量）
- DSL 节点：任意 `onClick/onDoubleClick.params.url`（`action=httpRequest` 或字面量 `openLink`）

**与 A0.5 的关系**：规则完全一致，两处均为硬阻止、不提供绕过选项。PC1.5 作为 publish 阶段的二道防线，防止配置推送到后台后被手动改出占位 URL。

**输出示例**：
```
🔴 发布终止：以下 URL 仍是占位符，上线后对应功能会失败：
- intercept[intercept_abc]: url = https://example.com/callback
- control[control_xxx]: table_url.url = https://PLACEHOLDER.invalid/...
- control[control_yyy] table_cell 里某 httpRequest 节点: params.url = https://your-api/submit

请先在 meego-point-config 中补齐真实 URL，重新 push 远端，再重试 publish。
```

**恢复路径**：用户替换为真实 URL → 通过 `meego-point-config` 重新推送（会再经过 A0.5 检查，这次 URL 真实故通过） → 回到 `plugin-publish` 重试，此时 PC1.5 扫描通过，发布继续。

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
