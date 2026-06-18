# Directus API 启动链路源码级说明

本文档从源码层面追踪 Directus API 的完整启动链路，从 `createApp` 到 `createServer`，详细分析各个阶段的执行顺序、关键栅栏和环境变量副作用。

---

## 一、总体架构概览

### 入口文件层次

```
startServer()
    └── createServer()
            └── createApp()  ← 核心启动逻辑
                    ├── 启动前校验（数据库、扩展、存储）
                    ├── 核心服务初始化
                    ├── Express App 配置
                    ├── 中间件挂载
                    ├── 路由注册
                    ├── 定时任务启动
                    └── 生命周期钩子
            ├── HTTP/HTTPS Server 创建
            ├── 请求指标收集
            ├── WebSocket 控制器（可选）
            └── Terminus 优雅关闭
```

### 关键源码文件

| 组件 | 文件路径 |
|------|---------|
| createApp | [app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L95-L433) |
| createServer | [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/server.ts#L37-L164) |
| 数据库校验 | [database/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/index.ts) |
| 扩展初始化 | [extensions/manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts) |

---

## 二、启动阶段详细分析（createApp 内部执行顺序）

### 阶段 1：启动前数据库与环境校验（L100-L142）

**执行顺序**：

```
1. validateDatabaseConnection()  → 数据库连通性校验
2. isInstalled()                 → Directus 表是否已安装
3. validateMigrations()          → 数据库迁移是否全部执行
4. SECRET / PUBLIC_URL 警告      → 关键环境变量检查
5. MCP_OAUTH_ENABLED 一致性检查  → 依赖 MCP_ENABLED=true
6. validateDatabaseExtensions()  → PostGIS/Spatialite 几何扩展
7. validateStorage()             → 存储驱动配置校验
```

**1.1 数据库连通性校验** ([validateDatabaseConnection](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/index.ts#L236-L251))

- 通过 `SELECT 1`（Oracle 用 `select 1 from DUAL`）测试连接
- **失败行为**：`process.exit(1)` 直接终止进程

**1.2 Directus 安装校验** ([isInstalled](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/index.ts#L277-L284))

- 检查 `directus_collections` 表是否存在
- **失败行为**：`process.exit(1)`，提示需要先运行 bootstrap

**1.3 迁移校验** ([validateMigrations](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/index.ts#L286-L318))

- 读取 `api/src/database/migrations/` 和扩展目录下的迁移文件
- 对比 `directus_migrations` 表中已记录的版本号
- **失败行为**：仅输出 WARN 日志（不终止进程）

**1.4 数据库扩展校验** ([validateDatabaseExtensions](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/index.ts#L323-L343))

- PostgreSQL：检查 PostGIS（几何类型支持）
- SQLite：检查 Spatialite
- **失败行为**：仅输出 WARN 日志，几何功能降级

---

### 阶段 2：核心服务初始化（L137-L149）

```
8.  LicenseManager.initialize()     → 许可证激活与状态同步
9.  EntitlementManager.initialize() → 功能权限初始化
10. registerAuthProviders()         → LDAP/SAML/OAuth2/OpenID/Local 注册
11. registerDeploymentDrivers()     → Netlify/Vercel 部署驱动
12. ensureDeploymentWebhooks()      → 部署 Webhook 端点
13. ExtensionManager.initialize()   → 扩展加载（核心阶段）
14. FlowManager.initialize()        → 工作流引擎初始化
```

**2.1 扩展初始化详解** ([ExtensionManager.initialize](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L189-L226))

内部 `load()` 方法执行：

```
a. syncExtensions()               → 从 EXTENSIONS_LOCATION 同步扩展
b. getExtensions()                → 扫描三种来源：
                                     ├── local (extensions/ 目录)
                                     ├── registry (.registry/ 目录)
                                     └── module (node_modules 包)
c. registerInternalOperations()   → 注册内置 Flow Operation（13种）
d. registerApiExtensions()        → 注册 API 扩展：
                                     ├── Hook（filter/action/init/schedule/embed）
                                     ├── Endpoint（自定义路由）
                                     ├── Operation（自定义 Flow 节点）
                                     └── Bundle（组合扩展）
e. generateExtensionBundle()      → 当 SERVE_APP=true 时，打包 App 扩展（Rollup/Rolldown）
```

---

### 阶段 3：Express App 基础配置（L150-L263）

```
15. express() 创建 App 实例
16. disable('x-powered-by')
17. set('trust proxy', IP_TRUST_PROXY)
18. 自定义 qs query parser（受 QUERYSTRING_MAX_PARSE_DEPTH 限制）
19. PRESSURE_LIMITER_ENABLED → handlePressure 中间件（过载保护）
20. Helmet CSP 安全头（Content-Security-Policy）
21. CROSS_ORIGIN_OPENER_POLICY_ENABLED → COOP 头
22. HSTS_ENABLED → HTTP Strict Transport Security
23. emitter.emitInit('app.before')       ← 扩展钩子
24. emitter.emitInit('middlewares.before') ← 扩展钩子
25. createExpressLogger()                → 请求日志
26. X-Powered-By: Directus 头
27. CORS_ENABLED → cors() 中间件
28. express.json() + rawBody 捕获（MAX_PAYLOAD_SIZE）
29. cookieParser()                       → Cookie 解析
30. extractToken                         → 令牌提取（关键栅栏）
```

**3.1 令牌提取中间件** ([extractToken](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/extract-token.ts#L14-L59))

按优先级提取令牌，写入 `req.token` 和 `req.tokenSource`：

| 来源 | 优先级 | 说明 |
|------|--------|------|
| `?access_token=` | 1 | Query 参数 |
| `Authorization: Bearer <token>` | 2 | Header，RFC6750 禁止多方式同时使用 |
| Cookie（SESSION_COOKIE_NAME） | 3 | 会话 Cookie，不受 RFC6750 限制 |

**关键规则**：如果同时存在 query 和 header 中的 token，抛出 `InvalidPayloadError`。

---

### 阶段 4：静态资源与基础路由（L266-L322）

```
31. GET / → ROOT_REDIRECT 重定向
32. GET /robots.txt → ROBOTS_TXT 内容
33. [SERVE_APP=true] Admin App 静态资源
34. RATE_LIMITER_GLOBAL_ENABLED → 全局限流
35. RATE_LIMITER_ENABLED → IP 级限流
36. GET /server/ping → 直接返回 'pong'（特殊位置）
37. /deployments/webhooks → 部署 Webhook 路由（无需认证）
38. [MCP_OAUTH_ENABLED=true] mcpOAuthPublicRouter → OAuth 公共端点
```

**⚠️ 注意**：`GET /server/ping` 位于认证中间件**之前**，是完全匿名的健康检查端点！

---

### 阶段 5：认证与核心中间件链（L328-L343）

```
39. authenticate           → 认证中间件（核心栅栏）
40. mcpOAuthGuard          → MCP OAuth 会话守卫
41. [MCP_OAUTH_ENABLED=true] mcpOAuthProtectedRouter
42. schema                 → 加载 Schema 到 req.schema
43. sanitizeQuery          → 查询参数规范化与校验
44. requestCounter         → API 请求计数（遥测）
45. cache                  → 响应缓存（读取侧）
46. emitter.emitInit('middlewares.after')
47. emitter.emitInit('routes.before')
```

**5.1 authenticate 详解** ([authenticate](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/authenticate.ts#L17-L62))

```
a. 创建 defaultAccountability（含 IP、User-Agent、Origin）
b. emitter.emitFilter('authenticate') → 扩展可自定义认证
c. 若扩展修改了 accountability → 直接使用扩展结果
d. 否则调用 getAccountabilityForToken(req.token)
   ├── JWT 验证 → 解析用户/角色/权限
   ├── Static Token 验证 → directus_users.token 字段
   └── 会话 Cookie → directus_sessions 表查询
e. 结果写入 req.accountability
f. 若令牌无效且是 Cookie 来源 → 清除 Cookie
```

**5.2 mcpOAuthGuard 详解** ([mcpOAuthGuard](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/mcp-oauth-guard.ts#L9-L21))

当 `req.accountability.oauth === true` 时：
- ✅ 仅允许访问 `/mcp/*` 路径（通过 `isMcpPath()` 判断）
- ✅ 仅允许 `GET` / `POST` 方法
- ❌ 其他路径一律抛出 `ForbiddenError`

**5.3 schema 中间件** ([schema](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/schema.ts#L2-L8))

- 调用 `getSchema()` 读取/缓存完整数据库 Schema（含 collections/fields/relations）
- 写入 `req.schema`，供后续所有服务层使用

**5.4 sanitizeQuery 中间件** ([sanitize-query](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/sanitize-query.ts#L10-L38))

- 规范化 `fields`、`filter`、`sort`、`limit`、`offset` 等查询参数
- 执行 `validateQuery()` 校验参数合法性
- 结果冻结写入 `req.sanitizedQuery`（不可变对象）

---

### 阶段 6：系统路由注册（L347-L407）

以下路由**按顺序**挂载，均已通过 authenticate + mcpOAuthGuard：

| 路径 | 说明 | 特性 |
|------|------|------|
| `/auth` | 认证（登录/登出/刷新/SSO） | 部分端点内部跳过认证 |
| `/graphql` | GraphQL API | 含 WebSocket 升级 |
| `/activity` | 活动日志 | |
| `/access` | 权限 | |
| `/assets` | 文件访问（缩略图） | |
| `/collections` | 集合 CRUD | |
| `/comments` | 评论 | |
| `/dashboards` | 仪表盘 | |
| `/deployments` | 部署管理 | |
| `/extensions` | 扩展管理 | |
| `/fields` | 字段配置 | |
| `/files/tus` | TUS 断点续传 | TUS_ENABLED 控制 |
| `/files` | 文件管理 | |
| `/flows` | 工作流 | |
| `/folders` | 文件夹 | |
| `/items` | 业务数据 CRUD | **核心业务路由** |
| `/license` | 许可证管理 | |
| `/mcp` | MCP AI 协议服务器 | MCP_ENABLED 控制 |
| `/ai` | AI Chat/Files | AI_ENABLED 控制 |
| `/metrics` | Prometheus 指标 | METRICS_ENABLED 控制 |
| `/notifications` | 通知 | |
| `/operations` | Flow 操作 | |
| `/panels` | 仪表盘面板 | |
| `/permissions` | 权限规则 | |
| `/policies` | 策略 | |
| `/presets` | 视图预设 | |
| `/translations` | 翻译 | |
| `/relations` | 关系配置 | |
| `/revisions` | 数据修订历史 | |
| `/roles` | 角色管理 | |
| `/mcp-oauth/clients` | MCP OAuth 客户端管理 | MCP_OAUTH_ENABLED 控制 |
| `/schema` | Schema 快照/对比/应用 | |
| `/server` | 服务器信息（info/health/setup/specs） | |
| `/settings` | 系统设置 | |
| `/shares` | 共享链接 | |
| `/users` | 用户管理 | |
| `/utils` | 工具（hash/import/export 等） | |
| `/versions` | 数据版本 | |

---

### 阶段 7：自定义端点与错误处理（L409-L418）

```
48. emitter.emitInit('routes.custom.before')
49. extensionManager.getEndpointRouter() → 所有自定义端点
50. emitter.emitInit('routes.custom.after')
51. notFoundHandler → 404 处理
52. errorHandler    → 全局错误处理（核心栅栏）
53. emitter.emitInit('routes.after')
```

**7.1 错误处理中间件** ([errorHandler](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/error-handler.ts#L22-L148))

特殊分支处理：

| 场景 | 行为 |
|------|------|
| MCP 路径 + 401 令牌错误 | 返回 RFC 6750 `WWW-Authenticate` 头，含 `invalid_token` 挑战 |
| DirectusError | 按状态码聚合，开发模式下暴露 stack 追踪 |
| 非 DirectusError + 管理员 | 暴露原始错误详情 |
| 非 DirectusError + 非管理员 | 返回通用 500 消息 |
| MethodNotAllowed | 自动添加 `Allow` 响应头 |

最后触发 `request.error` Filter Hook 允许扩展二次加工。

---

### 阶段 8：定时任务启动（L419-L428）

```
54. retentionSchedule()  → 数据保留策略（活动日志/修订历史清理）
55. telemetrySchedule()  → 遥测数据上报
56. tusSchedule()        → TUS 过期上传清理
57. metricsSchedule()    → 指标聚合
58. projectSchedule()    → 项目级定时任务
59. licenseSchedule()    → 许可证状态同步
60. [MCP_OAUTH_ENABLED=true] scheduleOAuthCleanup() → OAuth 授权码/过期 Token 清理
```

---

### 阶段 9：生命周期结束钩子

```
61. emitter.emitInit('app.after')
62. return app → createServer() 接收
```

---

## 三、createServer 层补充逻辑

### 3.1 HTTP Server 创建与指标收集 ([createServer](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/server.ts#L37-L103))

```typescript
server = http.createServer(app);  // 将 createApp() 注入

server.on('request', (req, res) => {
    // 为每个请求记录：
    //   - 字节数（bytesRead/bytesWritten）
    //   - 耗时（hrtime 纳秒级精度）
    //   - 完整请求/响应元数据
    // 触发 emitter.emitAction('response', ...)
});
```

### 3.2 WEBSOCKETS_ENABLED 副作用（L104-L110）

```typescript
if (WEBSOCKETS_ENABLED) {
    createSubscriptionController(server);  // GraphQL Subscriptions
    createWebSocketController(server);     // REST WebSocket API
    createLogsController(server);          // 实时日志流
    startWebSocketHandlers();              // 协作处理等高级功能
}
```

三个独立控制器（按 env 细分开关）：
- `WEBSOCKETS_GRAPHQL_ENABLED` → GraphQL 订阅
- `WEBSOCKETS_REST_ENABLED` → REST 风格 WebSocket
- `WEBSOCKETS_LOGS_ENABLED` → 日志流

### 3.3 Terminus 优雅关闭（L112-L163）

关闭阶段生命周期：
1. **beforeShutdown**：设置 `SERVER_ONLINE = false`
2. **onSignal**（SIGINT/SIGTERM/SIGHUP）：
   - 终止所有 WebSocket 控制器
   - 关闭协作处理器
   - 刷新所有缓冲遥测计数器
   - 关闭 AI 遥测
   - `database.destroy()` 归还连接池
3. **onShutdown**：触发 `server.stop` Action Hook

---

## 四、三种请求路径的关键栅栏分析

### 路径 A：带 Authorization 头的 GET /items/:collection

```
请求到达
  │
  ├─ 1. createServer request 事件 → 开始计时/度量
  │
  ├─ 2. PRESSURE_LIMITER → 过载保护（可选）
  ├─ 3. Helmet CSP → 安全响应头
  ├─ 4. CORS → 跨域预检（可选）
  ├─ 5. express.json() → Body 解析（GET 无 body，快速通过）
  ├─ 6. cookieParser()
  │
  ├─ 7. extractToken  ✅ 关键栅栏
  │     └── 从 Authorization: Bearer 提取 → req.token
  │         若同时有 query token → InvalidPayloadError
  │
  ├─ 8. RATE_LIMITER_GLOBAL → 全局限流（可选）
  ├─ 9. RATE_LIMITER → IP 级限流（可选）
  │
  │     ⚠️ /server/ping 在此处之前已响应，不会继续向下
  │
  ├─ 10. authenticate  ✅✅✅ 核心认证栅栏
  │      ├── emitFilter('authenticate') → 扩展可介入
  │      └── getAccountabilityForToken()
  │           ├── JWT 签名验证 + 过期检查
  │           ├── 用户/角色查询
  │           ├── 写入 req.accountability
  │           └── 失败 → InvalidCredentialsError → errorHandler
  │
  ├─ 11. mcpOAuthGuard  ✅  OAuth 会话栅栏
  │      └── 常规 JWT（accountability.oauth = false）→ 通过
  │
  ├─ 12. schema  → 加载完整 Schema → req.schema
  ├─ 13. sanitizeQuery  ✅  查询规范化
  │      ├── fields/filter/sort/limit 校验
  │      ├── 生成 req.sanitizedQuery（冻结对象）
  │      └── 失败 → next(error)
  │
  ├─ 14. requestCounter → 遥测 GET 计数（可选）
  ├─ 15. cache  ✅  缓存读取
  │      ├── 命中 → res.json(cachedData) → 响应结束
  │      └── 未命中 → 设置 CACHE_STATUS: MISS，继续
  │
  ├─ 16. 进入 /items 路由：
  │      ├── checkIsLocked('items')  ✅  许可证锁定检查
  │      ├── collectionExists  ✅  集合存在性 + singleton 标记
  │      ├── readHandler:
  │      │     ├── isSystemCollection → ForbiddenError（禁止直接访问系统表）
  │      │     ├── ItemsService.readByQuery()
  │      │     │    └── 权限层：process-ast 按角色过滤字段/行
  │      │     └── MetaService.getMetaForQuery()
  │      └── respond  ✅  响应发送 + 缓存写入
  │           ├── 计算是否可缓存（权限/TTL/大小）
  │           ├── 命中条件 → setCacheValue() 写入缓存层
  │           ├── 设置 Cache-Control + Vary 头
  │           └── res.json(payload)
  │
  ├─ 17. errorHandler（仅异常路径）
  │
  └─ 18. createServer 'finish' 事件 → 记录耗时 + bytes → emitAction('response')
```

---

### 路径 B：GET /server/ping

**极短路径，在认证中间件之前直接响应！**

```
请求到达
  │
  ├─ 1. createServer request 事件 → 计时
  ├─ 2. PRESSURE_LIMITER
  ├─ 3. Helmet CSP 头
  ├─ 4. CORS
  ├─ 5. express.json()
  ├─ 6. cookieParser()
  ├─ 7. extractToken（执行但未使用结果）
  │
  ├─ 8. RATE_LIMITER_GLOBAL
  ├─ 9. RATE_LIMITER
  │
  ├─ 10. ⚡  app.get('/server/ping', (_req, res) => res.send('pong'))
  │         └── 直接发送纯文本 'pong'
  │             不经过 authenticate
  │             不经过 schema
  │             不经过 cache
  │             不经过 errorHandler（正常路径）
  │
  └─ 11. createServer 'finish' 事件
```

**设计意图**：这是负载均衡器/容器健康检查的专用端点，不依赖数据库或认证体系。即使许可证锁定也能正常响应。

---

### 路径 C：MCP OAuth 请求（完整授权码流程）

MCP OAuth 有**公共路由**（认证前）和**受保护路由**（认证后）两组路径。以完整授权码交换流程为例：

#### C1：发现阶段 - GET /.well-known/oauth-protected-resource

```
请求到达
  │
  ├─ ... 基础中间件（extractToken 之前的所有步骤）
  │
  ├─ mcpOAuthPublicRouter（挂载于 authenticate 之前）
  │     ├── setCorsWildcard → Access-Control-Allow-Origin: *
  │     ├── checkOAuthSettings → 读取 settings.mcp_enabled + mcp_oauth_enabled
  │     │     └── 未启用 → 403 mcp_oauth_disabled
  │     └── McpOAuthService.getProtectedResourceMetadata()
  │           └── 返回 RFC 9728 元数据（含 authorization_servers 列表）
```

#### C2：授权页面 - GET /mcp-oauth/authorize

```
请求到达（浏览器导航）
  │
  ├─ mcpOAuthPublicRouter
  │     ├── oauthRateLimitMiddleware → 授权页面限流
  │     ├── checkOAuthSettings
  │     ├── ⚠️  手动检查会话 Cookie（无 authenticate 中间件）
  │     │     ├── 无 Cookie → 302 重定向到 /admin/login?redirect=...
  │     │     ├── 有 Cookie → getAccountabilityForToken() 验证
  │     │     └── 验证失败 → 302 重定向登录
  │     ├── 验证用户已登录且不是 OAuth 会话本身
  │     ├── McpOAuthService.validateAuthorization()
  │     │     ├── client_id 存在性 + redirect_uri 匹配校验
  │     │     ├── scope / response_type 校验
  │     │     ├── PKCE code_challenge 处理
  │     │     └── 生成 signed_params JWT（包含所有验证后的参数）
  │     └── renderConsentPage()
  │           ├── 渲染 HTML 同意页面（项目 Logo/名称/客户端信息/权限范围）
  │           ├── 显示 redirect_uri 风险指示器（localhost/IP/跨域）
  │           └── 表单 POST 目标：/mcp-oauth/authorize/decision
```

#### C3：授权决策 - POST /mcp-oauth/authorize/decision

```
请求到达（浏览器表单提交）
  │
  ├─ ... 经过 authenticate + mcpOAuthGuard
  │
  ├─ mcpOAuthProtectedRouter（在 authenticate 之后）
  │     ├── checkOAuthSettings
  │     ├── express.urlencoded()
  │     ├── rejectDuplicateParams → RFC 6749 禁止重复参数
  │     ├── requireCookieAuth  ✅  强制 Cookie 认证
  │     │     └── tokenSource !== 'cookie' → 403（防令牌窃取攻击）
  │     ├── requireSameOrigin  ✅  CSRF 防护
  │     │     ├── Origin 头必选
  │     │     └── 与 PUBLIC_URL.origin 严格匹配
  │     └── McpOAuthService.processDecision()
  │           ├── 校验 signed_params JWT（防止参数篡改）
  │           ├── 用户同意/拒绝处理
  │           ├── 同意 → 生成一次性授权码（code）
  │           ├── 拒绝 → 重定向 redirect_uri?error=access_denied
  │           └── 302 重定向 → redirect_uri?code=...&state=...&iss=...
```

#### C4：令牌交换 - POST /mcp-oauth/token

```
请求到达（后端对后端，Authorization 头含客户端凭证）
  │
  ├─ mcpOAuthPublicRouter
  │     ├── oauthRateLimitMiddleware
  │     ├── checkOAuthSettings
  │     ├── express.urlencoded()
  │     ├── rejectDuplicateParams
  │     ├── setCorsWildcard
  │     ├── setNoCacheHeaders → Cache-Control: no-store
  │     └── McpOAuthService.exchangeCode() / refreshToken()
  │           ├── 客户端认证（client_secret_basic / client_secret_post / none）
  │           ├── grant_type = authorization_code / refresh_token
  │           ├── 授权码一次性校验（消耗后立即失效）
  │           ├── redirect_uri 精确匹配
  │           ├── PKCE code_verifier 校验
  │           └── 返回 access_token(JWT) + refresh_token + id_token
  │                 JWT 中包含：scope: "mcp", oauth: true
```

#### C5：使用 OAuth Access Token 访问 MCP - POST /mcp

```
请求到达（Authorization: Bearer <OAuth JWT>）
  │
  ├─ extractToken → 从 Header 提取 OAuth token
  ├─ authenticate
  │     └── getAccountabilityForToken()
  │           ├── 验证 JWT 签名
  │           ├── 识别 oauth=true 标记
  │           └── accountability = { user: null, role: null, oauth: true, ... }
  │
  ├─ mcpOAuthGuard  ✅✅✅  关键守卫
  │     ├── 检测到 accountability.oauth === true
  │     ├── 检查 req.path 是否 /mcp/*
  │     ├── 检查 method 是否 GET/POST
  │     └── 检查通过 → next()
  │         不通过 → ForbiddenError
  │
  ├─ schema / sanitizeQuery / cache
  │
  ├─ 进入 /mcp 路由
  │     └── MCP Server 执行 JSON-RPC 2.0 工具调用
  │
  ├─ respond → 返回结果（若 MCP 未自定义响应）
  │
  └─ errorHandler 特殊分支：
        若路径是 /mcp + 401 令牌错误
          → 返回 WWW-Authenticate: Bearer error="invalid_token",
             authorization_server="...", resource="..."
          → 符合 RFC 6750 规范
```

---

## 五、关键环境变量副作用

### 5.1 SERVE_APP（[app.ts L280-L308](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L280-L308)）

**激活条件**：`SERVE_APP=true`（默认生产环境 true）

**副作用列表**：

| 阶段 | 副作用 |
|------|--------|
| 扩展初始化 | `ExtensionManager.generateExtensionBundle()`<br>→ 用 Rollup/Rolldown 打包所有 App 扩展到 `TEMP_PATH/app-extensions/`<br>→ 消耗额外 CPU/内存 |
| 路由层 | 注册三条路由：<br>`GET /admin` → 注入 base href + 扩展嵌入点的 index.html<br>`USE /admin` → 静态资源服务（31536000 秒强缓存）<br>`GET /admin/*` → SPA Fallback |
| HTML 注入 | 读取 `@directus/app` 包的 index.html，动态替换：<br>• `<base />` → 根据 PUBLIC_URL 重写<br>• `<!-- directus-embed-head -->` → 扩展 hook embed 注入<br>• `<!-- directus-embed-body -->` → 扩展 hook embed 注入 |

**性能影响**：首次启动时 Rollup 打包可能需要数秒（取决于扩展数量）。

---

### 5.2 WEBSOCKETS_ENABLED（[server.ts L104-L110](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/server.ts#L104-L110)）

**激活条件**：`WEBSOCKETS_ENABLED=true`

**副作用列表**：

| 子开关 | 控制器 | 行为 |
|--------|--------|------|
| `WEBSOCKETS_GRAPHQL_ENABLED` | `GraphQLSubscriptionController` | 在 HTTP Server 上监听 `upgrade` 事件，建立 GraphQL Subscription WebSocket 通道（graphql-ws 协议） |
| `WEBSOCKETS_REST_ENABLED` | `WebSocketController` | REST 风格的 WebSocket API，支持通过 WebSocket 发送 CRUD 请求并接收响应 |
| `WEBSOCKETS_LOGS_ENABLED` | `LogsController` | 实时日志流，前端可通过 WebSocket 订阅服务器日志输出 |
| 隐含 | `startWebSocketHandlers()` | 启动协作处理（协同编辑）等高级实时功能 |

**关闭阶段副作用**：
- `beforeShutdown` 后逐步关闭所有 WebSocket 连接
- 调用各 `terminate()` 方法优雅断开客户端
- 协作处理器 `terminate()` 保存未持久化状态

**资源影响**：
- 每个 WebSocket 连接占用 ~10-50KB 内存（取决于订阅数量）
- 连接数高时增加 Event Loop 压力

---

## 六、启动时序总览图

```
时间轴 →

 createApp()
     │
     ├─ [校验层] DB连接 → 安装检查 → 迁移 → 扩展 → 存储
     │
     ├─ [服务层] 许可证 → 认证 → 部署 → 扩展 → Flow
     │
     ├─ [配置层] Express → Helmet → CORS → BodyParser → Cookie
     │
     ├─ [提取层] extractToken (3种来源)
     │
     ├─ [前置路由] /server/ping  ← 无认证
     │              /deployments/webhooks
     │              mcpOAuthPublicRouter
     │
     ├─ [认证层] ╔══════════════════╗
     │          ║   authenticate   ║ ← JWT/Static/Cookie
     │          ╚══════════════════╝
     │          ╔══════════════════╗
     │          ║  mcpOAuthGuard   ║ ← OAuth会话仅限MCP
     │          ╚══════════════════╝
     │
     ├─ [预处理层] schema → sanitizeQuery → counter → cache
     │
     ├─ [路由层] 系统路由(40+) → 自定义端点 → 404 → errorHandler
     │
     └─ [定时层] 6个计划任务启动
             │
 createServer()
     │
     ├─ http.createServer(app)
     ├─ 'request' 事件 → 请求度量
     │
     ├─ [可选] WEBSOCKETS_ENABLED → 3个WS控制器
     │
     └─ createTerminus() → 优雅关闭钩子

 监听端口...
```

---

## 七、关键栅栏总结表

| 栅栏名称 | 位置 | 失败结果 | 影响范围 |
|----------|------|----------|----------|
| validateDatabaseConnection | 启动阶段 | exit(1) | 全服务 |
| isInstalled | 启动阶段 | exit(1) | 全服务 |
| PRESSURE_LIMITER | 请求层 | 503 Under Pressure | 全服务（过载时） |
| extractToken（多方法检测） | 请求层 | 400 InvalidPayload | 同时使用多方式传令牌的请求 |
| RATE_LIMITER | 请求层 | 429 Too Many Requests | 超阈值 IP/全局 |
| authenticate | 请求层 | 401 InvalidCredentials/Token | 所有受保护路由 |
| mcpOAuthGuard | 请求层 | 403 Forbidden | OAuth 会话访问非 MCP 路径 |
| sanitizeQuery | 请求层 | 400 系列 | 查询参数错误 |
| checkIsLocked | 路由层 | 403 ResourceRestricted | 许可证锁定时 |
| collectionExists | 路由层 | 403 CollectionForbidden | 不存在的集合 |
| permission 层 | 服务层 | 403 Forbidden | 行级/字段级权限不足 |
| errorHandler（MCP分支） | 响应层 | 401 WWW-Authenticate 头 | MCP 路径令牌错误 |
