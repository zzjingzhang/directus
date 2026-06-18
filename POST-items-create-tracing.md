# POST /items/:collection 创建记录源码行为追踪

> 追踪 Directus 中 `POST /items/:collection` 创建单条和批量记录时的完整源码流程。

---

## 目录

1. [请求路由与中间件链](#1-请求路由与中间件链)
2. [Controller 层检查](#2-controller-层检查)
3. [ItemsService.createOne 完整流程](#3-itemsservicecreateone-完整流程)
4. [ItemsService.createMany 完整流程](#4-itemsservicecreatemany-完整流程)
5. [PayloadService 预处理与类型转换](#5-payloadservice-预处理与类型转换)
6. [关系字段处理](#6-关系字段处理)
7. [Activity 与 Revision 生成](#7-activity-与-revision-生成)
8. [Nested Action 事件延迟发射](#8-nested-action-事件延迟发射)
9. [Cache 清理](#9-cache-清理)
10. [批量自增主键 Reset 时机](#10-批量自增主键-reset-时机)
11. [子关系写入失败时的回滚与事件分析](#11-子关系写入失败时的回滚与事件分析)

---

## 1. 请求路由与中间件链

### 路由注册

在 [app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L368) 中：

```typescript
app.use('/items', itemsRouter);
```

### 全局中间件顺序

请求到达 controller 之前经过的全局中间件（按顺序）：

1. **`authenticate`** — 在 [app.ts#L328](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L328) 注册，解析 `req.accountability`
2. **`sanitizeQuery`** — 在 [app.ts#L337](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L337) 注册，将 `req.query` 清洗后存入 `req.sanitizedQuery`

### Items Router 专用中间件

在 [items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/items.ts) 中：

```typescript
router.use(checkIsLocked('items'));       // L16 — 全路由级别
router.post('/:collection', collectionExists, handler, respond);  // L18-L62
```

| 中间件 | 位置 | 作用 |
|--------|------|------|
| `checkIsLocked('items')` | [is-locked.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/is-locked.ts#L10-L19) | 检查许可证锁定状态 |
| `collectionExists` | [collection-exists.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/collection-exists.ts#L10-L30) | 验证集合存在，设置 `req.collection` 和 `req.singleton` |
| `respond` | [respond.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/respond.ts#L16-L127) | 序列化响应、处理缓存头 |

---

## 2. Controller 层检查

### 2.1 许可证锁定检查 (`checkIsLocked`)

- 源码：[is-locked.ts#L10-L19](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/is-locked.ts#L10-L19)
- 逻辑：调用 `getLicenseManager().isLocked()`，若锁定则抛出 `ResourceRestrictedError`
- 时机：**所有 items 路由的最早关卡**，在路由匹配之前就已执行

### 2.2 集合存在性检查 (`collectionExists`)

- 源码：[collection-exists.ts#L10-L30](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/collection-exists.ts#L10-L30)
- 逻辑：
  1. 验证 `req.params.collection` 存在于 `req.schema.collections` 中，不存在则抛出 `createCollectionForbiddenError`
  2. 将 `req.params.collection` 赋值给 `req.collection`
  3. 检查是否为系统集合（`systemCollectionRows`），若是则使用其 `singleton` 标记；否则从 schema 中读取

### 2.3 系统集合写入拦截

- 源码：[items.ts#L22](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/items.ts#L22)
- 逻辑：

```typescript
if (isSystemCollection(req.params['collection']!)) throw new ForbiddenError();
```

`isSystemCollection` 定义在 [packages/system-data/src/collections/index.ts#L17-L20](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/system-data/src/collections/index.ts#L17-L20)：

```typescript
export function isSystemCollection(collection: string): boolean {
    if (!collection.startsWith('directus_')) return false;
    return systemCollectionNames.includes(collection);
}
```

**关键点**：以 `directus_` 开头但不在 `systemCollectionNames` 列表中的自定义集合不会被拦截；而 `directus_users`、`directus_files` 等系统集合的写入需走各自的专用 Service（通过 [get-service.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-service.ts#L34-L88) 分发）。

### 2.4 Singleton 拦截

- 源码：[items.ts#L24-L26](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/items.ts#L24-L26)

```typescript
if (req.singleton) {
    throw new RouteNotFoundError({ path: req.path });
}
```

Singleton 集合不支持 `POST` 创建，只能通过 `PATCH` 调用 `upsertSingleton`。

---

## 3. ItemsService.createOne 完整流程

### 3.1 入口与 MutationTracker 初始化

- 源码：[items.ts#L127-L132](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L127-L132)

```typescript
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    if (!opts.mutationTracker) opts.mutationTracker = this.createMutationTracker();
    if (!opts.bypassLimits) {
        opts.mutationTracker.trackMutations(1);
    }
    // ...
}
```

`createMutationTracker` 在 [items.ts#L90-L105](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L90-L105)：维护一个计数器，每次 `trackMutations(n)` 累加，超过 `MAX_BATCH_MUTATION` 环境变量则抛出 `InvalidPayloadError`。

**类型定义**：[items.ts (types)](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/items.ts#L22-L25)

### 3.2 事务边界

- 源码：[items.ts#L156](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L156)

```typescript
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
    // 所有数据库操作都在 trx 内
});
```

`transaction` 工具函数定义在 [transaction.ts#L14-L53](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/transaction.ts#L14-L53)：

- 若当前 knex 已在事务中（`knex.isTransaction === true`）且未强制新建，则复用现有事务
- 否则创建新事务 `knex.transaction(handler)`
- 事务失败时，针对特定数据库错误（CockroachDB 40001、SQLite SQLITE_BUSY、MySQL 死锁、PostgreSQL 40P01 等）会自动重试最多 3 次，指数退避

**事务范围包含**：filter hook → processPayload → processM2O → processA2O → INSERT → processO2M → user integrity check → activity/revision → auto-increment reset

### 3.3 Filter Hook (emitFilter)

- 源码：[items.ts#L161-L177](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L161-L177)

```typescript
const payloadAfterHooks = opts.emitEvents !== false
    ? await emitter.emitFilter(
        this.eventScope === 'items'
            ? ['items.create', `${this.collection}.items.create`]
            : `${this.eventScope}.create`,
        payload,
        { collection: this.collection },
        { database: trx, schema: this.schema, accountability: this.accountability },
    )
    : payload;
```

- 事件名：普通集合发射 `items.create` 和 `<collection>.items.create` 两个事件；系统集合发射 `<scope>.create`
- `emitFilter` 是**同步阻塞**的，会等待所有监听器依次执行完毕，返回修改后的 payload
- 此时在事务内，hook 可使用 `trx` 进行数据库操作

### 3.4 processPayload（权限校验 + 预设值注入）

- 源码：[items.ts#L179-L193](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L179-L193)
- 实现：[process-payload.ts#L29-L131](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-payload/process-payload.ts#L29-L131)

核心步骤：

1. **权限校验**（非 admin 用户）：
   - `fetchPolicies` → `fetchPermissions` 获取当前用户对目标集合的 `create` 权限
   - 若无权限则抛出 `createCollectionForbiddenError`
   - 检查 payload 中字段是否在允许的字段列表中，不允许则抛出 `createFieldsForbiddenError`
   - 将权限的 `validation` 规则收集

2. **字段级验证规则**：
   - 对非 nullable、无默认值的字段生成 `_submitted` + `_nnull` 验证
   - 对有 `validation` 属性的字段收集其验证过滤器
   - 所有验证规则合并后用 `validatePayload({ _and: rules }, payloadWithPresets)` 执行
   - 验证失败抛出 `FailedValidationError`

3. **预设值注入**：
   ```typescript
   const presets = (permissions ?? []).map((permission) => permission.presets);
   const payloadWithPresets = assign({}, ...presets, options.payload);
   ```
   - 先应用权限预设，再用实际 payload 覆盖（payload 优先级更高）

### 3.5 preMutationError 检查

- 源码：[items.ts#L195-L197](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L195-L197)

```typescript
if (opts.preMutationError) {
    throw opts.preMutationError;
}
```

允许调用方在事务开始后、实际变更前注入一个延迟抛出的错误，使整个事务回滚。

---

## 4. ItemsService.createMany 完整流程

### 4.1 入口

- 源码：[items.ts#L444-L516](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L444-L516)

```typescript
async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    if (!opts.mutationTracker) opts.mutationTracker = this.createMutationTracker();
    // ...
}
```

### 4.2 事务结构

```typescript
const { primaryKeys, nestedActionEvents } = await transaction(this.knex, async (knex) => {
    const previousSeatCount = await captureSeatCount(knex, opts.userIntegrityCheckFlags);
    const service = this.fork({ knex });
    // ...
    for (const [index, payload] of data.entries()) {
        // 循环调用 createOne
        const primaryKey = await service.createOne(payload, { ... });
        primaryKeys.push(primaryKey);
    }
    // User integrity check
    return { primaryKeys, nestedActionEvents };
});
```

**关键设计**：

1. **单一外层事务**：`createMany` 开启一个事务，所有 `createOne` 调用复用此事务
2. **fork 创建子服务**：通过 `this.fork({ knex })` 创建新 ItemsService 实例，共享事务连接
3. **事件收集**：每个 `createOne` 的 `bypassEmitAction` 回调将 action 事件收集到 `nestedActionEvents` 数组，而不是立即发射
4. **缓存延迟清理**：每个 `createOne` 传入 `autoPurgeCache: false`，避免逐条清理缓存

### 4.3 事务后操作

```typescript
// 1. 发射所有嵌套 action 事件
for (const nestedActionEvent of nestedActionEvents) {
    emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
}

// 2. 统一清理缓存
if (shouldClearCache(this.cache, opts, this.collection)) {
    await this.cache.clear();
}
```

---

## 5. PayloadService 预处理与类型转换

PayloadService 在 [payload.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts) 中定义。

### 5.1 processValues 调用时机

在 `createOne` 事务内部，关系处理之后：

```typescript
const payloadWithoutAliases = pick(payloadWithA2O, without(fields, ...aliases));
const payloadWithTypeCasting = await payloadService.processValues('create', payloadWithoutAliases);
```

- 源码：[items.ts#L225-L226](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L225-L226)
- **先剔除 alias 字段**（alias 字段不对应实际数据库列）
- 然后调用 `processValues('create', ...)` 执行类型转换

### 5.2 processValues 执行流程

- 源码：[payload.ts#L214-L283](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L214-L283)

1. **special 字段转换** — 遍历 schema 中带 `special` 标记的字段，依次调用对应 transformer

2. **Geometry 处理** — `processGeometries`：write 时将 GeoJSON 转为数据库原生格式（如 PostGIS 的 `ST_GeomFromGeoJSON`）

3. **日期处理** — `processDates`：
   - `date` 类型：write 时 `parseISO` 验证
   - `dateTime` 类型：write 时 `parseISO` 验证
   - `timestamp` 类型：write 时 `helpers.date.writeTimestamp` 转换

4. **对象/数组序列化** — create/update 时，将 payload 中的对象和数组 `JSON.stringify`，除非是 `Knex.Raw` 实例

### 5.3 Transformers 一览

| Special | create 行为 | 源码行 |
|---------|------------|--------|
| `hash` | 调用 `generateHash` 哈希值 | [payload.ts#L76-L84](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L76-L84) |
| `uuid` | 无值时生成 `randomUUID()` | [payload.ts#L85-L91](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L85-L91) |
| `cast-boolean` | create 时不转换 | [payload.ts#L92-L104](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L92-L104) |
| `cast-json` | create 时不转换 | [payload.ts#L105-L117](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L105-L117) |
| `cast-csv` | 数组转逗号分隔字符串 | [payload.ts#L152-L167](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L152-L167) |
| `user-created` | 注入当前用户 ID | [payload.ts#L122-L124](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L122-L124) |
| `date-created` | 注入当前时间戳 | [payload.ts#L138-L143](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L138-L143) |
| `encrypt` | 加密值 | [payload.ts#L169-L196](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L169-L196) |
| `conceal` | create 时不处理 | [payload.ts#L118-L119](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L118-L119) |

---

## 6. 关系字段处理

### 6.1 处理顺序

在 `createOne` 事务内，关系处理按以下固定顺序执行：

```
processM2O → processA2O → INSERT 主记录 → processO2M
```

- M2O 和 A2O 必须在 INSERT 之前处理，因为它们需要将嵌套对象替换为外键值
- O2M 必须在 INSERT 之后处理，因为需要主记录的主键值作为父 ID

### 6.2 processM2O (Many-to-One)

- 源码：[payload.ts#L665-L760](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L665-L760)

流程：
1. 从 schema 中筛选 `relation.collection === this.collection` 的关系
2. 仅处理 payload 中存在且值为对象的字段
3. 跳过 A2O 关系（`!relation.related_collection` 的为 A2O）
4. 检查关联记录是否存在（通过主键查询）
5. **存在则 `updateOne`，不存在则 `createOne`**
6. 将嵌套对象替换为关联记录的主键值：`payload[relation.field] = relatedPrimaryKey`
7. 收集嵌套的 revision IDs、action 事件、userIntegrityCheckFlags

### 6.3 processA2O (Any-to-One)

- 源码：[payload.ts#L551-L660](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L551-L660)

与 M2O 类似，但增加了：
1. 必须同时提供 `one_collection_field` 和 `one_allowed_collections` 元数据
2. 验证关联集合在允许列表中
3. 通过 `getService(relatedCollection, ...)` 选择正确的 Service

### 6.4 processO2M (One-to-Many)

- 源码：[payload.ts#L765-L1076](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts#L765-L1076)

两种输入模式：

**模式 A：数组形式**（`[{...}, {...}]`）
1. 遍历每条记录，检查是否已存在
2. 已存在的纯主键引用且已关联当前父记录 → 跳过
3. 其余记录设置 `relation.field = parentId` 后加入 `recordsToUpsert`
4. 调用 `service.upsertMany(recordsToUpsert, ...)`
5. 对不在 `savedPrimaryKeys` 中的已关联记录执行 **deselect 操作**：
   - `one_deselect_action === 'delete'` → `deleteByQuery`
   - 否则 → `updateByQuery` 将外键设为 null

**模式 B：Alterations 对象**（`{ create: [...], update: [...], delete: [...] }`）
1. 校验结构（Joi schema）
2. `create`：如果有 sort_field，自动计算排序值；调用 `createMany`
3. `update`：逐条调用 `updateOne`
4. `delete`：按 `one_deselect_action` 决定删除还是断开

---

## 7. Activity 与 Revision 生成

### 7.1 触发条件

- 源码：[items.ts#L333-L386](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L333-L386)

```typescript
if (
    opts.skipTracking !== true &&
    this.accountability &&
    this.schema.collections[this.collection]!.accountability !== null
) {
    // 创建 activity
}
```

三个条件必须同时满足：
1. `opts.skipTracking !== true`（未显式跳过）
2. `this.accountability` 存在（已认证请求）
3. 集合的 `accountability` 设置不为 `null`（即开启了审计追踪）

### 7.2 Activity 创建

```typescript
const activityService = new ActivityService({ knex: trx, schema: this.schema });
const activity = await activityService.createOne({
    action: Action.CREATE,
    user: this.accountability!.user,
    collection: this.collection,
    ip: this.accountability!.ip,
    user_agent: this.accountability!.userAgent,
    origin: this.accountability!.origin,
    item: primaryKey,
});
```

ActivityService 继承自 ItemsService，底层也使用 ItemsService 写入 `directus_activity` 表，但设置了 `autoPurgeCache: false` 和 `bypassLimits: true`。

### 7.3 Revision 创建

仅在 `accountability === 'all'` 时创建完整 revision：

```typescript
if (this.schema.collections[this.collection]!.accountability === 'all') {
    const revisionsService = new RevisionsService({ knex: trx, schema: this.schema });
    const relationalFields = getRelationsForCollection(this.schema, this.collection);
    const revisionPayload = await payloadService.prepareDelta(omit(payloadWithPresets, relationalFields));
    const revision = await revisionsService.createOne({
        activity: activity,
        collection: this.collection,
        item: primaryKey,
        data: revisionPayload,
        delta: revisionPayload,
    });
    // 将子关系的 revision 链接到父 revision
    const childrenRevisions = [...revisionsM2O, ...revisionsA2O, ...revisionsO2M];
    if (childrenRevisions.length > 0) {
        await revisionsService.updateMany(childrenRevisions, { parent: revision });
    }
}
```

**关键细节**：
- `prepareDelta` 使用 `processValues('read', ...)` 将 payload 转换为输出格式
- 关系字段从 revision 的 data/delta 中移除（`omit(payloadWithPresets, relationalFields)`）
- 子关系产生的 revision IDs 会被设置为当前 revision 的子节点（`parent` 字段）

---

## 8. Nested Action 事件延迟发射

### 8.1 事件收集机制

在 `createOne` 中：

```typescript
const nestedActionEvents: ActionEventParams[] = [];  // 事务前声明

const primaryKey = await transaction(this.knex, async (trx) => {
    // ... M2O/A2O/O2M 处理产生 nestedActionEventsM2O/A2O/O2M
    nestedActionEvents.push(...nestedActionEventsM2O);
    nestedActionEvents.push(...nestedActionEventsA2O);
    nestedActionEvents.push(...nestedActionEventsO2M);
    // ...
});

// 事务提交后发射
if (opts.emitEvents !== false) {
    // 主事件
    emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
    // 嵌套事件
    for (const nestedActionEvent of nestedActionEvents) {
        emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
    }
}
```

### 8.2 bypassEmitAction 回调

嵌套的 Service 调用（M2O/A2O/O2M 中的 create/update/delete）传入 `bypassEmitAction` 回调，将 action 事件收集到数组而非立即发射：

```typescript
bypassEmitAction: (params) =>
    opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
```

- 如果外层也传了 `bypassEmitAction`（如 `createMany` 场景），则继续向上冒泡
- 如果外层没有传（如直接调用 `createOne`），则收集到本地数组

### 8.3 emitAction 的异步特性

- 源码：[emitter.ts#L62-L72](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/emitter.ts#L62-L72)

```typescript
public emitAction(event: string | string[], meta: Record<string, any>, context: EventContext | null = null): void {
    const events = Array.isArray(event) ? event : [event];
    for (const event of events) {
        this.actionEmitter.emitAsync(event, { event, ...meta }, context ?? this.getDefaultContext()).catch((err) => {
            logger.warn(`An error was thrown while executing action "${event}"`);
            logger.warn(err);
        });
    }
}
```

`emitAction` 是**fire-and-forget**的：使用 `emitAsync` 异步执行所有监听器，错误仅记录日志，不影响调用方。这意味着 action 事件的失败不会导致事务回滚。

### 8.4 createMany 中的事件发射

```typescript
// createMany 中，所有 createOne 的事件都通过 bypassEmitAction 收集
// 事务提交后统一发射
for (const nestedActionEvent of nestedActionEvents) {
    emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
}
```

---

## 9. Cache 清理

### 9.1 shouldClearCache 判断

- 源码：[should-clear-cache.ts](file:////Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/should-clear-cache.ts)

```typescript
export function shouldClearCache(cache, opts?, collection?): cache is Keyv<any> {
    if (env['CACHE_AUTO_PURGE']) {
        if (collection && (env['CACHE_AUTO_PURGE_IGNORE_LIST'] as string[]).includes(collection)) {
            return false;
        }
        if (cache && opts?.autoPurgeCache !== false) {
            return true;
        }
    }
    return false;
}
```

条件：
1. `CACHE_AUTO_PURGE` 环境变量启用
2. 当前集合不在 `CACHE_AUTO_PURGE_IGNORE_LIST` 中
3. `opts.autoPurgeCache !== false`（默认为 undefined，即允许清理）

### 9.2 createOne 中的缓存清理

- 源码：[items.ts#L432-L434](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L432-L434)

```typescript
if (shouldClearCache(this.cache, opts, this.collection)) {
    await this.cache.clear();
}
```

在事务提交**之后**执行，清理整个缓存（非增量清理）。

### 9.3 createMany 中的缓存清理

- 源码：[items.ts#L511-L513](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L511-L513)

每个 `createOne` 传入 `autoPurgeCache: false`，只有最终 `createMany` 完成后统一清理一次。

---

## 10. 批量自增主键 Reset 时机

### 10.1 问题描述

当用户手动指定了自增主键的值时（如 `id: 42`），数据库的自增序列不会自动更新。后续不指定主键的 INSERT 可能因序列值落后而产生主键冲突。

### 10.2 createOne 中的处理

- 源码：[items.ts#L239-L251](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L239-L251)

```typescript
let autoIncrementSequenceNeedsToBeReset = false;
const pkField = this.schema.collections[this.collection]!.fields[primaryKeyField];

if (
    primaryKey &&
    pkField &&
    !opts.bypassAutoIncrementSequenceReset &&
    ['integer', 'bigInteger'].includes(pkField.type) &&
    pkField.defaultValue === 'AUTO_INCREMENT'
) {
    autoIncrementSequenceNeedsToBeReset = true;
}
```

条件：主键已提供 + 类型为 integer/bigInteger + 默认值为 AUTO_INCREMENT + 未被 bypass

Reset 执行时机在 INSERT **之后**、O2M 处理之后，仍在事务内：

```typescript
if (autoIncrementSequenceNeedsToBeReset) {
    await getHelpers(trx).sequence.resetAutoIncrementSequence(this.collection, primaryKeyField);
}
```

- 源码：[items.ts#L388-L390](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L388-L390)
- PostgreSQL 实现：[postgres.ts#L12-L17](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/helpers/sequence/dialects/postgres.ts#L12-L17)，使用 `SETVAL` 基于当前 MAX 值重置序列
- 默认实现（MySQL/SQLite 等）：[default.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/helpers/sequence/dialects/default.ts) 继承基类空操作，因为这些数据库不需要手动 reset

### 10.3 createMany 中的优化

- 源码：[items.ts#L464-L481](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L464-L481)

```typescript
for (const [index, payload] of data.entries()) {
    let bypassAutoIncrementSequenceReset = true;

    // 仅在当前项包含手动 PK 且是最后一条或下一条不含 PK 时才 reset
    if (payload[pkField] && (index === data.length - 1 || !data[index + 1]?.[pkField])) {
        bypassAutoIncrementSequenceReset = false;
    }

    const primaryKey = await service.createOne(payload, {
        ...opts,
        bypassAutoIncrementSequenceReset,
        // ...
    });
}
```

**优化逻辑**：批量创建时，仅当需要时才 reset 序列（避免每条记录都执行 `SETVAL`），具体来说：
- 如果当前项有手动 PK 但后续还有带手动 PK 的项 → 不 reset（bypass = true）
- 如果当前项有手动 PK 且是最后一项，或下一项不含 PK → 需要 reset（bypass = false）

---

## 11. 子关系写入失败时的回滚与事件分析

### 11.1 事务回滚范围

所有数据库写入（主记录 INSERT、M2O/A2O/O2M 关联记录、Activity、Revision、auto-increment reset）都在同一事务内。如果任何步骤抛出异常，Knex 事务会自动 ROLLBACK，**以下副作用全部回滚**：

| 回滚项 | 说明 |
|--------|------|
| 主记录 INSERT | 事务内，回滚 |
| M2O 关联的 create/update | 事务内，回滚 |
| A2O 关联的 create/update | 事务内，回滚 |
| O2M 关联的 upsert/delete/update | 事务内，回滚 |
| Activity 记录 | 事务内，回滚 |
| Revision 记录 | 事务内，回滚 |
| 子 Revision 的 parent 链接 | 事务内，回滚 |
| Auto-increment sequence reset | 事务内 DDL，回滚（PostgreSQL 的 SETVAL 在事务内可回滚） |
| User count integrity 验证 | 事务内，不产生副作用 |

### 11.2 不回滚的副作用

| 不回滚项 | 原因 |
|----------|------|
| Filter hook 中用户自定义的副作用 | `emitFilter` 在事务内执行，但 hook 作者可能在事务外产生了副作用（如 HTTP 调用） |
| Action 事件 | 延迟到事务提交后发射，若事务回滚则事件**不会被发射** |
| Cache 清理 | 在事务提交后执行，若事务回滚则**不会清理缓存** |
| `emitAction` 监听器的副作用 | fire-and-forget，即使已发射也无法回滚（但正常流程下事务回滚时不会发射） |

### 11.3 具体失败场景分析

#### 场景 A：M2O 关联创建失败

```
processM2O → service.createOne(relatedRecord) 抛出异常
```

- **回滚**：主记录尚未 INSERT（M2O 在 INSERT 之前），之前成功创建的 M2O 关联记录、所有 Activity/Revision 均回滚
- **事件**：尚未到达 action 事件发射点，不会有任何 action 事件被发射
- **Filter hook**：已执行，其外部副作用不可回滚

#### 场景 B：主记录 INSERT 失败（如唯一约束冲突）

```
trx.insert(payloadWithoutAliases) → 抛出数据库错误
```

- **回滚**：M2O/A2O 已创建的关联记录回滚
- **事件**：同上，未到发射点
- **注意**：数据库错误会被 `translateDatabaseError` 翻译为 Directus 错误

#### 场景 C：O2M 关联创建失败

```
processO2M → service.upsertMany/createMany/updateOne 抛出异常
```

- **回滚**：主记录和 M2O/A2O 关联记录全部回滚，因为都在同一事务中
- **事件**：主记录的 action 事件和所有嵌套事件均不会被发射

#### 场景 D：Activity/Revision 写入失败

- **回滚**：主记录和所有关联记录全部回滚
- **事件**：不会被发射

#### 场景 E：事务提交后、事件发射前进程崩溃

- **数据库**：已提交，不会回滚
- **事件**：丢失，不会重放
- **Cache**：未清理

### 11.4 事件发射时序总结

```
┌──────────────────── 事务开始 ────────────────────┐
│  1. emitFilter (items.create / collection.create) │
│  2. processPayload (权限校验 + 预设注入)            │
│  3. processM2O (嵌套事件收集到 nestedActionEvents)  │
│  4. processA2O (嵌套事件收集到 nestedActionEvents)  │
│  5. INSERT 主记录                                   │
│  6. processO2M (嵌套事件收集到 nestedActionEvents)  │
│  7. User integrity check                           │
│  8. Activity + Revision 写入                       │
│  9. Auto-increment sequence reset (如需)           │
└──────────────────── 事务提交 ────────────────────┘
         │
         ▼ (仅事务成功后)
┌──────────────────── 事务后 ──────────────────────┐
│  10. emitter.emitAction (主事件)                    │
│  11. emitter.emitAction (所有 nestedActionEvents)   │  ← fire-and-forget
│  12. cache.clear() (如果 CACHE_AUTO_PURGE)          │
└──────────────────────────────────────────────────┘
```

**核心原则**：
- **Filter 事件（emitFilter）**：事务内同步执行，可修改 payload，可感知事务状态
- **Action 事件（emitAction）**：事务提交后异步执行，fire-and-forget，不影响事务结果
- **事务回滚时**：所有 action 事件都不会被发射（因为发射逻辑在事务成功返回后才执行）
- **Filter hook 的外部副作用**：是唯一无法自动回滚的副作用，hook 作者需自行处理补偿逻辑

---

## 附：关键源码文件索引

| 文件 | 职责 |
|------|------|
| [controllers/items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/items.ts) | 路由定义与请求处理 |
| [middleware/is-locked.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/is-locked.ts) | 许可证锁定检查 |
| [middleware/collection-exists.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/collection-exists.ts) | 集合存在性验证 |
| [services/items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts) | ItemsService 核心 CRUD 逻辑 |
| [services/payload.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/payload.ts) | Payload 预处理、类型转换、关系处理 |
| [services/activity.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/activity.ts) | ActivityService |
| [services/revisions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/revisions.ts) | RevisionsService |
| [utils/transaction.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/transaction.ts) | 事务管理工具 |
| [utils/should-clear-cache.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/should-clear-cache.ts) | 缓存清理判断 |
| [utils/get-service.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-service.ts) | Service 分发工厂 |
| [utils/validate-keys.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/validate-keys.ts) | 主键类型校验 |
| [emitter.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/emitter.ts) | 事件发射器 |
| [permissions/modules/process-payload/process-payload.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-payload/process-payload.ts) | 权限校验与预设注入 |
| [database/helpers/sequence/dialects/postgres.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/database/helpers/sequence/dialects/postgres.ts) | PostgreSQL 自增序列重置 |
| [packages/system-data/src/collections/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/system-data/src/collections/index.ts) | 系统集合判断 |
| [packages/types/src/items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/items.ts) | MutationOptions/MutationTracker 类型定义 |
