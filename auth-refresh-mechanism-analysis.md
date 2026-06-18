# Directus 认证刷新机制深度分析

## 目录

1. [/auth/refresh 端点与 refresh_token 定位](#1-authrefresh-端点与-refresh_token-定位)
2. [从 directus_sessions 重建会话](#2-从-directus_sessions-重建会话)
3. [updateStatefulSession 并发刷新处理](#3-updatestatefulsession-并发刷新处理)
4. [SDK Authentication 组合件](#4-sdk-authentication-组合件)
5. [各类会话的差异对比](#5-各类会话的差异对比)

---

## 1. /auth/refresh 端点与 refresh_token 定位

### 1.1 认证模式概览

Directus 支持三种认证模式，不同模式下 `refresh_token` 的获取和传递方式不同：

| 模式 | refresh_token 来源 | access_token 传递 |
|------|-------------------|------------------|
| `json` | 请求体 `refresh_token` 字段 | JSON 响应体 |
| `cookie` | Cookie 中的刷新令牌 Cookie | JSON 响应体 |
| `session` | Cookie 中的会话 Cookie（需解码 JWT） | Cookie 中的会话 Cookie |

### 1.2 模式判定逻辑

模式判定在 [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/auth.ts#L69-L79) 的 `getCurrentMode` 函数中实现：

```typescript
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

**判定优先级：**
1. 请求体中显式指定的 `mode` 参数
2. 如果请求体包含 `refresh_token` 字段，自动判定为 `json` 模式
3. 默认使用 `cookie` 模式

### 1.3 refresh_token 定位机制

`getCurrentRefreshToken` 函数在 [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/auth.ts#L81-L100) 中实现，根据不同模式从不同位置提取 refresh_token：

**JSON 模式**
- 直接从请求体 `req.body.refresh_token` 获取
- 适用于移动端或无法使用 Cookie 的客户端

**Cookie 模式**
- 从 Cookie 中获取，Cookie 名称由环境变量 `REFRESH_TOKEN_COOKIE_NAME` 配置
- 通过 `req.cookies[env['REFRESH_TOKEN_COOKIE_NAME']]` 读取
- Cookie 配置见 [constants.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts#L66-L72) 的 `REFRESH_COOKIE_OPTIONS`

**Session 模式**
- 从 Cookie 中获取会话令牌（名称由 `SESSION_COOKIE_NAME` 配置）
- 会话令牌本身是一个 JWT，需要解码后提取 `session` 字段作为 refresh_token
- 使用 `isDirectusJWT` 验证是否为 Directus 签发的 JWT
- 使用 `verifyAccessJWT` 验证签名并解码 payload

```typescript
if (mode === 'session') {
    const token = req.cookies[env['SESSION_COOKIE_NAME'] as string];

    if (isDirectusJWT(token)) {
        const payload = verifyAccessJWT(token, getSecret());
        return payload.session;
    }
}
```

Session 模式下，access_token 和 refresh_token 是合二为一的——会话 JWT 既是访问令牌，又通过 `session` claim 引用了底层的会话记录。

### 1.4 刷新端点响应

根据模式的不同，响应中返回令牌的方式也不同：

- **json 模式**：access_token 和 refresh_token 都在 JSON 响应体中
- **cookie 模式**：access_token 在 JSON 响应体中，refresh_token 通过 Set-Cookie 设置
- **session 模式**：新的会话 JWT 通过 Set-Cookie 设置，响应体只返回 expires

---

## 2. 从 directus_sessions 重建会话

### 2.1 会话表结构

`directus_sessions` 表存储所有类型的会话记录，核心字段包括：

| 字段 | 说明 |
|------|------|
| `token` | 会话令牌（refresh_token 的值） |
| `user` | 关联的用户 ID（用户会话） |
| `share` | 关联的共享 ID（共享会话） |
| `expires` | 会话过期时间 |
| `next_token` | 指向下一个会话令牌（用于并发刷新） |
| `ip` | 客户端 IP |
| `user_agent` | 用户代理 |
| `origin` | 请求来源 |
| `oauth_client` | OAuth 客户端 ID（OAuth 会话） |

### 2.2 会话查询逻辑

在 [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L341-L375) 的 `refresh` 方法中，通过复杂的查询重建会话：

```typescript
const record = await this.knex
    .select({
        session_expires: 's.expires',
        session_next_token: 's.next_token',
        user_id: 'u.id',
        // ... 用户字段
        share_id: 'd.id',
        share_start: 'd.date_start',
        share_end: 'd.date_end',
    })
    .from('directus_sessions AS s')
    .leftJoin('directus_users AS u', 's.user', 'u.id')
    .leftJoin('directus_shares AS d', 's.share', 'd.id')
    .where('s.token', refreshToken)
    .andWhere('s.expires', '>=', new Date())
    .andWhere('s.oauth_client', null)
    .andWhere((subQuery) => {
        subQuery.whereNull('d.date_end').orWhere('d.date_end', '>=', new Date());
    })
    .andWhere((subQuery) => {
        subQuery.whereNull('d.date_start').orWhere('d.date_start', '<=', new Date());
    })
    .first();
```

**查询条件解析：**
1. `s.token = refreshToken` - 匹配会话令牌
2. `s.expires >= NOW()` - 会话未过期
3. `s.oauth_client IS NULL` - 排除 OAuth 会话（OAuth 有独立的刷新流程）
4. 共享时间窗口验证 - 共享的 `date_start` 和 `date_end` 在有效范围内

### 2.3 用户会话重建

当查询结果包含 `user_id` 时，进行用户会话重建：

**步骤 1：用户状态验证**
```typescript
if (record.user_id && record.user_status !== 'active') {
    await this.knex('directus_sessions').where({ token: refreshToken }).del();
    
    if (record.user_status === 'suspended') {
        throw new UserSuspendedError();
    } else {
        throw new InvalidCredentialsError();
    }
}
```
- 非 `active` 状态的用户直接删除会话并拒绝
- `suspended` 状态返回专门的错误信息
- 其他状态返回通用的无效凭证错误

**步骤 2：角色与权限重建**
```typescript
const roles = await fetchRolesTree(record.user_role, { knex: this.knex });
const globalAccess = await fetchGlobalAccess(
    { user: record.user_id, roles, ip: this.accountability?.ip ?? null },
    { knex: this.knex },
);
```
- `fetchRolesTree` 获取角色继承链
- `fetchGlobalAccess` 计算全局访问权限（app_access, admin_access）

**步骤 3：Auth Provider 刷新回调**
```typescript
if (record.user_id) {
    const provider = getAuthProvider(record.user_provider);
    await provider.refresh({ /* 用户信息 */ });
}
```
- 调用对应认证驱动的 `refresh` 方法
- 不同驱动（local、oauth2、ldap 等）可在此执行特定逻辑

### 2.4 共享会话重建

当查询结果包含 `share_id` 时，进行共享会话重建。共享会话的 token payload 具有特殊结构：

```typescript
if (record.share_id) {
    tokenPayload.share = record.share_id;
    tokenPayload.role = null;
    tokenPayload.app_access = false;
    tokenPayload.admin_access = false;
    delete tokenPayload.id;
}
```

**共享会话的特点：**
- 没有用户 ID（`id` 被删除）
- `role` 为 `null`
- `app_access` 和 `admin_access` 都为 `false`
- 包含 `share` 字段标识所属的共享记录
- 权限由共享记录本身控制，而非用户角色

共享会话的登录逻辑在 [shares.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/shares.ts#L60-L137) 的 `login` 方法中。

### 2.5 会话验证中间件

在实际请求处理中，通过 [get-accountability-for-token.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-accountability-for-token.ts) 将令牌转换为 accountability 对象：

```typescript
if ('session' in payload) {
    const { oauth_client } = await verifySessionJWT(payload);
    accountability.session = payload.session;
    // ... OAuth 相关处理
}

if (payload.share) accountability.share = payload.share;
if (payload.id) accountability.user = payload.id;
```

`verifySessionJWT` 函数在 [verify-session-jwt.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/verify-session-jwt.ts) 中实现，验证会话记录仍然存在且未过期。

---

## 3. updateStatefulSession 并发刷新处理

### 3.1 问题背景

在高并发场景下，多个请求可能同时使用同一个 refresh_token 进行刷新。如果简单地每次都生成新的 refresh_token 并替换旧的，会导致并发请求中只有一个成功，其他失败。

Directus 通过 `next_token` 机制和宽限期（grace period）来解决这个问题。

### 3.2 核心实现

`updateStatefulSession` 是 [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L499-L556) 中的私有方法，专门处理 session 模式下的会话刷新。

### 3.3 处理流程

**场景 1：会话已有 next_token（已被刷新过）**

```typescript
if (sessionRecord['session_next_token']) {
    await this.knex('directus_sessions')
        .update({
            expires: sessionExpiration,
        })
        .where({ token: newSessionToken });

    return newSessionToken;
}
```

- 如果当前会话已经有 `next_token`，说明它已经被另一个并发请求刷新过了
- 直接使用已有的 `next_token`，只更新它的过期时间
- 避免创建重复的新会话

**场景 2：首次刷新（无 next_token）**

**第一步：设置宽限期和 next_token**
```typescript
const GRACE_PERIOD = getMilliseconds(env['SESSION_REFRESH_GRACE_PERIOD'], 10_000);

const updatedSession = await this.knex('directus_sessions')
    .update(
        {
            next_token: newSessionToken,
            expires: new Date(Date.now() + GRACE_PERIOD),
        },
        ['next_token'],
    )
    .where({ token: oldSessionToken, next_token: null });
```

- 旧会话设置 `next_token` 指向下一个会话
- 旧会话的过期时间缩短为宽限期（默认 10 秒）
- 使用 `WHERE next_token IS NULL` 确保原子性——只有第一个请求能成功更新

**第二步：处理并发冲突**
```typescript
if (updatedSession.length === 0) {
    const { next_token } = await this.knex('directus_sessions')
        .select('next_token')
        .where({ token: oldSessionToken })
        .first();

    return next_token;
}
```
- 如果更新影响了 0 行，说明另一个并发请求已经先一步设置了 `next_token`
- 查询并返回已有的 `next_token`
- 这是乐观并发控制的典型应用

**第三步：创建新会话记录**
```typescript
await this.knex('directus_sessions').insert({
    token: newSessionToken,
    user: sessionRecord['user_id'],
    share: sessionRecord['share_id'],
    expires: sessionExpiration,
    ip: this.accountability?.ip,
    user_agent: this.accountability?.userAgent,
    origin: this.accountability?.origin,
    oauth_client: sessionRecord['oauth_client'],
});
```
- 只有第一个成功的请求会创建新的会话记录
- 新会话继承旧会话的用户、共享、OAuth 客户端等属性

### 3.4 宽限期的作用

宽限期（`SESSION_REFRESH_GRACE_PERIOD`，默认 10 秒）是并发刷新的关键设计：

1. **平滑过渡**：旧会话不会立即失效，在宽限期内仍然有效
2. **并发容错**：多个并发请求即使拿到的都是旧 token，在宽限期内都能成功刷新
3. **客户端重试**：如果客户端在刷新过程中遇到问题，有短暂的窗口期进行重试

### 3.5 与无状态刷新的对比

在非 session 模式下，刷新逻辑简单直接：

```typescript
// 无状态刷新（cookie/json 模式）
await this.knex('directus_sessions')
    .update({
        token: newRefreshToken,
        expires: refreshTokenExpiration,
    })
    .where({ token: refreshToken });
```

无状态模式只是简单地更新 token 和 expires，没有并发保护机制。

---

## 4. SDK Authentication 组合件

### 4.1 概述

SDK 的 authentication 组合件在 [composable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts) 中实现，提供了完整的客户端认证管理功能。

### 4.2 核心组件

```
┌──────────────────────────────────────────────────┐
│           authentication composable              │
├──────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌──────────────────────┐   │
│  │   storage   │    │  refreshPromise      │   │
│  │ (memory/自定义) │  │  (并发刷新锁)       │   │
│  └─────────────┘    └──────────────────────┘   │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │  refreshTimeout (自动刷新定时器)          │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

### 4.3 getToken - 获取访问令牌

```typescript
async getToken() {
    await refreshIfExpired().catch(() => {
        /* fail gracefully */
    });

    const data = await storage.get();
    return data?.access_token ?? null;
}
```

**行为特点：**
- 每次获取令牌前都会检查是否需要刷新
- 刷新失败时优雅降级（仍然返回当前存储的令牌）
- 返回 `null` 表示没有有效的访问令牌

### 4.4 refreshIfExpired - 智能刷新

```typescript
const refreshIfExpired = async () => {
    const authData = await storage.get();

    if (refreshPromise || !authData?.expires_at) {
        return activeRefresh();
    }

    if (authData.expires_at < new Date().getTime() + authConfig.msRefreshBeforeExpires) {
        refresh().catch((_err) => {
            /* throw err; */
        });
    }

    return activeRefresh();
};
```

**刷新判定逻辑：**
1. 如果已有正在进行的刷新（`refreshPromise` 存在），等待它完成
2. 如果没有过期时间，直接等待当前刷新（或无操作）
3. 如果令牌将在 `msRefreshBeforeExpires`（默认 30 秒）内过期，触发刷新
4. 返回当前刷新的 Promise

**并发控制：**
- `refreshPromise` 作为单例锁，确保同时只有一个刷新请求在进行
- `activeRefresh()` 辅助函数等待刷新完成并清理 `refreshPromise`

### 4.5 setCredentials - 设置凭证

```typescript
const setCredentials = async (data: AuthenticationData) => {
    const expires = data.expires ?? 0;
    data.expires_at = new Date().getTime() + expires;
    await storage.set(data);

    if (authConfig.autoRefresh && expires > authConfig.msRefreshBeforeExpires && expires < MAX_INT32) {
        if (refreshTimeout) clearTimeout(refreshTimeout);

        refreshTimeout = setTimeout(() => {
            refreshTimeout = null;
            refresh().catch((_err) => {
                /* throw err; */
            });
        }, expires - authConfig.msRefreshBeforeExpires);
    }
};
```

**功能：**
1. 计算绝对过期时间 `expires_at`
2. 将认证数据保存到 storage
3. 如果启用了自动刷新，设置定时器在令牌过期前自动刷新

**定时器注意事项：**
- 只在 `autoRefresh` 开启且 TTL 在合理范围内时设置
- `MAX_INT32` 限制防止 setTimeout 溢出（约 24 天）
- 每次设置新定时器前清除旧的

### 4.6 refresh - 执行刷新

```typescript
const refresh = async (options: Omit<RefreshOptions, 'refresh_token'> = {}) => {
    const awaitRefresh = async () => {
        const authData = await storage.get();

        const fetchOptions: RequestInit = {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
        };

        const body: Record<string, string> = { mode: options.mode ?? mode };

        if (mode === 'json' && authData?.refresh_token) {
            body['refresh_token'] = authData.refresh_token;
        }

        fetchOptions.body = JSON.stringify(body);
        const requestUrl = getRequestUrl(client.url, '/auth/refresh');
        const data = await request<AuthenticationData>(requestUrl.toString(), fetchOptions, client.globals.fetch);

        await resetStorage();
        await setCredentials(data);
        return data;
    };

    refreshPromise = awaitRefresh();
    return refreshPromise;
};
```

**刷新流程：**
1. 从 storage 读取当前认证数据
2. 根据模式决定 refresh_token 的传递方式（json 模式放 body，cookie/session 模式靠浏览器自动带 cookie）
3. 调用 `/auth/refresh` 接口
4. 重置旧的存储
5. 设置新凭证（会重新设置自动刷新定时器）

### 4.7 logout - 登出

```typescript
async logout(options: Omit<LogoutOptions, 'refresh_token'> = {}) {
    const authData = await storage.get();

    const fetchOptions: RequestInit = {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
    };

    const body: Record<string, string> = { mode: options.mode ?? mode };

    if (mode === 'json' && authData?.refresh_token) {
        body['refresh_token'] = authData.refresh_token;
    }

    fetchOptions.body = JSON.stringify(body);
    const requestUrl = getRequestUrl(client.url, '/auth/logout');
    await request(requestUrl.toString(), fetchOptions, client.globals.fetch);

    this.stopRefreshing();
    await resetStorage();
}
```

**登出流程：**
1. 调用 `/auth/logout` 接口通知服务端销毁会话
2. 停止自动刷新定时器
3. 清空本地存储

### 4.8 各函数配合关系

```
用户调用 getToken()
      │
      ▼
refreshIfExpired()
      │
      ├──► 已有刷新进行中？ ──是──► 等待刷新完成
      │                           │
      否                          │
      │                           │
      ▼                           │
是否即将过期？                      │
      │                           │
      是──► refresh() ◄───────────┘
      │       │
      否       ├──► 调用 /auth/refresh API
      │       │
      │       ├──► resetStorage()
      │       │
      │       └──► setCredentials()
      │               │
      │               ├──► 计算 expires_at
      │               ├──► storage.set()
      │               └──► 设置自动刷新定时器
      │
      ▼
返回 access_token
```

### 4.9 存储抽象

SDK 通过 `AuthenticationStorage` 接口支持自定义存储：

```typescript
export interface AuthenticationStorage {
    get: () => Promise<AuthenticationData | null> | AuthenticationData | null;
    set: (value: AuthenticationData | null) => Promise<unknown> | unknown;
}
```

默认实现是 [memoryStorage](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/utils/memory-storage.ts)，实际应用中可以替换为 localStorage、sessionStorage 或其他持久化方案。

---

## 5. 各类会话的差异对比

### 5.1 会话类型总览

| 类型 | user 字段 | share 字段 | oauth_client 字段 | 刷新入口 |
|------|----------|-----------|------------------|---------|
| 用户会话 | 有值 | NULL | NULL | `/auth/refresh` |
| 共享会话 | NULL | 有值 | NULL | `/auth/refresh` |
| OAuth 会话 | 有值 | NULL | 有值 | `/mcp-oauth/token` |

### 5.2 无效会话

**判定条件：**
- 会话记录不存在（token 不匹配）
- 会话已过期（`expires < NOW()`）
- 对应的用户已被删除（`user_id` 为 NULL 且 `share_id` 也为 NULL）
- 对应的共享已过期或被删除

**处理方式：**
- 抛出 `InvalidCredentialsError`
- 客户端需要重新登录

**相关代码**：[authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L373-L375)

### 5.3 OAuth Session（MCP OAuth 会话）

**特点：**
- `oauth_client` 字段不为空，关联到 `directus_oauth_clients`
- 通过 `/mcp-oauth/token` 端点刷新，而非 `/auth/refresh`
- 使用独立的刷新机制（refresh token rotation + reuse detection）
- 会话 token 存储的是 SHA-256 哈希，而非明文

**与普通用户会话的区别：**

| 特性 | 普通用户会话 | OAuth 会话 |
|------|------------|-----------|
| 刷新端点 | `/auth/refresh` | `/mcp-oauth/token` |
| refresh_token 存储 | 明文 | SHA-256 哈希 |
| 并发刷新策略 | next_token + 宽限期 | 单次使用 + 重用检测 |
| 刷新时创建新会话 | 是（session 模式） | 是（每次都轮换） |
| 重用检测 | 无（宽限期内允许） | 有（立即吊销） |

**重用检测机制：**
OAuth 会话在刷新时使用 `previous_session` 字段实现重用检测：
1. 刷新时删除旧会话记录
2. 更新 grant 的 `session` 为新哈希，`previous_session` 为旧哈希
3. 如果用旧的 refresh_token 再次刷新，会检测到 `previous_session` 匹配
4. 检测到重用后立即吊销整个 grant（删除 grant 和当前会话）

相关代码在 [mcp-oauth/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/mcp-oauth/index.ts#L1076-L1286) 的 `refreshToken` 方法中。

### 5.4 Inactive 用户

**用户状态枚举：**
```typescript
status: 'active' | 'suspended' | 'invited'
```

**各状态的含义：**
- `active`：正常用户，可以登录和使用
- `suspended`：因登录尝试次数过多等原因被暂时封禁
- `invited`：已邀请但未激活的用户

**刷新时的处理：**
```typescript
if (record.user_id && record.user_status !== 'active') {
    await this.knex('directus_sessions').where({ token: refreshToken }).del();
    
    if (record.user_status === 'suspended') {
        throw new UserSuspendedError();
    } else {
        throw new InvalidCredentialsError();
    }
}
```

**关键行为：**
1. 非 active 用户的会话会被立即删除
2. `suspended` 返回专门的错误码 `USER_SUSPENDED`
3. 其他状态（如 `invited`）返回通用的 `INVALID_CREDENTIALS`
4. 删除会话后即使用户重新激活也需要重新登录

### 5.5 Share 会话（共享会话）

**特点：**
- `share` 字段关联到 `directus_shares` 表
- 没有关联的用户（`user` 为 NULL）
- 权限由共享记录决定，而非用户角色
- 有独立的登录端点 `/shares/auth`

**与用户会话的对比：**

| 特性 | 用户会话 | 共享会话 |
|------|---------|---------|
| 关联实体 | directus_users | directus_shares |
| token 中的 id claim | 有（用户 ID） | 无 |
| token 中的 role claim | 有（用户角色） | null |
| app_access | 基于角色 | false |
| admin_access | 基于角色 | false |
| share claim | 无 | 有（共享 ID） |
| 登录端点 | `/auth/login` | `/shares/auth` |
| 刷新端点 | `/auth/refresh` | `/auth/refresh`（相同） |

**共享的额外限制：**
- `date_start` 和 `date_end` 时间窗口限制
- `max_uses` 使用次数限制
- 可选的 password 保护

相关代码在 [shares.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/shares.ts) 中。

### 5.6 会话清理

**自动清理机制：**
1. 登录时清理：每次登录后清理所有过期的会话
2. 刷新时清理：每次刷新后清理当前用户/共享的过期会话
3. 定时任务：OAuth 会话有专门的定时清理任务

**登录时的清理**（authentication.ts）：
```typescript
await this.knex('directus_sessions').delete().where('expires', '<', new Date());
```

**刷新时的清理**（authentication.ts）：
```typescript
await this.knex('directus_sessions')
    .delete()
    .where({
        user: record.user_id,
        share: record.share_id,
    })
    .andWhere('expires', '<', new Date());
```

---

## 附录：关键配置项

| 环境变量 | 默认值 | 说明 |
|---------|-------|------|
| `REFRESH_TOKEN_TTL` | 7d | refresh_token 有效期 |
| `ACCESS_TOKEN_TTL` | 15m | access_token 有效期 |
| `SESSION_COOKIE_TTL` | 7d | 会话 Cookie 有效期 |
| `SESSION_REFRESH_GRACE_PERIOD` | 10s | 会话刷新宽限期 |
| `REFRESH_TOKEN_COOKIE_NAME` | directus_refresh_token | 刷新令牌 Cookie 名称 |
| `SESSION_COOKIE_NAME` | directus_session | 会话 Cookie 名称 |

---

*文档生成时间：2026-06-19*
