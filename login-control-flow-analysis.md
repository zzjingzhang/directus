# Directus 本地账号登录控制流完整分析

## 概述

本文档详细分析 Directus 本地账号（local auth）从 HTTP 入口到 `AuthenticationService.login` 的完整控制流，覆盖所有关键校验环节和异常处理路径。

---

## 1. HTTP 入口层

### 1.1 路由注册

登录路由在 [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth.ts#L103) 中注册：

```typescript
// 始终注册默认本地认证，运行时根据配置决定是否禁用
router.use('/login', createLocalAuthRouter(DEFAULT_AUTH_PROVIDER));

// 按认证提供者注册路由
for (const authProvider of authProviders) {
    if (authProvider.driver === 'local') {
        authRouter = createLocalAuthRouter(authProvider.name);
    }
    router.use(`/login/${authProvider.name}`, authRouter);
}
```

### 1.2 本地认证路由创建

`createLocalAuthRouter` 在 [local.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth/drivers/local.ts#L49-L121) 中定义，创建 POST `/` 路由处理器。

---

## 2. 完整控制流详解

### 2.1 流程总览

```
HTTP POST /auth/login
    ↓
1. Payload Schema 校验 (Joi)
    ├─ 失败 → InvalidPayloadError → stall → 响应
    └─ 成功 → 继续
    ↓
2. 初始化 Accountability (IP, User-Agent, Origin)
    ↓
3. AuthenticationService.login() 调用
    ├─ 3.1 邮箱查找 getUserID()
    │   ├─ 无email → InvalidCredentialsError
    │   ├─ 找不到用户 → InvalidCredentialsError
    │   └─ 找到 → 返回 userId
    ├─ 3.2 查询用户完整信息
    ├─ 3.3 auth.login filter 钩子
    ├─ 3.4 用户状态检查 (status === 'active' && provider 匹配)
    │   └─ 失败 → InvalidCredentialsError
    ├─ 3.5 登录尝试次数限流 (Rate Limiter)
    │   └─ 超限 → 用户标记为 suspended + activity记录
    ├─ 3.6 provider.login() → argon2 密码校验
    │   └─ 失败 → InvalidCredentialsError
    ├─ 3.7 TFA 检查
    │   ├─ 有密钥无OTP → InvalidOtpError
    │   └─ 有密钥有OTP → 验证OTP → 失败 → InvalidOtpError
    ├─ 3.8 License 锁定检查 (非admin用户被阻断)
    │   └─ 锁定且非admin → ResourceRestrictedError
    ├─ 3.9 角色权限树与全局访问权限
    ├─ 3.10 生成 JWT Access Token
    ├─ 3.11 生成 Refresh Token 并写入 sessions 表
    ├─ 3.12 创建 LOGIN activity 记录
    ├─ 3.13 更新用户 last_access
    ├─ 3.14 重置登录尝试计数
    └─ 3.15 auth.login action 钩子 (success)
    ↓
4. 响应模式处理 (cookie/json/session)
    ↓
5. respond 中间件格式化响应
```

---

### 2.2 Payload 校验失败流程

**位置**: [local.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth/drivers/local.ts#L56-L89)

使用 Joi 进行 schema 校验：

```typescript
const userLoginSchema = Joi.object({
    email: Joi.string().email().required(),
    password: Joi.string().required(),
    mode: Joi.string().valid('cookie', 'json', 'session'),
    otp: Joi.string(),
}).unknown();

const { error } = userLoginSchema.validate(req.body);
if (error) {
    await stall(STALL_TIME, timeStart);  // 延迟响应，防止时序攻击
    throw new InvalidPayloadError({ reason: error.message });
}
```

**关键点**:
- `email` 必须是有效邮箱格式且必填
- `password` 必填
- `mode` 可选，仅支持 `cookie`/`json`/`session`
- `otp` 可选
- 校验失败后执行 `stall` 延迟，确保响应时间与成功路径一致

---

### 2.3 邮箱查找流程

**位置**: [local.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth/drivers/local.ts#L20-L36)

```typescript
async getUserID(payload: Record<string, any>): Promise<string> {
    if (!payload['email']) {
        throw new InvalidCredentialsError();
    }

    const user = await this.knex
        .select('id')
        .from('directus_users')
        .whereRaw('LOWER(??) = ?', ['email', payload['email'].toLowerCase()])
        .first();

    if (!user) {
        throw new InvalidCredentialsError();
    }

    return user.id;
}
```

**关键点**:
- 邮箱大小写不敏感匹配（`LOWER(email) = LOWER(payload.email)`）
- 邮箱缺失或用户不存在均抛出 `InvalidCredentialsError`
- 注意：此处与密码错误抛出相同错误，防止用户名枚举攻击

调用链：`AuthenticationService.login` → `provider.getUserID()`

---

### 2.4 Argon2 密码校验流程

**位置**: [local.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth/drivers/local.ts#L38-L46)

```typescript
async verify(user: User, password?: string): Promise<void> {
    if (!user.password || !(await argon2.verify(user.password, password as string))) {
        throw new InvalidCredentialsError();
    }
}

override async login(user: User, payload: Record<string, any>): Promise<void> {
    await this.verify(user, payload['password']);
}
```

**关键点**:
- 使用 `argon2` 库进行密码哈希验证
- 密码哈希存储在 `directus_users.password` 字段
- 空密码或验证失败均抛出 `InvalidCredentialsError`

调用链：`AuthenticationService.login` → `provider.login()` → `this.verify()`

---

### 2.5 登录尝试计数与用户 Suspended 流程

**位置**: [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L136-L194)

```typescript
const { auth_login_attempts: allowedAttempts } = await settingsService.readSingleton({
    fields: ['auth_login_attempts'],
});

if (allowedAttempts !== null) {
    loginAttemptsLimiter.points = allowedAttempts;
    
    try {
        await loginAttemptsLimiter.consume(user.id);
    } catch (error) {
        if (error instanceof RateLimiterRes && error.remainingPoints === 0) {
            // 超限：标记用户为 suspended
            await this.knex('directus_users').update({ status: 'suspended' }).where({ id: user.id });
            await getEntitlementManager().clearCache('sso_enabled', 'seats');

            // 记录 activity 和 revision
            if (this.accountability) {
                const activity = await this.activityService.createOne({
                    action: Action.UPDATE,
                    user: user.id,
                    ip: this.accountability.ip,
                    user_agent: this.accountability.userAgent,
                    origin: this.accountability.origin,
                    collection: 'directus_users',
                    item: user.id,
                });
                // ... 创建 revision 记录
            }

            user.status = 'suspended';
            await loginAttemptsLimiter.set(user.id, 0, 0);
        }
    }
}
```

**关键点**:
- 限流阈值从 `directus_settings.auth_login_attempts` 读取
- 使用 `rate-limiter-flexible` 库，支持 memory/redis 后端
- 超限后将用户 `status` 设为 `'suspended'`
- 创建 UPDATE activity 记录和 revision 审计记录
- 清除权限缓存（sso_enabled, seats）
- 重置限流计数，便于用户重新激活后正常登录

**用户状态检查**（登录失败路径）:
```typescript
if (user?.status !== 'active' || user?.provider !== providerName) {
    const loginError = new InvalidCredentialsError();
    emitStatus('fail', updatedPayload, user, loginError);
    await stall(STALL_TIME, timeStart);
    throw loginError;
}
```

注意：suspended 用户在第 129 行就被拦截，不会到达密码校验环节。

---

### 2.6 TFA（双因素认证）流程

**位置**: [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L204-L221)

**场景1：TFA密钥存在但OTP缺失**
```typescript
if (user.tfa_secret && !options?.otp) {
    const loginError = new InvalidOtpError();
    emitStatus('fail', updatedPayload, user, loginError);
    await stall(STALL_TIME, timeStart);
    throw loginError;
}
```

**场景2：TFA密钥存在且提供了OTP，但验证失败**
```typescript
if (user.tfa_secret && options?.otp) {
    const tfaService = new TFAService({ knex: this.knex, schema: this.schema });
    const otpValid = await tfaService.verifyOTP(user.id, options?.otp);

    if (otpValid === false) {
        const loginError = new InvalidOtpError();
        emitStatus('fail', updatedPayload, user, loginError);
        await stall(STALL_TIME, timeStart);
        throw loginError;
    }
}
```

**TFA验证实现** [tfa.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/tfa.ts#L18-L30):
```typescript
async verifyOTP(key: PrimaryKey, otp: string, secret?: string): Promise<boolean> {
    if (secret) {
        return authenticator.check(otp, secret);
    }
    const user = await this.knex.select('tfa_secret').from('directus_users').where({ id: key }).first();
    if (!user?.tfa_secret) {
        throw new InvalidPayloadError({ reason: `User "${key}" doesn't have TFA enabled` });
    }
    return authenticator.check(otp, user.tfa_secret);
}
```

**关键点**:
- 使用 `otplib` 库的 `authenticator.check()` 进行 TOTP 验证
- `tfa_secret` 存储在 `directus_users.tfa_secret` 字段
- 两种失败场景均抛出 `InvalidOtpError`
- 失败后同样执行 stall 延迟

---

### 2.7 License 锁定对非 Admin 的阻断

**位置**: [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L230-L234)

```typescript
if ((await getLicenseManager().isLocked()) && globalAccess.admin === false) {
    throw new ResourceRestrictedError({
        category: 'login',
    });
}
```

**License锁定判断** [manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts#L248-L252):
```typescript
public async isLocked() {
    const status = await this.getStatus();
    return status === 'locked';
}
```

**关键点**:
- 通过 `fetchGlobalAccess()` 获取用户是否有 admin 权限
- license 锁定状态由 `computeLicenseStatus()` 根据 license 有效性计算
- **admin 用户不受 license 锁定影响**，可继续登录进行问题排查
- 非 admin 用户抛出 `ResourceRestrictedError`（category: 'login'）

---

### 2.8 JWT 与 Session 写入流程

**位置**: [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L236-L300)

**Step 1: 构建 Token Payload**
```typescript
const tokenPayload: DirectusTokenPayload = {
    id: user.id,
    role: user.role,
    app_access: globalAccess.app,
    admin_access: globalAccess.admin,
};

// 检查是否需要强制设置 TFA
if (!user.tfa_secret) {
    const roleEnforcement = await this.knex
        .select('directus_policies.enforce_tfa')
        // ... 关联查询角色策略
        .where('directus_users.id', user.id)
        .where('directus_policies.enforce_tfa', true)
        .first();
    
    if (roleEnforcement) {
        tokenPayload.enforce_tfa = true;
    }
}

// session 模式下，将 refresh_token 嵌入 payload
if (options?.session) {
    tokenPayload.session = refreshToken;
}
```

**Step 2: 生成 Refresh Token**
```typescript
const refreshToken = nanoid(64);  // 64位随机字符串
const refreshTokenExpiration = new Date(Date.now() + getMilliseconds(env['REFRESH_TOKEN_TTL'], 0));
```

**Step 3: 触发 JWT 自定义声明钩子**
```typescript
const customClaims = await emitter.emitFilter(
    'auth.jwt',
    tokenPayload,
    { status: 'pending', user: user?.id, provider: providerName, type: 'login' },
    // ...
);
```

**Step 4: 签名 JWT Access Token**
```typescript
const TTL = env[options?.session ? 'SESSION_COOKIE_TTL' : 'ACCESS_TOKEN_TTL'] as StringValue | number;
const accessToken = jwt.sign(customClaims, getSecret(), {
    expiresIn: TTL,
    issuer: 'directus',
});
```

**Step 5: 写入 Session 表并清理过期 Session**
```typescript
await this.knex('directus_sessions').insert({
    token: refreshToken,
    user: user.id,
    expires: refreshTokenExpiration,
    ip: this.accountability?.ip,
    user_agent: this.accountability?.userAgent,
    origin: this.accountability?.origin,
});

await this.knex('directus_sessions').delete().where('expires', '<', new Date());
```

**关键点**:
- Refresh Token 使用 `nanoid(64)` 生成（64字符随机字符串）
- JWT 使用 `HS256` 算法，密钥来自 `getSecret()`
- Session 表记录包含 IP、User-Agent、Origin 等审计信息
- 每次登录自动清理过期 session

---

### 2.9 Activity 与 last_access 更新

**位置**: [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L302-L314)

**创建登录 Activity 记录**:
```typescript
if (this.accountability) {
    await this.activityService.createOne({
        action: Action.LOGIN,
        user: user.id,
        ip: this.accountability.ip,
        user_agent: this.accountability.userAgent,
        origin: this.accountability.origin,
        collection: 'directus_users',
        item: user.id,
    });
}
```

**更新用户 last_access 时间**:
```typescript
await this.knex('directus_users').update({ last_access: new Date() }).where({ id: user.id });
```

**关键点**:
- Activity 记录包含完整的访问上下文（IP、User-Agent、Origin）
- `last_access` 字段每次登录都更新，反映用户最近活动时间
- ActivityService 继承自 ItemsService，操作 `directus_activity` 表

---

### 2.10 登录成功后的清理

**位置**: [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L316-L320)

```typescript
if (allowedAttempts !== null) {
    await loginAttemptsLimiter.set(user.id, 0, 0);
}

await stall(STALL_TIME, timeStart);
```

**关键点**:
- 登录成功后重置登录尝试计数
- 执行 stall 确保总耗时与失败路径一致，防止时序攻击
- 触发 `auth.login` action 钩子（status: 'success'）

---

## 3. Cookie 模式与 JSON 模式对比

### 3.1 响应处理位置

**local.ts 登录响应**: [local.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth/drivers/local.ts#L91-L114)
**auth.ts refresh 响应**: [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth.ts#L121-L154)

### 3.2 模式判断逻辑

```typescript
// 登录接口的默认模式是 json
const mode: AuthenticationMode = req.body.mode ?? 'json';

// refresh 接口的模式判断
function getCurrentMode(req: Request): AuthenticationMode {
    if (req.body.mode) {
        return req.body.mode as AuthenticationMode;
    }
    if (req.body.refresh_token) {
        return 'json';
    }
    return 'cookie';
}
```

### 3.3 差异对比表

| 特性 | JSON 模式 (`mode: 'json'`) | Cookie 模式 (`mode: 'cookie'`) | Session 模式 (`mode: 'session'`) |
|------|---------------------------|--------------------------------|----------------------------------|
| **refresh_token 传输** | 返回在 JSON 响应体 `refresh_token` 字段 | 设置在 Cookie: `directus_refresh_token` | 不单独返回，嵌入 session cookie |
| **access_token 传输** | 返回在 JSON 响应体 `access_token` 字段 | 返回在 JSON 响应体 `access_token` 字段 | 设置在 Cookie: `directus_session_token` |
| **expires 字段** | JSON 响应体 `expires`（毫秒） | JSON 响应体 `expires`（毫秒） | JSON 响应体 `expires`（毫秒） |
| **Cookie 属性** | 无 | httpOnly, secure, sameSite, maxAge=REFRESH_TOKEN_TTL | httpOnly, secure, sameSite, maxAge=SESSION_COOKIE_TTL |
| **刷新时 token 来源** | 请求体 `refresh_token` 字段 | Cookie `directus_refresh_token` | Cookie `directus_session_token` 解析出 session |
| **JWT payload** | 标准 payload | 标准 payload | payload 含 `session` 字段 |
| **安全性** | 需前端安全存储 refresh_token | httpOnly 防止 XSS 窃取 refresh_token | 最安全，所有 token 均为 httpOnly |
| **适用场景** | 移动端、桌面端、SPA 无 Cookie 环境 | 传统 Web 应用、服务端渲染 | 需高安全性的 Web 应用 |

### 3.4 Cookie 配置详情

[constants.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts#L66-L80):

```typescript
export const REFRESH_COOKIE_OPTIONS: CookieOptions = {
    httpOnly: true,
    domain: env['REFRESH_TOKEN_COOKIE_DOMAIN'] as string,
    maxAge: getMilliseconds(env['REFRESH_TOKEN_TTL'] as string),
    secure: Boolean(env['REFRESH_TOKEN_COOKIE_SECURE']),
    sameSite: (env['REFRESH_TOKEN_COOKIE_SAME_SITE'] || 'strict') as 'lax' | 'strict' | 'none',
};

export const SESSION_COOKIE_OPTIONS: CookieOptions = {
    httpOnly: true,
    domain: env['SESSION_COOKIE_DOMAIN'] as string,
    maxAge: getMilliseconds(env['SESSION_COOKIE_TTL'] as string),
    secure: Boolean(env['SESSION_COOKIE_SECURE']),
    sameSite: (env['SESSION_COOKIE_SAME_SITE'] || 'strict') as 'lax' | 'strict' | 'none',
};
```

### 3.5 刷新凭证时的差异

**refresh 接口模式处理** [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth.ts#L121-L148):

```typescript
const mode = getCurrentMode(req);
const currentRefreshToken = getCurrentRefreshToken(req, mode);

// 获取 refresh token 的逻辑
function getCurrentRefreshToken(req: Request, mode: AuthenticationMode): string | undefined {
    if (mode === 'json') {
        return req.body.refresh_token;
    }
    if (mode === 'cookie') {
        return req.cookies[env['REFRESH_TOKEN_COOKIE_NAME'] as string];
    }
    if (mode === 'session') {
        const token = req.cookies[env['SESSION_COOKIE_NAME'] as string];
        if (isDirectusJWT(token)) {
            const payload = verifyAccessJWT(token, getSecret());
            return payload.session;  // 从 JWT payload 中提取 session
        }
    }
    return undefined;
}
```

---

## 4. 错误处理与 Stall 机制

### 4.1 所有失败路径均执行 Stall

```typescript
const STALL_TIME = env['LOGIN_STALL_TIME'] as number;
const timeStart = performance.now();

// ... 所有失败路径均调用:
await stall(STALL_TIME, timeStart);
```

**stall 实现**: 确保无论成功失败，总耗时至少为 `STALL_TIME`，防止通过响应时间差异进行用户枚举或密码爆破攻击。

### 4.2 事件钩子

登录流程中有两个关键事件钩子：

1. **Filter 钩子** - `auth.login`（状态 'pending'）: 可修改 payload
2. **Action 钩子** - `auth.login`（状态 'success' / 'fail'）: 审计、通知等

---

## 5. 关键文件索引

| 文件 | 核心职责 |
|------|----------|
| [local.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth/drivers/local.ts) | 本地认证驱动，payload 校验、邮箱查找、argon2 密码校验 |
| [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts) | AuthenticationService，登录主流程控制 |
| [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/auth.ts) | 路由注册、refresh/logout 处理 |
| [tfa.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/tfa.ts) | TFA OTP 验证 |
| [manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts) | License 管理，isLocked 检查 |
| [rate-limiter.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/rate-limiter.ts) | 登录尝试限流 |
| [constants.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts) | Cookie 选项配置 |
| [activity.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/activity.ts) | Activity 记录服务 |
