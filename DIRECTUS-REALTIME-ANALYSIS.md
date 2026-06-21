# Directus 实时链路完整分析

## 目录

1. [服务器启动与 WebSocket 控制器](#1-服务器启动与-websocket-控制器)
2. [连接认证机制](#2-连接认证机制)
3. [Items 消息处理](#3-items-消息处理)
4. [Subscribe 订阅分发](#4-subscribe-订阅分发)
5. [SDK Realtime 客户端](#5-sdk-realtime-客户端)
6. [完整流程图](#6-完整流程图)

---

## 1. 服务器启动与 WebSocket 控制器

### 1.1 启动流程

服务器启动入口位于 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/server.ts#L104-L110)，当 `WEBSOCKETS_ENABLED` 为 true 时，会依次创建三个控制器并启动处理器：

```typescript
if (toBoolean(env['WEBSOCKETS_ENABLED']) === true) {
    createSubscriptionController(server);   // GraphQL 订阅控制器
    createWebSocketController(server);      // REST WebSocket 控制器
    createLogsController(server);           // 日志 WebSocket 控制器
    startWebSocketHandlers();               // 启动消息处理器
}
```

### 1.2 WebSocket 控制器架构

控制器的核心继承关系：

- [SocketController](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/base.ts#L31-L457)（抽象基类）
  - [WebSocketController](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/rest.ts#L15-L59)（REST WebSocket）
  - [GraphQLSubscriptionController](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/graphql.ts)（GraphQL 订阅）
  - [LogsController](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/logs.ts)（日志）

### 1.3 基类核心功能

[SocketController](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/base.ts) 构造函数完成以下初始化：

1. **创建 WebSocketServer**：使用 `noServer: true` 模式，手动处理 upgrade
2. **初始化客户端集合**：`this.clients = new Set()`
3. **读取环境配置**：endpoint、认证模式、最大连接数
4. **创建限流器**：基于 `RATE_LIMITER_ENABLED` 配置
5. **绑定 upgrade 事件**：监听 HTTP server 的 upgrade 事件
6. **启动 token 检查定时器**：每 15 分钟检查一次客户端 token 状态

```typescript
constructor(httpServer: httpServer, configPrefix: string) {
    this.server = new WebSocketServer({ noServer: true, autoPong: false });
    this.clients = new Set();
    // ... 配置读取
    httpServer.on('upgrade', this.handleUpgrade.bind(this));
    this.checkClientTokens();
}
```

### 1.4 消息处理器

在 [startWebSocketHandlers()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/index.ts#L11-L39) 中启动各类处理器：

| 处理器 | 功能 | 启用条件 |
|--------|------|----------|
| [HeartbeatHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/heartbeat.ts) | 心跳检测 | `WEBSOCKETS_HEARTBEAT_ENABLED` |
| [ItemsHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/items.ts) | Items CRUD 操作 | REST 或 GraphQL 启用 |
| [SubscribeHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/subscribe.ts) | 订阅分发 | REST 启用 |
| [LogsHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/logs.ts) | 日志订阅 | `WEBSOCKETS_LOGS_ENABLED` |
| [CollabHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/collab/collab.ts) | 协同编辑 | `WEBSOCKETS_COLLAB_ENABLED` |

---

## 2. 连接认证机制

### 2.1 三种认证模式

WebSocket 支持三种认证模式，通过环境变量 `WEBSOCKETS_REST_AUTH` 配置：

| 模式 | 说明 | 升级时机 |
|------|------|----------|
| `public` | 公开模式，无需认证即可连接 | 立即升级 |
| `handshake` | 握手模式，连接后需在超时时间内发送 auth 消息 | 先升级，后认证 |
| `strict` | 严格模式，必须在 upgrade 时携带 token | 认证通过后才升级 |

模式定义见 [messages.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/messages.ts#L121)。

### 2.2 Upgrade 处理流程

[handleUpgrade()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/base.ts#L117-L178) 方法根据认证模式选择不同的升级路径：

```
请求到达
   │
   ├─► strict 模式 或 query 有 access_token 或 cookie 有 session？
   │      │
   │      └─► handleTokenUpgrade() ──► 验证 token ──► 成功则升级
   │                                  │
   │                                  └─► 失败返回 401
   │
   ├─► handshake 模式？
   │      │
   │      └─► handleHandshakeUpgrade() ──► 先升级
   │                                      等待 auth 消息
   │                                      超时则关闭
   │
   └─► public 模式
          └─► 直接升级，使用默认 accountability
```

### 2.3 三种认证凭证方式

[authenticateConnection()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/authenticate.ts#L15-L77) 函数支持三种认证凭证：

#### 方式一：email + password

```typescript
if ('email' in message && 'password' in message) {
    const authenticationService = new AuthenticationService({ schema: await getSchema() });
    const { accessToken, refreshToken } = await authenticationService.login(DEFAULT_AUTH_PROVIDER, message);
    access_token = accessToken;
    refresh_token = refreshToken;
}
```

- 调用 `AuthenticationService.login()` 进行登录
- 返回 `accessToken` 和 `refreshToken`
- 适用于初始登录场景

#### 方式二：refresh_token

```typescript
if ('refresh_token' in message) {
    const authenticationService = new AuthenticationService({ schema: await getSchema() });
    const { accessToken, refreshToken } = await authenticationService.refresh(message.refresh_token);
    access_token = accessToken;
    refresh_token = refreshToken;
}
```

- 调用 `AuthenticationService.refresh()` 刷新 token
- 返回新的 `accessToken` 和 `refreshToken`
- 适用于 token 过期后续期

#### 方式三：access_token

```typescript
if ('access_token' in message) {
    access_token = message.access_token;
}
```

- 直接使用传入的 access_token
- 不返回 refresh_token
- 适用于已有有效 token 的场景

### 2.4 OAuth Session 被拒绝的原因

在 [authenticate.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/authenticate.ts#L69-L71) 中明确禁止了 OAuth 会话：

```typescript
if (authenticationState.accountability.oauth) {
    throw new Error('OAuth sessions are not allowed on WebSocket connections');
}
```

**原因分析：**

1. **安全限制**：OAuth 会话（通过 MCP OAuth 流程创建）通常具有特定的 scope 和 audience 限制，设计用于受限的 API 访问而非完整的 WebSocket 实时连接
2. **会话类型区分**：在 [get-accountability-for-token.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-accountability-for-token.ts#L28-L48) 中，OAuth 会话会设置 `accountability.oauth` 属性，包含 `client`、`scopes`、`aud` 等信息
3. **路由守卫模式**：类似 [mcp-oauth-guard.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/mcp-oauth-guard.ts) 中间件只允许 OAuth 会话访问 MCP 端点，WebSocket 也遵循相同的安全策略
4. **刷新限制**：在 [authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts) 中，OAuth 会话的 refresh token 也会被拒绝

### 2.5 认证成功响应

认证成功后返回 `authenticationSuccess` 消息，格式定义见 [authenticate.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/authenticate.ts#L79-L94)：

```json
{
    "type": "auth",
    "status": "ok",
    "uid": "消息唯一标识",
    "refresh_token": "新的刷新令牌（如果适用）"
}
```

### 2.6 Token 过期检测

服务端通过 [setTokenExpireTimer()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/base.ts#L406-L432) 方法管理 token 过期：

1. 每个客户端连接后设置过期定时器
2. 过期时间从 JWT 的 `exp`  claim 中提取（见 [get-expires-at-for-token.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/utils/get-expires-at-for-token.ts)）
3. token 过期后发送 `TOKEN_EXPIRED` 错误
4. 等待客户端重新认证，超时则关闭连接
5. 每 15 分钟批量检查一次所有客户端的 token 状态

```typescript
setTokenExpireTimer(client: WebSocketClient) {
    // ... 清理旧定时器
    if (!client.expires_at) return;
    
    const expiresIn = client.expires_at * 1000 - Date.now();
    if (expiresIn > TOKEN_CHECK_INTERVAL) return; // 超过15分钟后再检查
    
    client.auth_timer = setTimeout(() => {
        client.accountability = null;
        handleWebSocketError(client, new TokenExpiredError(), 'auth');
        // 等待客户端重新认证
        waitForMessageType(client, 'auth', this.authentication.timeout).catch(() => {
            // 超时关闭
            if (this.authentication.mode !== 'public') {
                client.close();
            }
        });
    }, expiresIn);
}
```

---

## 3. Items 消息处理

### 3.1 ItemsHandler 架构

[ItemsHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/items.ts) 负责处理 WebSocket 上的 CRUD 操作，通过事件总线监听消息：

```typescript
emitter.onAction('websocket.message', ({ client, message }) => {
    if (getMessageType(message) !== 'items') return;
    // 处理 items 消息
});
```

### 3.2 支持的操作

| Action | 说明 | 支持的参数 |
|--------|------|-----------|
| `create` | 创建条目 | `data`（对象或数组）、`query` |
| `read` | 读取条目 | `id`、`ids`、`query` |
| `update` | 更新条目 | `id`、`ids`、`data`、`query` |
| `delete` | 删除条目 | `id`、`ids`、`query` |

消息格式定义见 [WebSocketItemsMessage](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/messages.ts#L71-L97)。

### 3.3 处理流程

以 `create` 操作为例：

1. 解析并验证消息
2. 检查集合是否存在且可访问
3. 创建 [ItemsService](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts) 实例
4. 执行具体操作（createOne/createMany）
5. 读取创建后的结果（应用 query 过滤）
6. 返回格式化的响应消息

```typescript
if (message.action === 'create') {
    const query = await sanitizeQuery(message?.query ?? {}, schema, accountability);
    
    if (Array.isArray(message.data)) {
        const keys = await service.createMany(message.data);
        result = await service.readMany(keys, query);
    } else {
        const key = await service.createOne(message.data);
        result = await service.readOne(key, query);
    }
}
```

---

## 4. Subscribe 订阅分发

### 4.1 SubscribeHandler 架构

[SubscribeHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/subscribe.ts) 是订阅功能的核心，主要职责：

- 管理客户端订阅
- 监听系统事件
- 过滤和分发事件

### 4.2 事件注册与总线

事件通过 [registerWebSocketEvents()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/hooks.ts#L8-L34) 注册，支持的集合包括：

- 所有用户集合（`items.*`）
- 系统集合：`directus_collections`、`directus_fields`、`directus_relations`、`directus_files` 等
- 排除：`directus_sessions`（安全原因）、`directus_migrations`

事件通过 [消息总线](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/hooks.ts#L148-L155) 发布：

```typescript
function registerAction(event: string, transform: (args: Record<string, any>) => WebSocketEvent) {
    const messenger = useBus();
    emitter.onAction(event, (data: Record<string, any>) => {
        messenger.publish('websocket.event', transform(data) as Record<string, any>);
    });
}
```

### 4.3 订阅存储结构

订阅按 collection 分组存储在 [subscriptions](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/subscribe.ts#L19) 对象中：

```typescript
subscriptions: Record<string, Set<Subscription>>;
// 例如: { "articles": Set<Subscription>, "users": Set<Subscription> }
```

[Subscription](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/types.ts#L23-L30) 类型定义：

```typescript
type Subscription = {
    uid?: string | number;     // 订阅唯一标识（客户端生成）
    query?: Query;             // 查询参数（字段、过滤等）
    item?: string | number;    // 特定条目 ID
    event?: SubscriptionEvent; // 事件类型过滤（create/update/delete）
    collection: string;        // 集合名称
    client: WebSocketClient;   // 关联的客户端
};
```

### 4.4 四层过滤机制

在 [dispatch()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/subscribe.ts#L108-L135) 方法中，事件按以下顺序过滤：

#### 第一层：collection 过滤

```typescript
const subscriptions = this.subscriptions[event.collection];
if (!subscriptions || subscriptions.size === 0) return;
```

直接按集合名称索引，O(1) 时间复杂度获取相关订阅。

#### 第二层：event 过滤

```typescript
if (subscription.event !== undefined && event.action !== subscription.event) {
    continue; // 跳过不匹配的事件类型
}
```

如果订阅指定了 `event`（如只监听 `create`），则只分发对应类型的事件。

#### 第三层：item 过滤

```typescript
if ('item' in subscription) {
    if ('keys' in event && !event.keys.includes(subscription.item)) continue;
    if ('key' in event && event.key !== subscription.item) continue;
}
```

如果订阅指定了具体的 `item` ID，则只分发该条目的事件。

> **注意**：`directus_fields` 和 `directus_relations` 集合不支持按具体 item 订阅。

#### 第四层：权限过滤（隐含）

在 [getPayload()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/utils/items.ts#L19-L55) 中通过 `ItemsService` 读取数据时，会自动应用权限过滤。如果用户无权访问某些数据，结果会为空。

```typescript
if (Array.isArray(result?.['data']) && result?.['data']?.length === 0) continue;
```

#### 第五层：uid 标识（客户端侧）

`uid` 是订阅的唯一标识，用于客户端区分不同的订阅。服务端在响应中带回 uid，客户端据此路由到对应的订阅回调。

### 4.5 初始 payload 与 event filter 的差异

订阅建立时的初始响应和后续事件响应存在重要差异：

| 特性 | 初始响应（init） | 事件响应（create/update/delete） |
|------|-----------------|--------------------------------|
| 触发时机 | 订阅建立时立即发送 | 数据变更时触发 |
| event 字段 | `'init'` | `'create'` / `'update'` / `'delete'` |
| 数据内容 | 基于 query 的完整查询结果 | 受事件影响的条目数据 |
| 是否受 event filter 影响 | **不受影响**，始终返回初始数据 | **受影响**，只返回匹配的事件 |
| 数据量 | 可能是多条（取决于 query） | create: 1 条，update/delete: 可能多条 |

在 [onMessage()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/subscribe.ts#L140-L199) 方法中：

```typescript
const data =
    subscription.event === undefined 
        ? await getPayload(subscription, accountability, schema)  // 完整初始数据
        : { event: 'init' };  // 只有 event 标识，无实际数据

// 注册订阅
this.subscribe(subscription);

// 发送初始响应
client.send(fmtMessage('subscription', data, subscription.uid));
```

**关键设计**：
- 如果订阅设置了 `event` 过滤，初始响应只返回 `{ event: 'init' }` 而不返回数据
- 如果订阅没有设置 `event` 过滤，初始响应返回完整的查询结果
- 这样设计是为了避免返回客户端不关心的数据（比如只关心 delete 事件就不需要初始数据）

### 4.6 事件格式

[WebSocketEvent](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/messages.ts#L99-L119) 定义：

```typescript
// create 事件
{ action: 'create', collection: string, payload?: object, key: string | number }

// update 事件
{ action: 'update', collection: string, payload?: object, keys: Array<string | number> }

// delete 事件
{ action: 'delete', collection: string, payload?: object, keys: Array<string | number> }
```

### 4.7 取消订阅

客户端断开连接时自动取消所有订阅：

```typescript
emitter.onAction('websocket.error', ({ client }) => this.unsubscribe(client));
emitter.onAction('websocket.close', ({ client }) => this.unsubscribe(client));
```

也可以主动发送 `unsubscribe` 消息取消特定订阅。

---

## 5. SDK Realtime 客户端

### 5.1 架构概述

SDK 的 realtime 模块位于 [sdk/src/realtime/](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/)，核心是 [realtime()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L44-L476) composable 函数。

### 5.2 配置选项

[WebSocketConfig](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/types.ts#L6-L22) 定义：

```typescript
interface WebSocketConfig {
    authMode?: 'public' | 'handshake' | 'strict';  // 认证模式
    reconnect?: { delay: number; retries: number } | false;  // 重连配置
    connect?: { timeout: number } | false;          // 连接超时
    heartbeat?: boolean;                            // 是否启用心跳
    debug?: boolean;                                // 调试日志
    url?: string;                                   // 自定义 WebSocket URL
}
```

默认配置：

```typescript
const defaultRealTimeConfig: WebSocketConfig = {
    authMode: 'handshake',
    heartbeat: true,
    debug: false,
    connect: { timeout: 10000 },  // 10秒
    reconnect: { delay: 1000, retries: 10 },  // 1秒延迟，最多10次
};
```

### 5.3 两种 Auth Mode

#### handshake 模式

在 [connect()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L294-L328) 中实现：

1. WebSocket 连接建立后
2. 调用 `self.getToken()` 获取 access token
3. 发送 auth 消息：`{ type: 'auth', access_token: '...' }`
4. 等待服务器响应认证结果
5. 认证成功则连接完成，失败则 reject

```typescript
if (config.authMode === 'handshake' && hasAuth(self)) {
    const access_token = await self.getToken();
    ws.send(auth({ access_token }));
    const confirm = await messageCallback(ws);
    
    if (confirm['type'] === 'auth' && confirm['status'] === 'ok') {
        debug('info', 'Authentication successful!');
    }
}
```

#### strict 模式

在 [withStrictAuth()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L68-L77) 中实现：

1. 连接前将 access_token 作为 URL 查询参数
2. WebSocket 握手时由服务端在 upgrade 阶段验证
3. 验证失败则连接直接失败

```typescript
const withStrictAuth = async (url: URL | string, currentClient: AuthWSClient<Schema>) => {
    const newUrl = new client.globals.URL(url);
    if (config.authMode === 'strict' && hasAuth(currentClient)) {
        const token = await currentClient.getToken();
        if (token) newUrl.searchParams.set('access_token', token);
    }
    return newUrl.toString();
};
```

### 5.4 心跳机制

#### 服务端心跳

[HeartbeatHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/heartbeat.ts) 实现：

1. 有客户端连接时启动定时器（`WEBSOCKETS_HEARTBEAT_PERIOD` 秒）
2. 定时向所有客户端发送 `ping` 消息
3. 等待客户端响应（任何消息都算存活）
4. 超时未响应的客户端会被关闭连接

```typescript
pingClients() {
    const pendingClients = new Set<WebSocketClient>(this.controller.clients);
    const timeout = setTimeout(() => {
        for (const client of pendingClients) {
            client.close();  // 关闭未响应的连接
        }
    }, HEARTBEAT_FREQUENCY);
    
    // 监听消息，收到任何消息都视为存活
    emitter.onAction('websocket.message', messageWatcher);
    
    // 发送 ping
    for (const client of pendingClients) {
        client.send(fmtMessage('ping'));
    }
}
```

#### 客户端心跳响应

在 [handleMessages()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L215-L219) 中：

```typescript
if (config.heartbeat && message['type'] === 'ping') {
    state.connection.send(pong());  // 回复 pong
    continue;
}
```

[pong()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/commands/pong.ts) 只是简单返回 `{ type: 'pong' }`。

### 5.5 TOKEN_EXPIRED 刷新

在 [handleAuthError()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L160-L199) 中处理：

```typescript
async function handleAuthError(message: WebSocketAuthError, currentClient: AuthWSClient<Schema>) {
    if (state.code !== 'open') return;
    
    if (message.error.code === 'TOKEN_EXPIRED') {
        debug('warn', 'Authentication token expired!');
        
        if (hasAuth(currentClient)) {
            const access_token = await currentClient.getToken();
            if (!access_token) {
                throw Error('No token for re-authenticating the websocket');
            }
            state.connection.send(auth({ access_token }));  // 重新认证
        }
    }
    
    // ... 其他错误处理
}
```

**流程**：
1. 收到 `TOKEN_EXPIRED` 错误
2. 调用 `getToken()` 获取新的 access token（auth 模块会自动刷新）
3. 发送新的 auth 消息重新认证
4. 认证成功后连接继续使用

### 5.6 重连机制

#### 触发重连

连接关闭时自动触发重连，见 [close 事件监听](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L344-L351)：

```typescript
ws.addEventListener('close', (evt: CloseEvent) => {
    eventHandlers['close'].forEach((handler') => handler.call(ws, evt));
    uid = generateUid();
    state = { code: 'closed' };
    reconnect(this);  // 触发重连
    if (!resolved) reject(evt);
});
```

#### 重连逻辑

[reconnect()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L95-L139) 函数：

```typescript
const reconnect = (self: WebSocketClient<Schema>) => {
    const reconnectPromise = new Promise<WebSocketInterface>((resolve, reject) => {
        if (!config.reconnect || wasManuallyDisconnected) return reject();
        
        if (reconnectState.attempts >= config.reconnect.retries) {
            reconnectState.attempts = -1;
            return reject();  // 达到最大重试次数
        }
        
        setTimeout(() => 
            self.connect()
                .then((ws) => {
                    // 重连后重新订阅
                    subscriptions.forEach((sub) => {
                        self.sendMessage(sub);
                    });
                    return ws;
                })
                .then(resolve)
                .catch(reject),
            Math.max(100, config.reconnect.delay)
        );
    });
    
    reconnectState.attempts += 1;
    // ...
};
```

**重连策略**：
- 延迟时间：`config.reconnect.delay`（默认 1000ms，最小 100ms）
- 最大重试次数：`config.reconnect.retries`（默认 10 次）
- 手动断开（`disconnect()`）不会自动重连
- 重连成功后重置重试次数

### 5.7 Resubscribe 机制

重连后有两个层面的重新订阅：

#### 层面一：连接层面的批量重订阅

在 `reconnect()` 函数中，连接成功后遍历 `subscriptions` 集合重新发送订阅消息：

```typescript
self.connect().then((ws) => {
    subscriptions.forEach((sub) => {
        self.sendMessage(sub);  // 重新发送每条订阅
    });
    return ws;
})
```

`subscriptions` 是一个 Set，保存了所有活跃的订阅配置。

#### 层面二：订阅生成器层面的恢复

在 [subscriptionGenerator()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts#L420-L461) 中：

```typescript
async function* subscriptionGenerator(): AsyncGenerator<...> {
    while (subscribed && state.code === 'open') {
        const message = await messageCallback(state.connection).catch(() => {});
        if (!message) continue;
        // ... 处理消息
        if (message['type'] === 'subscription' && message['uid'] === options.uid) {
            yield message;
        }
    }
    
    // 连接断开后，如果正在重连，等待并重试
    if (config.reconnect && reconnectState.active) {
        await reconnectState.active;
        if (state.code === 'open') {
            // 重新发送订阅
            state.connection.send(JSON.stringify({ ...options, collection, type: 'subscribe' }));
            yield* subscriptionGenerator();  // 递归恢复生成器
        }
    }
}
```

**两层机制的配合**：
1. 连接层的批量重订阅确保所有订阅都被重新发送
2. 生成器层的递归恢复确保正在迭代的订阅能继续接收消息
3. 两层都存在是为了保证可靠性，即使其中一层失败另一层也能工作

### 5.8 订阅 API 使用

```typescript
const { subscription, unsubscribe } = await client.subscribe('articles', {
    event: 'create',      // 可选：只监听特定事件
    query: { fields: ['id', 'title'] },  // 可选：查询参数
    uid: 'my-subscription',  // 可选：自定义订阅 ID
});

// 异步迭代接收消息
for await (const message of subscription) {
    console.log(message.event, message.data);
}

// 取消订阅
unsubscribe();
```

---

## 6. 完整流程图

### 6.1 连接建立与认证流程

```
客户端                          服务端
  │                              │
  │        HTTP Upgrade          │
  │─────────────────────────────►│
  │                              │
  │  strict 模式: 带 access_token │
  │  handshake 模式: 不带 token   │
  │  public 模式: 不带 token      │
  │                              │
  │  ◄────────────────────────── │
  │     101 Switching Protocols  │ (strict/public 直接成功)
  │                              │
  │                              │
  │  (handshake 模式)            │
  │  {type:"auth",               │
  │   access_token:"..."}        │
  │─────────────────────────────►│
  │                              │
  │  ◄────────────────────────── │
  │  {type:"auth", status:"ok"}  │
  │                              │
  │       连接已认证              │
```

### 6.2 订阅流程

```
客户端                          服务端                          事件总线
  │                              │                               │
  │  {type:"subscribe",          │                               │
  │   collection:"articles",     │                               │
  │   event:"create",            │                               │
  │   uid:"sub1"}                │                               │
  │─────────────────────────────►│                               │
  │                              │  添加到 subscriptions 集合    │
  │                              │  (按 collection 分组)         │
  │                              │                               │
  │  ◄────────────────────────── │                               │
  │  {type:"subscription",       │                               │
  │   event:"init",              │                               │
  │   uid:"sub1"}                │                               │
  │                              │                               │
  │                              │          数据变更              │
  │                              │  ◄────────────────────────── │
  │                              │       items.create 事件       │
  │                              │                               │
  │                              │  messenger.publish(           │
  │                              │    "websocket.event", ...)    │
  │                              │──────────────────────────────►│
  │                              │                               │
  │                              │  messenger.subscribe(         │
  │                              │    "websocket.event")         │
  │                              │◄──────────────────────────────│
  │                              │                               │
  │                              │  dispatch(event)              │
  │                              │  ─ 按 collection 过滤          │
  │                              │  ─ 按 event 过滤               │
  │                              │  ─ 按 item 过滤                │
  │                              │  ─ 按权限过滤                  │
  │                              │                               │
  │  ◄────────────────────────── │                               │
  │  {type:"subscription",       │                               │
  │   event:"create",            │                               │
  │   data: [...],               │                               │
  │   uid:"sub1"}                │                               │
```

### 6.3 重连与恢复流程

```
客户端                           服务端
  │                               │
  │   ◄────── 连接断开 ────────► │
  │                               │
  │  触发 reconnect()             │
  │  ─ 增加重试计数               │
  │  ─ 延迟 config.reconnect.delay│
  │                               │
  │        重新连接               │
  │  ──────────────────────────► │
  │                               │
  │  重新认证 (handshake/strict)  │
  │  ──────────────────────────► │
  │  ◄────────────────────────── │
  │       认证成功                │
  │                               │
  │  批量重订阅                   │
  │  (遍历 subscriptions 集合)    │
  │  {type:"subscribe", ...}     │
  │  ──────────────────────────► │
  │  {type:"subscribe", ...}     │
  │  ──────────────────────────► │
  │                               │
  │  订阅生成器递归恢复            │
  │  (继续 yield 消息)            │
```

---

## 7. 关键文件索引

| 文件 | 功能 |
|------|------|
| [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/server.ts) | 服务器启动入口 |
| [base.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/base.ts) | WebSocket 控制器基类 |
| [rest.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/rest.ts) | REST WebSocket 控制器 |
| [index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/index.ts) | 控制器工厂 |
| [authenticate.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/authenticate.ts) | 连接认证逻辑 |
| [messages.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/messages.ts) | 消息格式定义 |
| [types.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/types.ts) | TypeScript 类型定义 |
| [subscribe.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/subscribe.ts) | 订阅处理器 |
| [items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/items.ts) | Items CRUD 处理器 |
| [heartbeat.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/handlers/heartbeat.ts) | 心跳处理器 |
| [hooks.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/hooks.ts) | 事件钩子注册 |
| [items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/utils/items.ts) | 订阅数据获取工具 |
| [composable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/composable.ts) | SDK Realtime 客户端 |
| [types.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/types.ts) | SDK 类型定义 |
| [auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/commands/auth.ts) | SDK 认证命令 |
| [pong.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/sdk/src/realtime/commands/pong.ts) | SDK 心跳响应 |
