# DirectusMCP 工具暴露与权限边界分析

## 一、整体架构概览

DirectusMCP 是 Directus 中用于将 AI 工具暴露给 MCP (Model Context Protocol) 客户端的模块。它通过 `/mcp` 端点对外提供服务，支持三种用户身份边界：

- **普通认证用户**：通过常规会话或 API token 访问
- **OAuth 用户**：通过 MCP OAuth 流程获取 token 的第三方客户端
- **Admin 用户**：拥有 admin 权限的用户，可访问 admin 级工具

核心文件位置：
- MCP 服务器：[server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts)
- MCP 控制器：[index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/index.ts)
- MCP OAuth 控制器：[oauth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/oauth.ts)
- 工具定义集合：[index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/index.ts)
- OAuth 守卫中间件：[mcp-oauth-guard.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/mcp-oauth-guard.ts)

---

## 二、Settings 配置如何进入 DirectusMCP

### 2.1 配置项清单

Settings 表中与 MCP 相关的字段定义在 [settings.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/settings.ts#L83-L88)：

| 配置字段 | 类型 | 说明 |
|---------|------|------|
| `mcp_enabled` | boolean | MCP 功能总开关 |
| `mcp_oauth_enabled` | boolean | MCP OAuth 认证开关 |
| `mcp_allow_deletes` | boolean | 是否允许通过 MCP 执行删除操作 |
| `mcp_prompts_collection` | string \| null | 存储 prompt 的集合名称 |
| `mcp_system_prompt_enabled` | boolean | system-prompt 工具是否启用 |
| `mcp_system_prompt` | string \| null | 自定义 system prompt 内容 |

### 2.2 配置流向

配置从 Settings 表到 `DirectusMCP` 实例的完整流程：

**第一步：环境变量级别的开关（app.ts 挂载层）**

在 [app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L371-L373) 中，MCP 路由的挂载首先受环境变量控制：

```typescript
if (toBoolean(env['MCP_ENABLED']) === true) {
    app.use('/mcp', mcpRouter);
}

if (toBoolean(env['MCP_OAUTH_ENABLED']) === true) {
    app.use(mcpOAuthPublicRouter);
    // ...
    app.use(mcpOAuthProtectedRouter);
}
```

**第二步：Settings 级别的开关（控制器层）**

在 MCP 控制器 [index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/index.ts#L14-L58) 中，每次请求都会读取 Settings 并进行二次校验：

```typescript
const mcpHandler = asyncHandler(async (req, res) => {
    const settings = new SettingsService({ schema: req.schema });

    const {
        mcp_enabled,
        mcp_oauth_enabled,
        mcp_allow_deletes,
        mcp_prompts_collection,
        mcp_system_prompt,
        mcp_system_prompt_enabled,
    } = await settings.readSingleton({
        fields: [
            'mcp_enabled',
            'mcp_oauth_enabled',
            'mcp_allow_deletes',
            'mcp_prompts_collection',
            'mcp_system_prompt',
            'mcp_system_prompt_enabled',
        ],
    });

    // 总开关校验
    if (!mcp_enabled) {
        throw new ForbiddenError({ reason: 'MCP must be enabled' });
    }

    // OAuth 开关校验
    if (
        req.accountability?.oauth &&
        (toBoolean(env['MCP_OAUTH_ENABLED']) !== true || toBoolean(mcp_oauth_enabled) !== true)
    ) {
        throw new ForbiddenError({ reason: 'MCP OAuth must be enabled' });
    }

    // 构建设置传入 DirectusMCP
    const mcp = new DirectusMCP({
        promptsCollection: mcp_prompts_collection,
        allowDeletes: mcp_allow_deletes,
        systemPromptEnabled: mcp_system_prompt_enabled,
        systemPrompt: mcp_system_prompt,
    });

    mcp.handleRequest(req, res);
});
```

**第三步：DirectusMCP 构造函数接收配置**

在 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L40-L58) 的构造函数中，配置被保存为实例属性：

```typescript
export class DirectusMCP {
    promptsCollection?: string | null;
    systemPrompt?: string | null;
    systemPromptEnabled?: boolean;
    server: Server;
    allowDeletes?: boolean;

    constructor(options: MCPOptions = {}) {
        this.promptsCollection = options.promptsCollection ?? null;
        this.systemPromptEnabled = options.systemPromptEnabled ?? true;
        this.systemPrompt = options.systemPrompt ?? null;
        this.allowDeletes = options.allowDeletes ?? false;
        // ... MCP Server 初始化
    }
}
```

`MCPOptions` 类型定义在 [types.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/types.ts#L8-L13)。

---

## 三、工具列表过滤机制

### 3.1 全部工具清单

所有 MCP 工具定义在 [tools/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/index.ts) 中，共计 13 个工具：

```typescript
export const ALL_TOOLS: ToolConfig<any>[] = [
    system,           // system-prompt
    items,            // 数据项 CRUD
    files,            // 文件
    folders,          // 文件夹
    assets,           // 资源
    flows,            // 工作流（admin）
    triggerFlow,      // 触发工作流
    operations,       // 操作（admin）
    schema,           // 数据 Schema
    collections,      // 集合管理（admin）
    fields,           // 字段管理（admin）
    relations,        // 关系管理（admin）
];
```

### 3.2 Admin 工具标记

通过 `ToolConfig` 接口的 `admin?: boolean` 属性标记。标记为 `admin: true` 的工具有 5 个：

| 工具名称 | 文件 | 说明 |
|---------|------|------|
| collections | [collections/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/collections/index.ts#L45) | 集合管理 |
| fields | [fields/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/fields/index.ts#L76) | 字段管理 |
| relations | [relations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/relations/index.ts#L51) | 关系管理 |
| flows | [flows/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/flows/index.ts#L44) | 工作流管理 |
| operations | [operations/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/operations/index.ts#L48) | 操作管理 |

### 3.3 列表过滤逻辑

在 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L245-L261) 的 `ListToolsRequestSchema` 处理器中实现过滤：

```typescript
this.server.setRequestHandler(ListToolsRequestSchema, () => {
    const tools = [];

    for (const tool of getAllMcpTools()) {
        // 过滤 1：非 admin 用户排除 admin 工具
        if (req.accountability?.admin !== true && tool.admin === true) continue;
        
        // 过滤 2：system-prompt 工具受 systemPromptEnabled 控制
        if (tool.name === 'system-prompt' && this.systemPromptEnabled === false) continue;

        tools.push({
            name: tool.name,
            description: tool.description,
            inputSchema: z.toJSONSchema(tool.inputSchema),
            annotations: tool.annotations,
        });
    }

    return { tools };
});
```

**双重过滤机制：**

1. **Admin 工具过滤**：基于 `req.accountability.admin` 判断。非管理员用户在工具列表中看不到 admin 工具，也无法调用（调用时会再次校验）。

2. **System-prompt 工具过滤**：当 `mcp_system_prompt_enabled` 为 `false` 时，`system-prompt` 工具从列表中隐藏。该工具用于返回 Directus 的系统提示词，帮助 AI 理解如何使用其他工具。

---

## 四、callTool 调用与校验流程

### 4.1 完整调用流程

调用工具的处理逻辑位于 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L264-L322)，整体流程如下：

```
请求到达 → 工具存在性检查 → admin 权限校验 → system-prompt 特殊处理 
→ coerceJsonFields → schema 校验 → delete 操作禁止检查 → 调用 handler 
→ 构建 URL → 转换为 MCP 响应格式
```

### 4.2 详细步骤解析

**步骤 1：工具查找与存在性检查**

```typescript
const tool = findMcpTool(request.params.name);

if (!tool || (tool.name === 'system-prompt' && this.systemPromptEnabled === false)) {
    throw new InvalidPayloadError({ reason: `"${request.params.name}" doesn't exist in the toolset` });
}
```

即使工具未出现在列表中（如直接调用隐藏的 system-prompt），这里也会做二次检查。

**步骤 2：Admin 权限校验**

```typescript
if (req.accountability?.admin !== true && tool.admin === true) {
    throw new ForbiddenError({ reason: 'You must be an admin to access this tool' });
}
```

这是调用时的权限防线，与列表过滤形成双重保障。

**步骤 3：system-prompt 特殊处理**

```typescript
if (tool.name === 'system-prompt') {
    request.params.arguments = { promptOverride: this.systemPrompt };
}
```

当调用 `system-prompt` 工具时，将 settings 中配置的 `mcp_system_prompt` 作为参数注入。这意味着客户端无法自定义 system prompt 的内容，只能使用管理员配置的版本。

**步骤 4：coerceJsonFields - JSON 字段强制转换**

函数定义在 [tools/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/utils.ts#L11-L25)：

```typescript
const JSON_COERCE_FIELDS = ['data', 'keys', 'query', 'headers'];

export function coerceJsonFields(args: Record<string, unknown>): Record<string, unknown> {
    const coerced = { ...args };

    for (const field of JSON_COERCE_FIELDS) {
        if (typeof coerced[field] === 'string') {
            try {
                coerced[field] = parseJSON(coerced[field] as string);
            } catch {
                // Leave as-is if not valid JSON
            }
        }
    }

    return coerced;
}
```

**设计意图**：LLM 有时会将对象/数组参数以字符串化 JSON 的形式返回（例如把 `data` 字段的值作为字符串传递）。此函数将已知字段从 JSON 字符串强制解析回原生 JavaScript 值，提高 LLM 调用的容错性。

涉及的字段：`data`、`keys`、`query`、`headers`。

**步骤 5：Schema 校验**

```typescript
const { error, data: args } = tool.validateSchema?.safeParse(coercedArgs) ?? {
    data: coercedArgs,
};

if (error) {
    throw new InvalidPayloadError({ reason: fromZodError(error).message });
}

if (!isObject(args)) {
    throw new InvalidPayloadError({ reason: '"arguments" must be an object' });
}
```

每个工具都有两个 Schema：
- `inputSchema`：用于生成 MCP 工具描述（`z.toJSONSchema` 转换）
- `validateSchema`：用于实际参数校验（更严格，通常使用 `z.strictObject` 或 `z.discriminatedUnion`）

Schema 校验使用 Zod 的 `safeParse`，失败时通过 `fromZodError` 转换为人类可读的错误信息。

**步骤 6：Delete 操作禁止检查**

```typescript
if (this.allowDeletes === false && 'action' in args && args['action'] === 'delete') {
    throw new InvalidPayloadError({ reason: 'Delete actions are disabled' });
}
```

当 `mcp_allow_deletes` 为 `false`（默认值）时，任何 action 为 `delete` 的调用都会被拒绝。这是全局级别的删除防护。

**步骤 7：调用 Handler**

```typescript
const result = await tool.handler({
    args,
    schema: req.schema,
    accountability: req.accountability,
});
```

工具的 `handler` 函数接收三个参数：
- `args`：经过校验的参数
- `schema`：数据库 Schema 概览（`SchemaOverview`）
- `accountability`：用户身份信息（含用户 ID、角色、admin 标记等）

**步骤 8：构建资源 URL**

```typescript
const data = toArray(result?.data);

if (
    'action' in args &&
    ['create', 'update', 'read', 'import'].includes(args['action'] as string) &&
    result?.data &&
    data.length === 1
) {
    result.url = this.buildURL(tool, args, data[0]);
}
```

对于单条记录的创建、读取、更新操作，会自动构建指向 Directus Admin 后台的 URL，方便用户在浏览器中查看。

`buildURL` 方法通过工具的 `endpoint` 函数生成路径，最终格式为 `{PUBLIC_URL}/admin/content/{collection}/{id}`。

**步骤 9：转换为 MCP 响应格式**

`toToolResponse` 方法定义在 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L369-L386)：

```typescript
toToolResponse(result?: ToolResult) {
    const response: CallToolResult = {
        content: [],
    };

    if (!result || typeof result.data === 'undefined' || result.data === null) return response;

    if (result.type === 'text') {
        response.content.push({
            type: 'text',
            text: JSON.stringify({ raw: result.data, url: result.url }),
        });
    } else {
        response.content.push(result);
    }

    return response;
}
```

文本类型的结果会被包装为 JSON 字符串，包含 `raw`（原始数据）和 `url`（资源链接）两个字段。

**步骤 10：错误处理**

`toExecutionError` 方法将 Directus 错误转换为 MCP 格式的错误响应：

```typescript
toExecutionError(err: unknown) {
    // ... 遍历错误，提取 message 和 code
    return {
        isError: true,
        content: [{ type: 'text' as const, text: JSON.stringify(errors) }],
    };
}
```

---

## 五、用户身份边界与权限模型

### 5.1 三种用户身份

| 身份类型 | 识别方式 | 可访问范围 |
|---------|---------|-----------|
| 未认证用户 | 无 accountability | 拒绝访问（401） |
| 普通认证用户 | `accountability.user` 存在，`oauth` 为 falsy | 非 admin 工具 + 自有数据权限 |
| OAuth 用户 | `accountability.oauth` 存在 | 受 OAuth 范围限制 + 非 admin 工具 + 自有数据权限 |
| Admin 用户 | `accountability.admin === true` | 全部工具 + 全部数据权限 |

### 5.2 未认证用户拦截

在 `handleRequest` 方法入口处 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L91-L95)：

```typescript
if (!req.accountability?.user && !req.accountability?.role && req.accountability?.admin !== true) {
    this.sendUnauthorized(res);
    return;
}
```

未认证用户会收到 401 响应，并带有 `WWW-Authenticate` 头，包含 OAuth 发现信息（RFC 6750 / RFC 9728）。

### 5.3 OAuth 用户特殊限制

OAuth 用户在 `handleRequest` 中经过三层额外校验 [server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/server.ts#L98-L115)：

```typescript
if (oauth) {
    // 限制 1：Token 必须来自 Authorization header（RFC 6750）
    if (req.tokenSource !== 'header') {
        this.sendUnauthorized(res, 'invalid_request');
        return;
    }

    // 限制 2：必须包含 mcp:access scope
    if (!oauth.scopes.includes(MCP_ACCESS_SCOPE)) {
        this.sendUnauthorized(res, 'insufficient_scope', 403);
        return;
    }

    // 限制 3：audience 必须匹配 MCP 资源 URL
    const { resourceUrl } = getMcpUrls();
    if (!oauth.aud.includes(resourceUrl)) {
        this.sendUnauthorized(res, 'invalid_token');
        return;
    }
}
```

**MCP_ACCESS_SCOPE** 常量定义在 [utils.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/mcp/utils.ts#L4)，值为 `'mcp:access'`。

### 5.4 OAuth 全局守卫中间件

[mcp-oauth-guard.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/mcp-oauth-guard.ts) 确保 OAuth token 只能用于访问 MCP 端点：

```typescript
export function handler(req: Request, _res: Response, next: NextFunction): void {
    if (!req.accountability?.oauth) {
        next();
        return;
    }

    if (!isMcpPath(req.path) || (req.method !== 'GET' && req.method !== 'POST')) {
        next(new ForbiddenError());
        return;
    }

    next();
}
```

这个中间件在 [app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L329) 中挂载在 `authenticate` 中间件之后，所有路由之前。它确保 OAuth 颁发的 token 只能访问 `/mcp` 路径，不能访问其他 Directus API。

### 5.5 OAuth 设置校验

除了环境变量级别的开关，OAuth 端点还会在每次请求时检查 settings 中的开关：

- **MCP 控制器中**：`mcp_oauth_enabled` 必须为 true 才允许 OAuth 用户访问 `/mcp`
- **OAuth 控制器中**：`checkOAuthSettings` 中间件检查 `mcp_enabled` 和 `mcp_oauth_enabled`，对浏览器端点返回错误页面，对 API 端点返回 JSON 错误

---

## 六、Items 工具的权限复用

### 6.1 工具定义

Items 工具定义在 [tools/items/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/items/index.ts)，支持四种 action：`create`、`read`、`update`、`delete`。

### 6.2 权限复用机制

Items 工具通过实例化 `ItemsService` 并传入 `accountability` 来复用 Directus 现有的权限体系：

```typescript
const itemsService = new ItemsService(args.collection, {
    schema,
    accountability,
});
```

`ItemsService` 内部会根据 `accountability` 自动应用：
- 集合级别的权限检查（用户是否有权访问该集合）
- 字段级别的权限检查（用户能看到哪些字段）
- 行级别的权限检查（用户能看到哪些记录，通过权限规则过滤）
- 操作级别的权限检查（create/read/update/delete）

### 6.3 前置校验

在调用 `ItemsService` 之前，Items 工具还有两层额外校验：

```typescript
// 校验 1：禁止操作系统集合
if (isSystemCollection(args.collection)) {
    throw new InvalidPayloadError({ reason: 'Cannot provide a core collection' });
}

// 校验 2：集合必须存在于 schema 中（隐含权限检查）
if (args.collection in schema.collections === false) {
    throw new ForbiddenError();
}
```

**注意**：`args.collection in schema.collections === false` 抛出的是 `ForbiddenError` 而非 `NotFoundError`，这是一种安全措施——不向潜在攻击者泄露集合是否存在的信息。

### 6.4 查询 sanitize

Items 工具使用 `buildSanitizedQueryFromArgs` 函数对查询参数进行 sanitize 处理：

```typescript
const sanitizedQuery = await buildSanitizedQueryFromArgs(args, schema, accountability);
```

该函数定义在 [tools/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/utils.ts#L32-L53)，调用了 Directus 的 `sanitizeQuery` 工具，确保：
- 查询参数符合权限要求
- 过滤掉无权访问的字段
- 默认 `fields` 为 `'*'`

### 6.5 Singleton 处理

Items 工具特殊处理了单例集合（singleton collection）：
- create/update 使用 `upsertSingleton` 方法
- read 使用 `readSingleton` 方法
- 单例集合不支持批量操作（数组形式的 data 会报错）

---

## 七、Collections 工具的权限复用

### 7.1 工具定义

Collections 工具定义在 [tools/collections/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/collections/index.ts)，标记为 `admin: true`，同样支持四种 action：`create`、`read`、`update`、`delete`。

### 7.2 Admin 标记的意义

Collections 工具设置了 `admin: true`，意味着：
- 非 admin 用户在工具列表中看不到它
- 非 admin 用户直接调用会被拒绝（`ForbiddenError`）
- 只有 admin 用户可以使用该工具管理集合结构

### 7.3 权限复用机制

Collections 工具通过实例化 `CollectionsService` 来复用权限：

```typescript
const service = new CollectionsService({
    schema,
    accountability,
});
```

`CollectionsService` 是 Directus 的系统级服务，本身就要求 admin 权限才能执行 create/update/delete 操作。配合 MCP 层的 `admin: true` 标记，形成双重保障。

### 7.4 操作映射

| Action | CollectionsService 方法 | 说明 |
|--------|------------------------|------|
| create | `createMany` + `readMany` | 创建后读取返回 |
| read | `readMany` / `readByQuery` | 按 keys 或全部读取 |
| update | `updateBatch` + `readMany` | 批量更新后读取返回 |
| delete | `deleteMany` | 返回被删除的 keys |

与 Items 工具类似，create 和 update 操作后都会重新读取数据返回给调用方。

---

## 八、总结：权限边界防护层次

DirectusMCP 通过多层防护确保不同身份用户的边界：

### 8.1 环境层
- `MCP_ENABLED` 环境变量控制 MCP 路由是否挂载
- `MCP_OAUTH_ENABLED` 环境变量控制 OAuth 路由是否挂载

### 8.2 Settings 层
- `mcp_enabled`：MCP 功能总开关
- `mcp_oauth_enabled`：MCP OAuth 开关
- `mcp_allow_deletes`：删除操作总开关
- `mcp_system_prompt_enabled`：system-prompt 工具开关

### 8.3 身份认证层
- 未认证用户：401 拒绝
- OAuth 用户：仅限 header token、需 `mcp:access` scope、audience 必须匹配
- OAuth token 全局守卫：只能访问 `/mcp` 路径

### 8.4 工具列表层
- 非 admin 用户：看不到 admin 工具
- systemPromptEnabled=false 时：看不到 system-prompt 工具

### 8.5 工具调用层
- 工具存在性二次校验
- admin 工具权限二次校验
- 参数 Schema 校验
- delete 操作全局禁止校验（基于 settings）
- handler 内部的业务权限校验（ItemsService/CollectionsService 自带）

### 8.6 数据权限层
- 各 Service 基于 `accountability` 自动应用 Directus 权限体系
- 行级、字段级、集合级权限全部复用
- 系统集合额外禁止访问
