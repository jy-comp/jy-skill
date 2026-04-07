# mode=apply：校验 + 推送远端

## 前置

- 需要 plan 阶段生成的 `point.config.local-{timestamp}.json`
- 若无该文件，先执行 `mode=plan`

## A1：Schema 校验（local-config set）

```bash
npx @byted-meego/cli@builder local-config set --config '<point.config.local-{timestamp}.json 的完整 JSON 内容>'
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

- 成功 → 打印成功信息，继续执行清理
- 失败 → 报错展示，终止（不继续 verify）

## A3：清理临时文件

```
删除 point-schema.yaml
删除 point.config.local-remote.json
删除 point.config.local-{timestamp}.json
```

## A4：输出

```
✅ 配置已成功推送至远端
   变更：[添加/修改/删除] board[test_board_v1]
```
