# Directus Items 读取权限系统全流程分析

本文档详细分析普通用户读取 items 时，Directus 权限系统从 policy/access 查询、动态变量提取、`$CURRENT_USER`/`$CURRENT_ROLE` 数据加载、permission 条件处理，一路影响 GraphQL 或 REST 查询 AST 的完整链路。同时说明 admin、无 accountability、share、普通用户四种场景的处理分支差异，并明确字段不存在与字段无权限时的拒绝位置。

---

## 一、总体架构概览

### 1.1 调用入口

权限系统介入 items 读取的核心位置在 `ItemsService.readByQuery()` 方法中：

**REST 入口**：[items.ts:544-559](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts#L544-L559)
```
getAstFromQuery() → processAst() → runAst()
```

**GraphQL 入口**：[query.ts:15-50](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/query.ts#L15-L50)
```
resolveQuery() → gql.read() → ItemsService.readByQuery() → processAst()
```

GraphQL 最终也通过 `ItemsService` 调用相同的 `processAst()` 流程，因此 REST 和 GraphQL 共享同一套权限 AST 处理逻辑。

### 1.2 核心流程全景图

```
用户请求 (REST/GraphQL)
        ↓
ItemsService.readByQuery()
        ↓
getAstFromQuery() ──→ 构建查询 AST
        ↓
processAst() ──────────────┐
    │                      │
    ├─ fieldMapFromAst()   │ 构建字段访问路径映射
    │                      │
    ├─ [BRANCH 1] admin    │
    │  或 无 accountability├─ 仅 validatePathExistence()，跳过权限检查
    │                      │
    └─ [BRANCH 2] 普通用户 │
       │                   │
       ├─ fetchPolicies()  │ 查询关联策略（policy/access 表）
       │                   │
       ├─ fetchPermissions()───────────────┐
       │    │                              │
       │    ├─ fetchRawPermissions()       │ 查询 directus_permissions 表
       │    │                              │
       │    ├─ extractRequiredDynamic-     │ 从权限规则中提取 $CURRENT_* 所需字段
       │    │   VariableContext()          │
       │    │                              │
       │    ├─ fetchDynamicVariableData()  │ 加载用户/角色/策略的实际数据
       │    │   ├─ UsersService.readOne()  │ ($CURRENT_USER)
       │    │   ├─ RolesService.readOne/Many() ($CURRENT_ROLE(S))
       │    │   └─ PoliciesService.readMany() ($CURRENT_POLICIES)
       │    │                              │
       │    ├─ processPermissions()        │ 替换动态变量为实际值
       │    │   └─ parseFilter()/parsePreset()
       │    │                              │
       │    └─ [BRANCH share]              │ Share 场景走 getPermissionsForShare()
       │                                   │
       ├─ validatePathExistence()          │ 所有场景：字段存在性校验
       │                                   │
       ├─ validatePathPermissions()        │ 普通用户：字段权限校验
       │                                   │
       └─ injectCases()                    │ 注入条件 CASE/WHEN 到 AST
                            ───────────────┘
        ↓
runAst() ──→ 执行最终查询
```

---

## 二、核心函数详解

### 2.1 `processAst()` — AST 权限处理总入口

**文件**：[process-ast.ts:19-67](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/process-ast.ts#L19-L67)

```typescript
export async function processAst(options: ProcessAstOptions, context: Context) {
    const fieldMap: FieldMap = fieldMapFromAst(options.ast, context.schema);
    const collections = collectionsInFieldMap(fieldMap);

    if (!options.accountability || options.accountability.admin) {
        // BRANCH: admin 或 无 accountability
        for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
            validatePathExistence(path, collection, fields, context.schema);
        }
        return options.ast;
    }

    // BRANCH: 普通用户（含 share）
    const policies = await fetchPolicies(options.accountability, context);
    const permissions = await fetchPermissions({...}, context);
    const readPermissions = options.action === 'read' ? permissions : await fetchPermissions(...);

    for (...) { validatePathExistence(...); }
    for (...) { validatePathPermissions(path, permissions, ...); }   // 非读操作字段
    for (...) { validatePathPermissions(path, readPermissions, ...); } // 只读字段

    injectCases(options.ast, permissions);
    return options.ast;
}
```

**关键决策点**（第 25 行）：
- `!options.accountability`：系统内部调用（如 cron、hook 中无用户上下文）
- `options.accountability.admin`：管理员用户

两者均跳过权限检查，仅保留字段存在性校验。

### 2.2 `fetchPolicies()` — 策略查询（policy/access 关联）

**文件**：[fetch-policies.ts:16-59](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/lib/fetch-policies.ts#L16-L59)

此函数从 `directus_access` 表查询当前 accountability 关联的策略（policy）ID 列表。

#### 查询逻辑：

```typescript
// 1. 角色过滤
if (roles.length === 0) {
    // 无角色用户：匹配 Public 角色（role=null 且 user=null）
    roleFilter = { _and: [{ role: { _null: true } }, { user: { _null: true } }] };
} else {
    // 有角色用户：匹配任一角色
    roleFilter = { role: { _in: roles } };
}

// 2. 用户级策略叠加
const filter = user ? { _or: [{ user: { _eq: user } }, roleFilter] } : roleFilter;

// 3. 执行查询：读取 policy.id、policy.ip_access、role
const accessRows = await accessService.readByQuery({ filter, fields: ['policy.id', 'policy.ip_access', 'role'], ... });

// 4. IP 白名单过滤
const filteredAccessRows = filterPoliciesByIp(accessRows, ip);

// 5. 优先级排序（从低到高）：
//    - 父角色策略 → 子角色策略 → 用户专属策略
filteredAccessRows.sort((a, b) => {
    if (!a.role && !b.role) return 0;
    if (!a.role) return 1;   // 用户级策略（role=null）排最后，优先级最高
    if (!b.role) return -1;
    return roles.indexOf(a.role) - roles.indexOf(b.role);
});

return filteredAccessRows.map(({ policy }) => policy.id);
```

**注意**：`fetchPolicies` 通过 `withCache` 装饰器做了缓存，缓存键包含 `roles`、`user`、`ip`。

---

## 三、动态变量提取与数据加载

### 3.1 `extractRequiredDynamicVariableContextForPermissions()` — 动态变量需求提取

**文件**：[extract-required-dynamic-variable-context.ts:11-64](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/extract-required-dynamic-variable-context.ts#L11-L64)

遍历所有权限规则的三个字段：`permissions`（读权限过滤条件）、`validation`（写验证条件）、`presets`（预设值），通过 `deepMap` 递归扫描其中以 `$CURRENT_*` 开头的字符串引用。

```typescript
function extractPermissionData(val: any) {
    for (const placeholder of [
        '$CURRENT_USER', '$CURRENT_ROLE', '$CURRENT_ROLES', '$CURRENT_POLICIES'
    ]) {
        if (typeof val === 'string' && val.startsWith(`${placeholder}.`)) {
            // 例如 $CURRENT_USER.department → 提取 "department"
            permissionContext[placeholder].add(val.replace(`${placeholder}.`, ''));
        }
    }
}
```

**输出**：
```typescript
interface DynamicVariableContext {
    $CURRENT_USER: Set<string>;     // 需读取的用户字段，如 ['id', 'department']
    $CURRENT_ROLE: Set<string>;     // 需读取的角色字段
    $CURRENT_ROLES: Set<string>;    // 需读取的多角色字段
    $CURRENT_POLICIES: Set<string>; // 需读取的策略字段
}
```

### 3.2 `fetchDynamicVariableData()` — 实际数据加载

**文件**：[fetch-dynamic-variable-data.ts:14-94](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/fetch-dynamic-variable-data.ts#L14-L94)

根据提取出的字段需求，分别调用对应的 Service 从数据库加载数据，并做缓存。

#### 分支处理：

| 动态变量 | 触发条件 | 数据加载方式 |
|---------|---------|------------|
| `$CURRENT_USER` | `accountability.user` 存在 且 需要字段集非空 | `UsersService.readOne(user_id, { fields })` |
| `$CURRENT_ROLE` | `accountability.role` 存在 且 需要字段集非空 | `RolesService.readOne(role_id, { fields })` |
| `$CURRENT_ROLES` | `accountability.roles` 非空 且 需要字段集非空 | `RolesService.readMany(role_ids, { fields })` |
| `$CURRENT_POLICIES` | `policies.length > 0` | 有需要字段 → `PoliciesService.readMany()`；无 → 仅返回 `{ id }` 数组 |

**缓存机制**：通过 `filter-context-{TYPE}-{HASH}` 作为缓存 key，基于 `CACHE_ENABLED` 环境变量控制。

### 3.3 `processPermissions()` / `parseFilter()` — 动态变量替换

**文件**：[process-permissions.ts:10-19](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/process-permissions.ts#L10-L19)

**parseFilter 核心**：[parse-filter.ts:177-208](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/utils/shared/parse-filter.ts#L177-L208)

```typescript
function parseDynamicVariable(value, accountability, context) {
    if (value.startsWith('$CURRENT_USER')) {
        if (value === '$CURRENT_USER') return accountability?.user ?? null;
        return get(context, value, null);  // 从 $CURRENT_USER 数据对象取子路径
    }
    if (value.startsWith('$CURRENT_ROLE')) {
        if (value === '$CURRENT_ROLE') return accountability?.role ?? null;
        return get(context, value, null);
    }
    // $CURRENT_ROLES、$CURRENT_POLICIES 同理
}
```

**替换示例**：

**替换前**（权限规则存储在数据库中）：
```json
{
  "permissions": {
    "created_by": { "_eq": "$CURRENT_USER.id" },
    "department": { "_in": "$CURRENT_ROLE.accessible_departments" }
  }
}
```

**替换后**（传入实际数据）：
```json
{
  "permissions": {
    "created_by": { "_eq": "user-123" },
    "department": { "_in": ["sales", "marketing"] }
  }
}
```

---

## 四、四种场景的分支行为对比

### 4.1 四种场景定义

| 场景 | accountability 特征 | 典型触发 |
|-----|-------------------|---------|
| **Admin** | `accountability.admin === true` | 管理员用户登录，`fetchGlobalAccess()` 返回 admin=true |
| **无 Accountability** | `accountability === null` | 系统内部调用（如 hook、cron、CLI 工具）、服务间无上下文调用 |
| **Share** | `accountability.share` 非空（且 action=read 或 undefined） | 通过共享链接访问 `?share=xxx` |
| **普通用户** | accountability 非 null 非 admin 非 share | 普通登录用户、Public 用户访问 |

### 4.2 `processAst()` 中四种场景的分支

```
processAst(options, context)
        │
        ├─ fieldMapFromAst()    ← 所有场景都执行
        │
        ├─► !accountability || accountability.admin  [判断分支]
        │       │
        │       ├─ YES: 【场景：Admin 或 无Accountability】
        │       │     └─ 仅执行 validatePathExistence()
        │       │     └─ 直接 return ast（不做 fetchPolicies / 权限校验 / injectCases）
        │       │
        │       └─ NO: 【场景：普通用户 或 Share】
        │             ├─ fetchPolicies()    ← 查 access 关联策略
        │             ├─ fetchPermissions() ← 查权限规则 + 动态变量替换
        │             ├─ validatePathExistence()
        │             ├─ validatePathPermissions(permissions)   ← 操作级字段校验
        │             ├─ validatePathPermissions(readPermissions) ← 读级字段校验
        │             └─ injectCases()       ← 注入 CASE/WHEN 条件
        │
        └─ return processed_ast
```

### 4.3 `fetchPermissions()` 中四种场景的分支

**文件**：[fetch-permissions.ts:17-50](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/lib/fetch-permissions.ts#L17-L50)

```
fetchPermissions(options, context)
        │
        ├─ fetchRawPermissions()       ← 所有场景先查 directus_permissions 表（带缓存）
        │       │
        │       └─ 按 policy ∈ policies[]、action、collection 过滤
        │
        ├─► accountability 存在 && !bypassDynamicVariableProcessing  [判断分支]
        │       │
        │       ├─ YES: 【场景：普通用户 / Share / 非admin】
        │       │     ├─ extractRequiredDynamicVariableContextForPermissions()
        │       │     ├─ fetchDynamicVariableData()   ← 加载 $CURRENT_* 数据
        │       │     ├─ processPermissions()         ← 替换动态变量
        │       │     │
        │       │     └─► accountability.share 存在 && (action=read || undefined)  [子分支]
        │       │           │
        │       │           ├─ YES: 【场景：Share】
        │       │           │     └─ return getPermissionsForShare()
        │       │           │           │
        │       │           │           ├─ fetchShareInfo() → collection/item/role/user_created
        │       │           │           ├─ 构建 shareAccountability 和 userAccountability
        │       │           │           ├─ 双方都要 fetchGlobalAccess() 判断是否 admin
        │       │           │           ├─ 分别获取双方权限，按"交集"合并
        │       │           │           ├─ traverse() 生成关联关系的可达权限（防止循环）
        │       │           │           └─ 最终：limitedPermissions ∩ generatedPermissions
        │       │           │
        │       │           └─ NO: 【场景：普通用户】
        │       │                 └─ return processedPermissions
        │       │
        │       └─ NO: 【场景：Admin 或 无Accountability 或 跳过动态变量处理】
        │             └─ return permissions （原始权限，无动态变量替换）
```

### 4.4 Share 场景的特殊处理（`getPermissionsForShare()`）

**文件**：[get-permissions-for-share.ts:14-150](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/get-permissions-for-share.ts#L14-L150)

Share 场景需要**双重权限校验**（Share 角色权限 ∩ 创建者用户权限），具体组合逻辑：

| share admin | user admin | 实际权限 |
|-------------|-----------|---------|
| ✅ | ✅ | 完全 admin，所有字段 `['*']` |
| ❌ | ✅ | 使用 share 角色权限 + share fieldMap |
| ✅ | ❌ | 使用 创建者用户权限 + user fieldMap |
| ❌ | ❌ | 两者权限取交集（intersection），fieldMap 取两次 reduceSchema |

此外，Share 场景还会通过 `traverse()` 函数递归生成从共享根 item 可达的所有关联关系的过滤权限（基于主键链路限制），防止通过关联字段越权访问其他数据。

---

## 五、字段存在性 vs 字段权限 — 拒绝位置详解

### 5.1 拒绝位置总览

| 拒绝场景 | 执行函数 | 文件位置 | 所有场景都执行？ | 抛出错误类型 |
|---------|---------|---------|---------------|------------|
| **集合不存在** | `validatePathExistence()` | [validate-path-existence.ts:7-9](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/utils/validate-path/validate-path-existence.ts#L7-L9) | ✅ 是（包括 admin） | `ForbiddenError` |
| **字段不存在** | `validatePathExistence()` | [validate-path-existence.ts:13-17](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/utils/validate-path/validate-path-existence.ts#L13-L17) | ✅ 是（包括 admin） | `ForbiddenError` |
| **集合无权限** | `validatePathPermissions()` | [validate-path-permissions.ts:12-14](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts#L12-L14) | ❌ 仅普通用户/Share | `ForbiddenError` |
| **字段无权限** | `validatePathPermissions()` | [validate-path-permissions.ts:36-42](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts#L36-L42) | ❌ 仅普通用户/Share | `ForbiddenError` |

### 5.2 `validatePathExistence()` — 字段不存在校验

**执行时机**：在 `processAst()` 中，所有四种场景**都会**被调用。

**管理员也需要**：即使 admin 拥有全部权限，如果查询一个数据库中不存在的字段（如拼写错误 `titlle` 而非 `title`），也必须在此阶段被拒绝。

```typescript
export function validatePathExistence(path, collection, fields, schema) {
    // 1. 集合是否存在？
    const collectionInfo = schema.collections[collection];
    if (collectionInfo === undefined) {
        throw createCollectionForbiddenError(path, collection);
    }

    // 2. 每个字段是否存在于 schema 中？
    const nonExistentFields = requestedFields.filter(
        (field) => collectionInfo.fields[field] === undefined
    );

    if (nonExistentFields.length > 0) {
        throw createFieldsForbiddenError(path, collection, nonExistentFields);
    }
}
```

**错误信息示例**：
```
You don't have permission to access field "titlle" in collection "articles" or it does not exist. Queried in root.
```

> **注意错误信息的歧义性**：报错文案说"或它不存在"，但此时一定是字段不存在，因为 admin 场景根本不会检查权限。这是出于安全的模糊设计（防止攻击者枚举 schema）。

### 5.3 `validatePathPermissions()` — 字段无权限校验

**执行时机**：仅在非 admin 且 非 无 accountability 的场景中调用。

```typescript
export function validatePathPermissions(path, permissions, collection, fields) {
    // 1. 该集合是否有任何权限规则？
    const permissionsForCollection = permissions.filter(p => p.collection === collection);
    if (permissionsForCollection.length === 0) {
        throw createCollectionForbiddenError(path, collection);
    }

    // 2. 汇总所有权限规则允许的字段
    const allowedFields: Set<string> = new Set();
    for (const { fields } of permissionsForCollection) {
        if (!fields) continue;
        for (const field of fields) {
            if (field === '*') return;  // 通配符：所有字段都允许，直接返回
            allowedFields.add(field);
        }
    }

    // 3. 检查请求字段是否都在允许列表中
    const forbiddenFields = requestedFields.filter(
        (field) => allowedFields.has(field) === false
    );

    if (forbiddenFields.length > 0) {
        throw createFieldsForbiddenError(path, collection, forbiddenFields);
    }
}
```

**关键细节**：
- 允许多条权限规则**叠加**：如果 policy A 允许 `['id', 'title']`，policy B 允许 `['status']`，则最终可访问 `['id', 'title', 'status']`
- 任一规则含 `fields: ['*']` 则该集合所有字段都放行（第 26-28 行 early exit）
- 字段级别使用 `Set` 数据结构做 O(1) 查找

### 5.4 两次 validatePathPermissions 的区别

在 `processAst()` 中调用了两次：

```typescript
// 第 55-57 行：对"操作相关"字段使用操作 action 的权限（如 create/update/delete 的字段）
for (const [path, { collection, fields }] of fieldMap.other.entries()) {
    validatePathPermissions(path, permissions, collection, fields);
}

// 第 60-62 行：对"仅读取"字段使用 read action 的权限
// （即使当前 action 不是 read，嵌套关联字段的读取也需要 read 权限）
for (const [path, { collection, fields }] of fieldMap.read.entries()) {
    validatePathPermissions(path, readPermissions, collection, fields);
}
```

`fieldMap.other` 包含用于过滤、排序、聚合等 AST 查询路径中的字段（间接影响数据读取的字段）；`fieldMap.read` 包含直接出现在返回结果中的字段。

---

## 六、`injectCases()` — 条件权限如何注入 AST

当权限规则包含 `permissions` 字段（item-level 过滤条件，如"只能看自己创建的"）时，简单的字段级放行不足以控制行级访问。Directus 通过 SQL `CASE WHEN` 动态判定每行每列是否可访问。

**文件**：[inject-cases.ts:13-73](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/lib/inject-cases.ts#L13-L73)

### 6.1 核心逻辑

```typescript
export function injectCases(ast, permissions) {
    ast.cases = processChildren(ast.name, ast.children, permissions);
}

function processChildren(collection, children, permissions) {
    const requestedKeys = uniq(children.map(getUnaliasedFieldKey));
    const { cases, caseMap, allowedFields } = getCases(collection, permissions, requestedKeys);

    for (const child of children) {
        const fieldKey = getUnaliasedFieldKey(child);
        const globalWhenCase = caseMap['*'];
        const fieldWhenCase = caseMap[fieldKey];

        // 安全兜底：如果该字段既没有 * 通配规则，也没有专属规则 → 抛错
        if (!globalWhenCase && !fieldWhenCase) {
            throw new Error(`Cannot extract access permissions for field "${fieldKey}" in collection "${collection}"`);
        }

        // 如果该字段不在无条件放行列表中 → 附加 whenCase
        if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
            child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
        }

        // 递归处理 m2o/o2m/a2o/functionField 的子节点
        if (child.type === 'm2o') child.cases = processChildren(child.relation.related_collection!, child.children, permissions);
        if (child.type === 'o2m') child.cases = processChildren(child.relation.collection, child.children, permissions);
        // ...
    }

    return cases;
}
```

### 6.2 `getCases()` — 构建 cases 数组和 caseMap

**文件**：[get-cases.ts:6-57](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/lib/get-cases.ts#L6-L57)

每条带条件的权限规则 → 生成一个 case（过滤条件本身），并映射到该规则允许的所有字段。

```
规则 A: fields=['id','title'], permissions={status:{_eq:'published'}}
规则 B: fields=['*'],          permissions={created_by:{_eq:'$CURRENT_USER.id'}}
```

生成：
```
cases = [
  { status: { _eq: 'published' } },                 // cases[0]
  { created_by: { _eq: 'user-123' } }               // cases[1]
]
caseMap = {
  'id':    [0, 1],   // id 字段：规则 A 和规则 B 都允许
  'title': [0, 1],   // title 字段：规则 A 和规则 B 都允许（A 明确包含，B 通配）
  '*':     [1]       // 通配：规则 B 的 * 适用于所有字段
}
allowedFields = new Set()  // 无条件放行字段（permissions 为空的规则的字段）
```

最终生成的 SQL 伪代码：
```sql
CASE
  WHEN (status = 'published') OR (created_by = 'user-123') THEN title
  ELSE NULL  -- 条件不满足则该字段返回 NULL
END AS title
```

---

## 七、缓存机制

Directus 权限系统大量使用缓存以提升性能，关键缓存点：

| 缓存函数 | 缓存键构成 | 用途 |
|---------|-----------|------|
| `fetchPolicies` | `{ roles, user, ip }` | policy ID 列表查询结果 |
| `fetchRawPermissions` | `{ policies, action, collections, accountability.app, bypassMinimalAppPermissions }` | directus_permissions 原始查询结果 |
| `fetchDynamicVariableData` 内部 | `filter-context-{TYPE}-{HASH}`（HASH=cacheContext+fields 的 JSON hash） | `$CURRENT_USER` / `$CURRENT_ROLE` 等动态数据 |
| `fetchGlobalAccess` | `{ user, roles, ip }` | admin / app 全局权限标记 |

---

## 八、总结：四种场景的端到端差异矩阵

| 维度 | Admin | 无 Accountability | Share | 普通用户 |
|-----|-------|------------------|-------|---------|
| **processAst 分支** | 早返回（L25） | 早返回（L25） | 完整流程 | 完整流程 |
| **fetchPolicies** | ❌ 不执行 | ❌ 不执行 | ✅ 执行两次（share角色+创建者用户） | ✅ 执行 |
| **fetchRawPermissions** | ❌ 不执行 | ❌ 不执行 | ✅ 执行 | ✅ 执行 |
| **动态变量提取** | ❌ | ❌ | ✅（合并时各自执行） | ✅ |
| **$CURRENT_USER 加载** | ❌ | ❌ | ✅ 创建者用户数据 | ✅ 自身数据 |
| **$CURRENT_ROLE 加载** | ❌ | ❌ | ✅ 双方角色数据 | ✅ 自身角色 |
| **validatePathExistence** | ✅ 字段存在性校验 | ✅ 字段存在性校验 | ✅ 字段存在性校验 | ✅ 字段存在性校验 |
| **validatePathPermissions** | ❌ 跳过 | ❌ 跳过 | ✅ 合并后权限校验 | ✅ 权限校验 |
| **injectCases** | ❌ 跳过 | ❌ 跳过 | ✅ 注入行级条件 | ✅ 注入行级条件 |
| **字段不存在拒绝** | validatePathExistence | validatePathExistence | validatePathExistence | validatePathExistence |
| **字段无权限拒绝** | N/A（都允许） | N/A（都允许） | validatePathPermissions | validatePathPermissions |
| **行级过滤** | 无限制（除非自定义查询） | 无限制 | 双重权限交集 + share主键链路限制 | permissions 过滤条件 |

---

## 九、关键文件索引

| 文件 | 功能 |
|-----|------|
| [process-ast.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/process-ast.ts) | AST 权限处理主流程 |
| [fetch-permissions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/lib/fetch-permissions.ts) | 权限查询 + 动态变量处理编排 |
| [fetch-policies.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/lib/fetch-policies.ts) | policy/access 关联查询 |
| [fetch-raw-permissions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/fetch-raw-permissions.ts) | directus_permissions 表查询 |
| [extract-required-dynamic-variable-context.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/extract-required-dynamic-variable-context.ts) | 动态变量需求提取 |
| [fetch-dynamic-variable-data.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/fetch-dynamic-variable-data.ts) | $CURRENT_* 数据加载 |
| [process-permissions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/process-permissions.ts) | 动态变量替换编排 |
| [parse-filter.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/utils/shared/parse-filter.ts) | 动态变量替换核心实现 |
| [get-permissions-for-share.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/utils/get-permissions-for-share.ts) | Share 场景特殊权限计算 |
| [validate-path-existence.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/utils/validate-path/validate-path-existence.ts) | 字段不存在拒绝点 |
| [validate-path-permissions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts) | 字段无权限拒绝点 |
| [inject-cases.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/lib/inject-cases.ts) | CASE/WHEN 条件注入 AST |
| [get-cases.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/permissions/modules/process-ast/lib/get-cases.ts) | Cases 数组与映射构建 |
| [items.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/items.ts) | REST 服务层（processAst 调用点 L556） |
| [query.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/query.ts) | GraphQL 解析器 |
