# 订阅 Provider Auth 设计

最后更新：2026-05-08

## 决策：订阅绑定授权引用，不直接绑定密钥

**为了谁**：服务于 [框架记忆自动化需求](../requirements/task-memory-automation.md) 里的“外部订阅进入群消息流”，同时避免把真实 token 明文塞进群订阅配置。

**怎么设计**：把真实订阅拆成三层：

- `provider_account`：用户在本机连接过的 GitHub/GitLab/Sentry 账户或安装，记录 provider、base URL、账号展示名、权限摘要、状态和最后校验时间。
- `provider_credential`：真实 access token、refresh token、installation token、过期时间和 refresh 结果，放进本机安全存储；SQLite 只保存 `credential_ref`。
- `subscription`：继续保存 repo/project/org/filter/cursor/poll interval，只保存 `token_ref` 或后续 `auth_ref`，形如 `auth:acct_github_xxx`；现有 `env:GITHUB_TOKEN` 保留为开发 fallback。

**为什么这样**：订阅是“我要看哪个外部事件”，授权是“我用哪个身份看”。把两者分开后，一个 GitHub 账户可以服务多个 repo 订阅，token 轮换不会污染每条 subscription config。

## 决策：GitHub 优先走 GitHub App 用户 token，本地 MVP 不直接持有 App 私钥

**为了谁**：服务于 GitHub repo activity 从 PAT/env 升级到可安装、可撤销、权限更细的授权体验。

**怎么设计**：

- 本地 MVP 推荐 GitHub App user access token，授权方式优先 device flow，其次 localhost callback web flow。
- 用户 token 只访问“用户和 App 同时有权访问”的资源，repo 订阅可通过 `repository_id` 或安装仓库列表收窄。
- installation access token 留给未来“March 云中继/官方后端”或明确托管组件生成；纯本地 app 不默认保存 GitHub App private key。
- `env:GITHUB_TOKEN` 和 fine-grained PAT 仍保留为高级/开发模式，但 UI 要把它标记为 fallback。

**为什么这样**：GitHub 官方最佳实践把 installation token、user token 和 refresh token 区分得很清楚；本地设备保存 App 私钥的风险高于当前 MVP 收益。GitHub App user token 比 OAuth App/PAT 更适合后续 repo 安装、权限收窄和撤销。

## 决策：ProviderAuthResolver 是 connector 调用前的统一门面

**为了谁**：服务于 GitHub/GitLab/Sentry 三类 connector 的共同鉴权、刷新和错误恢复。

**怎么设计**：connector 不直接读环境变量，而是调用：

```js
resolveProviderAuth({
  provider: "github",
  tokenRef: subscription.token_ref,
  requiredAccess: { repository: "owner/repo", permissions: ["metadata:read"] },
  now,
})
```

返回：

```json
{
  "headers": { "Authorization": "Bearer <redacted>" },
  "accountId": "acct_github_123",
  "credentialRef": "cred_github_123",
  "expiresAt": "2026-05-08T13:00:00+08:00",
  "source": "github_app_user"
}
```

Resolver 负责：

- 兼容 `env:*` 旧引用。
- access token 即将过期时刷新并原子更新 credential。
- refresh 失败时返回结构化 `auth_required`，由 Orchestrator 停用相关订阅。
- 记录 `last_verified_at`、`last_error`，但日志永远不输出 token 文本。

**为什么这样**：当前 connector 已有统一 401/403/429/5xx 恢复策略；把 auth resolver 放在 poll 前面，可以让 token 刷新、权限校验、SAML/SSO 失效和用户重连共用同一条错误通道。

## Provider 策略

### GitHub

- 首选：GitHub App user access token。
- 生命周期：access token 默认 8 小时；refresh token 约 6 个月；refresh 后旧 token 失效。
- 安装身份：未来如果有 March 后端，可用 App JWT 换 installation access token；installation token 约 1 小时，适合无人值守自动化，不适合纯本地保存私钥。
- 权限：repo activity 只需读 metadata/events 类能力；后续 Actions/PR 深读再单独扩权限。

### GitLab

- 首选：GitLab OAuth user token；每个 account 带 `base_url`，兼容 self-managed GitLab。
- 生命周期：OAuth access token 约 2 小时，可用 refresh token 换新 token。
- fallback：`env:GITLAB_TOKEN` 或 PAT，继续走现有 `PRIVATE-TOKEN` header。

### Sentry

- 首选：Sentry internal integration token 或 org-level auth token，保留 `event:read` 最小权限。
- 生命周期：先按静态 token 管理，失效后要求用户重连；公共 OAuth app 后置。
- fallback：`env:SENTRY_TOKEN`。

## 本地 Token 生命周期

1. 用户在设置里选择“连接 provider”。
2. March 创建 `provider_account` pending 记录，并启动 device flow 或 localhost callback。
3. 授权完成后，March 把真实 credential 写入本机安全存储，SQLite 保存 `credential_ref` 和展示 metadata。
4. 创建订阅时，UI 选择 `provider_account`；subscription 只保存 `auth:acct_xxx`。
5. Orchestrator 轮询前调用 resolver；resolver 必要时刷新 token。
6. refresh 成功后原子更新 credential，并继续本次 poll。
7. refresh 失败、用户撤销、权限不足或 SSO 失效时，account 进入 `reconnect_required`，相关 subscription 写入 `disabled_reason = auth_required`。
8. 用户删除 account 时，March 清除 credential，并停用依赖它的订阅。

## 数据模型第一切片

新增表建议：

```sql
CREATE TABLE provider_accounts (
  id TEXT PRIMARY KEY,
  provider TEXT NOT NULL,
  auth_type TEXT NOT NULL,
  label TEXT NOT NULL,
  base_url TEXT,
  external_user_id TEXT,
  external_login TEXT,
  scopes_json TEXT NOT NULL DEFAULT '[]',
  permissions_json TEXT NOT NULL DEFAULT '{}',
  credential_ref TEXT NOT NULL,
  status TEXT NOT NULL,
  last_verified_at TEXT,
  last_error TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

现有 `subscriptions.token_ref` 第一阶段不改 schema，只扩展语义：

- `env:NAME`：现有环境变量 token。
- `auth:acct_xxx`：从 `provider_accounts` 解析 credential。

2026-05-08 第一切片已实现 `provider_accounts` 表，并让 `subscriptions.token_ref` 同时支持 `env:*` 和 `auth:acct_xxx`。等 UI/connector 稳定后，再考虑把 `token_ref` 重命名为更准确的 `auth_ref`。

## API 第一切片

- `GET /api/provider-accounts`
- `POST /api/provider-accounts/:provider/device/start`
- `POST /api/provider-accounts/:provider/device/complete`
- `DELETE /api/provider-accounts/:accountId`
- `POST /api/provider-accounts/:accountId/verify`

订阅 API 只需要允许 `tokenRef: "auth:acct_xxx"`，无需立即新增 subscription schema。

## UI 第一切片

- 右侧订阅面板已新增 Connected accounts 小区。
- GitHub/GitLab/Sentry 表单已新增 account selector，选择后写入 `auth:acct_*`。
- `env:*` 继续作为手填 fallback 保留在 token ref 输入里。
- 授权失效时，在 subscription card 展示“需要重新连接”，点击可聚焦到对应 account。

## 官方约束来源

- GitHub App user token、device flow 和 refresh token：<https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app>
- GitHub App installation token：<https://docs.github.com/en/enterprise-cloud@latest/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app>
- GitHub App credential best practices：<https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/best-practices-for-creating-a-github-app>
- GitHub OAuth app flow：<https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps>
- GitLab OAuth token flow：<https://docs.gitlab.com/api/oauth2/>
- Sentry API authentication：<https://docs.sentry.io/hosted/api/auth/>

## 下一步建议

第一实现已经完成：`provider_accounts`、`resolveProviderToken()`、`auth:*`/`env:*` 兼容、provider account API 和 `provider-auth-smoke`。

UI 第一切片已经完成：订阅面板展示 connected/reconnect accounts，三类表单可选择 provider account 并写入 `auth:acct_*`，`auth_required` subscription 可聚焦到对应 account，browser-smoke 已覆盖。

下一刀不要直接做完整 GitHub App 安装流。更稳的下一步是：

1. provider account 支持手动 verify。
2. subscription card 点击“需要重新授权”后接入真实 reconnect/device flow。
3. GitHub device flow 作为再后一刀接入。
