# mode=apply：在模板基础上实现功能代码

> CLI `update` 已在各 entry 路径生成了模板代码。apply 阶段在模板基础上**填充用户真实需求的功能代码**。

## A1：为每个点位实现功能代码

用 jq 取 entry 清单（不要 Read 整份 `plugin.config.json`）：

```bash
jq -r '.resources[].entry' plugin.config.json
```

对每个 entry 路径：

1. Read `entry` 文件（单文件，体量小）获取 CLI 已生成的模板代码
2. 根据 plan 阶段的实现方案，在模板基础上填充功能逻辑
3. 将修改后的代码写回 `entry` 路径

> ⚠️ `index.tsx` 入口包含"等 SDK 注入再挂载"的启动逻辑，**改动这段会导致插件不可用**。功能代码写在模板预留区域，不要替换整个入口文件。

（`plugin.config.json` 的维护规则见 [`../../meegle-plugin-shared/SKILL.md`](../../meegle-plugin-shared/SKILL.md#安全规则)。）

## A1.5：代码溯源审计（脚本化，跨平台）

A1 写完后、A2 输出前，**MUST 执行下述 bash 审计脚本**对所有改动文件做方法名存在性校验，避免 AI 在写代码时悄悄从 schema 字段名脑补 SDK 调用。

> 本步骤用纯 Bash + Grep 实现，不依赖 subagent / Agent 工具，任何支持 Skill 的 AI agent 均可执行。

### 审计脚本

```bash
TYPES="node_modules/@lark-project/js-sdk/dist/types/index.d.ts"
CHANGED_FILES=$(git diff --name-only -- 'src/features/**/*.ts' 'src/features/**/*.tsx' 2>/dev/null)
[ -z "$CHANGED_FILES" ] && CHANGED_FILES=$(find src/features -type f \( -name '*.ts' -o -name '*.tsx' \) 2>/dev/null)

if [ ! -f "$TYPES" ]; then
  echo "SKIP: js-sdk types 未安装 ($TYPES)，无法脚本审计，向用户报告后跳过 A1.5"
  exit 0
fi

while IFS= read -r FILE; do
  [ -z "$FILE" ] && continue
  grep -nE 'window\.JSSDK\.[a-zA-Z_]+\.[a-zA-Z_]+\s*\(' "$FILE" 2>/dev/null | while IFS=: read -r LINE_NUM LINE_CONTENT; do
    METHOD=$(echo "$LINE_CONTENT" | grep -oE 'window\.JSSDK\.[a-zA-Z_]+\.[a-zA-Z_]+' | head -1 | awk -F. '{print $NF}')
    [ -z "$METHOD" ] && continue
    if ! grep -qE "(^|[^a-zA-Z_])${METHOD}\s*[?:<(\[]" "$TYPES"; then
      echo "$FILE:$LINE_NUM | ${LINE_CONTENT# } | method '$METHOD' 在 js-sdk types 中未找到"
    fi
  done
done <<< "$CHANGED_FILES"
```

### 处理脚本输出

- **输出为空** → 审计通过 → 进入 A2 输出摘要
- **输出非空** → **暂停**，把每一条原文呈现给用户：

```
⚠️ 代码溯源审计发现以下可疑调用（方法名在 @lark-project/js-sdk types 中不存在）：

1. src/features/xxx/index.tsx:42
   代码: window.JSSDK.control.getValue()
   原因: 'getValue' 在 @lark-project/js-sdk 类型定义中不存在
         可能的真实 API：getCreateWorkItemFormItemValues

2. src/features/yyy/index.tsx:88
   代码: window.JSSDK.customField.saveData(...)
   原因: 'saveData' 在 types 中未找到
         可能的真实 API：saveFieldValue

请告诉我每条怎么处理：
- A. 按提示修改
- B. 我提供真实 API（贴文档/示例）
- C. 跳过这处功能（输出 TODO 占位）
```

> ⚠️ **禁止 orchestrator 自己"过滤"脚本输出**。所有条目必须**原文呈现**，不允许"我帮你筛选了下，问题不大"这种 sycophancy。
>
> ⚠️ **禁止用主 agent 自审代替脚本**——AI 对自己写的代码有偏见，容易放行。脚本是闸口，即使觉得"我这次写得挺稳"也要跑。

### 何时可以跳过 A1.5？

仅在以下条件全满足时可跳过（plan 阶段已显式记录）：

- 本次 apply 没有产生任何新的 `window.JSSDK.*` / SDK 调用
- 仅修改 UI、样式、纯前端逻辑（无 SDK 接触）

### 局限与补强

脚本审计只查"方法名存不存在于 types"，查不了：
- 方法签名参数是否正确
- 调用顺序 / 跨 API 约束
- source 注释是否合理

这些语义层问题靠 [`code-plan.md`](code-plan.md) 的"代码溯源协议 + 无源即停"在前置阶段堵住。脚本是**事后兜底闸**，不是全部防线。

## A2：输出摘要

```
✅ 代码生成完成
   修改文件：
   • ./src/features/board_web_rj4v_m/index.tsx

   下一步：执行语法检查（mode=verify）
```
