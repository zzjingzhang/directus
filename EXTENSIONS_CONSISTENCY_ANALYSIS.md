# Directus Registry Extension 一致性机制分析

本文档详细分析通过 `/extensions` 接口安装、重装、卸载或 patch 一个 registry extension 时，`ExtensionService` 和 `ExtensionManager` 如何保持数据库 settings、bundle 子项、filesystem 包和 App source chunk 的一致性。

---

## 1. 核心组件概览

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| `ExtensionsService` | [extensions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts) | 业务逻辑层，协调数据库操作与扩展管理器 |
| `ExtensionManager` | [manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts) | 扩展核心管理器，负责加载、注册、重载扩展 |
| `InstallationManager` | [manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/installation/manager.ts) | 文件系统层面的安装/卸载 |
| Extensions Controller | [extensions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts) | HTTP 路由层，定义接口与权限 |

---

## 2. 四种操作的完整流程

### 2.1 安装 (Install)

**接口**: `POST /extensions/registry/install`

**执行顺序**（自上而下）：

| 步骤 | 执行位置 | 操作内容 |
|------|----------|----------|
| 1 | Controller [L184-L206](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L184-L206) | Admin 权限校验 → 调用 `service.install(extensionId, versionId)` |
| 2 | `ExtensionsService.preInstall()` [L41-L77](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L41-L77) | 描述扩展信息 → EXTENSIONS_LIMIT 计数校验 |
| 3 | `ExtensionsService.install()` [L79-L102](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L79-L102) | 写入数据库 `directus_extensions` 表（主扩展） → 若为 bundle，批量写入子项 |
| 4 | `ExtensionManager.install()` [L231-L248](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L231-L248) | 调用 `InstallationManager.install()` 下载解压到 `.registry/` |
| 5 | `ExtensionManager.broadcastReloadNotification()` [L269-L271](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L269-L271) | 广播 `extensions.reload` 消息到其他进程 |
| 6 | `ExtensionManager.reload({ skipSync: true })` [L338-L398](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L338-L398) | `unload()` → `load()` → `generateExtensionBundle()` 重建 App Chunk |
| 7 | Event | 触发 `extensions.installed` action |

**一致性保证**:
- **数据库 settings**: 步骤 3 先写入 DB（主扩展 + bundle 子项）
- **Bundle 子项**: 步骤 3 通过 `createMany` 批量写入，每条记录的 `bundle` 字段指向父扩展 ID
- **Filesystem 包**: 步骤 4 下载 tarball 解压到 `extensions/.registry/{versionId}/`
- **App source chunk**: 步骤 6 的 `generateExtensionBundle()` 重新 Rollup 打包所有 App 扩展到 `$TEMP_PATH/app-extensions/`

---

### 2.2 重装 (Reinstall)

**接口**: `POST /extensions/registry/reinstall`

**执行顺序**：

| 步骤 | 执行位置 | 操作内容 |
|------|----------|----------|
| 1 | Controller [L208-L230](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L208-L230) | Admin 校验 → 调用 `service.reinstall(id)` |
| 2 | `ExtensionsService.reinstall()` [L123-L143](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L123-L143) | 读取 settings → 校验 `source === 'registry'` → 校验 `bundle === null`（禁止单独重装子项） → `preInstall()` 校验 limit → 调用 `ExtensionManager.install(versionId)` |
| 3 | 同安装步骤 4-7 | 覆盖文件系统 → 广播 → reload → 重建 App Chunk |

**与安装的区别**：不重新写入数据库 settings（保留原有 enabled 等状态），仅覆盖文件系统并触发 reload。

---

### 2.3 卸载 (Uninstall)

**接口**: `DELETE /extensions/registry/uninstall/:pk`

**执行顺序**：

| 步骤 | 执行位置 | 操作内容 |
|------|----------|----------|
| 1 | Controller [L232-L254](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L232-L254) | Admin 校验 → 调用 `service.uninstall(pk)` |
| 2 | `ExtensionsService.uninstall()` [L104-L121](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L104-L121) | 读取 settings → 校验 `source === 'registry'` → **校验 `bundle === null`（禁止单独卸载 bundle child）** → `deleteOne(id)` 删除 DB → `ExtensionManager.uninstall(folder)` |
| 3 | `ExtensionsService.deleteOne()` [L237-L240](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L237-L240) | 删除主扩展记录 → **级联删除所有 `bundle = id` 的子项记录** |
| 4 | `ExtensionManager.uninstall()` [L250-L267](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L250-L267) | `InstallationManager.uninstall()` 删除 `.registry/{folder}/` → `broadcastReloadNotification()` → `reload({ skipSync: true })` 重建 App Chunk |

**一致性保证**:
- **数据库 settings**: 步骤 3 级联删除主记录和所有 bundle 子项
- **Bundle 子项**: 禁止单独卸载子项（步骤 2 校验），只能通过卸载父 bundle 级联删除
- **Filesystem 包**: 步骤 4 删除整个目录
- **App source chunk**: reload 时 `generateExtensionBundle()` 重新打包

---

### 2.4 Patch (updateOne)

**接口**: `PATCH /extensions/:pk`

**执行顺序**：

| 步骤 | 执行位置 | 操作内容 |
|------|----------|----------|
| 1 | Controller [L256-L292](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L256-L292) | Admin 校验 → 调用 `service.updateOne(id, body)` |
| 2 | `ExtensionsService.updateOne()` [L201-L235](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L201-L235) | 事务包裹 → **校验只允许 `meta` 字段变更** → 更新 DB → 若变更了 `enabled`，调用 `checkBundleAndSyncStatus()` 同步 bundle 状态 |
| 3 | `ExtensionManager.reload()` → `broadcastReloadNotification()` [L230-L232](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L230-L232) | **先 reload，reload 完成后再 broadcast**（注意：异步，不等待） |

**一致性保证**:
- **数据库 settings**: 事务内更新，若 enabled 变更则同步 bundle 父子状态
- **Bundle 子项**: `checkBundleAndSyncStatus()` 处理父/子 enabled 联动（详见第 6 节）
- **Filesystem 包**: 不涉及（patch 只改元数据）
- **App source chunk**: reload 时 `generateExtensionBundle()` 依据 `extensionsSettings.enabled` 过滤后重新打包

---

## 3. Admin 权限校验

所有写操作（安装/重装/卸载/PATCH/DELETE）和 registry 相关读操作都在 Controller 层进行强制 Admin 校验：

```typescript
if (req.accountability && req.accountability.admin !== true) {
    throw new ForbiddenError();
}
```

涉及位置：
- `GET /extensions/registry` [L48-L120](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L48-L120)
- `GET /extensions/registry/account/:pk` [L122-L151](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L122-L151)
- `GET /extensions/registry/extension/:pk` [L153-L182](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L153-L182)
- `POST /extensions/registry/install` [L184-L206](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L184-L206)
- `POST /extensions/registry/reinstall` [L208-L230](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L208-L230)
- `DELETE /extensions/registry/uninstall/:pk` [L232-L254](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L232-L254)
- `PATCH /extensions/:pk` [L256-L292](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L256-L292)
- `DELETE /extensions/:pk` [L294-L317](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L294-L317)

注意：`GET /extensions/` 和 `GET /extensions/sources/:chunk` **没有** 显式 Admin 校验（详见第 8 节）。

---

## 4. EXTENSIONS_LIMIT 计数校验

在 `preInstall()` 方法中执行 [L57-L74](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L57-L74)：

```typescript
const limit = env['EXTENSIONS_LIMIT'] ? Number(env['EXTENSIONS_LIMIT']) : null;

if (limit !== null) {
    const currentlyInstalledCount = this.extensionsManager.extensions.length;

    // Bundle 按子项数量计数，而非按 1 个计数
    const points = version.bundled.length ?? 1;

    const afterInstallCount = currentlyInstalledCount + points;

    if (afterInstallCount >= limit) {
        throw new LimitExceededError({ category: 'Extensions' });
    }
}
```

**关键规则**:
- 环境变量 `EXTENSIONS_LIMIT` 为 null 时不限制
- `currentlyInstalledCount` 包含 `local + registry + module` 三类扩展的总数
- **Bundle 按子项数量 `bundled.length` 计费**，防止通过 bundle 绕过限制
- 比较条件是 `>=`（等于即超限）

---

## 5. 禁止单独卸载 Bundle Child

在 `ExtensionsService.uninstall()` [L113-L117](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L113-L117) 和 `reinstall()` [L132-L136](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L132-L136) 中：

```typescript
if (settings.bundle !== null) {
    throw new InvalidPayloadError({
        reason: 'Cannot uninstall sub extensions of bundles separately',
    });
}
```

同时在 `ExtensionsService.deleteOne()` [L237-L240](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L237-L240) 中实现级联删除：

```typescript
async deleteOne(id: string) {
    await this.extensionsItemService.deleteOne(id);
    await this.extensionsItemService.deleteByQuery({ filter: { bundle: { _eq: id } } });
}
```

**设计意图**: bundle 子项的生命周期完全依附于父 bundle，不能独立管理。

---

## 6. updateOne 只允许 Meta 变更

在 `ExtensionsService.updateOne()` [L201-L235](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L201-L235) 中：

```typescript
if (!isObject(data.meta)) {
    throw new InvalidPayloadError({ reason: `"meta" is required` });
}

await service.extensionsItemService.updateOne(id, data.meta);
```

**约束**:
- 请求体必须包含 `meta` 字段且为对象
- 只有 `meta` 内的字段（即 `directus_extensions` 表字段：`enabled`、`folder`、`bundle` 等）会被更新
- `schema` 等其他字段被完全忽略，不允许通过 API 修改

---

## 7. Bundle Enabled 状态同步

由 `ExtensionsService.checkBundleAndSyncStatus()` [L251-L288](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L251-L288) 实现，在事务内执行。

### 7.1 切换父 Bundle 的 enabled

当扩展本身是 bundle（`bundle === null && schema.type === 'bundle'`）：

```sql
UPDATE directus_extensions 
SET enabled = :newStatus 
WHERE bundle = :extensionId OR id = :extensionId
```

即 **父 bundle 和所有子项同步切换为相同状态**。

### 7.2 切换子项的 enabled

当操作对象是 bundle 中的某个子项时：

1. **校验父 bundle 是否为 partial**: 若 `parent.schema.partial === false`，抛出 `UnprocessableContentError`（非 partial bundle 不允许单独切换子项）。
2. **自动同步父 bundle 状态**:
   - 若存在至少一个 enabled 子项 → 父 bundle 设为 enabled
   - 若所有子项都 disabled → 父 bundle 设为 disabled

---

## 8. managerReload 与 broadcastReloadNotification 的先后顺序

### 8.1 安装/卸载场景

在 `ExtensionManager.install()` [L231-L248](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L231-L248) 和 `uninstall()` [L250-L267](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L250-L267) 中：

```typescript
await this.broadcastReloadNotification({ partialSync: syncFolder });  // 先广播
await this.reload({ skipSync: true });                                  // 再本地 reload
```

**顺序：先 broadcast，再 reload**。原因：
- 本进程直接操作了文件系统，无需 sync（`skipSync: true`）
- 其他进程需要先收到消息，然后从远程存储 sync 最新文件

### 8.2 Patch 场景

在 `ExtensionsService.updateOne()` [L230-L232](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/extensions.ts#L230-L232) 中：

```typescript
this.extensionsManager.reload().then(() => {
    this.extensionsManager.broadcastReloadNotification();
});
```

**顺序：先 reload，完成后再 broadcast（异步，不阻塞响应）**。原因：
- Patch 只改数据库 settings，不涉及文件系统变更，其他进程只需 reload 重新读取 DB 即可

### 8.3 消息订阅端

在 `ExtensionManager.initialize()` [L217-L225](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L217-L225)：

```typescript
this.messenger.subscribe(this.reloadChannel, (payload) => {
    // 忽略自己发出的消息
    if (payload.origin === this.processId) return;
    this.reload(options);
});
```

---

## 9. /controllers/extensions.ts 中三类接口的权限对比

### 9.1 Registry 接口（`/extensions/registry/*`）

| 接口 | 方法 | Admin 校验 | 说明 |
|------|------|-----------|------|
| `/registry` | GET | ✅ 是 | 从 marketplace 搜索扩展列表 |
| `/registry/account/:pk` | GET | ✅ 是 | 查询 marketplace 账户信息 |
| `/registry/extension/:pk` | GET | ✅ 是 | 查询 marketplace 扩展详情 |
| `/registry/install` | POST | ✅ 是 | 安装 registry 扩展 |
| `/registry/reinstall` | POST | ✅ 是 | 重装 registry 扩展 |
| `/registry/uninstall/:pk` | DELETE | ✅ 是 | 卸载 registry 扩展 |

### 9.2 Sources 接口（`/extensions/sources/:chunk`）

| 接口 | 方法 | Admin 校验 | 说明 |
|------|------|-----------|------|
| `/sources/:chunk` | GET | ❌ 否 | 返回 App 扩展打包后的 JS chunk（供前端 App 加载） |

- **无显式权限校验**，但经过全局 `authenticate` 中间件（在 [app.ts L328](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L328) 注册），要求至少有有效登录态
- 有 `Cache-Control` 缓存头（由 `EXTENSIONS_CACHE_TTL` 控制）
- chunk 名白名单校验：只返回 `appExtensionChunks` 数组中存在的文件

### 9.3 通用/本地扩展相关接口

| 接口 | 方法 | Admin 校验 | 说明 |
|------|------|-----------|------|
| `/` | GET | ❌ 控制器内无显式校验 | 读取所有已安装扩展列表 |
| `/:pk` | PATCH | ✅ 是 | 更新扩展 meta（enabled 等） |
| `/:pk` | DELETE | ✅ 是 | 删除扩展记录 |

- `GET /extensions/` 使用了 `useCollection('directus_extensions')` 中间件 [L31](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/extensions.ts#L31)，后续由权限系统（policies）控制访问
- 但实际该路由未经过 `validate-collection` 权限中间件，结合 `ExtensionsService.readAll()` 直接查询 `directus_extensions` 表，**实际权限取决于全局策略配置**

---

## 10. App Source Chunk 生成机制

在 `ExtensionManager.generateExtensionBundle()` [L569-L614](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L569-L614) 中：

1. 调用 `generateExtensionsEntrypoint()` 生成虚拟入口代码，按 `extensionsSettings.enabled` 过滤扩展
2. 使用 Rollup（或 Rolldown，由 `EXTENSIONS_ROLLDOWN` 控制）打包
3. 输出到 `$TEMP_PATH/app-extensions/` 目录
4. 将所有 chunk 文件名存入 `this.appExtensionChunks`

`GET /extensions/sources/:chunk` 路由通过 `getAppExtensionChunk()` [L408-L425](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L408-L425) 读取这些临时文件返回给前端。

**触发重建的时机**: 每次 `reload()` → `load()` 时都会重新调用 `generateExtensionBundle()`。

---

## 11. 四层一致性总结

| 层级 | 存储位置 | 变更时机 | 同步机制 |
|------|----------|----------|----------|
| 数据库 Settings | `directus_extensions` 表 | install / uninstall / patch | ExtensionsService 直接 CRUD，bundle 子项级联 |
| Bundle 子项 | `directus_extensions.bundle` 字段 | install 创建 / uninstall 级联删除 / patch 同步 enabled | `deleteOne` 级联 + `checkBundleAndSyncStatus` 同步状态 |
| Filesystem 包 | `extensions/.registry/{versionId}/` | install 下载解压 / uninstall 删除 / reinstall 覆盖 | InstallationManager 操作文件系统 |
| App Source Chunk | `$TEMP_PATH/app-extensions/*.js` | 每次 reload 时重建 | `generateExtensionBundle()` Rollup 重新打包 |

所有四个层级的最终一致性通过 **reload() + broadcastReloadNotification()** 保证所有进程重新加载。
