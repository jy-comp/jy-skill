# Checkpoint 更新协议

> **前置**：本段是 meego-shared 主文件"Checkpoint（进度追踪）"的完整规则。子 skill 在被 plugin-workflow 编排且需要写 checkpoint 时按需 Read。

## 适用条件

子 skill 执行前读取项目根目录 `.plugin-workflow-state.json`：
- **存在** → 当前处于 workflow 编排中，每步 CLI 前后更新此文件（走本文件协议）
- **不存在** → 子 skill 被独立调用，不需要写 checkpoint（跳过本文件）

## 更新协议

```
CLI 命令执行前：
  → 写入 nextCommand、nextStep，lastCommandStatus 设为 "running"

CLI 命令执行后（成功）：
  → lastCommand = 刚执行的命令
  → lastCommandStatus = "success"
  → nextCommand/nextStep 更新为下一步

CLI 命令执行后（失败）：
  → lastCommand = 刚执行的命令
  → lastCommandStatus = "failed"
  → 保留 nextCommand/nextStep 不变（重试时使用）
```

## 恢复语义

当 workflow 从 checkpoint 恢复并调用子 skill 时，子 skill 应检查 `lastCommand` + `lastCommandStatus`：
- 上一步 `"success"` → 跳过该步，执行下一步
- 上一步 `"failed"` → 重试该步
- 上一步 `"running"` → 未知状态，重新执行该步
