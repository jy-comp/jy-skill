# mode=apply：校验 + 推送远端

## 前置

- 需要 plan 阶段生成的 `plugin.temp.local-{timestamp}.json`
- 若无该文件，先执行 `mode=plan`

## A1：Schema 校验（local-config set）

```bash
npx @byted-meego/cli@builder local-config set --config '<plugin.temp.local-{timestamp}.json 的完整 JSON 内容>'
```

> `--config` 参数值是完整 JSON 字符串，不是文件路径。

### 成功

继续执行 A2。

### 失败 — 错误分类处理

| 错误类型 | 处理方式 |
|----------|---------|
| 必填字段缺失 | 向用户补充询问，更新配置文件，**重新执行 A1** |
| 枚举值非法 | 列出合法值，询问用户选择，更新后**重新执行 A1** |
| key 重复冲突 | AI 自动在 key 后追加数字后缀（如 `_2`、`_3`）生成不冲突的 key，更新配置后**重新执行 A1** |
| name 超长 | 提示最大长度限制，要求用户缩短，**重新执行 A1** |
| 未知错误 | 展示原始错误给用户，共同分析后**重新执行 A1** |

循环直至 set 成功。

## A2：推送远端（update）

```bash
npx @byted-meego/cli@builder update --source-type=local
```

- 成功 → 继续执行 A3
- 失败 → 报错展示，终止（不继续后续步骤）

## A3：拉取远端配置到本地（update）

推送成功后，**必须**再执行一次 update 将远端配置（含模板等）同步回本地 `plugin.config.json`：

```bash
npx @byted-meego/cli@builder update
```

> **禁止直接修改 `plugin.config.json`。** 始终通过 `update` 命令让 CLI 自动同步，确保本地配置与远端一致。

- 成功 → 继续执行清理
- 失败 → 报错展示，但不影响已推送的远端配置

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
