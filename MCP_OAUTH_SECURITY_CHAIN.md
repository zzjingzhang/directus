# MCP OAuth 安全链路追踪文档

## 1. 概览

Directus MCP OAuth 实现了一套完整的 OAuth 2.1 授权服务器，保护 `/mcp` 端点。安全链路涵盖：

```
Discovery (RFC 9728/8414)
    ↓
Dynamic Client Registration (RFC 7591) / CIMD
    ↓
Authorize (RFC 6749) + PKCE S256 (RFC 7636)
    ↓
Consent Page + Signed Consent JWT
    ↓
Authorization Decision
    ↓
Token Exchange (authorization_code / refresh_token)
    ↓
Protected /mcp Resource Access
```

涉及的关键文件：

- 控制器：[oauth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts)
- 服务层：[index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts)
- MCP 守卫：[mcp-oauth-guard.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/mcp-oauth-guard.ts)
- MCP Server：[server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts)
- 应用挂载：[app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts)
- Redirect 工具：[redirect.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/utils/redirect.ts)
- Loopback 工具：[loopback.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/utils/loopback.ts)
- MCP 工具：[utils.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/utils.ts)

---

## 2. 端点与路由挂载

### 2.1 公共路由（mounted BEFORE authenticate）

在 [app.ts#L324-L326](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L324-L326) 中挂载 `mcpOAuthPublicRouter`：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/.well-known/oauth-protected-resource*` | GET | RFC 9728 受保护资源发现 |
| `/.well-known/oauth-authorization-server*` | GET | RFC 8414 授权服务器发现 |
| `/mcp-oauth/authorize` | GET | 授权请求 + Consent 页面 |
| `/mcp-oauth/register` | POST | RFC 7591 动态客户端注册 (DCR) |
| `/mcp-oauth/token` | POST | Token 颁发 (code/refresh) |
| `/mcp-oauth/revoke` | POST | RFC 7009 Token 撤销 |

### 2.2 受保护路由（mounted AFTER authenticate）

在 [app.ts#L331-L333](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L331-L333) 中挂载 `mcpOAuthProtectedRouter`：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/mcp-oauth/authorize/decision` | POST | Consent 决策端点（表单提交） |

### 2.3 MCP 资源端点

在 [app.ts#L371-L373](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L371-L373) 中挂载：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/mcp` | GET/POST | MCP JSON-RPC 受保护资源 |

---

## 3. OAuth 开关：环境变量 + Settings 双重校验

MCP OAuth 的启用受到**环境变量**和**数据库 Settings** 的双重保护，任何一层关闭都会导致功能不可用。

### 3.1 第一层：环境变量（路由挂载级）

在 [app.ts#L127-L132](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L127-L132) 中进行前置依赖校验：

```typescript
if (env['MCP_OAUTH_ENABLED'] === true) {
    if (toBoolean(env['MCP_ENABLED']) !== true) {
        logger.warn('MCP_OAUTH_ENABLED requires MCP_ENABLED=true. OAuth disabled.');
        env['MCP_OAUTH_ENABLED'] = false;
    }
}
```

路由挂载条件（全部使用 `env['MCP_OAUTH_ENABLED'] === true` 判断）：
- `mcpOAuthPublicRouter`：[app.ts#L324-L326](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L324-L326)
- `mcpOAuthProtectedRouter`：[app.ts#L331-L333](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L331-L333)
- `mcpOAuthClientsRouter`：[app.ts#L397-L399](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L397-L399)
- OAuth cleanup 定时任务：[app.ts#L426-L428](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L426-L428)

### 3.2 第二层：Settings（请求级中间件）

在 [oauth.ts#L248-L279](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts#L248-L279) 的 `checkOAuthSettings` 中间件中，对每个 OAuth 请求从 `directus_settings` 表读取并校验：

```typescript
const settings = await settingsService.readSingleton({ fields: ['mcp_enabled', 'mcp_oauth_enabled'] });

if (toBoolean(settings?.mcp_enabled) !== true || toBoolean(settings?.mcp_oauth_enabled) !== true) {
    // 返回 403 错误...
}
```

该中间件作用于：
- Discovery 端点
- Authorize 端点
- DCR 注册端点
- Token 端点
- Revoke 端点
- Decision 端点

### 3.3 DCR/CIMD 各自的双重开关

在 [index.ts#L300-L311](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L300-L311) 中，DCR 注册有独立的双重开关：

```typescript
// 环境变量
if (!toBoolean(env['MCP_OAUTH_DCR_ENABLED'])) {
    throw new OAuthError(404, 'not_found', 'Dynamic client registration is not available');
}
// Settings
if (!toBoolean(settings?.mcp_oauth_dcr_enabled)) {
    throw new OAuthError(404, 'not_found', 'Dynamic client registration is not available');
}
```

CIMD 同理，见 [index.ts#L498-L507](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L498-L507)。

### 3.4 Authorization Server Metadata 中的条件暴露

在 [index.ts#L244-L277](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L244-L277) 中，discovery 元数据的 `registration_endpoint` 和 `client_id_metadata_document_supported` 仅在对应开关启用时才出现在响应中。

---

## 4. redirect_uri 校验与 Loopback 端口例外

### 4.1 redirect_uri 校验规则

在 [redirect.ts#L61-L111](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/utils/redirect.ts#L61-L111) 中实现 `validateRedirectUri`：

| 校验项 | 规则 |
|--------|------|
| 类型 | 必须为 string |
| 长度 | ≤ 255 字符 |
| 格式 | 必须是合法 URL |
| Fragment | 不允许 (RFC 6749 §3.1.2) |
| Userinfo | 不允许 (username/password) |
| 协议 | 必须是 `https:` **例外**: loopback 允许 `http:` |
| 自定义 Scheme | `MCP_OAUTH_ALLOWED_CUSTOM_REDIRECTS` 配置白名单 |
| 域名白名单 | `MCP_OAUTH_ALLOWED_REDIRECT_DOMAINS` **例外**: loopback 和自定义 scheme 绕过 |

Loopback 主机判定在 [loopback.ts#L5-L6](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/utils/loopback.ts#L5-L6)：

```typescript
export function isLoopbackHost(hostname: string): boolean {
    return hostname === 'localhost' || hostname === '127.0.0.1' || hostname === '[::1]';
}
```

### 4.2 Loopback 端口例外（RFC 8252 §7.3）

在 [redirect.ts#L119-L154](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/utils/redirect.ts#L119-L154) 中实现 `matchRedirectUri`：

- **非 loopback**: 要求**精确字符串匹配** (RFC 6749 §3.1.2)
- **Loopback**: 允许**任意端口**，但其他部分必须匹配

```typescript
if (isLoopbackHost(regUrl.hostname) && isLoopbackHost(reqUrl.hostname)) {
    return (
        regUrl.protocol === reqUrl.protocol &&
        regUrl.hostname === reqUrl.hostname &&
        regUrl.pathname === reqUrl.pathname &&
        // 注意：此处不比较 port，即允许任意端口
        regUrl.search === reqUrl.search &&
        regUrl.hash === reqUrl.hash
    );
}
```

这符合 RFC 8252 §7.3 规定：原生应用绑定到临时端口时，授权服务器不得限制端口号。

### 4.3 DCR 注册时的校验

在 [index.ts#L363-L376](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L363-L376) 中，所有注册的 redirect_uri 都通过 `validateRedirectUri` 校验，确保后续授权请求时使用的 URI 已提前验证。

---

## 5. PKCE S256、scope=mcp:access、resource 匹配

### 5.1 PKCE S256 强制启用

PKCE 相关常量在 [index.ts#L135-L139](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L135-L139)：

```typescript
const CODE_VERIFIER_RE = /^[A-Za-z0-9\-._~]{43,128}$/;     // RFC 7636 §4.1
const CODE_CHALLENGE_S256_RE = /^[A-Za-z0-9_-]{43}$/;       // RFC 7636 §4.2
```

**Authorize 阶段校验** ([index.ts#L579-L590](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L579-L590))：
- `code_challenge` 必填
- 必须匹配 S256 格式正则
- `code_challenge_method` 必须精确等于 `"S256"`

Discovery 元数据也仅声明 S256 支持：[index.ts#L264](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L264)

**Code Exchange 阶段校验** ([index.ts#L939-L948](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L939-L948))：

```typescript
const computedChallenge = this.hashToken(params.code_verifier!, 'base64url');
const storedChallenge = codeRecord['code_challenge'] as string;

if (
    computedChallenge.length !== storedChallenge.length ||
    !crypto.timingSafeEqual(Buffer.from(computedChallenge), Buffer.from(storedChallenge))
) {
    rejectCode({}, 'PKCE verification failed');
}
```

使用 `crypto.timingSafeEqual` 进行定时安全比较，防止侧信道攻击。

### 5.2 scope 必须包含 `mcp:access`

Scope 常量定义在 [utils.ts#L4](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/utils.ts#L4)：

```typescript
export const MCP_ACCESS_SCOPE = 'mcp:access';
```

**Authorize 阶段** ([index.ts#L592-L601](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L592-L601))：

```typescript
const parsedScopes = parseOAuthScope(scope);
const requestedScopes = parsedScopes.length > 0 ? parsedScopes : [MCP_ACCESS_SCOPE];

if (!requestedScopes.includes(MCP_ACCESS_SCOPE)) {
    throw new OAuthError(400, 'invalid_scope', 'Scope must include mcp:access', true);
}

const normalizedScope = MCP_ACCESS_SCOPE;  // 只授予我们支持的 scope
```

**Refresh Token 阶段** ([index.ts#L1132-L1143](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L1132-L1143)) 也做了同样的校验，并额外限制不能超出原始授权的 scope。

### 5.3 resource 参数匹配（RFC 8707）

在 [index.ts#L610-L622](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L610-L622)：

```typescript
const { resourceUrl: expectedResource } = getMcpUrls();
const requireResource = env['MCP_OAUTH_REQUIRE_RESOURCE'] === true;
const resolvedResource = resource || (!requireResource ? expectedResource : null);

if (!resolvedResource) {
    throw new OAuthError(400, 'invalid_target', 'resource is required', true);
}

if (resolvedResource !== expectedResource) {
    throw new OAuthError(400, 'invalid_target', 'resource does not match the protected resource', true);
}
```

`getMcpUrls()` 在 [utils.ts#L9-L17](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/utils.ts#L9-L17) 中返回 `{PUBLIC_URL}/mcp` 作为资源标识。

最终颁发的 Access Token 中 `aud` 声明绑定到此 resource URL：[index.ts#L1928](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L1928)

---

## 6. Consent JWT 的 typ/aud/session_hash 校验

### 6.1 Consent JWT 常量定义

在 [index.ts#L37-L40](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L37-L40)：

```typescript
const CONSENT_JWT_TYP = 'directus-mcp-consent+jwt';   // 防止令牌混淆
const CONSENT_JWT_AUD = 'mcp-oauth-authorize-decision'; // 绑定到决策端点
```

### 6.2 Consent JWT 签名

在 [index.ts#L624-L643](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L624-L643)：

```typescript
const signedParams = jwt.sign(
    {
        typ: CONSENT_JWT_TYP,          // ← typ 声明
        aud: CONSENT_JWT_AUD,          // ← aud 声明
        sub: userId,                    // 当前用户
        session_hash: sessionHash,      // ← 会话绑定
        client_id, redirect_uri,
        code_challenge, code_challenge_method,
        scope, resource, state,
    },
    consentKey,  // HMAC-SHA256，域分离密钥
    { expiresIn: '5m', algorithm: 'HS256' },
);
```

域分离密钥通过 [index.ts#L2007-L2009](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L2007-L2009) 生成：

```typescript
private getConsentKey(): Buffer {
    return crypto.createHmac('sha256', getSecret()).update('mcp-oauth-consent-v1').digest();
}
```

### 6.3 Consent JWT 验证

在 [index.ts#L689-L721](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L689-L721) 的 `processDecision` 中：

1. **签名 + 过期 + aud 校验** ([index.ts#L698-L704](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L698-L704))：
   ```typescript
   claims = jwt.verify(signed_params, consentKey, {
       algorithms: ['HS256'],
       audience: CONSENT_JWT_AUD,  // aud 校验
   });
   ```

2. **typ 校验** ([index.ts#L706-L709](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L706-L709))：
   ```typescript
   if (claims['typ'] !== CONSENT_JWT_TYP) {
       throw new OAuthError(400, 'invalid_request', 'Invalid consent token type');
   }
   ```

3. **sub 校验**（防跨用户重放）[index.ts#L711-L714](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L711-L714)：
   ```typescript
   if (claims['sub'] !== userId) {
       throw new OAuthError(400, 'invalid_request', 'Consent token user mismatch');
   }
   ```

4. **session_hash 校验**（防跨会话重放）[index.ts#L716-L721](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L716-L721)：
   ```typescript
   const currentSessionHash = this.hashToken(sessionToken);
   if (claims['session_hash'] !== currentSessionHash) {
       throw new OAuthError(400, 'invalid_request', 'Session binding mismatch');
   }
   ```

### 6.4 Decision 端点的额外安全层

Decision 端点在 [oauth.ts#L590-L603](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts#L590-L603) 还有两层中间件保护：

- `requireCookieAuth` ([oauth.ts#L95-L102](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts#L95-L102))：拒绝 bearer-token 认证，强制使用 cookie，防止通过 API 提交 consent 的令牌窃取攻击
- `requireSameOrigin` ([oauth.ts#L109-L135](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts#L109-L135))：Origin 必须与 `PUBLIC_URL` 同源，防止 CSRF

---

## 7. Confidential Client Secret 哈希

### 7.1 Secret 生成与哈希存储

在 [index.ts#L455-L463](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L455-L463)：

```typescript
const isConfidential = authMethod !== 'none';
let clientSecret: string | undefined;
let clientSecretHash: string | null = null;

if (isConfidential) {
    clientSecret = crypto.randomBytes(CLIENT_SECRET_BYTES).toString('base64url');  // 256 bits
    clientSecretHash = this.hashToken(clientSecret);                               // SHA-256 hex
}
```

存储在 [index.ts#L469-L481](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L469-L481) 的 `directus_oauth_clients.client_secret_hash` 字段中，**仅存储哈希，明文只返回一次给客户端**。

### 7.2 Hash 函数

在 [index.ts#L2011-L2013](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L2011-L2013)：

```typescript
private hashToken(token: string, encoding: 'hex' | 'base64url' = 'hex'): string {
    return crypto.createHash('sha256').update(token).digest(encoding);
}
```

### 7.3 Secret 验证（定时安全比较）

在 [index.ts#L1702-L1721](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L1702-L1721) 的 `verifySecret`：

```typescript
private verifySecret(providedSecret: string, clientRecord: Record<string, unknown>): void {
    const storedHash = clientRecord['client_secret_hash'] as string | null;

    if (!storedHash || !SHA256_HEX_RE.test(storedHash)) {
        throw new OAuthError(401, 'invalid_client', 'Client authentication failed');
    }

    const computedHash = this.hashToken(providedSecret);
    const hashA = Buffer.from(computedHash, 'hex');
    const hashB = Buffer.from(storedHash, 'hex');

    if (hashA.length !== hashB.length || !crypto.timingSafeEqual(hashA, hashB)) {
        throw new OAuthError(401, 'invalid_client', 'Client authentication failed');
    }
}
```

安全要点：
- 使用 `SHA256_HEX_RE` (/^[0-9a-f]{64}$/) 格式守卫确保存储值格式正确
- 通过 `crypto.timingSafeEqual` 进行恒定时间比较，防止时序攻击
- 错误信息统一返回"Client authentication failed"，防止信息泄露

### 7.4 认证方法强制执行

在 [index.ts#L1642-L1700](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L1642-L1700) 的 `authenticateClient` 中：

- `none`（公共客户端）：禁止任何 Authorization header 和 body 中的 client_secret
- `client_secret_basic`：必须通过 Basic Authorization header 传递，禁止 body 传递
- `client_secret_post`：必须通过 body 传递，禁止 Authorization header

---

## 8. tokenSource 必须为 header

### 8.1 tokenSource 识别

在 [extract-token.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/extract-token.ts) 中，请求 token 的来源被标记为：
- `'cookie'`：来自 session cookie
- `'header'`：来自 `Authorization: Bearer`
- `'query'`：来自 URL query 参数
- `null`：无 token

### 8.2 MCP 资源端点的 tokenSource 限制

在 [server.ts#L97-L102](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L97-L102)：

```typescript
if (oauth) {
    if (req.tokenSource !== 'header') {
        this.sendUnauthorized(res, 'invalid_request');
        return;
    }
    // ...
}
```

这意味着：**通过 OAuth accountability 访问 `/mcp` 时，token 必须来自 `Authorization` header**，不允许通过 cookie 或 query 参数传递。这遵循 RFC 6750 推荐的 Bearer Token 使用方式，防止：
- 通过 URL 泄漏 access token（历史记录、日志、Referer 头）
- CSRF 攻击（cookie 会被浏览器自动发送）

### 8.3 Consent Decision 端点的反向限制

作为对比，[oauth.ts#L95-L102](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts#L95-L102) 的 `requireCookieAuth` 强制 Decision 端点**只能使用 cookie**，防止通过 API token 提交 consent。

---

## 9. mcpOAuthGuard：限制 OAuth Accountability 只能访问 MCP 路径

### 9.1 守卫代码

在 [mcp-oauth-guard.ts#L9-L21](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/mcp-oauth-guard.ts#L9-L21)：

```typescript
export function handler(req: Request, _res: Response, next: NextFunction): void {
    if (!req.accountability?.oauth) {
        next();  // 普通会话直接放行
        return;
    }

    if (!isMcpPath(req.path) || (req.method !== 'GET' && req.method !== 'POST')) {
        next(new ForbiddenError());
        return;
    }

    next();
}
```

### 9.2 挂载位置

在 [app.ts#L328-L329](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L328-L329)：

```typescript
app.use(authenticate);
app.use(mcpOAuthGuard);
```

守卫位于 `authenticate` 之后、**所有业务路由之前**，是全局生效的。

### 9.3 判定条件

当 `req.accountability.oauth` 存在（即当前会话是通过 OAuth 登录的）时：

| 条件 | 要求 | 说明 |
|------|------|------|
| `isMcpPath(req.path)` | `true` | 路径必须是 `/mcp` 或 `/mcp/` |
| `req.method` | `GET` 或 `POST` | MCP 协议仅使用这两种方法 |

`isMcpPath` 定义在 [utils.ts#L20-L22](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/utils.ts#L20-L22)：

```typescript
export function isMcpPath(path: string): boolean {
    return path === '/mcp' || path === '/mcp/';
}
```

### 9.4 设计意图

该守卫实现了**最小权限原则**：
1. OAuth 颁发的 token 的唯一用途是访问 MCP 资源
2. 即使 OAuth token 被窃取，攻击者也无法访问 Directus 的其他 API（如 `/items`、`/users`、`/settings` 等）
3. 限制 HTTP 方法为 GET/POST，与 MCP JSON-RPC 协议实际使用的方法一致

### 9.5 MCP 控制器中的二次校验

在 [index.ts#L43-L48](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/index.ts#L43-L48) 中，MCP 控制器还有一层额外校验：

```typescript
if (
    req.accountability?.oauth &&
    (toBoolean(env['MCP_OAUTH_ENABLED']) !== true || toBoolean(mcp_oauth_enabled) !== true)
) {
    throw new ForbiddenError({ reason: 'MCP OAuth must be enabled' });
}
```

确保即使 OAuth 会话存在，如果 OAuth 功能后来被关闭，也无法继续使用。

---

## 10. 安全链路总结表

| 安全层 | 机制 | 位置 |
|--------|------|------|
| 功能开关 | 环境变量 + Settings 双重校验 | app.ts / oauth.ts checkOAuthSettings |
| Discovery | 仅在开关启用时暴露端点 | index.ts getAuthorizationServerMetadata |
| DCR | 开关双重校验 + 域名白名单 + redirect_uri 预验证 | index.ts registerClient |
| redirect_uri | HTTPS 强制 + loopback 端口例外 + 精确匹配 | redirect.ts validateRedirectUri / matchRedirectUri |
| PKCE | S256 强制 + 定时安全比较 | index.ts validateAuthorization / exchangeCode |
| Scope | 必须包含 `mcp:access`，仅授予已知 scope | index.ts validateAuthorization |
| Resource | 必须匹配 `/mcp`，aud 声明绑定 | index.ts validateAuthorization / issueMcpAccessToken |
| Consent | JWT typ/aud/sub/session_hash 多重绑定 + 5min TTL | index.ts validateAuthorization / processDecision |
| Decision | Cookie 认证 + Same Origin + CSRF 防护 | oauth.ts requireCookieAuth / requireSameOrigin |
| Client Secret | SHA-256 哈希存储 + 定时安全比较 | index.ts verifySecret |
| Token Transport | MCP 资源仅接受 header (RFC 6750) | server.ts handleRequest |
| Accountability Scope | OAuth 会话仅允许访问 `/mcp` (GET/POST) | mcp-oauth-guard.ts handler |
