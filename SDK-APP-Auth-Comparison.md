# SDK 层与 App 层 REST 请求认证分工对比

本文档详细对比 Directus SDK 层和 App 层在 REST 请求认证处理上的职责分工与协作机制。

---

## 一、SDK 层认证机制

### 1.1 `createDirectus()` 与 `with` 组合件

**核心文件**：[client.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/client.ts)

`createDirectus(url, options)` 创建一个基础客户端，包含三个核心属性：

```typescript
{
  globals,          // 注入的全局对象（fetch, WebSocket, URL, logger）
  url,              // Directus 服务 URL（URL 对象）
  with(createExtension)  // 组合件挂载方法
}
```

**`with` 组合件的合成原理**（[client.ts#L26-L31](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/client.ts#L26-L31)）：

```typescript
with(createExtension) {
    return {
        ...this,              // 展开当前 client 的所有属性
        ...createExtension(this),  // 执行组合件函数，注入当前 client 上下文，合并返回的新能力
    };
}
```

这是一个典型的**装饰器模式**实现。每个组合件（如 `rest()`、`authentication()`）是一个高阶函数，接收当前 client 作为参数，返回包含新方法和属性的对象，通过对象展开实现能力叠加。

**调用示例**（App 层）：[sdk.ts#L52-L54](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/sdk.ts#L52-L54)

```typescript
export const sdk = createDirectus(getPublicURL(), { globals: { fetch: baseClient.native } })
    .with(authentication('session', { credentials: 'include', msRefreshBeforeExpires: SDK_AUTH_REFRESH_BEFORE_EXPIRES }))
    .with(rest({ credentials: 'include' }));
```

最终合成的 client 同时具备：基础属性 + authentication 能力 + REST 能力。

---

### 1.2 `rest` 组合件

**核心文件**：[rest/composable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/composable.ts)

rest 组合件负责所有 REST API 请求的发起，在认证方面承担以下职责：

#### 1.2.1 Content-Type 设置

**代码位置**：[rest/composable.ts#L22-L31](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/composable.ts#L22-L31)

```typescript
// 所有 API 请求默认设置 Content-Type
if (!options.headers) {
    options.headers = {};
}

if ('Content-Type' in options.headers === false) {
    options.headers['Content-Type'] = 'application/json';
}
```

默认注入 `application/json`，适用于绝大多数 JSON 请求体场景。

#### 1.2.2 multipart 删除 header

**代码位置**：[rest/composable.ts#L28-L31](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/composable.ts#L28-L31)

```typescript
else if (options.headers['Content-Type'] === 'multipart/form-data') {
    // 让 fetch 函数自行处理 multipart boundaries
    delete options.headers['Content-Type'];
}
```

**设计原因**：`multipart/form-data` 请求需要包含正确的 `boundary` 参数（如 `Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryxxx`）。如果手动设置该 header 而不携带正确 boundary，将导致服务端解析失败。因此删除手动设置的 header，让底层 fetch 实现根据 FormData 自动生成带 boundary 的 Content-Type。

#### 1.2.3 Bearer Token 注入

**代码位置**：[rest/composable.ts#L34-L40](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/composable.ts#L34-L40)

```typescript
if ('getToken' in this && 'Authorization' in options.headers === false) {
    const token = await (this.getToken as StaticTokenClient<Schema>['getToken'])();

    if (token) {
        options.headers['Authorization'] = `Bearer ${token}`;
    }
}
```

关键点：
- **`this` 动态绑定**：使用 `this` 而非 `client`，以访问经过 `with()` 叠加后的 `getToken` 方法（来自 authentication 组合件）
- **条件注入**：仅当 client 上存在 `getToken` 方法且请求未手动设置 Authorization header 时才注入
- **异步获取**：`getToken()` 是异步的，内部可能触发 token 刷新逻辑

**辅助工具** `withToken()`：[with-token.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/helpers/with-token.ts) 提供命令级别的 token 注入，用于静态 token 场景或覆盖默认 token。

#### 1.2.4 Hook 机制

**代码位置**：[rest/composable.ts#L58-L77](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/composable.ts#L58-L77)

两层 Hook，按以下顺序执行：

| 层级 | 触发时机 | 配置来源 | 作用 |
|------|----------|----------|------|
| 命令级 onRequest | 请求发送前 | RestCommand 中设置 | 单个请求的定制化修改 |
| 全局 onRequest | 命令级之后 | `rest({ onRequest })` 配置 | 所有请求的统一修改（如加 header） |
| 命令级 onResponse | 收到响应后 | RestCommand 中设置 | 单个请求的响应处理 |
| 全局 onResponse | 命令级之后 | `rest({ onResponse })` 配置 | 所有响应的统一处理 |

**辅助工具** `withOptions()`：[with-options.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/helpers/with-options.ts) 用于向 RestCommand 注入 onRequest hook 或额外 fetch options。

---

### 1.3 `authentication` 组合件

**核心文件**：[auth/composable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts)

authentication 组合件负责 token 生命周期的完整管理。

#### 1.3.1 `expires_at` 管理

**代码位置**：[auth/composable.ts#L73-L89](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts#L73-L89)

```typescript
const setCredentials = async (data: AuthenticationData) => {
    const expires = data.expires ?? 0;
    data.expires_at = new Date().getTime() + expires;  // 计算绝对过期时间戳
    await storage.set(data);
    // ... 设置 autoRefresh
};
```

`expires_at` 是**服务端返回的 expires（相对毫秒数）加上当前客户端时间**得到的绝对过期时间戳，存储于 storage（默认 memoryStorage，可配置）。

#### 1.3.2 `autoRefresh` 机制

**代码位置**：[auth/composable.ts#L78-L88](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts#L78-L88)

```typescript
if (authConfig.autoRefresh && expires > authConfig.msRefreshBeforeExpires && expires < MAX_INT32) {
    if (refreshTimeout) clearTimeout(refreshTimeout);

    refreshTimeout = setTimeout(() => {
        refreshTimeout = null;
        refresh().catch((_err) => { /* fail gracefully */ });
    }, expires - authConfig.msRefreshBeforeExpires);
}
```

自动刷新流程：
1. 默认 `msRefreshBeforeExpires = 30000ms`（30秒）
2. 在 token 实际过期前 30 秒触发 `refresh()`
3. 用 `setTimeout` 调度，因 setTimeout 最大支持 32 位整数（约24天），故超长 token 不启用自动刷新
4. `MAX_INT32 = 2^31 - 1` 作为安全上限

**刷新并发控制**：[auth/composable.ts#L49-L71](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts#L49-L71)

```typescript
let refreshPromise: Promise<AuthenticationData> | null = null;

const activeRefresh = async () => {
    try { await refreshPromise; } finally { refreshPromise = null; }
};

const refreshIfExpired = async () => {
    const authData = await storage.get();
    if (refreshPromise || !authData?.expires_at) {
        return activeRefresh();  // 已有刷新进行中，等待其完成
    }
    if (authData.expires_at < new Date().getTime() + authConfig.msRefreshBeforeExpires) {
        refresh().catch(...);  // 接近过期，主动刷新
    }
    return activeRefresh();
};
```

通过 `refreshPromise` 单例模式确保**同一时间只有一个刷新请求在执行**，避免并发刷新导致 token 竞争。

**refresh 请求本身**：[auth/composable.ts#L91-L127](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts#L91-L127)

- 直接调用 `/auth/refresh` 端点，使用 `request()` 工具函数
- json 模式下携带 `refresh_token`，cookie/session 模式依赖浏览器 cookie
- 成功后 `resetStorage()` 清空旧凭证，`setCredentials()` 写入新凭证并重新调度 autoRefresh

#### 1.3.3 `logout` 实现

**代码位置**：[auth/composable.ts#L162-L189](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts#L162-L189)

```typescript
async logout(options: Omit<LogoutOptions, 'refresh_token'> = {}) {
    const authData = await storage.get();
    // 构建 POST /auth/logout 请求
    // json 模式下携带 refresh_token
    const requestUrl = getRequestUrl(client.url, '/auth/logout');
    await request(requestUrl.toString(), fetchOptions, client.globals.fetch);

    this.stopRefreshing();   // 清除自动刷新定时器
    await resetStorage();    // 清空所有 token 存储
},
stopRefreshing() {
    if (refreshTimeout) clearTimeout(refreshTimeout);
},
```

#### 1.3.4 `getToken()` 对外接口

**代码位置**：[auth/composable.ts#L195-L202](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts#L195-L202)

```typescript
async getToken() {
    await refreshIfExpired().catch(() => { /* fail gracefully */ });
    const data = await storage.get();
    return data?.access_token ?? null;
}
```

这是 SDK 暴露给外部调用方（如 App 层 axios）的核心接口：
- 内部调用 `refreshIfExpired()`，若 token 即将过期则触发刷新并等待
- 刷新失败**静默降级**（catch 空处理），返回可能的 null

---

## 二、App 层认证机制

### 2.1 ofetch `baseClient` 与 `/auth/refresh` 的 requestQueue 暂停

**核心文件**：[sdk.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/sdk.ts)

App 层用 ofetch 创建了一个自定义 fetch 客户端，作为 SDK 的底层 fetcher：

```typescript
const baseClient = ofetch.create({
    retry: 0,
    ignoreResponseError: true,
    async onRequest({ request, options }) {
        const requestsStore = useRequestsStore();
        const id = requestsStore.startRequest();
        (options as OptionsWithId).id = id;
        const path = getUrlPath(request);

        return new Promise((resolve) => {
            if (path && path === '/auth/refresh') {
                requestQueue.pause();   // 刷新请求优先执行，暂停队列
                return resolve();
            }
            requestQueue.add(() => resolve());  // 其他请求入队等待
        });
    },
    // ... onResponse / onResponseError / onRequestError 均调用 endRequest
});
```

**暂停机制原理**（[sdk.ts#L22-L29](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/sdk.ts#L22-L29)）：

| 请求类型 | 行为 | 目的 |
|----------|------|------|
| `/auth/refresh` | `requestQueue.pause()` 后立即 resolve | 刷新请求**立即执行**，不再排队；同时暂停队列阻止后续请求发出 |
| 其他请求 | `requestQueue.add(() => resolve())` | 进入队列等待，确保刷新完成前不发送可能携带过期 token 的请求 |

SDK 的 fetch 被替换为 `baseClient.native`，因此所有 SDK 请求（包括 authentication 内部的 login/refresh/logout）都会经过此拦截逻辑。

---

### 2.2 axios api 排队等待 `sdk.getToken`

**核心文件**：[api.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts)

App 层同时维护一个 axios 实例（用于部分非 SDK 的 API 调用），其请求拦截器实现排队 + 等待 token：

#### 2.2.1 requestQueue 配置

**代码位置**：[api.ts#L12-L17](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts#L12-L17)

```typescript
export let requestQueue: PQueue = new PQueue({
    concurrency: 5,          // 最多 5 个并发请求
    intervalCap: 5,          // 每 500ms 最多 5 个请求
    interval: 500,
    carryoverConcurrencyCount: true,
});
```

使用 `p-queue` 库实现**限流 + 并发控制**的请求队列。

#### 2.2.2 请求拦截器排队与等待 token

**代码位置**：[api.ts#L25-L46](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts#L25-L46)

```typescript
export const onRequest = (config: InternalAxiosRequestConfig): Promise<InternalAxiosRequestConfig> => {
    const requestsStore = useRequestsStore();
    const id = requestsStore.startRequest();
    const requestConfig: InternalRequestConfig = { ...config, id };

    return new Promise((resolve) => {
        requestQueue.add(async () => {
            // 关键：等待当前活跃的 token 刷新完成
            await sdk.getToken().catch(() => { /* fail gracefully */ });

            if (requestConfig.measureLatency) requestConfig.start = performance.now();
            return resolve(requestConfig);
        });
    });
};
```

**流程要点**：
1. 每个 axios 请求进入拦截器后返回 Promise，暂不真正发送
2. 将请求处理函数加入 `requestQueue`，受并发/限流控制
3. 队列执行时，首先 `await sdk.getToken()`——此调用内部会执行 `refreshIfExpired()`，确保：
   - 如果 token 即将过期，等待刷新完成
   - 如果已有刷新在进行，等待其结束
4. token 就绪后才 resolve 请求 config，真正发出请求

#### 2.2.3 队列恢复

**代码位置**：[api.ts#L63-L71](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts#L63-L71)

```typescript
export function resumeQueue() {
    if (!requestQueue.isPaused) return;
    requestQueue.start();
}
```

`resumeQueue()` 在 App 层 `refresh()` 函数的 finally 块中被调用（见下文），确保刷新结束后被暂停的队列恢复执行。

---

### 2.3 登录 / 刷新失败触发 logout 与 dehydrate

**核心文件**：[auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/auth.ts)、[hydrate.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/hydrate.ts)

#### 2.3.1 `refresh()` 失败 → `logout`

**代码位置**：[auth.ts#L103-L126](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/auth.ts#L103-L126)

```typescript
export async function refresh({ navigate }: LogoutOptions = { navigate: true }): Promise<void> {
    const appStore = useAppStore();
    if (!firstRefresh && !appStore.authenticated) return;

    try {
        // 本地 token 未过期时仅验证会话，否则执行刷新
        if (appStore.accessTokenExpiry && Date.now() < appStore.accessTokenExpiry - SDK_AUTH_REFRESH_BEFORE_EXPIRES) {
            await sdk.request(readMe({ fields: ['id'] }));
            return;
        }

        const response = await sdk.refresh();
        appStore.accessTokenExpiry = Date.now() + (response.expires ?? 0);
        appStore.authenticated = true;
        firstRefresh = false;
    } catch {
        // 刷新失败 → 触发注销
        await logout({ navigate, reason: LogoutReason.SESSION_EXPIRED });
    } finally {
        resumeQueue();  // 无论成功失败，都恢复队列
    }
}
```

**关键逻辑**：
- App 层在 SDK `refresh()` 之外额外维护了本地的 `accessTokenExpiry`，用于快速判断是否需要刷新
- 刷新失败（无论何种原因，如网络错误、refresh_token 过期、服务端拒绝）统一在 catch 块中调用 `logout`，原因标记为 `SESSION_EXPIRED`
- `finally` 中 `resumeQueue()` 保证即使刷新失败，被暂停的请求队列也能恢复（避免应用死锁）

#### 2.3.2 `login()` 流程

**代码位置**：[auth.ts#L36-L84](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/auth.ts#L36-L84)

```typescript
export async function login({ credentials, provider, share }: LoginParams): Promise<void> {
    const appStore = useAppStore();
    const serverStore = useServerStore();
    let response: AuthenticationData;

    // 根据 share / email / identifier 选择不同登录方式
    if (share) {
        await sdk.request(authenticateShare(share, password, 'session'));
        response = await sdk.refresh();   // 初始化 auto-refresh 定时器
    } else {
        if (email) {
            response = await sdk.login({ email, password }, loginOptions);
        } else if (identifier) {
            await sdk.request(login());
            response = await sdk.refresh();  // identifier 登录后手动 refresh 以初始化
        }
    }

    appStore.accessTokenExpiry = Date.now() + (response.expires ?? 0);
    appStore.authenticated = true;
    serverStore.hydrate();
    await hydrate();  // 登录成功后重新水化所有 store
}
```

注意：部分登录流程（share、identifier）登录后会**额外调用一次 `sdk.refresh()`**，其目的是触发 SDK 内部的 `setCredentials()`，从而初始化 `autoRefresh` 定时器。

#### 2.3.3 `logout()` 与 `dehydrate()`

**代码位置**：[auth.ts#L141-L174](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/auth.ts#L141-L174)

```typescript
export async function logout(options: LogoutOptions = {}): Promise<void> {
    const appStore = useAppStore();
    const logoutOptions = { navigate: true, reason: LogoutReason.SIGN_OUT, ...options };

    sdk.stopRefreshing();  // 停止 SDK 层自动刷新定时器

    // 仅主动登出（SIGN_OUT）时调用后端 /auth/logout
    if (logoutOptions.reason === LogoutReason.SIGN_OUT) {
        try { await sdk.logout(); } catch { /* 用户已登出，忽略 */ }
    }

    appStore.authenticated = false;
    await dehydrate();  // 清空所有 store 状态

    if (logoutOptions.navigate === true) {
        router.push({ path: `/login`, query: { reason: logoutOptions.reason } });
    }
}
```

**登出原因区分**：

| 原因 | 是否调用后端 `/auth/logout` | 触发场景 |
|------|-----------------------------|----------|
| `SIGN_OUT` | 是 | 用户主动点击退出按钮 |
| `SESSION_EXPIRED` | 否 | 刷新失败、token 过期等被动场景 |

#### 2.3.4 `dehydrate()` 脱水机制

**核心文件**：[hydrate.ts#L94-L106](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/hydrate.ts#L94-L106)

```typescript
export async function dehydrate(stores = useStores()): Promise<void> {
    const appStore = useAppStore();
    if (appStore.hydrated === false) return;

    for (const store of stores) {
        await store.dehydrate?.();  // 逐个调用各 store 的 dehydrate 方法
    }

    await onDehydrateExtensions();  // 通知扩展脱水
    appStore.hydrated = false;
}
```

涉及 14 个 store（collections、fields、user、requests、presets、settings、server、relations、permissions、insights、flows、notifications、ai、license），每个 store 负责清理自身状态（如清空用户信息、权限缓存、字段定义等）。

---

## 三、层级分工总览

| 关注点 | SDK 层职责 | App 层职责 |
|--------|------------|------------|
| **Client 合成** | 提供 `createDirectus()` 和 `with()` 组合件机制 | 按需要组装 `authentication` + `rest` 等组合件 |
| **Content-Type** | 设置默认 `application/json`；multipart 时删除 header 让 fetch 处理 boundary | 不直接处理，依赖 SDK |
| **Bearer 注入** | `rest.request()` 中调用 `this.getToken()` 自动注入 `Authorization` header | 不直接处理；`withToken()` 可手动覆盖 |
| **Hook** | 提供命令级 + 全局的 `onRequest` / `onResponse` Hook | 可通过 `rest({ onRequest, onResponse })` 配置全局 Hook |
| **Token 存储** | `storage`（默认 memory，可自定义）维护 `access_token`/`refresh_token`/`expires`/`expires_at` | 不直接存储 token；用 cookie（session 模式）或维护本地 `accessTokenExpiry` 作快速判断 |
| **expires_at** | `setCredentials()` 中计算绝对时间戳并持久化 | 存储副本用于快速过期判断，不参与刷新决策 |
| **autoRefresh** | `setTimeout` 提前 30s 自动刷新，`refreshPromise` 单例防并发 | 监听 tab 空闲/活跃事件控制 SDK 的 `stopRefreshing()` 与重启刷新 |
| **logout** | 调用 `/auth/logout`，清理 storage，停止定时器 | 区分主动/被动登出，决定是否调用 SDK logout；执行 `dehydrate`，路由跳转 |
| **请求队列** | 无队列概念，直接发请求 | ofetch `baseClient` 对 `/auth/refresh` 暂停队列；axios `requestQueue` 排队等待 `sdk.getToken()` |
| **失败降级** | `getToken()` 和 `refresh()` 失败时静默 catch，返回 null/不抛错 | `refresh()` catch 中触发 `logout(SESSION_EXPIRED)` → `dehydrate()` 清空状态并跳登录页 |
| **状态清理** | 仅清理认证相关 storage | `dehydrate()` 遍历 14 个 store 清理所有应用状态 |

---

## 四、完整请求流程时序（刷新场景）

```
  App (axios)                 App (ofetch baseClient)          SDK (rest + auth)          Server
      |                               |                             |                     |
      |  请求 A                        |                             |                     |
      |----> onRequest ----------------|                             |                     |
      |       (加入 requestQueue)      |                             |                     |
      |                               |                             |                     |
      |  [定时器触发]                   |                             |                     |
      |                               |                             |                     |
      |                               |  sdk.refresh() 调用          |                     |
      |                               |---------------------------->|                     |
      |                               |                             |-- POST /auth/refresh|
      |                               |                             |------------------->|
      |                               |                             |                     |
      |                               |  onRequest: path=/auth/refresh                    |
      |                               |  → requestQueue.pause()     |                     |
      |                               |  → 立即 resolve             |                     |
      |                               |                             |<-------------------|
      |                               |                             |   新 token          |
      |                               |                             |                     |
      |                               |         setCredentials()    |                     |
      |                               |         (更新 expires_at,    |                     |
      |                               |          重设定时器)         |                     |
      |                               |                             |                     |
      |                               |<----------------------------|                     |
      |     finally: resumeQueue()    |                             |                     |
      |<------------------------------|                             |                     |
      |                               |                             |                     |
      |  队列恢复，请求 A 出队执行      |                             |                     |
      |  await sdk.getToken()         |                             |                     |
      |  → refreshIfExpired() → 已新   |                             |                     |
      |  → 返回新 token                |                             |                     |
      |                               |                             |                     |
      |---- 携带新 Bearer token 发请求 -->                        |                     |
```

---

## 五、关键文件索引

| 层级 | 文件 | 核心职责 |
|------|------|----------|
| SDK | [sdk/src/client.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/client.ts) | `createDirectus()` 和 `with()` 组合件机制 |
| SDK | [sdk/src/rest/composable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/composable.ts) | `rest()` 组合件：Content-Type、Bearer 注入、Hook |
| SDK | [sdk/src/rest/helpers/with-token.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/helpers/with-token.ts) | 命令级 token 注入工具 |
| SDK | [sdk/src/rest/helpers/with-options.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/helpers/with-options.ts) | 命令级 options/hook 注入工具 |
| SDK | [sdk/src/auth/composable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/auth/composable.ts) | `authentication()`：expires_at、autoRefresh、logout、getToken |
| SDK | [sdk/src/utils/request.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/utils/request.ts) | 底层 fetch 封装与错误处理 |
| App | [app/src/sdk.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/sdk.ts) | ofetch baseClient，`/auth/refresh` 暂停队列 |
| App | [app/src/api.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts) | axios 实例，requestQueue 排队等待 `sdk.getToken()` |
| App | [app/src/auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/auth.ts) | `login()` / `refresh()` / `logout()`，刷新失败触发登出 |
| App | [app/src/hydrate.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/hydrate.ts) | `hydrate()` / `dehydrate()`，store 状态的水化与脱水 |
