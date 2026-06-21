# GraphQL fieldName 到 ItemsService/GraphQLService 方法映射分析

## 概述

本文档详细分析 Directus 中 GraphQL fieldName 如何通过命名约定映射到 `ItemsService` 或 `GraphQLService` 的具体方法。整个流程涉及 Schema 定义、Resolver 解析、参数解析、查询构建、服务调用和错误处理等多个环节。

---

## 1. fieldName 命名映射规则

### 1.1 Query（查询）命名模式

在 [read.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/read.ts) 中定义了以下查询字段命名模式：

| fieldName 模式 | 说明 | 对应服务方法 |
|---------------|------|-------------|
| `{collection}` | 普通集合查询（非 singleton 返回数组，singleton 返回单个对象） | `ItemsService.readByQuery()` 或 `readSingleton()` |
| `{collection}_by_id` | 按 ID 查询单条记录 | `ItemsService.readOne()` |
| `{collection}_aggregated` | 聚合查询（count、sum、avg、min、max 等） | `ItemsService.readByQuery()` 配合 `query.aggregate` |
| `{collection}_by_version` | 按版本查询（仅 scope=items 时可用） | `ItemsService.readOne()` 配合 `query.versionRaw=true` |

**代码位置**：[read.ts#L626-L714](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/read.ts#L626-L714)

### 1.2 Mutation（变更）命名模式

在 [write.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/write.ts) 中定义了以下变更字段命名模式：

#### Create 操作
| fieldName 模式 | 说明 | 对应服务方法 |
|---------------|------|-------------|
| `create_{collection}_items` | 批量创建多条记录 | `ItemsService.createMany()` |
| `create_{collection}_item` | 创建单条记录 | `ItemsService.createOne()` |

#### Update 操作
| fieldName 模式 | 说明 | 对应服务方法 |
|---------------|------|-------------|
| `update_{collection}` | 更新 singleton 集合 | `GraphQLService.upsertSingleton()` |
| `update_{collection}_batch` | 批量更新（数据中包含 ID） | `ItemsService.updateBatch()` |
| `update_{collection}_items` | 按 IDs 批量更新 | `ItemsService.updateMany()` |
| `update_{collection}_item` | 按 ID 更新单条 | `ItemsService.updateOne()` |

#### Delete 操作
| fieldName 模式 | 说明 | 对应服务方法 |
|---------------|------|-------------|
| `delete_{collection}_items` | 按 IDs 批量删除 | `ItemsService.deleteMany()` |
| `delete_{collection}_item` | 按 ID 删除单条 | `ItemsService.deleteOne()` |

**代码位置**：[write.ts#L45-L202](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/write.ts#L45-L202)

### 1.3 命名解析流程

#### Query Resolver 中的命名解析

在 [query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/query.ts#L15-L41) 中：

```typescript
let collection = info.fieldName;
if (gql.scope === 'system') collection = `directus_${collection}`;

const isAggregate = collection.endsWith('_aggregated') && collection in gql.schema.collections === false;
if (isAggregate) {
    collection = collection.slice(0, -11); // 去掉 "_aggregated" 后缀
    query = await getAggregateQuery(...);
} else {
    if (collection.endsWith('_by_id') && collection in gql.schema.collections === false) {
        collection = collection.slice(0, -6); // 去掉 "_by_id" 后缀
    }
    query = await getQuery(...);
    if (collection.endsWith('_by_version') && collection in gql.schema.collections === false) {
        collection = collection.slice(0, -11); // 去掉 "_by_version" 后缀
        query.versionRaw = true;
    }
}
```

#### Mutation Resolver 中的命名解析

在 [mutation.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/mutation.ts#L13-L32) 中：

```typescript
const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
let collection = info.fieldName.substring(action.length + 1);
if (gql.scope === 'system') collection = `directus_${collection}`;

const singleton =
    collection.endsWith('_batch') === false &&
    collection.endsWith('_items') === false &&
    collection.endsWith('_item') === false &&
    collection in gql.schema.collections;

const single = collection.endsWith('_items') === false && collection.endsWith('_batch') === false;
const batchUpdate = action === 'update' && collection.endsWith('_batch');

// 去除后缀
if (collection.endsWith('_batch')) collection = collection.slice(0, -6);
if (collection.endsWith('_items')) collection = collection.slice(0, -6);
if (collection.endsWith('_item')) collection = collection.slice(0, -5);
```

---

## 2. Read Resolver 实现详解

### 2.1 核心处理流程

`resolveQuery` 函数是所有查询的统一入口，定义在 [query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/query.ts#L15-L82)：

```typescript
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo): Promise<Partial<Item> | null> {
    // 1. 解析 collection 名称
    // 2. 替换 fragments
    // 3. 解析 args
    // 4. 构建 query（普通查询或聚合查询）
    // 5. 调用 gql.read()
    // 6. 处理 group 聚合结果
}
```

### 2.2 Fragment 替换

**函数**：`replaceFragmentsInSelections`，定义在 [replace-fragments.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/replace-fragments.ts#L8-L47)

**作用**：将 GraphQL 查询中的 FragmentSpread 替换为实际的选择集，支持嵌套 fragments。

**处理逻辑**：
1. 遍历所有 selections
2. 遇到 `FragmentSpread` 时，查找对应的 `FragmentDefinition`
3. 递归替换 fragment 内部的 fragments
4. 将替换后的 selections 包装在 `InlineFragment` 中以保留 `typeCondition`
5. 对于 `Field` 或 `InlineFragment`，递归处理其内部的 `selectionSet`

```typescript
if (selection.kind === 'FragmentSpread') {
    const fragment = fragments[selection.name.value]!;
    const replacedSelections = replaceFragmentsInSelections(fragment.selectionSet.selections, fragments);
    return [
        {
            kind: 'InlineFragment' as const,
            typeCondition: fragment.typeCondition,
            selectionSet: {
                kind: 'SelectionSet' as const,
                selections: replacedSelections ?? [],
            },
        } as SelectionNode,
    ];
}
```

### 2.3 parseArgs - 参数解析

**函数**：`parseArgs`，定义在 [parse-args.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/parse-args.ts#L10-L42)

**作用**：将 GraphQL AST 中的 `ArgumentNode` 转换为 JavaScript 对象结构。GraphQL 原生的 `args` 只包含顶层参数，而此函数递归解析整个参数树。

**支持的 ValueNode 类型**：
- `Variable` - 变量引用
- `ListValue` - 列表
- `ObjectValue` - 对象
- `NullValue` - null
- `StringValue` / `IntValue` / `FloatValue` / `BooleanValue` / `EnumValue` - 基本类型

```typescript
const parse = (node: ValueNode): any => {
    switch (node.kind) {
        case 'Variable': return variableValues[node.name.value];
        case 'ListValue': return node.values.map(parse);
        case 'ObjectValue': return Object.fromEntries(node.fields.map((node) => [node.name.value, parse(node.value)]));
        // ... 其他类型处理
    }
};
```

### 2.4 getQuery - 查询构建

**函数**：`getQuery`，定义在 [parse-query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/parse-query.ts#L16-L197)

**作用**：将 GraphQL AST 的 SelectionSet 转换为 Directus 的 `Query` 对象，包含 `fields`、`filter`、`sort`、`limit`、`offset`、`page`、`search`、`deep`、`alias` 等属性。

**核心处理步骤**：

1. **参数净化**：调用 `sanitizeQuery` 净化输入参数
2. **别名解析**：`parseAliases` 解析字段别名
3. **字段解析**：`parseFields` 递归解析选择集
   - 处理 `InlineFragment`（M2A 关系用 `:` 分隔）
   - 处理嵌套关系（用 `.` 分隔）
   - 处理 `_func` 函数字段（如 `year(date)`、`json(field, path)`）
   - 处理字段级别的参数（filter、sort、limit 等），存入 `query.deep`
4. **函数替换**：`replaceFuncs` 替换过滤器和 deep 中的函数
5. **M2A 替换**：`filterReplaceM2A` 和 `filterReplaceM2ADeep` 处理多态关联
6. **验证**：`validateQuery` 验证查询有效性

**关键字段解析逻辑**：
```typescript
// 嵌套关系字段用 "." 连接
if (parent) {
    current = `${parent}.${current}`;
}

// M2A 关系用 ":" 连接
if (isM2A) {
    current = `${parent}:${selection.typeCondition!.name.value}`;
}

// 函数字段处理
if (current.endsWith('_func')) {
    const rootField = current.slice(0, -5);
    for (const subSelection of selection.selectionSet.selections) {
        children.push(`${subSelection.name!.value}(${rootField})`);
    }
}
```

### 2.5 GraphQLService.read 方法

**函数**：`GraphQLService.read`，定义在 [index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts#L103-L116)

**作用**：根据集合类型和参数，路由到正确的服务方法。

```typescript
async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
    const service = getService(collection, { ... });

    // singleton 集合走 readSingleton
    if (this.schema.collections[collection]!.singleton)
        return await service.readSingleton(query, { stripNonRequested: false });

    // 有 ID 走 readOne
    if (id) return await service.readOne(id, query, { stripNonRequested: false });

    // 否则走 readByQuery
    return await service.readByQuery(query, { stripNonRequested: false });
}
```

---

## 3. Mutation Resolver 实现详解

### 3.1 核心处理流程

`resolveMutation` 函数是所有变更的统一入口，定义在 [mutation.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/mutation.ts#L9-L92)：

```typescript
export async function resolveMutation(
    gql: GraphQLService,
    args: Record<string, any>,
    info: GraphQLResolveInfo,
): Promise<Partial<Item> | boolean | undefined> {
    // 1. 解析 action 和 collection
    // 2. 替换 fragments
    // 3. 构建 query（用于变更后的回读）
    // 4. 路由到具体的服务方法
    // 5. 根据 hasQuery 决定是否回读数据
}
```

### 3.2 Mutation 到服务方法的映射

#### Create 操作

```typescript
if (single) {
    if (action === 'create') {
        const key = await service.createOne(args['data']);
        return hasQuery ? await service.readOne(key, query) : true;
    }
} else {
    if (action === 'create') {
        const keys = await service.createMany(args['data']);
        return hasQuery ? await service.readMany(keys, query) : true;
    }
}
```

#### Update 操作

```typescript
if (singleton && action === 'update') {
    return await gql.upsertSingleton(collection, args['data'], query);
}

if (single) {
    if (action === 'update') {
        const key = await service.updateOne(args['id'], args['data']);
        return hasQuery ? await service.readOne(key, query) : true;
    }
} else {
    if (action === 'update') {
        const keys: PrimaryKey[] = [];
        if (batchUpdate) {
            keys.push(...(await service.updateBatch(args['data'])));
        } else {
            keys.push(...(await service.updateMany(args['ids'], args['data'])));
        }
        return hasQuery ? await service.readMany(keys, query) : true;
    }
}
```

#### Delete 操作

```typescript
if (single) {
    if (action === 'delete') {
        await service.deleteOne(args['id']);
        return { id: args['id'] }; // 总是返回 id
    }
} else {
    if (action === 'delete') {
        const keys = await service.deleteMany(args['ids']);
        return { ids: keys }; // 总是返回 ids
    }
}
```

---

## 4. 特殊处理逻辑

### 4.1 Aggregate Group 处理

**位置**：[query.ts#L73-L79](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/query.ts#L73-L79)

当查询包含 `groupBy` 参数时，聚合查询结果需要特殊处理：

```typescript
if (query.group) {
    const aggregateKeys = Object.keys(query.aggregate ?? {});

    result['map']((payload: Item) => {
        payload['group'] = omit(payload, aggregateKeys);
    });
}
```

**处理逻辑**：
- 聚合查询返回的 payload 顶层包含分组字段和聚合字段
- 将分组字段复制到 `group` 属性中
- 保留顶层的分组字段（因为聚合选择集只包含这些字段）

**示例**：
```
Before: { year_date: 2023, count: { id: 42 } }
After:  { year_date: 2023, count: { id: 42 }, group: { year_date: 2023 } }
```

### 4.2 Singleton Upsert 处理

**函数**：`GraphQLService.upsertSingleton`，定义在 [index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts#L121-L144)

Singleton 集合的更新走特殊流程：

```typescript
async upsertSingleton(
    collection: string,
    body: Record<string, any> | Record<string, any>[],
    query: Query,
): Promise<Partial<Item> | boolean> {
    const service = getService(collection, { ... });

    try {
        await service.upsertSingleton(body);

        // 如果有字段选择，回读数据
        if ((query.fields || []).length > 0) {
            const result = await service.readSingleton(query);
            return result;
        }

        // 无字段选择返回 true
        return true;
    } catch (err: any) {
        throw formatError(err);
    }
}
```

### 4.3 Aggregate Query 构建

**函数**：`getAggregateQuery`，定义在 [aggregate-query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/aggregate-query.ts#L11-L56)

聚合查询的查询构建与普通查询不同，它解析选择集中的聚合函数：

```typescript
query.aggregate = {};

for (let aggregationGroup of selections) {
    if ((aggregationGroup.kind === 'Field') !== true) continue;
    if (aggregationGroup.name.value.startsWith('__')) continue;
    if (aggregationGroup.name.value === 'group') continue; // 跳过 group 字段

    const aggregateProperty = aggregationGroup.name.value as keyof Aggregate;

    query.aggregate[aggregateProperty] =
        aggregationGroup.selectionSet?.selections
            .filter((selectionNode) => !(selectionNode as FieldNode)?.name.value.startsWith('__'))
            .map((selectionNode) => {
                selectionNode = selectionNode as FieldNode;
                return selectionNode.name.value;
            }) ?? [];
}
```

**支持的聚合函数**（定义在 [read.ts#L563-L619](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/read.ts#L563-L619)）：
- `group` - 分组字段（特殊处理）
- `countAll` - 统计总数
- `count` / `countDistinct` - 按字段统计
- `avg` / `avgDistinct` - 平均值（仅数字字段）
- `sum` / `sumDistinct` - 求和（仅数字字段）
- `min` / `max` - 最小/最大值（仅数字字段）

---

## 5. 返回值处理规则

### 5.1 无 Selection 返回 true

**位置**：[mutation.ts#L44-L80](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/mutation.ts#L44-L80)

对于 create 和 update 操作，如果查询没有选择任何字段（`hasQuery = false`），则直接返回 `true`：

```typescript
const hasQuery = (query.fields || []).length > 0;

if (single) {
    if (action === 'create') {
        const key = await service.createOne(args['data']);
        return hasQuery ? await service.readOne(key, query) : true;
    }
    // ... update 同理
}
```

**适用场景**：
- `create_{collection}_item` / `create_{collection}_items`
- `update_{collection}` / `update_{collection}_item` / `update_{collection}_items` / `update_{collection}_batch`

### 5.2 Delete 操作返回 id/ids

**位置**：[mutation.ts#L58-L85](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/mutation.ts#L58-L85)

Delete 操作无论是否有 selection，都返回固定结构：

```typescript
if (single) {
    if (action === 'delete') {
        await service.deleteOne(args['id']);
        return { id: args['id'] }; // 返回 { id: ... }
    }
} else {
    if (action === 'delete') {
        const keys = await service.deleteMany(args['ids']);
        return { ids: keys }; // 返回 { ids: [...] }
    }
}
```

**Schema 定义**（[write.ts#L169-L181](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/write.ts#L169-L181)）：

```typescript
DeleteCollectionTypes['many'] = schemaComposer.createObjectTC({
    name: `delete_many`,
    fields: {
        ids: new GraphQLNonNull(new GraphQLList(GraphQLID)),
    },
});

DeleteCollectionTypes['one'] = schemaComposer.createObjectTC({
    name: `delete_one`,
    fields: {
        id: new GraphQLNonNull(GraphQLID),
    },
});
```

---

## 6. 错误处理机制

### 6.1 formatError - Directus 错误转 GraphQL 错误

**函数**：`formatError`，定义在 [format.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/errors/format.ts#L8-L16)

**作用**：将 Directus 的 `DirectusError` 转换为 GraphQL 的 `GraphQLError`，确保错误能被 GraphQL 正确返回。

```typescript
export function formatError(error: DirectusError | DirectusError[]): GraphQLError {
    if (Array.isArray(error)) {
        set(error[0]!, 'extensions.code', error[0]!.code);
        return new GraphQLError(error[0]!.message, undefined, undefined, undefined, undefined, error[0]);
    }

    set(error, 'extensions.code', error.code);
    return new GraphQLError(error.message, undefined, undefined, undefined, undefined, error);
}
```

**使用位置**：
- [mutation.ts#L90](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/mutation.ts#L90) - mutation resolver 的 catch 块
- [index.ts#L142](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts#L142) - upsertSingleton 的 catch 块

### 6.2 processError - 执行后错误处理

**函数**：`processError`，定义在 [process-error.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/process-error.ts#L6-L53)

**作用**：在 GraphQL 执行完成后，统一处理返回的错误，根据用户权限控制错误信息的暴露程度。

```typescript
const processError = (
    accountability: Accountability | null,
    error: Readonly<GraphQLError & { originalError: GraphQLError | DirectusError | Error | undefined }>,
): GraphQLFormattedError => {
    logger.error(error);

    let originalError = error.originalError;
    if (originalError && 'originalError' in originalError) {
        originalError = originalError.originalError;
    }

    if (isDirectusError(originalError)) {
        // Directus 错误：返回完整信息
        return {
            message: originalError.message,
            extensions: {
                code: originalError.code,
                ...(originalError.extensions ?? {}),
            },
            ...(error.locations && { locations: error.locations }),
            ...(error.path && { path: error.path }),
        };
    } else {
        if (accountability?.admin === true) {
            // 管理员：返回详细错误信息
            return {
                message: error.message,
                extensions: { code: 'INTERNAL_SERVER_ERROR' },
                ...(error.locations && { locations: error.locations }),
                ...(error.path && { path: error.path }),
            };
        } else {
            // 普通用户：隐藏错误详情
            return {
                message: 'An unexpected error occurred.',
                extensions: { code: 'INTERNAL_SERVER_ERROR' },
            };
        }
    }
};
```

**使用位置**：[index.ts#L82](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts#L82)

### 6.3 GraphQLValidationError - 验证错误

**定义**：[errors/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/errors/index.ts#L2)

在 Schema 验证阶段抛出，用于 GraphQL 文档验证错误。

### 6.4 GraphQLExecutionError - 执行错误

**定义**：[errors/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/errors/index.ts#L1)

在 GraphQL 执行阶段（execute 调用）抛出的错误。

### 6.5 错误处理流程图

```
GraphQL 请求
    ↓
validate() - Schema 验证
    ↓ （验证失败）
throw GraphQLValidationError
    ↓ （验证通过）
execute() - 执行 GraphQL
    ↓ （执行中异常）
catch → throw GraphQLExecutionError
    ↓ （Resolver 中异常）
formatError() → 转为 GraphQLError
    ↓ （执行完成后）
result.errors?.map(processError())
    ↓
返回格式化后的错误
```

---

## 7. 整体架构图

```
                    GraphQL Request
                         │
                         ▼
              ┌─────────────────────┐
              │  GraphQLService     │
              │  .execute()         │
              └─────────┬───────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
  validate()     generateSchema()   execute()
         │              │              │
         │              │              └─────────┐
         │              │                        │
         │              ▼                        ▼
         │    ┌──────────────────┐      Resolver Functions
         │    │ getReadableTypes │      ┌─────────────────┐
         │    │ getWritableTypes │      │  resolveQuery   │
         │    └─────────┬────────┘      │ resolveMutation │
         │              │               └─────────┬───────┘
         │              │                         │
         ▼              ▼                         ▼
  addPathToValidationError              ┌─────────────────────┐
         │                              │ replaceFragmentsInSelections
         ▼                              │ parseArgs           │
 GraphQLValidationError                 │ getQuery / getAggregateQuery
                                        └─────────┬───────────┘
                                                  │
                            ┌─────────────────────┼─────────────────────┐
                            ▼                     ▼                     ▼
                  GraphQLService.read    GraphQLService.upsertSingleton   ItemsService
                  ┌─────────────────┐   ┌─────────────────────────┐   ┌─────────────┐
                  │ readSingleton   │   │ service.upsertSingleton │   │ createOne   │
                  │ readOne         │   │ service.readSingleton   │   │ createMany  │
                  │ readByQuery     │   └─────────────────────────┘   │ updateOne   │
                  └─────────────────┘                                 │ updateMany  │
                                                                      │ updateBatch │
                                                                      │ deleteOne   │
                                                                      │ deleteMany  │
                                                                      └─────────────┘
                                                                           │
                                                                           ▼
                                                                  formatError() (异常时)
                                                                           │
                                                                           ▼
                                                                  processError() (返回前)
                                                                           │
                                                                           ▼
                                                                   Formatted Response
```

---

## 8. 关键文件索引

| 文件 | 说明 |
|------|------|
| [index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/index.ts) | GraphQLService 核心类，execute/read/upsertSingleton |
| [schema/read.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/read.ts) | Query 字段定义和命名映射 |
| [schema/write.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/write.ts) | Mutation 字段定义和命名映射 |
| [resolvers/query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/query.ts) | Read Resolver 实现 |
| [resolvers/mutation.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/mutation.ts) | Mutation Resolver 实现 |
| [schema/parse-args.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/parse-args.ts) | GraphQL 参数解析 |
| [schema/parse-query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/schema/parse-query.ts) | Directus Query 构建 |
| [utils/replace-fragments.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/replace-fragments.ts) | Fragment 替换 |
| [utils/aggregate-query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/aggregate-query.ts) | 聚合查询构建 |
| [errors/format.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/errors/format.ts) | DirectusError 转 GraphQLError |
| [utils/process-error.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/utils/process-error.ts) | 执行后错误处理 |
| [services/items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts) | ItemsService 核心实现 |
