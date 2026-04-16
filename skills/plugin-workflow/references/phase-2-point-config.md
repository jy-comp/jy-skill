# Phase 2：点位配置

把 Phase 0 采集的用户上下文交给 meego-point-config 处理。**编排层不预判、不拉 schema、不填默认值、不替用户做选型**——点位识别、schema 切片、用户交互确认、URL 询问、MCP 业务语义查询全部在 meego-point-config/references/plan.md 里完成。编排层这里插手哪一项，那一项的护栏就会被绕过。

## 2.1 调用 meego-point-config

**Checkpoint 写入**：`{ "phase": 2, "step": "2.1", "stepName": "meego-point-config", "nextStep": "2.1 调用 meego-point-config pipeline" }`

**恢复检查**：若从 checkpoint 恢复，读 `.lpm-cache/state.json` 里 meego-point-config 自己维护的子状态——由 meego-point-config 决定从哪一步继续，编排层不替它解读 `local-config set` / `update` 的成功与否。

调用时把 Phase 0 采集的两项作为 hint 传入：

- `context.originalRequirement` —— 用户原话（逐字），作为 P1 识别的输入
- `context.mentionedPointType` —— 用户主动提到的点位类型或 null，作为 P1 的可选提示

```
调用 /meego-point-config mode=pipeline
```

meego-point-config 完成后会返回配置后的点位清单，写回 checkpoint: `{ "step": "2.1", "stepName": "meego-point-config", "status": "success" }`。

## 2.2 阶段输出

```
✅ 点位配置完成
   点位：[type] key / name（由 meego-point-config 回填）
   下一步：正在为你生成代码...
```

**自动进入 Phase 3，无需用户确认。**
