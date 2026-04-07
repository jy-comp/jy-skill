# mode=verify：拉取远端验证

## 前置

- apply 阶段必须已执行（远端有数据）

## V1：拉取远端配置

```bash
npx @byted-meego/cli@builder local-config get --remote
```

验证远端配置可读，并确认刚才推送的变更已生效。

## V2：解读结果

### 成功

确认远端数据包含预期的点位配置：

```
✅ 远端配置验证通过
   board: 2 个点位
   view: 1 个点位
   ...
```

### 失败（远端缺失预期数据）

```
❌ 远端配置验证失败
   预期的 board[test_board_v1] 未在远端找到
```

处理：
- 确认 apply 阶段是否真正成功（检查 update 命令输出）
- 若 apply 确认成功但远端无数据：可能是 API 延迟，等待后重试 verify
- 若持续失败：报告给用户，不自动修复

## V3：清理临时文件（强制）

> **MUST — 此步骤不可跳过。** verify 是 point-config 流程的最后一步，以下临时文件已无任何下游消费者，必须立即删除。如果文件不存在则跳过，不报错。

```bash
rm -f point-schema.yaml point.config.local-remote.json
```

清理完成后再输出 verify 结果。
