# mode=apply：校验 + 推送远端

## 前置

- 需要 plan 阶段生成的 `plugin.temp.local-{timestamp}.json`
- 若无该文件，先执行 `mode=plan`

**Checkpoint 恢复检查**（仅在 `.plugin-workflow-state.json` 存在时）：
- `lastCommand` 含 `local-config set` + `"success"` → 跳过 A1，直接执行 A2
- `lastCommand` 含 `update --source-type=local` + `"success"` → 跳过 A1+A2，直接执行 A3
- `lastCommand` 含 `update --source-type=remote` + `"success"` → 跳过 A1+A2+A3，直接执行 A4
- （兼容老 checkpoint）`lastCommand` 含 `update`（不带 source-type，老命名）+ `"success"` → 等同 `--source-type=remote`，跳过 A1+A2+A3，直接执行 A4
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

## A0.5：Runtime URL 占位符兜底 lint（CRITICAL — 不可跳过）

即将提交的 JSON 里，所有 **Runtime URL**（分类定义见 `meego-shared/SKILL.md` 的"禁止编造 URL"章节）必须真实可达，否则对应功能上线后会失败。此步骤是 plan 阶段 P3 询问的防御性兜底——防止直接从 apply 入口进入绕过 plan，或配置文件在 plan 之后被手动改动。

### 扫描范围（遍历提交 JSON 的所有类型）

**顶层 URL 字段**：
- `intercept[].url`
- `listen_event[].url`
- `control[].url`（"新建页可见" Webhook）

**数据接口 URL 字段**（仅当该点位的 DSL 声明了变量时才需要真实）：
- `control[].platform.web.table_url.url` —— 条件：`table_cell` 归一化后 `definitions.data` 非空
- `customField[].platform.web.table_data_url` —— 条件：`table_layout` 归一化后 `definitions.data` 非空
- **未来任何新增的、共享 `PlatformWebForControl.table_cell.dslSchema` 的点位类型** —— 扫 `platform.web.<dsl-field>.definitions.data` 与其配套 URL 字段，规则自动适用

**DSL 节点 URL 字段**（扫 template 树的每个节点）：
- `props.onClick.params.url` / `props.onDoubleClick.params.url`，当 `action` 是 `httpRequest` 或 `openLink`（字面量 URL 时）
- 递归进入 `children[]`

**`table_cell` / `table_layout` 的三种形态都要扫**：
- 完整对象 `{definitions, template}` → 扫 template 树
- 裸 template（顶层直接 `type`） → 扫整棵树
- JSON 字符串 → 先 `JSON.parse` 再扫（parse 失败则跳过，让 schema 自己报错）

### 触发警示的 URL 特征

值匹配以下任一可疑模式（大小写不敏感）即判定为"占位符"：

- `example.com` / `example.org` / `example.net`
- `your-server` / `your-domain` / `your-api` / `your-host`
- `placeholder` / `<placeholder`
- `.invalid` / `.test` / `.example`（RFC 2606 保留 TLD，永远无法解析）
- `localhost` / `127.0.0.1` / `0.0.0.0`（生产环境无法访问）
- `mock` / `fake` / `dummy` / `foo.bar` / `sample`
- 空字符串或仅空白（对"必填"的 Runtime URL 而言）

### 处理流程

1. 若发现任一可疑 URL，**立即暂停**，列出清单（按点位分组）：
   ```
   🔴 以下 URL 看起来是占位符，上线后对应功能会失败：
   - intercept[intercept_abc]: url = https://example.com/callback
   - control[control_xxx]: table_url.url = https://PLACEHOLDER.invalid/...
     （该点位 table_cell 声明了变量：varName1, varName2）
   - control[control_yyy].table_cell.template.children[0].props.onClick.params.url
     = https://your-api.test/submit
     （action=httpRequest）
   - customField[ft_zzz]: table_data_url 缺失（table_layout 声明了变量）
   ```

2. **硬阻止，不提供绕过选项**：要求用户提供真实 URL → 修改配置后重新进入 A0。不允许"先占位、后替换"。

**此步骤无论从 pipeline / plugin-workflow / 单独 apply 入口进入均须执行**。即使 plan 阶段已问过，A0.5 仍要再扫一遍——文件可能被手动改过。

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
| URL 模式违规（`must match "^https?://"`） | **禁止自创占位 URL**（包括 `.invalid`/`.test`/`.example` 等 RFC 保留 TLD）。暂停流程向用户索取真实 URL 或授权 placeholder 写法，见 `meego-shared/SKILL.md` 的"规则冲突时的兜底"章节，确认后**重新执行 A1** |
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
> 跳过 A3 的后果：plugin.config.json 中没有 resources、本地 src/ 没有对应 entry 文件，下游的 `plugin-code-gen` skill 会在第一步就找不到代码模板。

```bash
npx @byted-meego/cli@builder update --source-type=remote
```

> **禁止直接修改 `plugin.config.json`。** 始终通过 `update --source-type=remote` 命令让 CLI 自动同步，确保本地配置与远端一致。

A3 完成后 CLI 会做以下三件事（对照验证）：
1. 拉取远端最新配置（含 A2 推送进去的点位）
2. 在 `plugin.config.json.resources` 数组中补齐每个点位对应的 resource id + entry 路径
3. 在 `src/features/<resourceId>/index.tsx` 生成模板代码

- 成功 → **Checkpoint**：`{ lastCommand: "update --source-type=remote", lastCommandStatus: "success", nextStep: "A4 清理临时文件" }` → 继续执行清理
- 失败 → **Checkpoint**：`{ lastCommand: "update --source-type=remote", lastCommandStatus: "failed" }` → 报错展示，但不影响已推送的远端配置

## A4：清理临时文件（强制）

> **MUST — 此步骤不可跳过。** apply 完成后 `plugin.temp.local-{timestamp}.json` 已无任何下游消费者，必须立即删除。如果文件不存在则跳过，不报错。

```bash
rm -f plugin.temp.local-*.json
```

## A5：输出

```
✅ 配置已成功推送至远端
   变更：[添加/修改/删除] page[test_board_v1]
```
