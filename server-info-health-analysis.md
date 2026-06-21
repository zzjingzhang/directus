# /server/info 与 /server/health 端点分析

## 一、路由与控制器

### 1.1 路由注册

两个端点均定义在 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/server.ts)：

| 端点 | 路径 | 控制器方法 | 可配置开关 |
|------|------|-----------|-----------|
| info | `GET /server/info` | `ServerService.serverInfo()` | 始终启用 |
| health | `GET /server/health` | `ServerService.health()` | `HEALTHCHECK_ENABLED` (默认 `true`) |

关键代码：[server.ts#L63-L98](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/server.ts#L63-L98)

### 1.2 控制器 vs 服务层

- 控制器 ([server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/server.ts)) 只负责路由注册和响应包装
- 实际逻辑都在服务层 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts) 的 `ServerService` 类中

---

## 二、/server/info 字段返回条件分析

服务层核心方法：`serverInfo()` —— [server.ts#L56-L194](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L56-L194)

### 2.1 所有状态下均返回的字段

| 字段 | 说明 |
|------|------|
| `project` | 项目基本信息（名称、描述、Logo、主题、语言、注册设置等） |
| `license` | `{ source, entitlements }` — 始终返回 |

`license` 字段结构：
```typescript
{
  source: licenseManager.getSource(),          // 许可证来源
  entitlements: getEntitlementManager().getAppEntitlements()  // 应用权限
}
```
代码位置：[server.ts#L188-L191](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L188-L191)

### 2.2 基于 setup 状态的字段

当 `isSetupCompleted === false` 时（即 `directus_users` 表为空），额外返回：

| 字段 | 条件 | 说明 |
|------|------|------|
| `setup` | `!isSetupCompleted` | `{ license_complete, owner_complete }` — 安装向导进度 |
| `version` | `this.accountability?.user \|\| !isSetupCompleted` | 版本号（已登录用户或未安装时可见） |

代码位置：[server.ts#L87-L92](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L87-L92)

### 2.3 基于已登录状态的字段（`this.accountability?.user` 为 true）

当请求携带有效用户身份时，返回以下字段：

#### 2.3.1 MCP / AI / OAuth 相关

| 字段 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `mcp_enabled` | `MCP_ENABLED` | `true` | MCP 功能是否启用 |
| `ai_enabled` | `AI_ENABLED` | `true` | AI 功能是否启用 |
| `mcp_oauth_enabled` | `MCP_OAUTH_ENABLED` | `false` | MCP OAuth 是否启用 |
| `mcp_oauth_dcr_enabled` | `MCP_OAUTH_DCR_ENABLED` | `false` | MCP OAuth DCR (Dynamic Client Registration) |
| `mcp_oauth_cimd_enabled` | `MCP_OAUTH_CIMD_ENABLED` | `false` | MCP OAuth CIMD |

代码位置：[server.ts#L94-L99](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L94-L99)

#### 2.3.2 autoSave

| 字段 | 环境变量 | 说明 |
|------|---------|------|
| `autoSave.revisionInterval` | `AUTOSAVE_REVISION_INTERVAL` | 自动保存修订版本间隔 |

#### 2.3.3 files

| 字段 | 环境变量 | 说明 |
|------|---------|------|
| `files.mimeTypeAllowList` | `FILES_MIME_TYPE_ALLOW_LIST` | 允许上传的 MIME 类型列表 |

代码位置：[server.ts#L105-L107](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L105-L107)

#### 2.3.4 rateLimit & rateLimitGlobal

| 字段 | 环境变量开关 | 条件 | 结构 |
|------|-------------|------|------|
| `rateLimit` | `RATE_LIMITER_ENABLED` | `true` 时返回对象，`false` 时返回 `false` | `{ points, duration }` |
| `rateLimitGlobal` | `RATE_LIMITER_GLOBAL_ENABLED` | `true` 时返回对象，`false` 时返回 `false` | `{ points, duration }` |

代码位置：[server.ts#L109-L125](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L109-L125)

后端中间件对应实现：
- IP 限流：[rate-limiter-ip.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/rate-limiter-ip.ts)
- 全局限流：[rate-limiter-global.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/rate-limiter-global.ts)

#### 2.3.5 extensions

| 字段 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `extensions.limit` | `EXTENSIONS_LIMIT` | `null` | 允许安装的扩展数量上限 |

代码位置：[server.ts#L127-L129](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L127-L129)

#### 2.3.6 queryLimit

| 字段 | 环境变量 | 说明 |
|------|---------|------|
| `queryLimit.default` | `QUERY_LIMIT_DEFAULT` | 默认分页大小 |
| `queryLimit.max` | `QUERY_LIMIT_MAX` | 最大分页大小（无穷大时为 `-1`） |

代码位置：[server.ts#L131-L134](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L131-L134)

#### 2.3.7 websocket

**总开关**：`WEBSOCKETS_ENABLED` — 为 `false` 时整个 `websocket` 字段为 `false`

当 `WEBSOCKETS_ENABLED === true` 时：

| 子字段 | 开关环境变量 | 结构 | 特殊条件 |
|--------|-------------|------|---------|
| `websocket.rest` | `WEBSOCKETS_REST_ENABLED` | `{ authentication, path }` 或 `false` | — |
| `websocket.graphql` | `WEBSOCKETS_GRAPHQL_ENABLED` | `{ authentication, path }` 或 `false` | — |
| `websocket.heartbeat` | `WEBSOCKETS_HEARTBEAT_ENABLED` | `period(毫秒)` 或 `false` | — |
| `websocket.collaborativeEditing` | `WEBSOCKETS_COLLAB_ENABLED` | `boolean` | — |
| `websocket.logs` | `WEBSOCKETS_LOGS_ENABLED` | `{ allowedLogLevels }` 或 `false` | **需要 `accountability.admin === true`** |

> **注意**：`websocket.logs` 是唯一需要 admin 权限的 websocket 子字段。非 admin 用户即使 `WEBSOCKETS_LOGS_ENABLED` 为 `true`，也会收到 `false`。

代码位置：[server.ts#L136-L167](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L136-L167)

#### 2.3.8 uploads

| 字段 | 条件 | 结构 |
|------|------|------|
| `uploads.maxConcurrency` | `FILE_UPLOADS.MAX_CONCURRENCY` 存在且非 `Infinity` | `{ maxConcurrency }` |
| `uploads.tus` + `uploads.chunkSize` | `RESUMABLE_UPLOADS.ENABLED === true` | 追加 `{ tus: true, chunkSize }` |

常量定义位置：[constants.ts#L97-L109](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts#L97-L109)

代码位置：[server.ts#L169-L181](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L169-L181)

### 2.4 /server/info 字段返回条件总览表

| 字段 | 未登录未安装 | 未登录已安装 | 已登录非admin | 已登录admin |
|------|:-----------:|:----------:|:------------:|:----------:|
| `project` | ✅ | ✅ | ✅ | ✅ |
| `license` | ✅ | ✅ | ✅ | ✅ |
| `setup` | ✅ | ❌ | ❌ | ❌ |
| `version` | ✅ | ❌ | ✅ | ✅ |
| `mcp_enabled` | ❌ | ❌ | ✅ | ✅ |
| `ai_enabled` | ❌ | ❌ | ✅ | ✅ |
| `mcp_oauth_*` | ❌ | ❌ | ✅ | ✅ |
| `autoSave` | ❌ | ❌ | ✅ | ✅ |
| `files` | ❌ | ❌ | ✅ | ✅ |
| `rateLimit` | ❌ | ❌ | ✅ | ✅ |
| `rateLimitGlobal` | ❌ | ❌ | ✅ | ✅ |
| `extensions` | ❌ | ❌ | ✅ | ✅ |
| `queryLimit` | ❌ | ❌ | ✅ | ✅ |
| `websocket` (不含logs) | ❌ | ❌ | ✅ | ✅ |
| `websocket.logs` | ❌ | ❌ | ❌ | ✅ |
| `uploads` | ❌ | ❌ | ✅ | ✅ |

---

## 三、/server/health 端点分析

服务层核心方法：`health()` —— [server.ts#L196-L443](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L196-L443)

### 3.1 前置权限检查

```typescript
if (isUnauthenticated(this.accountability)) {
    throw new ForbiddenError();
}
```

`isUnauthenticated` 判断逻辑（[is-unauthenticated.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/is-unauthenticated.ts)）：
- `accountability === null` → 视为 admin（`false`，不拦截）
- `accountability === undefined` → 未认证（`true`，拦截）
- `accountability.role === null && accountability.user === null` → 未认证（`true`，拦截）

**结论**：health 端点仅允许**已认证用户**访问，未登录直接返回 403。

### 3.2 健康检查缓存

- 使用 KV store 缓存，TTL 默认为 5 分钟（`HEALTHCHECK_CACHE_TTL`）
- 缓存 key：`directus:healthcheck` + `health`
- 命中缓存时直接返回，不再执行实际检查

### 3.3 健康检查项

由 `HEALTHCHECK_SERVICES` 环境变量控制启用哪些服务检查：

| 服务 | 检查项 | 说明 |
|------|--------|------|
| `database` | `{client}:responseTime`、`connectionsAvailable`、`connectionsUsed` | 数据库响应时间（阈值默认 150ms）、连接池状态 |
| `redis` | `redis:responseTime` | Redis 响应时间（阈值默认 150ms），仅当配置了 Redis 时启用 |
| `storage` | `storage:{location}:responseTime` | 各存储位置响应时间（阈值默认 750ms） |
| `email` | `email:connection` | 邮件服务连接（仅当 `EMAIL_VERIFY_SETUP=true` 时启用） |

### 3.4 状态聚合逻辑

总体 `status` 取值：`'ok' | 'warn' | 'error'`

判断优先级：
1. `SERVER_ONLINE === false` → `error`
2. 任一子项 `error` → 总体 `error`
3. 任一子项 `warn` 且总体目前为 `ok` → 总体 `warn`
4. 否则 → `ok`

### 3.5 非 admin 用户信息收敛（核心）

```typescript
return this.accountability?.admin === true ? data : { status: data.status };
```

代码位置：[server.ts#L212](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L212) 和 [server.ts#L260](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L260)

**收敛规则**：
- **admin 用户**：收到完整 `ServerHealth` 对象，包含 `status`、`releaseId`、`serviceId`、`checks`（所有检查详情）
- **非 admin 用户**：仅收到 `{ status }`，不含任何服务详情、版本信息或检查指标

类型定义见 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/server.ts#L13-L20)：
```typescript
export type ServerHealth = {
    status: ServerHealthStatus;
    releaseId: string;
    serviceId: string;
    checks: { [service: string]: ServerHealthCheck[] };
};
```

SDK 类型也体现了这一点（[health.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/rest/commands/server/health.ts#L3-L10)）：
```typescript
export type ServerHealthOutput = {
    status: 'ok' | 'warn' | 'error';
    releaseId?: string;      // 非 admin 时缺失
    serviceId?: string;      // 非 admin 时缺失
    checks?: { ... };        // 非 admin 时缺失
};
```

---

## 四、AppServerStore（前端）中的字段使用

Store 定义位置：[server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/stores/server.ts)

### 4.1 Store 初始化

`hydrate()` 方法同时请求两个接口：
```typescript
const [serverInfoResponse, authResponse] = await Promise.all([
    api.get(`/server/info`),
    api.get('/auth?sessionOnly'),
]);
```
代码位置：[server.ts#L145-L149](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/stores/server.ts#L145-L149)

### 4.2 各字段在前端的使用场景

| 字段 | 使用文件 | 用途 |
|------|---------|------|
| `info.project` | 多个视图组件 | 项目名称、Logo、主题、公共注册配置等 UI 展示 |
| `info.ai_enabled` | [private-view-sidebar.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/views/private/private-view/components/private-view-sidebar.vue) | 控制 AI 侧边栏是否显示，以及添加 `ai-disabled` CSS class |
| `info.license` | [private-view.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/views/private/private-view/components/private-view.vue)、[private-view-nav-footer.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/views/private/private-view/components/private-view-nav-footer.vue) | 判断是否显示 license onboarding 弹窗；根据 `display_powered_by` 控制页脚"Powered by"显示 |
| `info.setup` / `info.setupCompleted` | [private-view.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/views/private/private-view/components/private-view.vue) | 判断安装流程是否完成，控制 license onboarding 弹窗 |
| `info.mcp_enabled`、`info.mcp_oauth_*` | Store 定义，暂无直接消费代码 | 预留 MCP 功能开关 |
| `info.files.mimeTypeAllowList` | [use-mime-type-filter.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/composables/use-mime-type-filter.ts) | 全局 MIME 类型过滤，与组件级 allowList 取交集后限制文件上传 |
| `info.rateLimit` | [server.ts hydrate()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/stores/server.ts#L171-L181) | 动态调整前端请求队列的并发节奏（详见第五章） |
| `info.queryLimit.default` / `info.queryLimit.max` | [fetch-all.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/utils/fetch-all.ts)、[use-revisions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/composables/use-revisions.ts) | 控制批量拉取数据时的分页大小；`max === -1` 时尝试单请求获取全部 |
| `info.websocket.collaborativeEditing` | [use-collab.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/composables/use-collab.ts) | 判断是否建立协同编辑 WebSocket 连接 |
| `info.websocket.logs.allowedLogLevels` | [logs.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/modules/settings/routes/system-logs/logs.vue) | 系统日志页面根据允许的日志级别构建过滤选项 |
| `info.uploads.tus` / `info.uploads.chunkSize` | [upload-file.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/utils/upload-file.ts) | 启用 TUS 分块上传，设置分块大小；否则回退到普通 FormData 上传 |
| `info.uploads.maxConcurrency` | 前端保留，未见直接消费 | 上传并发上限信息 |
| `info.extensions.limit` | [extension-install.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/modules/settings/routes/marketplace/routes/extension/components/extension-install.vue) | 判断已安装扩展数是否达上限，禁用安装按钮 |
| `info.autoSave.revisionInterval` | Store 定义，供自动保存功能使用 | 自动保存修订版本的时间间隔 |

---

## 五、rateLimit 如何改变前端请求队列的并发节奏

### 5.1 请求队列基础配置

请求队列使用 `p-queue` 库实现，初始配置在 [api.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts#L12-L17)：

```typescript
export let requestQueue: PQueue = new PQueue({
    concurrency: 5,          // 最大并发数
    intervalCap: 5,          // 每个时间窗口内最多处理的请求数
    interval: 500,           // 时间窗口（毫秒）
    carryoverConcurrencyCount: true,
});
```

初始策略：每 500ms 最多 5 个请求，最多同时 5 个并发。

### 5.2 请求进入队列的流程

所有 axios 请求（`api.ts`）和 SDK 请求（`sdk.ts`）都通过拦截器进入队列：

**axios 拦截器**（[api.ts#L25-L46](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts#L25-L46)）：
```typescript
export const onRequest = (config) => {
    return new Promise((resolve) => {
        requestQueue.add(async () => {
            await sdk.getToken().catch(() => {});
            return resolve(requestConfig);
        });
    });
};
```

**SDK onRequest**（[sdk.ts#L16-L30](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/sdk.ts#L16-L30)）：
```typescript
async onRequest({ request, options }) {
    return new Promise((resolve) => {
        if (path === '/auth/refresh') {
            requestQueue.pause();  // token 刷新时暂停队列
            return resolve();
        }
        requestQueue.add(() => resolve());
    });
}
```

### 5.3 根据 rateLimit 动态调整队列

在 `useServerStore.hydrate()` 中（[server.ts#L171-L181](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/stores/server.ts#L171-L181)）：

```typescript
if (serverInfoResponse.data.data?.rateLimit !== undefined) {
    if (serverInfoResponse.data.data?.rateLimit === false) {
        // 后端未启用限流：无限制队列
        await replaceQueue();
    } else {
        const { duration, points } = serverInfoResponse.data.data.rateLimit;
        const intervalCap = 1;
        // 计算"每 1 点配额"对应的间隔时间
        const interval = Math.ceil((duration * 1000) / points);
        await replaceQueue({ intervalCap, interval });
    }
}
```

`replaceQueue` 实现（[api.ts#L68-L71](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/api.ts#L68-L71)）：
```typescript
export async function replaceQueue(options?: Options<any, QueueAddOptions>) {
    await requestQueue.onIdle();   // 等待当前队列清空
    requestQueue = new PQueue(options);  // 用新配置重建队列
}
```

### 5.4 速率调整算法

后端返回的 rateLimit 结构：`{ points, duration }`
- `points`：窗口内允许的请求数
- `duration`：窗口时长（秒）

前端将其转换为 p-queue 配置：
```
intervalCap = 1
interval = ceil(duration * 1000 / points)   // 毫秒
```

**实际效果**：
- 后端配置 `points=50, duration=10` → 前端 `interval=200ms`，即每 200ms 最多发送 1 个请求（整体速率 = 5 req/s）
- 后端未启用限流 → 前端使用默认 p-queue 配置（concurrency=5, intervalCap=5, interval=500ms）

这种方式确保前端请求速率不会超过后端 IP 限流的配额，避免触发 `HitRateLimitError`（429 Too Many Requests）。

### 5.5 特殊处理：token 刷新

SDK 请求拦截器中，当路径为 `/auth/refresh` 时：
```typescript
if (path && path === '/auth/refresh') {
    requestQueue.pause();
    return resolve();
}
```
刷新 token 请求不受队列限制，直接执行，并暂停队列避免其他请求在 token 失效期间发出。

---

## 六、调用链总览

```
前端 App
  │
  ├── useServerStore.hydrate()
  │     ├── GET /server/info ────────► ServerService.serverInfo()
  │     │     └── 根据 auth / setup / admin 权限返回差异化字段
  │     │           ├── rateLimit → replaceQueue() 调整请求节奏
  │     │           ├── queryLimit → fetch-all / use-revisions 分页
  │     │           ├── websocket → use-collab 协同编辑 / 日志页面
  │     │           ├── files.mimeTypeAllowList → useMimeTypeFilter
  │     │           ├── uploads → upload-file.ts (TUS vs FormData)
  │     │           ├── extensions.limit → marketplace 安装限制
  │     │           └── ai_enabled / license → UI 条件渲染
  │     │
  │     └── GET /auth?sessionOnly
  │
  └── api / sdk 请求
        └── requestQueue (p-queue)
              ├── 初始: concurrency=5, intervalCap=5, interval=500ms
              └── hydrate 后: 根据 rateLimit 动态调整 interval
```
