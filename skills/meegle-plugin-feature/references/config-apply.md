# mode=apply：校验 + 推送远端

## 前置

- 需要 plan 阶段生成的 `.lpm-cache/config/draft-{timestamp}.json`
- 若无该文件，先执行 `mode=plan`

**Checkpoint 恢复检查**（仅在 `.lpm-cache/state.json` 存在时）：
- `lastCommand` 含 `local-config set` + `"success"` → 跳过 A1，直接执行 A2
- `lastCommand` 含 `update --source-type=local` + `"success"` → 跳过 A1+A2，直接执行 A3
- `lastCommand` 含 `update --source-type=remote` + `"success"` → 跳过 A1+A2+A3，直接执行 A4
- （兼容老 checkpoint）`lastCommand` 含 `update`（不带 source-type，老命名）+ `"success"` → 等同 `--source-type=remote`，跳过 A1+A2+A3，直接执行 A4
- 其他 → 从 A1 开始

## A0：Diff 检查 — 删除确认（CRITICAL · 根原则 2 数据完整性 · config.apply 实施点）

> **规则定义**：见 [`../../meegle-plugin-shared/SKILL.md`](../../meegle-plugin-shared/SKILL.md) 的"删除点位前置检查协议"。本节是该规则在 Stage Config 的 config.apply 阶段的强制执行步骤，描述"如何检查"而不复述"为什么"。

**无论从哪个入口进入 apply（pipeline、meegle-plugin-workflow、单独调用），此步骤都 MUST 执行。**

**基线**：对比 `.lpm-cache/config/remote.json`（plan 阶段 `lpm local-config get --remote` 拉取的远端当前状态）——这个基线能防并发覆盖（别人在 plan 和 apply 之间刚推过点位）。若文件缺失，重跑 `lpm local-config get --remote` 刷新。

**用 jq 做 key 集合比较，不要 Read 整 JSON 进上下文**：

```bash
jq -r '[.[][].key] | sort | .[]' .lpm-cache/config/remote.json > /tmp/remote-keys.txt
jq -r '[.[][].key] | sort | .[]' .lpm-cache/config/draft-<timestamp>.json > /tmp/draft-keys.txt

# 远端有但 draft 没有的 key = 即将被删除的点位
comm -23 /tmp/remote-keys.txt /tmp/draft-keys.txt
```

处理：
- **输出为空**（无 key 减少）→ 直接进 A1
- **输出有 key**（有点位会被删）→ 逐个 jq 取 `type + key + name` 展示给用户确认：
  ```bash
  jq --arg k "<key>" 'to_entries[] | .key as $t | .value[] | select(.key==$k) | {type: $t, key, name}' .lpm-cache/config/remote.json
  ```
  等用户明确确认再进 A1；用户拒绝 → 修正 draft 后重跑 A0。

> **不可跳过**。即使 plan 阶段已确认过删除，apply 仍要再走一次——draft 可能被手改，apply 也可能不是从 plan 连续进入。

## A0.5：Runtime URL 占位符硬阻（CLI 强制，无需 AI 手动扫）

**规则体已下沉到 CLI**：`local-config set` 和 `update --source-type=local` 在提交前会自动扫 Runtime URL 占位符，命中占位符模式（`example.com` / `your-server` / `.test`/`.example`/`.invalid` / `localhost` / `<PLACEHOLDER:...>` 等）即 stderr 打印详细违规清单并 `exit 1`，不允许绕过。

**AI 的职责**：
- plan 阶段 P3 已主动询问用户真实 URL，正常走流程即可。
- 遇到 CLI 的 Runtime URL 占位符校验报错 → 转呈用户 → 用户提供真实 URL → 改配置后重试。**不要尝试"换个占位符绕过"——会被同一套规则再次拦下**。

**CLI 扫描范围**：
- 顶层：`intercept[].url` / `listen_event[].url` / `control[].url`
- 数据接口（仅当 DSL `definitions.data` 非空时必填）：`control[].platform.web.table_url.url` / `customField[].platform.web.table_data_url`
- DSL 节点：递归扫 `table_cell` / `table_layout` 的 template 树里所有 `props.onClick.params.url` / `props.onDoubleClick.params.url`（当 `action=httpRequest`，或 `action=openLink` 且 URL 非 `{{varName}}` 引用时）

此 lint 覆盖所有调用路径：pipeline / meegle-plugin-workflow / 单独 apply 入口进入均会触发，因为规则在 CLI 层,AI 的流程分支无法绕开。

## A1：Schema 校验（local-config set）

**Checkpoint**：执行前写入 `{ nextCommand: "local-config set ...", nextStep: "A1 Schema 校验", lastCommandStatus: "running" }`

```bash
npx @byted-meego/cli@builder local-config set --from .lpm-cache/config/draft-{timestamp}.json
```

> `--from` 接受 draft 文件路径（相对或绝对均可）。CLI 成功推送后会自动删除该 draft 以及 `.lpm-cache/config/remote.json` 基线——AI 不需要手动清理。

### 成功

**Checkpoint**：`{ lastCommand: "local-config set", lastCommandStatus: "success", nextCommand: "update --source-type=local", nextStep: "A2 推送远端" }`

继续执行 A2。

### 失败 — 错误分类处理

**Checkpoint**：`{ lastCommand: "local-config set", lastCommandStatus: "failed" }`

| 错误类型 | 处理方式 |
|----------|---------|
| 必填字段缺失 | 向用户补充询问，更新配置文件，**重新执行 A1** |
| 枚举值非法 | 列出合法值，询问用户选择，更新后**重新执行 A1** |
| key 重复冲突 | AI 自动在 key 后追加数字后缀（如 `_2`、`_3`）生成不冲突的 key，更新配置后**重新执行 A1** |
| name 超长 | 提示最大长度限制，要求用户缩短，**重新执行 A1** |
| URL 模式违规（`must match "^https?://"`） | **禁止自创占位 URL**（包括 `.invalid`/`.test`/`.example` 等 RFC 保留 TLD）。暂停流程向用户索取真实 URL 或授权 placeholder 写法，见 `meegle-plugin-shared/SKILL.md` 的"规则冲突时的兜底"章节，确认后**重新执行 A1** |
| Token 缺失（`must have required property 'token'`） | 当 `table_url.url` / `url` 有值时，对应 `token` 由 CLI 自动生成（36 位 UUID），AI **不需要**手写。若仍报错，说明 CLI 版本过旧或生成链路失败，如实上报用户排查；**切勿**自行伪造 token 值 |
| 数值字段类型冲突（`must be number`） | schema 已把 `style.width/height/margin/padding/borderRadius/fontSize/...` 改为 `oneOf: [number, string]`，允许模板表达式（如 `"{{$container.width}}"`）。若仍报错，说明 CLI 本地 schema 版本过旧，引导用户更新 `@byted-meego/cli@builder` 到最新 builder 版本后**重新执行 A1** |
| `table_cell must be object` | schema 已支持 `table_cell` 同时接受 object 与 JSON string（向后兼容），CLI 归一化前置。若报错说明 schema 版本过旧，同上引导用户升级 CLI 后**重新执行 A1**；**切勿**为了绕过而擅自把 string 改写成 object（会丢失"向后兼容"测试目的） |
| 未知错误 | 展示原始错误给用户，共同分析后**重新执行 A1** |

循环直至 set 成功。

> **通用原则：** 校验失败时 AI 的默认动作是"停下来问用户"，不是"悄悄发明绕过方案"。任何自主修正都必须（1）不改变被测语义；（2）明确在对话中说明；（3）记入最终产物的 TODO 标记。

## A2：推送远端（update）

**Checkpoint**：执行前写入 `{ nextCommand: "update --source-type=local", nextStep: "A2 推送远端", lastCommandStatus: "running" }`

```bash
npx @byted-meego/cli@builder update --source-type=local
```

- 成功 → **Checkpoint**：`{ lastCommand: "update --source-type=local", lastCommandStatus: "success", nextCommand: "update --source-type=remote", nextStep: "A3 拉取远端配置" }` → 继续执行 A3
- 失败 → **Checkpoint**：`{ lastCommand: "update --source-type=local", lastCommandStatus: "failed" }` → 报错展示，终止

## A3：拉取远端配置到本地（update --source-type=remote）

**Checkpoint**：执行前写入 `{ nextCommand: "update --source-type=remote", nextStep: "A3 拉取远端配置", lastCommandStatus: "running" }`

> ⚠️ **此步骤不可跳过**。A2 的 `--source-type=local` 只**推送配置到后端**，**不会**生成 `plugin.config.json` 中的 `resources` 数组和本地 entry 模板代码——这些是后端基于配置生成的产物，必须再走一次 `--source-type=remote` 才会被 CLI 拉下来。
>
> 跳过 A3 的后果：plugin.config.json 中没有 resources、本地 src/ 没有对应 entry 文件，下游的 Stage Code 会在 code.setup 的第一步就找不到代码模板。

```bash
npx @byted-meego/cli@builder update --source-type=remote
```

> `update --source-type=remote` 让 CLI 把远端配置同步回本地，保证本地与远端一致——不要自己改 `plugin.config.json`（规则见 [`../../meegle-plugin-shared/SKILL.md`](../../meegle-plugin-shared/SKILL.md#安全规则)）。

A3 完成后 CLI 会做以下三件事（对照验证）：
1. 拉取远端最新配置（含 A2 推送进去的点位）
2. 在 `plugin.config.json.resources` 数组中补齐每个点位对应的 resource id + entry 路径
3. 在 `src/features/<resourceId>/index.tsx` 生成模板代码

- 成功 → **Checkpoint**：`{ lastCommand: "update --source-type=remote", lastCommandStatus: "success", nextStep: "A4 输出" }` → 继续执行输出
- 失败 → **Checkpoint**：`{ lastCommand: "update --source-type=remote", lastCommandStatus: "failed" }` → 报错展示，但不影响已推送的远端配置

## A4：输出

```
✅ 配置已成功推送至远端
   变更：[添加/修改/删除] page[test_board_v1]
```
