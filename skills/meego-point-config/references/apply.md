# mode=apply：校验 + 推送远端

## 前置

- 需要 plan 阶段生成的 `plugin.temp.local-{timestamp}.json`
- 若无该文件，先执行 `mode=plan`

**Checkpoint 恢复检查**（仅在 `.plugin-workflow-state.json` 存在时）：
- `lastCommand` 含 `local-config set` + `"success"` → 跳过 A1，直接执行 A2
- `lastCommand` 含 `update --source-type=local` + `"success"` → 跳过 A1+A2，直接执行 A3
- `lastCommand` 含 `update` (不含 --source-type) + `"success"` → 跳过 A1+A2+A3，直接执行 A4
- 其他 → 从 A1 开始

## A0：Diff 检查 — 删除确认（CRITICAL — 不可跳过）

**无论从哪个入口进入 apply（pipeline、plugin-workflow、单独调用），此步骤都 MUST 执行。**

1. 用 `local-config get --remote` 获取远端当前完整配置
2. 对比即将提交的 `plugin.temp.local-{timestamp}.json` 与远端配置：
   - 逐类型对比：远端有哪些点位类型，提交的 JSON 中是否都包含
   - 逐实例对比：同一类型下，远端有哪些 key，提交的 JSON 中是否都包含
3. **如果发现任何点位减少**（无论是整个类型缺失还是某个 key 缺失）：
   - **立即暂停**，向用户列出即将被删除的点位清单（类型 + key + name）
   - **等待用户明确确认**后才能继续执行 A1
   - 如果用户拒绝删除，修正配置文件后重新执行 A0
4. 如果没有点位减少，直接继续执行 A1

> **禁止跳过此步骤。** 即使 plan 阶段已经确认过删除，apply 阶段仍须再次检查——因为配置文件可能在 plan 之后被修改，或者当前 apply 并非从 plan 阶段进入。

## A1：Schema 校验（local-config set）

**Checkpoint**：执行前写入 `{ nextCommand: "local-config set ...", nextStep: "A1 Schema 校验", lastCommandStatus: "running" }`

```bash
npx @byted-meego/cli@builder local-config set --config '<plugin.temp.local-{timestamp}.json 的完整 JSON 内容>'
```

> `--config` 参数值是完整 JSON 字符串，不是文件路径。

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
| 未知错误 | 展示原始错误给用户，共同分析后**重新执行 A1** |

循环直至 set 成功。

## A2：推送远端（update）

**Checkpoint**：执行前写入 `{ nextCommand: "update --source-type=local", nextStep: "A2 推送远端", lastCommandStatus: "running" }`

```bash
npx @byted-meego/cli@builder update --source-type=local
```

- 成功 → **Checkpoint**：`{ lastCommand: "update --source-type=local", lastCommandStatus: "success", nextCommand: "update", nextStep: "A3 拉取远端配置" }` → 继续执行 A3
- 失败 → **Checkpoint**：`{ lastCommand: "update --source-type=local", lastCommandStatus: "failed" }` → 报错展示，终止

## A3：拉取远端配置到本地（update）

**Checkpoint**：执行前写入 `{ nextCommand: "update", nextStep: "A3 拉取远端配置", lastCommandStatus: "running" }`

推送成功后，**必须**再执行一次 update 将远端配置（含模板等）同步回本地 `plugin.config.json`：

```bash
npx @byted-meego/cli@builder update
```

> **禁止直接修改 `plugin.config.json`。** 始终通过 `update` 命令让 CLI 自动同步，确保本地配置与远端一致。

- 成功 → **Checkpoint**：`{ lastCommand: "update", lastCommandStatus: "success", nextStep: "A4 清理临时文件" }` → 继续执行清理
- 失败 → **Checkpoint**：`{ lastCommand: "update", lastCommandStatus: "failed" }` → 报错展示，但不影响已推送的远端配置

## A4：清理临时文件（强制）

> **MUST — 此步骤不可跳过。** apply 完成后 `plugin.temp.local-{timestamp}.json` 已无任何下游消费者，必须立即删除。如果文件不存在则跳过，不报错。

```bash
rm -f plugin.temp.local-*.json
```

## A5：输出

```
✅ 配置已成功推送至远端
   变更：[添加/修改/删除] board[test_board_v1]
```
