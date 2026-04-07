# mode=verify：TypeScript 语法预检 + 自动修复

## V1：执行 tsc --noEmit 预检（L1）

在沙盒中执行：
```bash
npx tsc --noEmit
```

预期耗时约 10s，通过 SSE 展示实时输出。

### 成功（exit code 0）

```
✅ TypeScript 语法检查通过，代码质量 L1 合格
   下一步：执行 plugin-publish skill 发布版本
```

### 失败（有语法错误）→ 进入 L2 自动修复

## V2：AI 自动修复（L2，最多 2 轮）

**第 1 轮修复**：

1. 读取 tsc 完整错误输出（文件名 + 行号 + 错误类型）
2. AI 分析错误原因（import 路径错误、类型不匹配等）
3. 生成修复后的代码，通过 `/files` 接口覆盖写入
4. 重新执行 `tsc --noEmit`

若第 1 轮成功 → 通过，继续发布。

**第 2 轮修复**：

若第 1 轮仍失败，再尝试一次（方法同上）。

若第 2 轮成功 → 通过，继续发布。

## V3：降级兜底（L3）

若 2 轮自动修复后仍有错误：

```
⚠️ 代码存在语法错误，AI 自动修复未能完全解决。

错误详情：
  • src/features/board/main/index.tsx:12 - 类型 'string' 不可分配给类型 'number'
  • ...

建议：
1. 下载代码 zip，在本地用 VS Code 手动修复
2. 本地修复后：npm install && npx tsc --noEmit 验证
3. 修复完成后：npx @byted-meego/cli@builder release 发布

[下载代码 zip]
```

**L3 降级时不阻止用户继续**：用户可以选择忽略错误强制发布（webpack 会再做一次报错），或下载本地修复。

## V4：引导本地调试

verify 通过（L1 或 L2 修复后通过）后，引导用户启动调试：

```bash
npx @byted-meego/cli@builder start --source_type local --auto
```

`--auto` 参数会自动根据 `getLocalConfig` 中的当前域名拼接调试 URL，并在浏览器中打开调试页面。

```
✅ TypeScript 语法检查通过，代码质量合格

下一步：启动本地调试服务器
  运行：npx @byted-meego/cli@builder start --source_type local --auto
  CLI 将自动打开浏览器调试页面
```

用户在浏览器中确认插件效果后，进入下一步：plugin-publish 发布插件。
