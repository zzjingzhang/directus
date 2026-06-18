# Directus 集合级访问合并与前端权限解析全链路分析

> 围绕 `directus_permissions` 和 `directus_access` 两张核心表，分析后端如何合并集合级访问结果，以及 App 端如何将 `/permissions/me` 的响应转化为 `hasPermission` 和 `preset` 动态字段。

---

## 目录

1. [数据模型概览](#1-数据模型概览)
2. [后端：/permissions/me 全链路](#2-后端permissionsme-全链路)
3. [mergePermissions 核心合并策略](#3-mergepermissions-核心合并策略)
4. [字段合并策略 mergeFields](#4-字段合并策略-mergefields)
5. [三种访问级别的判定逻辑](#5-三种访问级别的判定逻辑)
6. [Preset 合并与解析](#6-preset-合并与解析)
7. [动态变量 Hydrate 机制](#7-动态变量-hydrate-机制)
8. [App 端权限 Store 与 hasPermission](#8-app-端权限-store-与-haspermission)
9. [前端可见字段控制 isFieldAllowed](#9-前端可见字段控制-isfieldallowed)
10. [后端策略变化对前端可见行为的影响](#10-后端策略变化对前端可见行为的影响)

---

## 1. 数据模型概览

### directus_access

`directus_access` 是策略（Policy）与用户/角色之间的关联表。每一行代表一条"某策略被赋予某角色或某用户"的记录：

| 字段 | 含义 |
|------|------|
| `id` | 主键 |
| `policy` | 外键 → `directus_policies.id` |
| `role` | 外键 → `directus_roles.id`（可为 null，表示 Public 策略） |
| `user` | 外键 → `directus_users.id`（可为 null，表示角色级策略） |

一个用户最终权限 = 其所有角色关联的策略 ∪ 其个人关联的策略，按优先级排序后合并。

### directus_permissions

`directus_permissions` 存储每条策略下对某个集合、某个动作的细粒度控制：

| 字段 | 含义 |
|------|------|
| `id` | 主键 |
| `policy` | 外键 → `directus_policies.id` |
| `collection` | 目标集合名 |
| `action` | 动作：`read` / `create` / `update` / `delete` / `share` |
| `permissions` | 行级过滤条件（Filter），`{}` 表示无限制，`null` 表示无此权限 |
| `validation` | 字段级验证规则 |
| `presets` | 字段默认值（含动态变量如 `$CURRENT_USER.id`） |
| `fields` | 允许访问的字段列表，`["*"]` 表示全部字段 |

> **关键区别**：`permissions: {}`（空对象）= 无过滤条件（全量可见），`permissions: null` = 无此权限行。

---

## 2. 后端：/permissions/me 全链路

### 2.1 路由入口

定义于 [permissions.ts](api/src/controllers/permissions.ts#L92-L106)：

```
GET /permissions/me → fetchAccountabilityCollectionAccess(accountability, context)
```

核心逻辑分两条路径：

```
                    ┌─────────────────┐
                    │ accountability   │
                    │  .admin === true │
                    └───────┬─────────┘
                            │
              ┌─────────────┴──────────────┐
              │ Yes                        │ No
              ▼                            ▼
    返回全量 CollectionAccess        fetchPolicies()
    (所有集合 × 所有动作             ↓
     access='full',                 fetchPermissions()
     fields=['*'])                  ↓
                              extractDynamicVariables()
                              fetchDynamicVariableData()
                              processPermissions()
                              ↓
                              构建 CollectionAccess
```

### 2.2 fetchPolicies — 策略收集

源码：[fetch-policies.ts](api/src/permissions/lib/fetch-policies.ts#L16-L59)

```
输入: { user, roles, ip }
  │
  ├─ 查询 directus_access：
  │    WHERE role IN (user.roles) OR user = currentUser
  │
  ├─ filterPoliciesByIp：按 ip_access 过滤
  │
  └─ 排序（优先级从低到高）：
       1. 父角色策略
       2. 子角色策略
       3. 用户个人策略
  │
  输出: 排序后的 policy ID 列表
```

排序方向至关重要——后合入的策略（用户个人策略）优先级更高，在 preset 合并时会覆盖先前的值。

### 2.3 fetchRawPermissions — 原始权限查询

源码：[fetch-raw-permissions.ts](api/src/permissions/utils/fetch-raw-permissions.ts#L27-L61)

```
输入: { policies, action?, collections?, accountability? }
  │
  ├─ 查询 directus_permissions：
  │    WHERE policy IN (policies)
  │    AND action = ? (可选)
  │    AND collection IN (?) (可选)
  │
  ├─ 按 policy 排序顺序排列
  │
  └─ 如果 accountability.app === true：
       追加 appAccessMinimalPermissions
       （App 运行所需的最小权限集合）
  │
  输出: Permission[] 原始权限列表
```

### 2.4 appAccessMinimalPermissions — App 最小权限

源码：[app-access-permissions/index.ts](packages/system-data/src/app-access-permissions/index.ts#L18-L21)

当用户通过 App 访问时（`accountability.app === true`），系统自动追加一组最小权限，确保 App 界面正常运作。包括：

- `directus_users.read` / `directus_users.update`（限本人）
- `directus_roles.read`
- `directus_files.read` / `create` / `update` / `delete`
- `directus_shares.read` / `create` / `update` / `delete`
- `directus_flows.read`（仅 trigger=manual）
- `directus_folders` / `directus_dashboards` / `directus_panels` 的完整 CRUD
- Schema 相关的系统集合权限

这些权限的 `policy: null`、`system: true`，在合并时与用户策略权限叠加。

### 2.5 动态变量处理管线

源码：[fetch-permissions.ts](api/src/permissions/lib/fetch-permissions.ts#L17-L49)

```
rawPermissions
  │
  ├─ extractRequiredDynamicVariableContextForPermissions
  │   扫描所有 permissions/validation/presets
  │   收集 $CURRENT_USER.* / $CURRENT_ROLE.* / $CURRENT_ROLES.* / $CURRENT_POLICIES.* 引用
  │
  ├─ fetchDynamicVariableData
  │   根据收集到的字段列表，从 DB 查询：
  │   - $CURRENT_USER → UsersService.readOne
  │   - $CURRENT_ROLE → RolesService.readOne
  │   - $CURRENT_ROLES → RolesService.readMany
  │   - $CURRENT_POLICIES → PoliciesService.readMany
  │
  └─ processPermissions
      使用 parseFilter / parsePreset 替换动态变量为实际值
```

#### parseDynamicVariable 核心替换规则

源码：[parse-filter.ts](packages/utils/shared/parse-filter.ts#L177-L208)

| 动态变量 | 替换结果 |
|---------|---------|
| `$CURRENT_USER` | `accountability.user`（用户 ID） |
| `$CURRENT_USER.xxx` | `context.$CURRENT_USER.xxx`（从 DB 查到的用户字段值） |
| `$CURRENT_ROLE` | `accountability.role`（角色 ID） |
| `$CURRENT_ROLE.xxx` | `context.$CURRENT_ROLE.xxx` |
| `$CURRENT_ROLES` | `accountability.roles`（角色 ID 数组） |
| `$CURRENT_ROLES.xxx` | `context.$CURRENT_ROLES.xxx` |
| `$CURRENT_POLICIES` | 策略 ID 数组 |
| `$CURRENT_POLICIES.xxx` | `context.$CURRENT_POLICIES.xxx` |
| `$NOW` / `$NOW(...)` | 当前时间戳（支持偏移量） |

> **注意**：动态变量在后端 `/permissions/me` 调用链中已被解析为实际值。App 端收到的 preset 中不会再有 `$CURRENT_USER.xxx` 这样的占位符——除非 preset 值在解析后仍然是字符串（即原始值不以 `$CURRENT_` 开头）。

---

## 3. mergePermissions 核心合并策略

源码：[merge-permissions.ts](api/src/permissions/utils/merge-permissions.ts#L12-L51)

### 3.1 三种策略

| 策略 | 行为 | 典型场景 |
|------|------|---------|
| `or` | 同一 `collection__action` 的权限宽松合并，任一满足即可访问 | 多角色策略合并 |
| `and` | 同一 `collection__action` 的权限严格合并，所有条件必须同时满足 | Share 权限与用户权限交叉 |
| `intersection` | 仅保留在所有列表中都存在的 `collection__action`，然后用 `and` 合并 | Share 场景中用户与分享目标的权限交集 |

### 3.2 合并流程

```
mergePermissions(strategy, ...permissionLists)
  │
  ├─ 如果 strategy === 'intersection':
  │   1. 取各列表的 collection__action 键集的交集
  │   2. 先用 'or' 策略去重各子列表
  │   3. 过滤只保留交集中的键
  │   4. 将 strategy 改为 'and' 继续处理
  │
  └─ 按 collection__action 分组
      对每组调用 mergePermission(strategy, current, new)
```

### 3.3 mergePermission 单条合并细节

源码：[merge-permissions.ts](api/src/permissions/utils/merge-permissions.ts#L53-L126)

#### permissions（行级过滤）合并

```
策略 = 'or' 时：
  current {} + new {status: 'published'}  → {}  （空对象 = 无限制，在 or 中优先级最高）
  current {author: 'me'} + new {status: 'published'}  → {_or: [{author: 'me'}, {status: 'published'}]}
  current {_or: [A, B]} + new C  → {_or: [A, B, C]}  （追加到已有 _or）

策略 = 'and' 时：
  current {author: 'me'} + new {status: 'published'}  → {_and: [{author: 'me'}, {status: 'published'}]}
  current {_and: [A, B]} + new C  → {_and: [A, B, C]}  （追加到已有 _and）
```

**核心规则**：在 `or` 策略下，空对象 `{}` 代表"无过滤条件"，会覆盖其他任何条件——因为"无限制 OR 有限制 = 无限制"。

#### validation（字段验证）合并

与 permissions 完全相同的逻辑，只是作用于 validation 字段。

#### fields 合并

委托给 `mergeFields`（见第 4 节）。

#### presets 合并

```typescript
if (newPerm.presets) {
    presets = merge({}, presets, newPerm.presets);  // lodash deep merge
}
```

- 后合入的策略的 preset 值覆盖先前的同名键
- 嵌套对象深度合并，不覆盖
- 这就是为什么策略排序（用户策略 > 角色策略）很重要

---

## 4. 字段合并策略 mergeFields

源码：[merge-fields.ts](api/src/permissions/utils/merge-fields.ts#L1-L27)

### 4.1 规则表

| 场景 | `and` 策略（交集） | `or` 策略（并集） |
|------|-------------------|------------------|
| 一方为 `[]`（空/无权限） | `[]`（无字段可见） | 返回另一方 |
| 一方含 `*`（全字段） | 返回另一方（全量 ∩ X = X） | `['*']`（全量 ∪ X = 全量） |
| 两方都含 `*` | `['*']` | `['*']` |
| 两方都是具体字段 | `intersection(A, B)` | `union(A, B)` |

### 4.2 关键语义

- `fields: null` 在 `mergeFields` 入口被当作 `[]` 处理
- `fields: []` 在 `and` 策略下意味着"没有任何字段可访问"——即操作虽然存在但无法操作任何字段
- `fields: ['*']` 在 `or` 策略下会吞并另一方——即"全字段 OR 限字段 = 全字段"
- 最终结果如果含 `*`，统一为 `['*']`

### 4.3 在 CollectionAccess 构建中的字段合并

源码：[fetch-accountability-collection-access.ts](api/src/permissions/modules/fetch-accountability-collection-access/fetch-accountability-collection-access.ts#L64-L70)

```typescript
if (perm.fields && info.fields?.[0] !== '*') {
    info.fields = uniq([...(info.fields || []), ...(perm.fields || [])]);
    if (info.fields.includes('*')) {
        info.fields = ['*'];
    }
}
```

注意这里的合并是**并集**语义（`uniq`），且一旦某条权限含 `*`，整体变为 `['*']`。这是因为 `/permissions/me` 返回的是"用户在此集合上能看到的字段的超集"——用户需要知道所有可能可见的字段。

> **与 mergeFields 的区别**：`mergeFields` 用于后端请求时合并多条策略的权限约束（可交集可并集），而 CollectionAccess 构建中用的是纯并集——因为前端需要知道"用户是否有机会看到这个字段"。

---

## 5. 三种访问级别的判定逻辑

源码：[fetch-accountability-collection-access.ts](api/src/permissions/modules/fetch-accountability-collection-access/fetch-accountability-collection-access.ts#L16-L79)

### 5.1 Admin 全量访问

```typescript
if (accountability.admin) {
    return mapValues(context.schema.collections, () =>
        Object.fromEntries(
            PERMISSION_ACTIONS.map((action) => [action, { access: 'full', fields: ['*'] }])
        )
    );
}
```

- **跳过所有策略查询和权限合并**
- 直接遍历 schema 中所有集合
- 每个集合的每个动作都是 `{ access: 'full', fields: ['*'] }`
- 没有 `presets`（admin 不需要默认值限制）

Admin 判定由 [is-admin.ts](api/src/utils/is-admin.ts#L1-L8) 完成：

```typescript
export function isAdmin(accountability?: Accountability | null) {
    if (accountability === null) return true;   // system 内部调用
    if (accountability?.admin === true) return true;
    return false;
}
```

### 5.2 Partial（条件访问）

```typescript
// 初始设为 'full'（因为有权限行存在）
infos[perm.collection]![perm.action]!.access = 'full';

// 如果有非空过滤条件，降级为 'partial'
if (info.access === 'full' && perm.permissions && Object.keys(perm.permissions).length > 0) {
    info.access = 'partial';
}
```

**判定条件**：
- 存在该 collection + action 的权限行
- 且该权限行的 `permissions` 字段是非空对象（有过滤条件）

**前端含义**：用户可以看到该集合/动作，但只能操作满足过滤条件的记录。

### 5.3 None（无访问）

```typescript
// 初始化时所有动作默认为 'none'
infos[perm.collection] = {
    read: { access: 'none' },
    create: { access: 'none' },
    update: { access: 'none' },
    delete: { access: 'none' },
    share: { access: 'none' },
};
```

**判定条件**：
- 没有任何策略为该 collection + action 提供权限行
- 或者用户不关联任何策略

**前端含义**：该集合/动作完全不可见。

### 5.4 三级判定流程图

```
对每个 (collection, action)：
  │
  ├─ 无权限行？ → access = 'none'
  │
  ├─ 有权限行，permissions === null？ → 不应出现（null 意味着无此行）
  │
  ├─ 有权限行，permissions === {} ？ → access = 'full'
  │
  └─ 有权限行，permissions 非空？ → access = 'partial'
```

> **注意**：同一 collection+action 如果有多条权限行，只要任一条的 `permissions` 非空，就会是 `partial`。`full` 要求所有行的 `permissions` 都为空对象 `{}`。

---

## 6. Preset 合并与解析

### 6.1 后端 Preset 合并

在 `mergePermission` 中：

```typescript
if (newPerm.presets) {
    presets = merge({}, presets, newPerm.presets);  // lodash deep merge
}
```

在 `fetchAccountabilityCollectionAccess` 中：

```typescript
if (perm.presets) {
    info.presets = { ...(info.presets ?? {}), ...perm.presets };
}
```

两层合并的区别：

| 层级 | 合并方式 | 深度 | 用途 |
|------|---------|------|------|
| `mergePermission` | `lodash.merge()` | 深度合并 | 合并同一 collection+action 的多条策略权限 |
| `CollectionAccess` 构建 | `{...spread}` | 浅合并 | 合并不同策略的 preset 到最终响应 |

> **后合入的策略优先**——这就是策略排序的意义。用户级策略的 preset 会覆盖角色级策略的同名键。

### 6.2 后端 Preset 动态变量解析

在 `processPermissions` 中：

```typescript
presets: parsePreset(permission.presets, accountability, permissionsContext)
```

`parsePreset` 使用 `deepMap` 遍历 preset 对象，将所有 `$CURRENT_USER.xxx` / `$CURRENT_ROLE.xxx` 等替换为从 DB 查到的实际值。

### 6.3 前端 Preset 二次解析

源码：[permissions.ts](app/src/stores/permissions.ts#L27-L38)

```typescript
this.permissions = mapValues(response.data.data, (collectionPermission) => {
    Object.values(collectionPermission).forEach((actionPermission) => {
        if (actionPermission.presets) {
            actionPermission.presets = parsePreset(actionPermission.presets);
        }
    });
    return collectionPermission;
});
```

前端 `parsePreset` 源码：[parse-preset.ts](app/src/utils/parse-preset.ts#L1-L22)

```typescript
export function parsePreset(preset) {
    const { currentUser } = useUserStore();
    const accountability = { role: currentUser.role?.id, user: currentUser.id };
    return parsePresetShared(preset, accountability, {
        $CURRENT_ROLE: currentUser.role,
        $CURRENT_USER: currentUser,
    });
}
```

> **为什么需要二次解析？** 后端 `processPermissions` 在 `/permissions/me` 调用链中已经解析了动态变量。但 App 端仍执行 `parsePreset`，这是为了处理边缘情况——例如后端因 `bypassDynamicVariableProcessing` 跳过了解析，或 preset 中嵌套了尚未展开的动态变量引用。前端的 `currentUser` 对象比后端解析时更完整（通过 `hydrateAdditionalFields` 补充了额外字段）。

---

## 7. 动态变量 Hydrate 机制

### 7.1 后端 Hydrate

源码：[fetch-dynamic-variable-data.ts](api/src/permissions/utils/fetch-dynamic-variable-data.ts#L14-L94)

```
extractRequiredDynamicVariableContextForPermissions(permissions)
  │
  ├─ 扫描 permissions/validation/presets 中的动态变量引用
  │   收集到 DynamicVariableContext:
  │   {
  │     $CURRENT_USER: Set<'id', 'email', ...>,
  │     $CURRENT_ROLE: Set<'id', ...>,
  │     $CURRENT_ROLES: Set<'id', ...>,
  │     $CURRENT_POLICIES: Set<'id', ...>,
  │   }
  │
  └─ fetchDynamicVariableData
      ├─ $CURRENT_USER → UsersService.readOne(user, { fields })
      ├─ $CURRENT_ROLE → RolesService.readOne(role, { fields })
      ├─ $CURRENT_ROLES → RolesService.readMany(roles, { fields })
      └─ $CURRENT_POLICIES → PoliciesService.readMany(policies, { fields })
```

**按需查询**：只查询权限规则中实际引用的字段，而非全量加载用户/角色数据。

### 7.2 前端 Hydrate

源码：[permissions.ts](app/src/stores/permissions.ts#L16-L25)

```typescript
async hydrate() {
    const response = await api.get('/permissions/me');
    const fields = getNestedDynamicVariableFields(response.data.data);

    if (fields.length > 0) {
        await userStore.hydrateAdditionalFields(fields);
    }

    this.permissions = mapValues(response.data.data, ...);
}
```

`getNestedDynamicVariableFields` 扫描 preset 中残留的 `$CURRENT_USER.xxx` 和 `$CURRENT_ROLE.xxx` 引用：

```typescript
if (value.startsWith('$CURRENT_USER.')) {
    fields.add(value.replace('$CURRENT_USER.', ''));
} else if (value.startsWith('$CURRENT_ROLE.')) {
    fields.add(value.replace('$CURRENT_ROLE.', 'role.'));
}
```

然后调用 [hydrateAdditionalFields](app/src/stores/user.ts#L96-L104)：

```typescript
const hydrateAdditionalFields = async (fields: string[]) => {
    const { data } = await api.get('/users/me', { params: { fields } });
    currentUser.value = merge({}, currentUser, data.data);
};
```

**流程**：
1. 从 `/permissions/me` 响应的 preset 中提取动态变量引用
2. 识别需要从用户/角色上获取的额外字段
3. 请求 `/users/me?fields=xxx,yyy` 获取这些字段
4. 合并到 `currentUser` 中
5. 用完整的 `currentUser` 重新解析 preset

---

## 8. App 端权限 Store 与 hasPermission

### 8.1 Permissions Store

源码：[permissions.ts](app/src/stores/permissions.ts#L10-L76)

```typescript
state: { permissions: {} as CollectionAccess }

actions:
  - hydrate()          // 请求 /permissions/me，解析 preset，补充 hydrate
  - dehydrate()        // 重置
  - getPermission(collection, action)  // 返回 ActionPermission | null
  - hasPermission(collection, action)  // 返回 boolean
```

### 8.2 hasPermission 实现逻辑

```typescript
hasPermission(collection, action) {
    const userStore = useUserStore();
    if (userStore.isAdmin) return true;
    return (this.getPermission(collection, action)?.access ?? 'none') !== 'none';
}
```

**判定矩阵**：

| 条件 | 结果 |
|------|------|
| `userStore.isAdmin === true` | `true`（不查 permissions） |
| `getPermission()` 返回 `null` | `false`（集合不在响应中） |
| `access === 'none'` | `false` |
| `access === 'partial'` | `true`（有条件访问） |
| `access === 'full'` | `true`（完全访问） |

> **关键**：`hasPermission` 不区分 `partial` 和 `full`——它只关心"是否有权限"。区分两者是 UI 层的职责。

### 8.3 isAdmin 前端判定

源码：[user.ts](app/src/stores/user.ts#L26)

```typescript
const isAdmin = computed(() => unref(currentUser)?.admin_access === true || false);
```

`admin_access` 来自 `/policies/me/globals` 端点的响应，在 user store hydrate 时合并到 `currentUser`。

### 8.4 useCollectionPermissions 组合式函数

源码：[use-collection-permissions.ts](app/src/composables/use-permissions/collection/use-collection-permissions.ts#L19-L38)

```typescript
export function useCollectionPermissions(collection) {
    const readAllowed = isActionAllowed(collection, 'read');
    const createAllowed = isActionAllowed(collection, 'create');
    const updateAllowed = isActionAllowed(collection, 'update');
    const deleteAllowed = isActionAllowed(collection, 'delete');
    const sortAllowed = isSortAllowed(collection);
    const archiveAllowed = isArchiveAllowed(collection, updateAllowed);
    const revisionsAllowed = isRevisionsAllowed();
    return { readAllowed, createAllowed, updateAllowed, deleteAllowed,
             sortAllowed, archiveAllowed, revisionsAllowed };
}
```

其中 `isActionAllowed` 源码：[is-action-allowed.ts](app/src/composables/use-permissions/collection/lib/is-action-allowed.ts#L6-L16)

```typescript
export const isActionAllowed = (collection, action) => {
    const { hasPermission } = usePermissionsStore();
    return computed(() => {
        if (!collectionValue) return false;
        return hasPermission(collectionValue, action);
    });
};
```

---

## 9. 前端可见字段控制 isFieldAllowed

源码：[is-field-allowed.ts](app/src/composables/use-permissions/utils/is-field-allowed.ts#L1-L7)

```typescript
export const isFieldAllowed = (permission: ActionPermission, field: string) => {
    if (!permission.fields) return false;
    return permission.fields.includes('*') || permission.fields.includes(field);
};
```

**语义**：
- `fields` 为 `undefined` / `null` → 不可见任何字段
- `fields` 含 `'*'` → 所有字段可见
- `fields` 为具体列表 → 只有列表中的字段可见

### isSortAllowed 示例

源码：[is-sort-allowed.ts](app/src/composables/use-permissions/collection/lib/is-sort-allowed.ts#L8-L27)

```typescript
export const isSortAllowed = (collection) => {
    const userStore = useUserStore();
    const { getPermission } = usePermissionsStore();

    return computed(() => {
        if (userStore.isAdmin) return true;
        const permission = getPermission(collectionValue, 'update');
        if (!permission) return false;
        return isFieldAllowed(permission, sortField);
    });
};
```

排序需要 `update` 权限且排序字段在允许列表中。

---

## 10. 后端策略变化对前端可见行为的影响

### 10.1 mergePermissions 策略切换

| 变更方向 | 对 CollectionAccess 的影响 | 对前端可见行为的影响 |
|---------|---------------------------|-------------------|
| 策略从 `or` 变为 `and` | `permissions` 从 `_or` 变为 `_and`，`access` 从 `full` 降为 `partial` | 更多集合显示为"条件访问"；部分集合从可见变为不可见 |
| 策略从 `and` 变为 `or` | `permissions` 从 `_and` 变为 `_or`，`access` 可能从 `partial` 升为 `full` | 更少条件限制；部分集合从 `partial` 变为 `full` |
| `intersection` 变为 `or` | 保留更多 collection__action 对 | 更多集合/动作出现在前端 |
| `or` 变为 `intersection` | 只保留公共的 collection__action | 前端大量集合/动作消失 |

### 10.2 fields 合并策略变化

| 变更 | 影响 |
|------|------|
| 从 `union` 变为 `intersection` | `fields` 列表缩短，更多字段在前端被 `isFieldAllowed` 判定为不可见 |
| 从 `intersection` 变为 `union` | `fields` 列表增长，更多字段可见 |
| 引入 `null` 处理变化 | 如果 `null` 从"无字段"变为"全字段"，会导致 `isFieldAllowed` 行为反转 |
| `['*']` 语义变化 | 前端 `isFieldAllowed` 的 `includes('*')` 分支会受影响 |

### 10.3 preset 合并策略变化

| 变更 | 影响 |
|------|------|
| 合并顺序改变 | 后合入的策略 preset 覆盖前者——若用户策略与角色策略的合入顺序反转，preset 默认值会改变 |
| 从 deep merge 变为 shallow merge | 嵌套 preset 值可能被整体替换而非字段级合并 |
| preset 动态变量解析时机改变 | 如果后端不再解析动态变量，前端 `parsePreset` 需承担全部解析职责，且需确保 `hydrateAdditionalFields` 收集到足够字段 |

### 10.4 access 级别判定变化

| 变更 | 前端影响 |
|------|---------|
| `full` 判定条件收紧 | 原本 `full` 的集合变为 `partial`，前端 UI 可能显示过滤指示器 |
| `none` 判定条件放宽 | 原本不可见的集合变为可见 |
| 新增 `access` 级别 | 前端类型系统需同步更新，否则未知值被当作 `none` |

### 10.5 Admin 绕过路径

Admin 用户的权限完全由 `accountability.admin === true` 决定，不经过任何策略合并逻辑。因此：

- **mergePermissions 策略变化不影响 Admin 用户**
- **appAccessMinimalPermissions 不影响 Admin 用户**
- Admin 的前端 `hasPermission` 始终返回 `true`，`isFieldAllowed` 不会被调用（因为 `isAdmin` 直接短路）

### 10.6 影响传播链路总结

```
后端 mergePermissions 策略变化
  │
  ├─ 影响 fetchAccountabilityCollectionAccess 的输出
  │   ├─ access: 'none' | 'partial' | 'full'
  │   ├─ fields: string[] | undefined
  │   └─ presets: Record<string, any> | undefined
  │
  ├─ → /permissions/me 响应变化
  │
  └─ → App permissions store hydrate 结果变化
      │
      ├─ hasPermission → 控制 CRUD 按钮显示/隐藏
      ├─ getPermission.access === 'partial' → 控制 UI 过滤提示
      ├─ isFieldAllowed → 控制字段可见性（表单字段、列表列）
      └─ parsePreset → 控制新建记录的默认值
```

---

## 附录：关键源码文件索引

| 文件 | 职责 |
|------|------|
| [merge-permissions.ts](api/src/permissions/utils/merge-permissions.ts) | 核心权限合并逻辑 |
| [merge-fields.ts](api/src/permissions/utils/merge-fields.ts) | 字段列表合并 |
| [fetch-permissions.ts](api/src/permissions/lib/fetch-permissions.ts) | 权限获取管线（含动态变量处理） |
| [fetch-raw-permissions.ts](api/src/permissions/utils/fetch-raw-permissions.ts) | 原始权限查询 + App 最小权限追加 |
| [fetch-policies.ts](api/src/permissions/lib/fetch-policies.ts) | 策略收集与排序 |
| [fetch-accountability-collection-access.ts](api/src/permissions/modules/fetch-accountability-collection-access/fetch-accountability-collection-access.ts) | /permissions/me 核心：构建 CollectionAccess |
| [extract-required-dynamic-variable-context.ts](api/src/permissions/utils/extract-required-dynamic-variable-context.ts) | 动态变量引用提取 |
| [fetch-dynamic-variable-data.ts](api/src/permissions/utils/fetch-dynamic-variable-data.ts) | 动态变量数据获取 |
| [process-permissions.ts](api/src/permissions/utils/process-permissions.ts) | 动态变量替换 |
| [parse-filter.ts](packages/utils/shared/parse-filter.ts) | Filter/Preset 动态变量解析 |
| [with-app-minimal-permissions.ts](api/src/permissions/lib/with-app-minimal-permissions.ts) | App 最小权限注入 |
| [app-access-permissions/index.ts](packages/system-data/src/app-access-permissions/index.ts) | App 最小权限定义 |
| [is-admin.ts](api/src/utils/is-admin.ts) | Admin 判定 |
| [permissions.ts (controller)](api/src/controllers/permissions.ts) | /permissions/me 路由 |
| [permissions.ts (store)](app/src/stores/permissions.ts) | App 权限 Store |
| [user.ts (store)](app/src/stores/user.ts) | App 用户 Store + hydrateAdditionalFields |
| [parse-preset.ts](app/src/utils/parse-preset.ts) | 前端 preset 解析 |
| [permissions.ts (types)](app/src/types/permissions.ts) | ActionPermission / CollectionPermission 类型 |
| [permissions.ts (types package)](packages/types/src/permissions.ts) | CollectionAccess / CollectionPermissions 类型 |
| [is-field-allowed.ts](app/src/composables/use-permissions/utils/is-field-allowed.ts) | 字段可见性判定 |
| [is-action-allowed.ts](app/src/composables/use-permissions/collection/lib/is-action-allowed.ts) | 动作可见性判定 |
| [use-collection-permissions.ts](app/src/composables/use-permissions/collection/use-collection-permissions.ts) | 集合级权限组合式函数 |
| [get-permissions-for-share.ts](api/src/permissions/utils/get-permissions-for-share.ts) | Share 场景权限合并 |
