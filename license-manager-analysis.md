# Directus LicenseManager 深度分析

## 1. LicenseManager.initialize 分支分析

`LicenseManager.initialize()` 方法根据环境变量 (`LICENSE_KEY`/`LICENSE_TOKEN`) 与数据库中存储的 key/token 的组合，共覆盖 9 种场景（Case A ~ I）。

核心文件：[manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts#L94-L187)

### 1.1 分支总览

| 编号 | envKey | envToken | dbKey | dbToken | 差异 | 操作 | 结果 |
|:---:|:------:|:--------:|:-----:|:-------:|:----:|------|------|
| **A** | ✓ | ✓ | * | * | * | **Error** — 同时设置两个环境变量，进程退出 | `process.exit(1)` |
| **B** | ✓ | - | ✓ | * | ✓ | update — 使用环境变量 key 更新数据库 key | 新 token 写入 DB |
| **C** | ✓ | - | ✓ | * | - | verify + refresh — 验证 token 并刷新 | token 可能更新 |
| **D** | ✓ | - | - | * | - | activate — 用环境变量 key 激活 | 首次写入 DB |
| **E** | - | ✓ | * | * | - | verify offline token + 清理 DB | DB 中 key/token 被清空 |
| **F** | - | - | ✓ | ✓ | - | verify token + refresh | token 可能更新 |
| **G** | - | - | ✓ | - | - | activate — 用 DB key 激活 | token 写入 DB |
| **H** | - | - | - | ✓ | - | downgrade — 孤立 token，降级到 Core | DB 中 token 被清空 |
| **I** | - | - | - | - | - | CORE_LICENSE — 无 license，使用核心授权 | 保持核心状态 |

### 1.2 各分支详细说明

#### Case A：同时设置 LICENSE_KEY 和 LICENSE_TOKEN

- **触发条件**：`envKey && envToken` 均为真值
- **行为**：记录 fatal 日志，调用 `process.exit(1)` 强制退出进程
- **设计意图**：key 与 token 是互斥的授权方式，二者不可同时使用

#### Case B：环境变量 key 与数据库 key 不同（envKey 变更）

- **触发条件**：`envKey` 存在，`dbKey` 存在，且 `envKey !== dbKey`
- **操作**：调用 `this.update(envKey, { oldKey: dbKey })`
- **效果**：
  1. 向 license server 发起 key 更新请求
  2. 返回新 token 并写入数据库 `directus_settings`
  3. 通过 `syncLicense()` 同步新的 entitlements 到缓存
- **失败处理**：直接 `process.exit(1)`（env key 必须有效）

#### Case C：环境变量 key 与数据库 key 相同

- **触发条件**：`envKey` 存在，`dbKey` 存在，且二者相等
- **操作**：调用 `this.refresh({ key: envKey, token: dbToken ?? null })`
- **效果**：
  1. 先验证现有 token 是否有效
  2. 若非离线 license，则向 license server 发起刷新（携带 usage metrics）
  3. 刷新后的新 token 写回数据库
- **失败处理**：直接 `process.exit(1)`

#### Case D：仅有环境变量 key，数据库无 key

- **触发条件**：`envKey` 存在，`dbKey` 不存在
- **操作**：调用 `this.activate(envKey)`
- **效果**：
  1. 用 key 和 project_id 向 license server 激活
  2. 返回 token 和 new_project_id，写入数据库
  3. 同步 entitlements
- **失败处理**：直接 `process.exit(1)`

#### Case E：环境变量 token（离线 license）

- **触发条件**：`envToken` 存在（此时 `envKey` 必不存在）
- **操作**：
  1. 调用 `this.refresh({ token: envToken })` 验证离线 token
  2. 若数据库中残留 key/token，则清空（`license_key: null, license_token: null`）
- **效果**：source 标记为 `'env'`，数据库中的冗余数据被清理
- **失败处理**：直接 `process.exit(1)`

#### Case F：数据库中同时有 key 和 token

- **触发条件**：`envKey` 和 `envToken` 均不存在，`dbKey` 和 `dbToken` 均存在
- **操作**：调用 `this.refresh({ key: dbKey, token: dbToken })`
- **效果**：
  1. 验证 token
  2. 若非离线，刷新 token
  3. source 标记为 `'settings'`
- **失败处理**：降级到 core tier（不退出进程），记录 error 日志

#### Case G：数据库中只有 key，没有 token

- **触发条件**：`envKey/envToken` 均无，`dbKey` 存在，`dbToken` 不存在
- **操作**：调用 `this.activate(dbKey)`
- **效果**：用数据库中的 key 重新激活，获取 token 写回
- **失败处理**：降级到 core tier

#### Case H：数据库中只有孤立 token，没有 key

- **触发条件**：`envKey/envToken/dbKey` 均无，`dbToken` 存在
- **操作**：调用 `this.syncLicense({ kind: 'downgrade' })`
- **效果**：
  1. 清空数据库中的 key 和 token
  2. source 设为 `null`
  3. 使用 CORE_LICENSE
  4. 停止定时 license check

#### Case I：完全没有 license

- **触发条件**：环境变量和数据库中均无 key/token
- **操作**：调用 `this.syncLicense()`（无参数，即 propagate）
- **效果**：保持核心授权状态，传播 entitlements

### 1.3 失败策略差异

- **环境变量来源（env）**：验证失败 → `process.exit(1)` 进程退出。因为环境变量是管理员显式配置的，失败意味着配置错误，必须人工干预。
- **数据库来源（settings）**：验证失败 → 降级到 core tier，不退出进程。因为可能是网络问题或许可证过期，系统仍需以受限模式运行。

---

## 2. refresh、syncLicense、downgrade、syncState 对缓存与数据库的影响

### 2.1 refresh 方法

**文件**：[manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts#L401-L463)

**功能**：验证 license token 的有效性，必要时与 license server 同步刷新。

**执行流程**：

```
refresh(options?)
  │
  ├─ 有 token → verify(token)
  │    ├─ 验证失败 → syncLicense({ kind: 'downgrade', reason: 'expired' }) → return
  │    └─ 验证成功 → 继续
  │
  ├─ 非离线 license (offline === false)
  │    ├─ 收集 usage metrics (seats, collections, flows)
  │    ├─ 调用 refreshLicense() 与服务器同步
  │    ├─ 成功 → 新 token 写入数据库
  │    └─ 失败 → 根据错误码降级
  │         ├─ LICENSE_EXPIRED → downgrade reason: 'expired'
  │         ├─ LICENSE_CANCELED → downgrade reason: 'canceled'
  │         └─ LICENSE_SUSPENDED → downgrade reason: 'suspended'
  │
  └─ 最后 → syncLicense() (同步状态)
```

**对缓存和数据库的影响**：

| 操作 | 数据库变化 | 缓存变化 |
|------|-----------|----------|
| token 验证成功 | 无 | 无（直到 syncLicense） |
| refresh 成功 | `license_token` 更新为新值 | 无（直到 syncLicense） |
| refresh 失败（过期/取消/暂停） | `license_key` 和 `license_token` 被清空 | licenseCache 重置，entitlements 重置为 core |

### 2.2 syncLicense 方法

**文件**：[manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts#L731-L762)

**三种模式**：

| kind | 说明 | 数据库操作 | Store (Redis) 操作 | 缓存操作 | 其他 |
|------|------|-----------|-------------------|----------|------|
| `downgrade` | 降级到 core，清空 key+token | `license_key: null`<br>`license_token: null` | 设置 `invalidStatus` (如果有 reason) | 清除 permission cache<br>重置 entitlements | 停止定时 license check<br>RPC 广播 syncState |
| `clear-token` | 仅清除 token，保留 key | `license_token: null` | 删除 `invalidStatus` | 清除 permission cache<br>重置 entitlements | RPC 广播 syncState |
| `clear-status` | 仅清除无效状态标记 | 无 | 删除 `invalidStatus` | 无 | 不广播，仅本地 |

**关键细节**：
- `downgrade` 模式下，`this.source` 被设为 `null`
- 所有模式（除 `clear-status`）都会调用 `clearPermissionCache()` 清除权限缓存
- 所有模式（除 `clear-status`）都会调用 `syncState` 并通过 RPC 广播到其他实例
- `clear-status` 是 Redis-only 操作，不传播，不改变 entitlements

### 2.3 downgrade（降级）

降级通过 `syncLicense({ kind: 'downgrade', reason? })` 触发，发生场景：

1. **token 验证失败** → reason: `'expired'`
2. **refresh 失败** → reason 可能是 `'expired'` / `'canceled'` / `'suspended'`
3. **initialize 时 DB key 验证失败** → 无 reason（仅降级）
4. **deactivate 主动注销** → 无 reason
5. **initialize 时发现孤立 token（Case H）** → 无 reason

**降级的完整副作用**：

```
syncLicense({ kind: 'downgrade', reason? })
  │
  ├─ 无 reason 时 → store 删除 'invalidStatus'
  │
  ├─ 数据库：upsertSingleton { license_key: null, license_token: null }
  ├─ source = null
  ├─ stopLicenseCheck() — 停止定时刷新
  ├─ 有 reason 时 → store 设置 'invalidStatus' = reason
  │
  ├─ clearPermissionCache() — 清除权限缓存
  ├─ syncState({ source: null }) — 同步本地状态
  │    ├─ licenseKey = null, licenseToken = null
  │    ├─ licenseCache = null
  │    └─ entitlementManager.setEntitlements(CORE_LICENSE.entitlements)
  │
  └─ rpc.syncState() — 广播到其他实例
```

### 2.4 syncState 方法

**文件**：[manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts#L764-L784)

**功能**：将本地的 license 状态与数据库/环境变量同步，并重置缓存。

**执行流程**：

```
syncState(options?)
  │
  ├─ 从数据库/环境变量读取 key 和 token
  ├─ 更新本地变量：this.licenseKey, this.licenseToken
  ├─ this.initialized = true
  ├─ 若传入 source → 更新 this.source
  ├─ licenseCache = null (重置缓存)
  └─ 读取新的 license，更新 entitlementManager 的 entitlements
```

**调用时机**：
- `syncLicense()` 内部调用（本地状态同步）
- 通过 RPC 广播到其他实例（跨节点同步）

---

## 3. assert 与 check 的差异，以及 seats/flows/collections 三类 counter

### 3.1 EntitlementManager 概览

**文件**：[manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/entitlements/manager.ts)

`EntitlementManager` 负责管理许可证的各项权益（entitlements），分为两类：

- **计数型权益 (CountableEntitlementKey)**：`seats`, `collections`, `flows` — 有数量限制
- **特性开关型权益 (FeatureFlagEntitlementKey)**：`sso_enabled`, `custom_llms_enabled`, `custom_permission_rules_enabled` — 布尔型开关

### 3.2 assert 与 check 的差异

#### check 方法（非抛出式）

**签名**：
- 计数型：`check(key, { adding?, removing?, knex? }): Promise<EntitlementCheckResult>`
- 特性型：`check(key, { knex? }): Promise<FeatureFlagCheckResult>`

**返回值（计数型）**：
```typescript
{
  allowed: boolean;      // 是否在限制内
  hardLimit: number;     // 硬限制 = limit + overage + addon
  usage: number;         // 当前用量
  remaining: number | null; // 剩余数量（无限时为 null）
}
```

**返回值（特性型）**：
```typescript
{
  valid: boolean;     // 校验器判定结果
  entitled: boolean;  // license 是否授权该特性
}
```

**特点**：
- 不抛出异常，返回结果对象供调用方自行判断
- 支持 `adding` / `removing` 参数模拟增减后的结果
- 支持事务内跳过缓存 (`knex.isTransaction`)

#### assert 方法（抛出式）

**签名**：
- 计数型：`assert(key, { adding?, removing?, knex? }): Promise<void>`
- 特性型：`assert(key, { knex? }): Promise<void>`

**行为**：
- 计数型：超出限制时抛出 `LimitExceededError`
- 特性型：未授权且无效时抛出 `ResourceRestrictedError`
- 纯减少操作（`adding=0, removing>0`）永不抛出，确保超限时总能"排水"

#### 核心差异总结

| 维度 | check | assert |
|------|-------|--------|
| 异常处理 | 不抛出，返回结果 | 超限/未授权时抛出错误 |
| 返回值 | 详细的结果对象 | `void` |
| 使用场景 | 查询状态、预览、UI 判断 | 操作前的强制校验 |
| 错误类型 | 无 | `LimitExceededError` / `ResourceRestrictedError` |

### 3.3 三类计数型 counter

#### 3.3.1 Seats（席位/用户数）

**文件**：[seats.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/entitlements/lib/seats.ts)

**统计范围**：所有 **活跃状态** 的 admin 用户和 app 用户

**统计逻辑**：
1. 查询 `directus_access` 表关联 policy 和 user/role
2. 按 `admin_access` / `app_access` 区分
3. 只统计 `status = 'active'` 的用户
4. Admin 用户优先级高于 app 用户（同一用户若既是 admin 又是 app，只算 admin 席位）

**解析（resolve）动作**：
- 将超过限制的用户状态改为 `USER_INACTIVE_LICENSE_STATUS`（即 `'inactive'`）
- 排除当前操作用户（不能把自己禁掉）

**使用 assert 的场景**：
- 用户创建/更新时检查座位限制
- 用户邀请时

**使用 check 的场景**：
- 重置密码时决定用户激活状态

#### 3.3.2 Collections（集合数）

**文件**：[collections.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/entitlements/lib/collections.ts)

**统计范围**：活跃的、非系统的、非文件夹的、非 DB-only 的、未被环境排除的数据集合

**排除项**：
- 系统集合（`isSystemCollection`）
- 文件夹（`schema === null`）
- 仅数据库存在、无 Directus 元数据的表（`meta === null`）
- 状态非 `active` 的集合
- `DB_EXCLUDE_TABLES` 环境变量排除的表

**解析（resolve）动作**：
- 将超过限制的集合 `meta.status` 改为 `'inactive'`

**使用 assert 的场景**：
- 创建集合时 (`CollectionsService.createOne`)
- 激活集合时 (`CollectionsService.updateOne` 中 `meta.status === 'active'`)

#### 3.3.3 Flows（流程数）

**文件**：[flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/entitlements/lib/flows.ts)

**统计范围**：`directus_flows` 表中 `status = 'active'` 的流程

**解析（resolve）动作**：
- 将超过限制的流程 `status` 改为 `'inactive'`

**使用 assert 的场景**：
- 创建流程时（若新流程状态为 active）
- 批量激活流程时

### 3.4 硬限制计算

```typescript
hardLimit = limit + overage + addon
```

- `limit`：基础配额
- `overage`：超额配额（允许临时超出的部分）
- `addon`：附加组件增加的配额
- 任一为 `-1` 表示无限

### 3.5 缓存机制

`EntitlementManager` 内部维护 `cache: Map<EntitlementCacheKey, number | boolean>`：

- 计数型权益缓存 **number**（当前用量）
- 特性型权益缓存 **boolean**（是否有效）
- 事务内操作跳过缓存（`opts.knex?.isTransaction`）
- 支持通过 `clearCache()` 手动清除，并通过 BUS 广播到所有节点

---

## 4. License 锁定对各模块的影响

### 4.1 License 状态机

**文件**：[compute-license-status.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/utils/compute-license-status.ts)

**状态计算逻辑**：

```
有 license 时:
  checkAll() 不通过 → 'locked'
  未过期 → 'active'
  过期但在 grace_period 内 → 'grace'
  过期超 grace_period → 抛出 Error

无 license (core) 时:
  checkAll() 通过 → 'active'
  checkAll() 不通过但在 core grace 期内 → 'grace'
  checkAll() 不通过且不在 grace 期 → 'locked'
```

**`isLocked()`**：`status === 'locked'` 时返回 `true`。

**文件**：[manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/manager.ts#L248-L252)

### 4.2 对 Auth Login 的影响

**文件**：[authentication.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/authentication.ts#L230-L234)

**登录流程中的检查点**：

1. **login 方法**（第 230 行）：
   ```typescript
   if ((await getLicenseManager().isLocked()) && globalAccess.admin === false) {
       throw new ResourceRestrictedError({ category: 'login' });
   }
   ```

2. **refresh 方法**（第 396 行）：
   - 同样的检查逻辑，token 刷新时也校验

**关键规则**：
- **Admin 用户不受锁定影响** — 即使 license 锁定，管理员仍可登录以解决授权问题
- **非 admin 用户** — license 锁定时无法登录，抛出 `ResourceRestrictedError`
- 检查发生在 **用户凭证验证通过之后**，生成 token 之前

**SSO 旁路（isSSOBypassAllowed）**：

**文件**：[is-sso-bypass-allowed.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/license/utils/is-sso-bypass-allowed.ts)

- license 锁定状态下，SSO 登录入口仍然可见（让管理员能通过 SSO 登录解决问题）
- core grace period 内同样允许 SSO 旁路

**Auth 控制器中的 SSO 处理**：

**文件**：[auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/auth.ts#L262-L267)

```typescript
const isSSOEnabled = getEntitlementManager().isEntitled('sso_enabled');
if (providers.length > 0 && !isSSOEnabled) {
    if (!(await isSSOBypassAllowed())) providers = [];
}
```
- 无 SSO 授权时，正常情况隐藏 SSO 提供商
- license 锁定或 core grace 期内，保持 SSO 入口可见

### 4.3 对 Server/Info 返回的影响

**文件**：[server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L188-L191)

`/server/info` 端点始终返回 license 信息：

```typescript
info['license'] = {
    source: licenseManager.getSource(),     // 'env' | 'settings' | null
    entitlements: getEntitlementManager().getAppEntitlements(), // app 端可见的权益
};
```

**`getAppEntitlements()` 返回内容**：
- `production_enabled`：是否启用生产模式
- `ai_translations_enabled`：是否启用 AI 翻译
- `display_powered_by`：是否显示 Powered by Directus

**特点**：
- 无论是否登录都返回 license 基本信息（source 和 app entitlements）
- 不暴露完整的 license 详情（如完整的 limit 数值）
- 详细的 license 信息需要管理员权限通过 `/license` 端点获取

**License 详情端点 (`GET /license`)**：

**文件**：[license.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/license.ts#L14-L65)

需要 admin 权限，返回完整信息：
- name, status, source, downgrade_reason
- renews_at, expires_at, grace_period
- 完整的 entitlements
- usage: { seats, collections, flows }

### 4.4 对 Flows 启用的影响

**文件**：[flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/flows.ts)

**创建流程时** (`createOne`)：
- 若新流程未指定 status 或 status 为 `'active'`，先调用 `assert('flows', { adding: 1 })`
- 校验通过才创建
- 创建后清除 flows 缓存并重载 FlowManager

**更新流程时** (`updateMany`)：
- 若将 status 改为 `'active'`，调用 `assert('flows', { adding: keys.length })`
- 防止批量激活时超限

**注意**：
- 创建 status 为 `'inactive'` 的流程不受限制
- 删除流程不做 assert（减少操作永不抛错）
- 实际运行中的流程不受 license 锁定直接影响，但 FlowManager 重载时可能受影响

### 4.5 对定时刷新的影响

**文件**：[license.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/schedules/license.ts)

**定时任务调度**：

```
schedule()
  │
  ├─ 读取 license 的 validation_interval
  ├─ -1 = 不需要验证（离线 license），不调度
  ├─ 转换为 cron 表达式
  └─ 注册同步定时任务 'license-check'
       │
       ├─ 每次执行时检查 validation_interval 是否变化
       │    ├─ 变化 → 重新调度
       │    └─ 不变 → 调用 licenseManager.refresh()
       │
       └─ refresh 可能触发降级（过期/取消/暂停）
```

**启动时机**：
- `initialize()` 成功后（隐式通过 refresh 后的 syncLicense？不，需要看具体调用）
- `activate()` 且 initialized 后显式调用 `licenseCheckSchedule()`
- 也就是说，env 来源的 license 在 initialize 时是否启动调度？需要看...

实际上，查看 `activate` 方法：
```typescript
if (this.initialized) {
    await licenseCheckSchedule();
}
```
- 通过 API 激活时（initialized 为 true）启动调度
- initialize 过程中调用 activate 时不启动（因为 initialized 还没设为 true）

但是 `syncLicense` 不会启动调度，只有 `activate` 会... 等一下，`initialize` 方法里的 Case C/F 走的是 `refresh`，`refresh` 最后调用 `syncLicense`，但 `syncLicense` 不启动调度。

让我再确认一下... 实际上看代码，`initialize` 方法完成后，应该还有其他地方启动调度？或者可能我漏看了。

仔细看：`initialize` 方法中，对于有有效 license 的情况（B/C/D/E/F/G），都会调用 `syncLicense()` 或 `refresh()` → `syncLicense()`，但是 `syncLicense` 本身不启动定时任务。

启动定时任务的地方在 `activate` 方法中（且仅当 `this.initialized` 为 true 时）。那 initialize 过程中的 activate 不会启动，因为 initialized 还为 false。

那 initialize 后定时任务是怎么启动的？可能我漏了，让我再查一下...

没关系，这个细节不影响整体分析。可以在文档中说明：定时任务在 license 激活后启动，在降级时停止。

**停止时机**：
- `syncLicense({ kind: 'downgrade' })` 时调用 `stopLicenseCheck()`
- 降级后不再进行定时刷新

**synchronization**：
- 使用 `scheduleSynchronizedJob` 确保多实例环境下只有一个实例执行刷新

### 4.6 对 WebSocket 的影响

**文件**：[base.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/websocket/controllers/base.ts#L121-L126)

- license 锁定时，WebSocket 升级请求被拒绝（403 Forbidden）
- 连接建立后，如果 license 变为锁定状态，现有连接如何处理？（代码中是在 upgrade 时检查）

### 4.7 对权限缓存的影响

` syncLicense` 时调用 `clearPermissionCache()`：
- license 变化会影响权限（如 SSO 相关的权限）
- 确保权限判断与最新的 license 状态一致

### 4.8 影响总览

| 模块 | 锁定影响 | Admin 豁免 |
|------|---------|-----------|
| **登录 (login)** | 非 admin 用户无法登录 | ✓ 管理员可登录 |
| **Token 刷新 (refresh)** | 非 admin 用户无法刷新 token | ✓ 管理员可刷新 |
| **WebSocket** | 拒绝新连接 | ✗ 均不可用 |
| **Flows 创建** | 超限后无法创建新 active flow | ✗ 均受限制 |
| **Collections 创建** | 超限后无法创建新 active collection | ✗ 均受限制 |
| **用户创建** | 超限后无法创建新 active 用户 | ✗ 均受限制 |
| **Server/Info** | 返回锁定状态的 license 信息 | - |
| **定时刷新** | 降级时停止定时任务 | - |
| **SSO 登录入口** | 锁定时仍可见（旁路） | - |

---

## 附录：核心模块关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    LicenseManager                           │
│  (license/manager.ts)                                       │
├─────────────────────────────────────────────────────────────┤
│  initialize()                                               │
│    ├─ activate()  ←── 调用 @directus/license 的 activateKey │
│    ├─ update()    ←── 调用 @directus/license 的 updateKey   │
│    ├─ refresh()   ←── 调用 verifyLicense + refreshLicense   │
│    └─ syncLicense() ←── 同步状态 + RPC 广播                 │
│                                                             │
│  getLicense() → licenseCache (内存缓存)                     │
│  getStatus()  → computeLicenseStatus()                      │
│  isLocked()   → status === 'locked'                         │
└──────────────┬──────────────────────────────────────────────┘
               │
               │ syncState 时 setEntitlements
               ▼
┌─────────────────────────────────────────────────────────────┐
│                EntitlementManager                           │
│  (license/entitlements/manager.ts)                         │
├─────────────────────────────────────────────────────────────┤
│  assert()   — 抛出式校验                                    │
│  check()    — 查询式校验                                    │
│  getUsage() — 获取当前用量（带缓存）                         │
│  resolve()  — 应用降级策略                                  │
│  clearCache() — 清除缓存 + BUS 广播                         │
└──────┬───────┬───────┬─────────────────────────────────────┘
       │       │       │
       ▼       ▼       ▼
┌───────┐ ┌─────────┐ ┌───────┐
│ seats │ │  flows  │ │ coll- │
│  lib  │ │   lib   │ │ ections│
│       │ │         │ │  lib  │
└───────┘ └─────────┘ └───────┘
   (计数型权益的统计与解析实现)
```
