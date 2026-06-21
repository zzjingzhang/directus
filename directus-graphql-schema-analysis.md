# Directus GraphQL Schema 生成、缓存与裁剪机制分析

## 目录

- [1. 整体架构概览](#1-整体架构概览)
- [2. 请求解析：GET / POST 与 Mutation 拒绝](#2-请求解析get--post-与-mutation-拒绝)
- [3. GRAPHQL_INTROSPECTION 开关](#3-graphql_introspection-开关)
- [4. Schema 缓存机制](#4-schema-缓存机制)
  - [4.1 缓存 Key 结构](#41-缓存-key-结构)
  - [4.2 缓存实现与失效](#42-缓存实现与失效)
- [5. 并发生成 Semaphore](#5-并发生成-semaphore)
- [6. Schema 裁剪策略](#6-schema-裁剪策略)
  - [6.1 Admin 或无 Accountability：全量 Schema](#61-admin-或无-accountability全量-schema)
  - [6.2 普通用户：按 Action 裁剪 Schema](#62-普通用户按-action-裁剪-schema)
- [7. System Collection 过滤](#7-system-collection-过滤)
- [8. SDL 输出与缓存](#8-sdl-输出与缓存)
- [9. 生成流程总览](#9-生成流程总览)
- [10. 关键文件索引](#10-关键文件索引)

---

## 1. 整体架构概览

Directus 的 GraphQL 服务分为两个独立的 scope：

- **`/graphql/system`** — 系统集合（`directus_*`）的 GraphQL 入口
- **`/graphql/`** — 用户业务集合（items）的 GraphQL 入口

两个入口共享同一套 Schema 生成、缓存与裁剪逻辑，只是通过 `scope` 参数区分集合范围。

核心入口文件：
- 控制器：[graphql.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/graphql.ts)
- 服务：[index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts)
- Schema 生成：[schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts)
- 缓存：[schema-cache.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema-cache.ts)

---

## 2. 请求解析：GET / POST 与 Mutation 拒绝

GraphQL 请求通过 `parseGraphQL` 中间件解析，定义于 [middleware/graphql.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/graphql.ts)。

### 2.1 GET 请求解析

GET 请求从 URL query 参数中提取：

- `query` — GraphQL 查询字符串
- `variables` — JSON 格式的变量（需 parseJSON，失败则抛 `InvalidQueryError`）
- `operationName` — 操作名

```typescript
if (req.method === 'GET') {
    query = (req.query['query'] as string | undefined) || null;
    // ... variables 解析
    operationName = (req.query['operationName'] as string | undefined) || null;
}
```

### 2.2 POST 请求解析

POST 请求从请求体（`req.body`）中提取相同字段。

### 2.3 GET Mutation 拒绝

出于安全考虑（防止 CSRF、遵循 HTTP 方法语义），**GET 请求只能执行 query 操作**，mutation / subscription 操作会被拒绝：

```typescript
const operationAST = getOperationAST(document, operationName);

// Mutations can't happen through GET requests
if (req.method === 'GET' && operationAST?.operation !== 'query') {
    throw new MethodNotAllowedError({
        allowed: ['POST'],
        current: 'GET',
    });
}
```

此外，mutation 操作还会禁用响应缓存：

```typescript
if (operationAST?.operation === 'mutation') {
    res.locals['cache'] = false;
}
```

### 2.4 Token 限制

查询解析时会受到 `GRAPHQL_QUERY_TOKEN_LIMIT`（默认 5000）的限制，防止过大的查询攻击：

```typescript
document = parse(new Source(query), {
    maxTokens: Number(env['GRAPHQL_QUERY_TOKEN_LIMIT']),
});
```

---

## 3. GRAPHQL_INTROSPECTION 开关

环境变量 `GRAPHQL_INTROSPECTION`（默认 `true`）控制 GraphQL 的内省（introspection）功能。

该开关在**两个层面**生效：

### 3.1 验证规则层面

在 [services/graphql/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts) 中，若关闭内省，则添加 `NoSchemaIntrospectionCustomRule` 到验证规则中：

```typescript
const validationRules = Array.from(specifiedRules);

if (env['GRAPHQL_INTROSPECTION'] === false) {
    validationRules.push(NoSchemaIntrospectionCustomRule);
}
```

这会使得执行 `__schema` / `__type` 等内省查询时抛出验证错误。

### 3.2 System Resolver 层面

在 [resolvers/system.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/system.ts) 中，`server_specs_graphql` 查询字段（用于通过 GraphQL 查询获取 SDL）也受此开关控制：

```typescript
if (env['GRAPHQL_INTROSPECTION'] !== false) {
    schemaComposer.Query.addFields({
        server_specs_graphql: {
            type: GraphQLString,
            args: { scope: /* items | system */ },
            resolve: /* 返回对应 scope 的 SDL */,
        },
    });
}
```

### 3.3 默认值

| 变量 | 默认值 | 定义位置 |
|------|--------|----------|
| `GRAPHQL_INTROSPECTION` | `true` | [defaults.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L172-L172) |

---

## 4. Schema 缓存机制

### 4.1 缓存 Key 结构

缓存 Key 由四部分组成，定义于 [schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts#L61-L61)：

```
${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}
```

各部分含义：

| 组成部分 | 说明 | 可能值 |
|----------|------|--------|
| `scope` | GraphQL 作用域 | `items` 或 `system` |
| `type` | 返回类型 | `schema`（GraphQLSchema 对象）或 `sdl`（字符串） |
| `role` | 当前用户角色 ID | 字符串 / `undefined`（无 accountability） |
| `user` | 当前用户 ID | 字符串 / `undefined`（无 accountability） |

这种设计意味着：
- 不同 scope 的 schema 相互独立缓存
- `schema` 和 `sdl` 形式的 schema 分开缓存
- 不同用户 / 角色拥有不同的缓存条目（因为权限不同，裁剪后的 schema 不同）

### 4.2 缓存实现与失效

缓存使用 `mnemonist` 的 `LRUMap` 实现，定义于 [schema-cache.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema-cache.ts)：

```typescript
export const cache = new LRUMap<string, GraphQLSchema | string>(
    Number(env['GRAPHQL_SCHEMA_CACHE_CAPACITY'] ?? 100)
);
```

缓存容量由 `GRAPHQL_SCHEMA_CACHE_CAPACITY` 控制，默认 100 条。

**缓存失效机制**：监听 `schemaChanged` 事件，当数据库 schema 发生变化时清空整个缓存：

```typescript
bus.subscribe('schemaChanged', () => {
    cache.clear();
});
```

---

## 5. 并发生成 Semaphore

为避免在缓存未命中时大量并发请求同时生成 schema 造成资源耗尽，Directus 使用 **Semaphore（信号量）** 控制并发生成数量，定义于 [schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts#L52-L52)：

```typescript
const semaphore = new Semaphore(
    (env['GRAPHQL_SCHEMA_GENERATION_MAX_CONCURRENT'] as number) ?? 5
);
```

默认最大并发数为 `5`，可通过 `GRAPHQL_SCHEMA_GENERATION_MAX_CONCURRENT` 调整。

### 5.1 双重检查模式

生成 schema 采用**双重检查锁定**模式（在 semaphore 前后各检查一次缓存），避免不必要的重复生成：

```typescript
async function generateSchema(gql, type = 'schema') {
    const key = `${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}`;

    const cachedSchema = cache.get(key);
    if (cachedSchema) return cachedSchema; // 第一次检查

    return semaphore.runExclusive(async () => {
        const cachedSchema = cache.get(key);
        if (cachedSchema) return cachedSchema; // 第二次检查（获取锁后）

        // ... 实际生成逻辑 ...

        cache.set(key, gqlSchema);
        return gqlSchema;
    });
}
```

---

## 6. Schema 裁剪策略

Schema 裁剪是指根据用户权限，从完整 schema 中移除无权限访问的集合、字段和关系。

### 6.1 Admin 或无 Accountability：全量 Schema

当满足以下任一条件时，直接使用**完整的 sanitized schema**（不进行权限裁剪），四种 action 共用同一份 schema：

- `!gql.accountability` — 无 accountability 信息（如系统内部调用）
- `gql.accountability.admin === true` — 管理员用户

```typescript
if (!gql.accountability || gql.accountability.admin) {
    schema = {
        read: sanitizedSchema,
        create: sanitizedSchema,
        update: sanitizedSchema,
        delete: sanitizedSchema,
    };
}
```

这里的 `sanitizedSchema` 是经过 [sanitizeGraphqlSchema](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/sanitize-gql-schema.ts) 清洗后的 schema，主要过滤：
- 以 `__` 开头的集合名（GraphQL 内省保留）
- 不符合 GraphQL 命名规范的集合名（`/^[_A-Za-z][_0-9A-Za-z]*$/`）
- GraphQL 保留关键字（`Query`、`Mutation`、`String`、`Int` 等）
- 指向不存在集合的关系

### 6.2 普通用户：按 Action 裁剪 Schema

对于普通用户，会针对 `read`、`create`、`update`、`delete` 四种 action 分别独立裁剪 schema：

```typescript
schema = {
    read: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap(
        { accountability: gql.accountability, action: 'read' },
        { schema: gql.schema, knex: gql.knex },
    )),
    create: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap(
        { accountability, action: 'create' },
        { schema, knex },
    )),
    update: /* 同理 */,
    delete: /* 同理 */,
};
```

#### 6.2.1 `fetchAllowedFieldMap` — 获取允许的字段映射

定义于 [fetch-allowed-field-map.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/fetch-allowed-field-map/fetch-allowed-field-map.ts)。

返回类型：`Record<string, string[]>`，即 `集合名 → 允许的字段数组`。

对于 admin 用户，直接返回所有集合的所有字段。
对于普通用户，通过 `fetchPolicies` 和 `fetchPermissions` 获取权限策略，然后汇总每个集合的允许字段（去重）。

#### 6.2.2 `reduceSchema` — 根据字段映射裁剪 Schema

定义于 [reduce-schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/reduce-schema.ts)。

裁剪逻辑包括：

1. **集合级别**：如果集合不在 fieldMap 中，整个集合被移除
2. **字段级别**：如果字段不在集合的允许字段列表中（且不是 `*`），该字段被移除
3. **关系级别**：如果关系涉及的集合或字段不被允许，该关系被移除

除了 fieldMap 裁剪外，还会额外计算 `inconsistentFields`（不一致字段），用于处理字段权限不一致的情况。

---

## 7. System Collection 过滤

虽然权限裁剪已经过滤了大部分内容，但还有一层 **scope 级别的集合过滤**，由 `scopeFilter` 函数实现，定义于 [schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts#L184-L193)：

```typescript
const scopeFilter = (collection) => {
    if (gql.scope === 'items' && isSystemCollection(collection.collection)) 
        return false;

    if (gql.scope === 'system') {
        if (isSystemCollection(collection.collection) === false) return false;
        if (SYSTEM_DENY_LIST.includes(collection.collection)) return false;
    }

    return true;
};
```

### 7.1 过滤规则

| Scope | 规则 |
|-------|------|
| `items` | 只包含非系统集合（排除所有 `directus_*`） |
| `system` | 只包含系统集合，且排除 `SYSTEM_DENY_LIST` 中的集合 |

### 7.2 SYSTEM_DENY_LIST

定义于 [schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts#L40-L47)：

```typescript
export const SYSTEM_DENY_LIST = [
    'directus_collections',
    'directus_fields',
    'directus_relations',
    'directus_migrations',
    'directus_sessions',
    'directus_extensions',
];
```

这些集合在 system scope 中**不作为普通集合暴露**，而是通过自定义 resolver（如 `collections`、`fields`、`relations` 查询字段）提供更丰富的 API，定义于 [resolvers/system.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/system.ts)。

### 7.3 READ_ONLY

定义于 [schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts#L49-L49)：

```typescript
export const READ_ONLY = ['directus_activity', 'directus_revisions'];
```

这些系统集合是只读的，在 mutation（create/update/delete）中会被过滤掉。

### 7.4 集合命名方式

在 system scope 中，集合名称会去掉 `directus_` 前缀：

```typescript
const collectionName = gql.scope === 'items' 
    ? collection.collection 
    : collection.collection.substring(9); // 去掉 "directus_"
```

例如：`directus_users` → `users`（仅在 system scope 中）。

---

## 8. SDL 输出与缓存

`generateSchema` 函数支持两种输出类型：

- `'schema'`（默认）：返回 `GraphQLSchema` 对象
- `'sdl'`：返回 SDL（Schema Definition Language）字符串

### 8.1 SDL 生成

```typescript
if (type === 'sdl') {
    const sdl = schemaComposer.toSDL();
    cache.set(key, sdl);
    return sdl;
}

const gqlSchema = schemaComposer.buildSchema();
cache.set(key, gqlSchema);
return gqlSchema;
```

### 8.2 SDL 缓存特点

1. **独立缓存**：因为缓存 key 中包含 `type`，所以 `sdl` 和 `schema` 是分开缓存的，互不影响
2. **相同的生成流程**：SDL 输出同样经过完整的权限裁剪、scope 过滤、system resolver 注入等流程
3. **主要用途**：`server_specs_graphql` 查询（内省开关控制）和外部工具获取 schema 定义

---

## 9. 生成流程总览

```
请求到达
  │
  ▼
parseGraphQL 中间件
  ├── GET: 从 query 解析 query/variables/operationName
  ├── POST: 从 body 解析
  ├── 解析 document (受 maxTokens 限制)
  ├── 检查: GET 请求不允许 mutation
  └── mutation 操作禁用响应缓存
  │
  ▼
GraphQLService.execute()
  │
  ▼
getSchema() → generateSchema()
  │
  ├── 计算缓存 key: scope_type_role_user
  ├── 检查 LRU 缓存 → 命中则直接返回
  │
  └── 未命中 → Semaphore 排队
        │
        ├── 再次检查缓存（双重检查）
        │
        ├── sanitizeGraphqlSchema (过滤非法名称、保留字)
        │
        ├── 判断权限级别
        │     ├── admin 或 无 accountability → 全量 schema（4 个 action 共用）
        │     └── 普通用户 → 按 read/create/update/delete 分别 fetchAllowedFieldMap + reduceSchema
        │
        ├── 计算 inconsistentFields
        │
        ├── getReadableTypes (构建可读类型)
        ├── getWritableTypes (构建可写类型)
        │
        ├── system scope → injectSystemResolvers (注入系统查询和 mutation)
        │
        ├── scopeFilter 过滤集合（items 排除系统集合 / system 排除非系统 + DENY_LIST）
        │
        ├── 构建 Query 字段 (集合查询、by_id、aggregated、by_version)
        ├── 构建 Mutation 字段 (create / update / delete，排除 READ_ONLY)
        │
        ├── type === 'sdl' → 生成 SDL 字符串并存入缓存
        └── type === 'schema' → buildSchema 并存入缓存
  │
  ▼
validate (验证查询，含内省规则)
  │
  ▼
execute (执行查询)
  │
  ▼
返回结果
```

---

## 10. 关键文件索引

| 文件 | 作用 |
|------|------|
| [api/src/controllers/graphql.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/graphql.ts) | GraphQL 路由控制器，定义 /system 和 / 两个入口 |
| [api/src/middleware/graphql.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/graphql.ts) | 请求解析中间件（GET/POST 解析、mutation 拒绝、token 限制） |
| [api/src/services/graphql/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts) | GraphQLService 主类（execute、getSchema、内省验证规则） |
| [api/src/services/graphql/schema/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/index.ts) | generateSchema 核心函数（缓存 key、semaphore、裁剪、scope 过滤、SDL 输出） |
| [api/src/services/graphql/schema-cache.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema-cache.ts) | LRU 缓存实现与 schemaChanged 失效机制 |
| [api/src/services/graphql/utils/sanitize-gql-schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/sanitize-gql-schema.ts) | Schema 清洗（过滤非法名称、保留字、无效关系） |
| [api/src/services/graphql/resolvers/system.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/system.ts) | System scope 自定义 resolver（server_info、collections、fields 等） |
| [api/src/utils/reduce-schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/reduce-schema.ts) | 根据权限字段映射裁剪 schema |
| [api/src/permissions/modules/fetch-allowed-field-map/fetch-allowed-field-map.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/fetch-allowed-field-map/fetch-allowed-field-map.ts) | 获取指定 action 下允许的字段映射 |
| [packages/env/src/constants/defaults.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts) | 环境变量默认值（GRAPHQL_INTROSPECTION 等） |
