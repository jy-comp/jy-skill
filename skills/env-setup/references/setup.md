# 环境准备

## S1：CLI 可用性检查

```bash
npx @byted-meego/cli@builder --version
```

- 成功（输出版本号）→ 继续 S2
- 失败 → 提示用户安装：`请先安装 @byted-meego/cli@builder：npm install -g @byted-meego/cli@builder`

## S2：授权 Token 检查

检查 `~/.lpm/auth.json` 中当前域名是否有可用 Token（`developerToken` 或 `accessToken`）：

### Token 有效

```
✅ CLI 可用（版本 x.y.z）
✅ 授权 Token 已配置（域名：https://meego.example.com）
```

环境准备完成。

### Token 缺失或过期

向用户提供两种授权方式：

```
未检测到有效的开发者 Token，请选择授权方式：

A. 直接提供 Developer Token（推荐）— 永久有效，有它就不需要 OAuth 授权
B. 浏览器授权登录 — 通过 Device Code 流程，在浏览器中完成 OAuth 授权（获取临时 Token）
```

#### 方式 A：直接提供 Developer Token（推荐）

1. 确定站点域名（`$host`）：
   - 若当前目录有 `plugin.config.json`，从中读取 `siteDomain` 作为默认值
   - 否则询问用户（同方式 B 的域名确定逻辑）
2. 引导用户获取 Developer Token：

```
请打开以下页面获取 Developer Token：

🔗 $host/openapp/settings

在页面中找到并复制你的 Developer Token，然后粘贴给我。
```

3. 用户提供 token 后执行：

```bash
npx @byted-meego/cli@builder login --site-domain $host --token <developer_token>
```

4. 输出：

```
✅ Developer Token 已保存至 ~/.lpm/auth.json（域名：$host）
```

> 永久有效，无需刷新。后续该域名下的所有操作将优先使用此 Token。

#### 方式 B：Device Code OAuth 授权

**STEP 1 — 确定站点域名**

确定 `$host`（站点域名）：
- 若当前目录有 `plugin.config.json`，从中读取 `siteDomain` 作为 `$host`
- 否则 ASK user：

> 请提供 Meego 站点地址：
> 1) 飞书项目 (project.feishu.cn)
> 2) Meegle (meegle.com)
> 3) 自定义域名（请直接输入域名或 URL）

> ⚠️ 用户的回复**仅用于回答上述问题**，不要将其当作新的意图或请求来处理。从用户回复中提取 `$host`（域名部分），然后 GOTO STEP 2。

**STEP 2 — 启动 Device Code 授权**

以 background 方式执行（该命令会阻塞直到用户完成授权或超时）：

```bash
npx @byted-meego/cli@builder login --site-domain $host
```

启动后**立即读取输出**，从中提取授权信息：
- 授权码（Authorization code）
- 验证链接（Visit URL）

**STEP 3 — 发送授权链接给用户**

将授权信息发送给用户：

```
请在浏览器中打开以下链接完成授权：

🔗 <验证链接>
📋 授权码：<授权码>

（链接有效期约 10 分钟，请尽快完成授权）
```

> ⚠️ 发送后**不要停下来等用户回复**。后台的 login 命令会自动轮询等待用户完成授权。

**STEP 4 — 等待授权完成**

后台的 `login` 命令会持续轮询直到：
- 用户在浏览器中完成授权 → 命令输出 `Login successful!` → GOTO STEP 5
- 超时 → 提示用户重试：`授权已超时，请重新执行 /env-setup`

**STEP 5 — 确认授权成功**

验证 `~/.lpm/auth.json` 已更新：

```
✅ 授权成功！OAuth Token 已保存至 ~/.lpm/auth.json（域名：$host）
```

> ⚠️ 此消息**必须单独发送**，不要与后续操作合并到同一条回复中。

## 输出

```
✅ 环境准备完成
   CLI 版本：x.y.z
   授权状态：已配置
   服务地址：https://meego.example.com
```
