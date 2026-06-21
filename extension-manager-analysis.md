# ExtensionManager 完整生命周期分析

> 基于 [manager.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts) 核心源码，梳理从 `initialize` 到 `load`、`registerApiExtensions`、`generateExtensionBundle` 和 `reload` 的完整过程。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [三类 Extension 来源](#2-三类-extension-来源)
3. [Settings 表与 enabled 决定机制](#3-settings-表与-enabled-决定机制)
4. [initialize → load 完整流程](#4-initialize--load-完整流程)
5. [registerApiExtensions 注册流程](#5-registerapiextensions-注册流程)
6. [Bundle Entry 如何生成到 App 扩展 Chunk](#6-bundle-entry-如何生成到-app-扩展-chunk)
7. [Hook / Endpoint / Operation / Bundle 的注册与卸载](#7-hook--endpoint--operation--bundle-的注册与卸载)
8. [Sandboxed API Extension 的导入限制与 Host Function](#8-sandboxed-api-extension-的导入限制与-host-function)
9. [Bus Reload 如何避免处理自己发出的消息](#9-bus-reload-如何避免处理自己发出的消息)
10. [Watcher 与热重载机制](#10-watcher-与热重载机制)

---

## 1. 整体架构概览

`ExtensionManager` 是 Directus 扩展系统的核心调度器，以单例形式通过 [getExtensionManager()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/index.ts) 导出。它管理着扩展从发现、加载、注册到卸载的完整生命周期。

**扩展类型分类**（定义于 [extensions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/constants/src/extensions.ts)）：

| 分类 | 类型 | 运行环境 |
|------|------|----------|
| App 扩展 | `interface`, `display`, `layout`, `module`, `panel`, `theme` | 浏览器端 |
| API 扩展 | `hook`, `endpoint` | 服务端 |
| Hybrid 扩展 | `operation` | 两端均有入口 |
| Bundle 扩展 | `bundle` | 可包含以上任意类型 |

**核心状态变量**：

| 变量 | 类型 | 用途 |
|------|------|------|
| `localExtensions` | `Map<string, Extension>` | 本地文件系统扩展（key 为 folder 名） |
| `registryExtensions` | `Map<string, Extension>` | Registry 安装的扩展（key 为 versionId） |
| `moduleExtensions` | `Map<string, Extension>` | npm 依赖中的扩展（key 为包名） |
| `extensionsSettings` | `ExtensionSettings[]` | 从数据库读取的扩展启用状态 |
| `appExtensionChunks` | `string[]` | App 扩展 bundle 的 chunk 文件名列表 |
| `unregisterFunctionMap` | `Map<string, PromiseCallback>` | 每个扩展的注销回调 |
| `reloadQueue` | `PQueue({ concurrency: 1 })` | 防竞态的串行重载队列 |

---

## 2. 三类 Extension 来源

扩展通过 [getExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/get-extensions.ts) 函数从三个来源加载：

### 2.1 Local（本地文件系统）

```typescript
const localExtensions = await resolveFsExtensions(getExtensionsPath());
```

- **路径来源**：由 [getExtensionsPath()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/get-extensions-path.ts) 决定。若配置了 `EXTENSIONS_LOCATION` 环境变量，则使用 `$TEMP_PATH/extensions`（远程同步的本地缓存），否则使用 `EXTENSIONS_PATH`（默认 `./extensions`）
- **Map key**：文件夹名（即 folder name）
- **local 标记**：`true`
- **扫描方式**：通过 [resolveFsExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/extensions/src/node/utils/get-extensions.ts#L68) 列出根目录下的所有非隐藏文件夹，逐个读取 `package.json`，用 `ExtensionManifest` 解析验证后构建 `Extension` 对象

### 2.2 Registry（扩展市场安装）

```typescript
const registryExtensions = await resolveFsExtensions(join(getExtensionsPath(), '.registry'));
```

- **路径来源**：`$EXTENSIONS_PATH/.registry/` 子目录，每个子文件夹以 versionId 命名
- **Map key**：versionId（文件夹名）
- **local 标记**：`true`（因为它们最终也落在文件系统上，仍可被 watcher 监控）
- **扫描方式**：同样使用 `resolveFsExtensions()`，只是根目录指向 `.registry` 子文件夹
- **安装/卸载**：由 [InstallationManager](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/installation/manager.ts) 负责

### 2.3 Module（npm 依赖）

```typescript
const moduleExtensions = await resolveModuleExtensions(env['PACKAGE_FILE_LOCATION'] as string);
```

- **路径来源**：项目根目录 `package.json` 中的 `dependencies`
- **Map key**：npm 包名
- **local 标记**：`true`（但不会被 watcher 监控，因为不在 extensions 目录内）
- **扫描方式**：通过 [resolveModuleExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/extensions/src/node/utils/get-extensions.ts#L111) 读取 `package.json` 的 `dependencies` 列表，逐个 `resolvePackage()` 定位实际路径，检查其 `package.json` 中是否包含 `directus: {}` 扩展声明，有则解析为 `Extension`

**三者合并后的 `extensions` getter**：

```typescript
public get extensions() {
    return [...this.localExtensions.values(), ...this.registryExtensions.values(), ...this.moduleExtensions.values()];
}
```

---

## 3. Settings 表与 enabled 决定机制

扩展的启用/禁用状态存储在数据库的 `directus_extensions` 表中，对应类型 [ExtensionSettings](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/extensions/app-extension-config.ts#L61)：

```typescript
interface ExtensionSettings {
    id: string;
    source: 'module' | 'registry' | 'local';
    enabled: boolean;
    bundle: string | null;   // bundle 扩展的 id，非 bundle 为 null
    folder: string;          // 文件夹名（local）或包名（module）或 versionId（registry）
}
```

### 3.1 加载过程

[getExtensionsSettings()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/get-extensions-settings.ts) 在 `load()` 阶段被调用，流程如下：

1. **读取现有设置**：通过 `ExtensionsService` 从数据库读取所有 `directus_extensions` 记录，按 `source` 分为 `localSettings`、`registrySettings`、`moduleSettings`
2. **增量同步**：遍历三个来源的扩展 Map，与现有设置对比：
   - **已存在**：保留原有设置（包括 `enabled` 状态），如果是 bundle 类型还需更新其子条目
   - **不存在**：调用 `generateSettingsEntry()` 创建新记录，**默认 `enabled: true`**
3. **清理**：移除数据库中已不再安装的扩展记录
4. **Bundle 子条目**：bundle 扩展的每个 entry（hook/endpoint/operation）都有独立的设置行，`bundle` 字段指向 bundle 的 `id`

### 3.2 enabled 在注册时的作用

在 [registerApiExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L676) 中，每个扩展在注册前会查找其对应的 settings：

```typescript
const { id, enabled } = this.extensionsSettings.find(
    (settings) => settings.source === source && settings.folder === folder,
) ?? { enabled: false };

if (!enabled) return;  // 未启用则跳过注册
```

**关键细节**：如果找不到对应 settings 记录，默认 `enabled: false`，该扩展不会被注册。

### 3.3 Bundle 内部的 enabled 控制

Bundle 扩展本身有一个 settings 条目（`bundle: null`），每个子条目也有独立的 settings（`bundle: <bundleId>`）。在 [registerBundleExtension()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L801) 中：

```typescript
const extensionEnabled = (extensionName: string) => {
    const settings = this.extensionsSettings.find(
        (settings) => settings.source === source && settings.folder === extensionName && settings.bundle === bundleId,
    );
    if (!settings) return false;
    return settings.enabled;
};
```

每个子条目独立判断 `enabled`，未启用的子条目会被跳过。

---

## 4. initialize → load 完整流程

### 4.1 initialize()

[initialize()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L189) 是入口方法：

```
initialize(options?)
  │
  ├─ 合并 options 与 defaultOptions
  │    defaultOptions = { schedule: true, watch: EXTENSIONS_AUTO_RELOAD }
  │
  ├─ Watcher 初始化（如果 watch: true 且尚未初始化）
  │    └─ initializeWatcher() → 启动 chokidar 文件监听
  │
  ├─ 首次加载（if !isLoaded）
  │    └─ load({ forceSync: true })
  │
  ├─ 更新 Watcher 监听列表
  │    └─ updateWatchedExtensions([...this.extensions])
  │
  └─ 订阅 Bus Reload 通道
       └─ messenger.subscribe('extensions.reload', callback)
```

### 4.2 load()

[load()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L276) 是核心加载方法：

```
load(options?)
  │
  ├─ 1. 远程同步（if EXTENSIONS_LOCATION 已配置）
  │    └─ syncExtensions(options)  ← 从远程存储拉取到本地
  │
  ├─ 2. 扫描三来源扩展
  │    └─ getExtensions() → { local, registry, module }
  │       ├─ localExtensions   = resolveFsExtensions(extensionsPath)
  │       ├─ registryExtensions = resolveFsExtensions(extensionsPath/.registry)
  │       └─ moduleExtensions  = resolveModuleExtensions(packageFileLocation)
  │
  ├─ 3. 加载 Settings
  │    └─ getExtensionsSettings({ local, registry, module })
  │       → 从 DB 读取/创建/更新 ExtensionSettings
  │
  ├─ 4. 注册 API 扩展（并行）
  │    ├─ registerInternalOperations()  ← 内置 operation
  │    └─ registerApiExtensions()       ← hook/endpoint/operation/bundle
  │
  ├─ 5. 生成 App 扩展 Bundle（if SERVE_APP）
  │    └─ generateExtensionBundle()  → rollup/rolldown 打包
  │
  ├─ 6. 标记 isLoaded = true
  │
  └─ 7. 发出事件
       └─ emitter.emitAction('extensions.load', { extensions })
```

### 4.3 unload()

[unload()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L319) 用于卸载所有已注册的扩展：

```
unload()
  │
  ├─ unregisterApiExtensions()   ← 执行所有 unregisterFunctionMap 中的回调
  ├─ localEmitter.offAll()       ← 清除扩展间本地事件监听
  ├─ isLoaded = false
  └─ emitter.emitAction('extensions.unload', { extensions })
```

---

## 5. registerApiExtensions 注册流程

[registerApiExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L676) 遍历三个来源的所有扩展，根据类型分发注册：

```
registerApiExtensions()
  │
  └─ 对每个 source (module, registry, local):
     └─ 对每个 extension:
        │
        ├─ 查找 ExtensionSettings: { id, enabled }
        │   └─ find where source === source && folder === folder
        │   └─ 若无匹配 → { enabled: false } → 跳过
        │
        └─ if enabled → 按 extension.type 分发:
           ├─ 'hook'      → registerHookExtension(extension)
           ├─ 'endpoint'  → registerEndpointExtension(extension)
           ├─ 'operation' → registerOperationExtension(extension)
           └─ 'bundle'    → registerBundleExtension(extension, source, id)
```

**关键**：App 类型扩展（`interface`, `display`, `layout`, `module`, `panel`, `theme`）不在 `registerApiExtensions` 中注册，它们仅被打包到 App Bundle 中。

---

## 6. Bundle Entry 如何生成到 App 扩展 Chunk

### 6.1 入口点生成

[generateExtensionsEntrypoint()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/extensions/src/node/utils/generate-extensions-entrypoint.ts) 负责生成一段虚拟的 JavaScript 入口代码，供 rollup/rolldown 打包：

**筛选逻辑**：

1. 遍历三个来源的扩展 Map
2. 对每个扩展，查找其 settings（按 `source + folder` 匹配）
3. **App / Hybrid 扩展**：如果 `enabled === true`，加入 `appOrHybridExtensions` 数组
4. **Bundle 扩展**：遍历其 `entries`，仅保留类型为 App/Hybrid 且 `enabled` 的子条目，构建 `appBundle` 对象

**生成的代码结构**：

```javascript
// 1. 导入所有 App/Hybrid 扩展
import interface0 from './path/to/extension/dist/index.js';
import operation0 from './path/to/operation/dist/app.js';

// 2. 导入 Bundle 扩展的 App 入口（按类型解构）
import {interfaces as interfaceBundle0, operations as operationBundle0} from './path/to/bundle/dist/app.js';

// 3. 按类型聚合导出
export const interfaces = [interface0, ...interfaceBundle0];
export const displays = [];
export const layouts = [];
export const modules = [];
export const panels = [];
export const themes = [];
export const operations = [operation0, ...operationBundle0];
```

**Bundle 子条目的 enabled 判定**：

```typescript
const enabled = settings.find(
    (setting) => setting.source === source && setting.folder === entry.name && setting.bundle === settingsForExtension.id,
)?.enabled ?? false;
```

Bundle 的每个子条目需要在 settings 中同时匹配 `source`、`folder`（entry.name）和 `bundle`（bundle 的 id），三者缺一不可。

### 6.2 Bundle 打包

[generateExtensionBundle()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L569) 使用 rollup 或 rolldown 将入口代码打包：

```typescript
const bundle = await rollDirection({
    input: 'entry',
    external: Object.values(sharedDepsMapping),    // 共享依赖作为外部依赖
    makeAbsoluteExternalsRelative: false,
    plugins: [
        virtual({ entry: entrypoint }),             // 虚拟入口模块
        alias({ entries: internalImports }),         // 重写 @directus/extensions-sdk 等为 App 中已有的构建产物
        nodeResolve({ browser: true }),             // 浏览器端模块解析
    ],
});
```

**共享依赖映射**（[getSharedDepsMapping()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/get-shared-deps-mapping.ts)）：

[APP_SHARED_DEPS](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/extensions/src/shared/constants/shared-deps.ts) 定义了保证在 App 全局可用的依赖：

```typescript
const APP_SHARED_DEPS = ['@directus/extensions-sdk', 'vue', 'vue-router', 'vue-i18n', 'pinia'] as const;
```

这些依赖会被映射到 `@directus/app/dist/assets/` 中对应的 chunk 文件路径（如 `vue.xxxxxxxx.entry.js`），通过 `alias` 插件在打包时替换导入路径，避免重复打包。

### 6.3 产物存储

打包结果写入 `$TEMP_PATH/app-extensions/` 目录，chunk 文件名保存在 `appExtensionChunks` 数组中。App 通过 `getAppExtensionChunk()` 读取这些文件。

---

## 7. Hook / Endpoint / Operation / Bundle 的注册与卸载

### 7.1 Hook 注册与卸载

**注册**（[registerHookExtension()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L715)）：

1. 若 `sandbox.enabled`，走沙箱注册路径（见第 8 节）
2. 否则，通过 `importFileUrl()` 动态导入扩展模块
3. 调用 [registerHook()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L885) 注册，传入 `HookConfig` 回调

**registerHook() 内部注册的 5 类钩子**：

| 钩子类型 | 注册方式 | 卸载方式 |
|----------|----------|----------|
| `filter` | `emitter.onFilter(event, handler)` | `emitter.offFilter(event, handler)` |
| `action` | `emitter.onAction(event, handler)` | `emitter.offAction(event, handler)` |
| `init` | `emitter.onInit(event, handler)` | `emitter.offInit(name, handler)` |
| `schedule` | `scheduleSynchronizedJob(name:cronIndex, cron, handler)` | `job.stop()` |
| `embed` | 推入 `hookEmbedsHead` / `hookEmbedsBody` 数组 | 从数组中 splice 移除 |

**卸载**：将所有 unregister 回调存入 `unregisterFunctionMap`，在 `unload()` 时统一执行，同时调用 `deleteFromRequireCache()` 清除模块缓存。

### 7.2 Endpoint 注册与卸载

**注册**（[registerEndpointExtension()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L741)）：

1. 若 `sandbox.enabled`，走沙箱路径
2. 否则，动态导入扩展模块，获取 `EndpointConfig`
3. 调用 [registerEndpoint()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L977)

**registerEndpoint() 内部**：

```typescript
const scopedRouter = express.Router();
this.endpointRouter.use(`/${routeName}`, scopedRouter);
endpointRegistrationCallback(scopedRouter, { services, env, database, emitter, logger, getSchema });
```

每个 endpoint 获得一个独立的 scoped Router，挂载到 `endpointRouter` 的 `/${routeName}` 路径下。

**卸载**：

```typescript
this.endpointRouter.stack = this.endpointRouter.stack.filter((layer) => scopedRouter !== layer.handle);
```

直接从 Router 的 stack 中移除对应的 layer。

### 7.3 Operation 注册与卸载

**注册**（[registerOperationExtension()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L771)）：

1. 若 `sandbox.enabled`，走沙箱路径
2. 否则，动态导入扩展的 `api` 入口（`operation.entrypoint.api`）
3. 调用 [registerOperation()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L1007)

**registerOperation() 内部**：

```typescript
const flowManager = getFlowManager();
flowManager.addOperation(config.id, config.handler);
```

将 operation 的 handler 注册到 FlowManager 中。

**卸载**：

```typescript
flowManager.removeOperation(config.id);
```

### 7.4 Bundle 注册与卸载

**注册**（[registerBundleExtension()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L801)）：

Bundle 不支持沙箱模式，统一走动态导入：

1. 导入 bundle 的 `api` 入口（`bundle.entrypoint.api`），获取 `BundleConfig`
2. `BundleConfig` 结构：

```typescript
type BundleConfig = {
    endpoints: { name: string; config: EndpointConfig }[];
    hooks: { name: string; config: HookConfig }[];
    operations: { name: string; config: OperationApiConfig }[];
};
```

3. 对每个子条目，通过 `extensionEnabled(name)` 检查其独立 enabled 状态
4. 启用的子条目分别调用 `registerHook()`、`registerEndpoint()`、`registerOperation()`
5. 所有子条目的 unregister 回调汇聚到一个数组

**卸载**：

```typescript
await Promise.all(unregisterFunctions.map((fn) => fn()));
deleteFromRequireCache(bundlePath);
```

### 7.5 统一卸载流程

[unregisterApiExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L1022) 在 `unload()` 中被调用：

```typescript
private async unregisterApiExtensions(): Promise<void> {
    const unregisterFunctions = Array.from(this.unregisterFunctionMap.values());
    await Promise.all(unregisterFunctions.map((fn) => fn()));
}
```

执行所有已注册的注销回调（包括 emitter 解绑、router 移除、FlowManager 移除、sandbox isolate 销毁等），然后清除模块缓存。

---

## 8. Sandboxed API Extension 的导入限制与 Host Function

### 8.1 沙箱创建流程

[registerSandboxedApiExtension()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L616) 使用 `isolated-vm` 创建隔离沙箱：

```
registerSandboxedApiExtension(extension)
  │
  ├─ 1. 创建 Isolate
  │    new ivm.Isolate({ memoryLimit: EXTENSIONS_SANDBOX_MEMORY })
  │    └─ onCatastrophicError → process.abort()
  │
  ├─ 2. 创建 Context
  │    context.global.setSync('process', { env: { NODE_ENV } }, { copy: true })
  │
  ├─ 3. 编译扩展代码为 Module
  │    isolate.compileModule(extensionCode, { filename })
  │
  ├─ 4. 实例化 SDK Module
  │    instantiateSandboxSdk(isolate, requestedScopes)
  │
  ├─ 5. 实例化扩展 Module
  │    module.instantiate(context, (specifier) => {
  │        if (specifier !== 'directus:api') {
  │            throw new Error('Imports other than "directus:api" are prohibited');
  │        }
  │        return sdkModule;
  │    })
  │
  ├─ 6. 执行模块
  │    module.evaluate({ timeout: sandboxTimeout })
  │
  ├─ 7. 获取默认导出
  │    module.namespace.get('default', { reference: true })
  │
  ├─ 8. 生成沙箱入口代码
  │    generateApiExtensionsSandboxEntrypoint(type, name, endpointRouter)
  │
  ├─ 9. 在 Context 中执行入口代码
  │    context.evalClosure(code, [cb, ...hostFunctions.map(fn => new ivm.Reference(fn))])
  │
  └─ 10. 注册卸载回调
         └─ isolate.dispose() + unregisterFunction()
```

### 8.2 导入限制：仅允许 `directus:api`

在步骤 5 的 `module.instantiate()` 回调中，**specifier 必须等于 `'directus:api'`**，否则抛出异常：

```typescript
await module.instantiate(context, (specifier) => {
    if (specifier !== 'directus:api') {
        throw new Error('Imports other than "directus:api" are prohibited in API extension sandboxes');
    }
    return sdkModule;
});
```

这意味着沙箱扩展的代码中只能有 `import ... from 'directus:api'` 这一条导入语句，任何其他模块导入（包括 Node.js 内置模块、npm 包等）都会被拒绝。

### 8.3 SDK 提供的 Host Function

[instantiateSandboxSdk()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/sandbox/sdk/instantiate.ts) 创建的 SDK Module 导出三个函数：

| SDK 函数 | 参数 | 是否异步 | 权限控制 |
|----------|------|----------|----------|
| `log` | `message` | 否 | `requestedScopes.log` |
| `sleep` | `milliseconds` | 是 | `requestedScopes.sleep` |
| `request` | `url`, `options` | 是 | `requestedScopes.request`（URL 白名单 + HTTP 方法白名单） |

每个 SDK 函数都是 Host Function，通过 `ivm.Reference` 从宿主注入到沙箱中。调用时会检查 `requestedScopes`，未授权的 scope 会抛出异常。

### 8.4 Host Function 的桥接机制

**[generateHostFunctionReference()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/sandbox/generate-host-function-reference.ts)** 负责生成在沙箱内调用 Host Function 的代理代码：

- **同步调用**：`$i.applySync(undefined, [args], { arguments: { reference: true }, result: { copy: true } })`
- **异步调用**：`await $i.apply(undefined, [args], { arguments: { reference: true }, result: { copy: true, promise: true } })`

其中 `$i` 是通过 `evalClosure` 注入的 `ivm.Reference` 引用，`i` 由 `numberGenerator()` 递增分配。

### 8.5 沙箱入口代码

[generateApiExtensionsSandboxEntrypoint()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/sandbox/generate-api-extensions-sandbox-entrypoint.ts) 为不同扩展类型生成在沙箱内执行的桥接代码：

**Hook 沙箱入口**：

```javascript
const extensionExport = $0.deref();
const filter = ($1, $2) => $1.applySync(undefined, [$1, $2], { ... });
const action = ($3, $4) => $3.applySync(undefined, [$3, $4], { ... });
return extensionExport({ filter, action });
```

宿主端的 `registerFilterGenerator()` 和 `registerActionGenerator()` 提供了实际的 `filter`/`action` Host Function，它们将沙箱回调通过 `callReference()` 桥接到 `emitter.onFilter` / `emitter.onAction`。

**Endpoint 沙箱入口**：生成 `router` 代理对象（get/post/put/patch/delete 方法），宿主端通过 `registerRouteGenerator()` 将请求转发到沙箱回调。

**Operation 沙箱入口**：直接调用 `registerOperation(id, handler)`，宿主端通过 `registerOperationGenerator()` 注册到 FlowManager。

### 8.6 错误隔离与 [wrap()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/sandbox/sdk/utils/wrap.ts)

`isolated-vm` 不允许沙箱捕获宿主抛出的异常。`wrap()` 函数将 Host Function 的结果包装为 `{ result, error }` 形式，使沙箱内可以正确判断和重新抛出错误：

```typescript
return async (...args) => {
    try {
        return { result: await util(...args), error: false };
    } catch (error) {
        return { result: sanitizedError, error: true };
    }
};
```

### 8.7 超时与内存限制

- **内存**：`EXTENSIONS_SANDBOX_MEMORY`（默认 MB）限制沙箱的 V8 堆大小
- **超时**：`EXTENSIONS_SANDBOX_TIMEOUT`（默认 ms）限制沙箱代码执行时间
- **调用超时**：[callReference()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/lib/sandbox/register/call-reference.ts) 在调用沙箱回调时也使用 `sandboxTimeout`
- **内存溢出**：如果 `RangeError`（内存超限），直接 `process.abort()`

---

## 9. Bus Reload 如何避免处理自己发出的消息

### 9.1 问题背景

在多实例部署中，当某个实例安装/卸载扩展时，需要通知其他实例重新加载扩展。这通过消息总线（Bus）实现。但如果不过滤自己发出的消息，会导致实例在已经执行了重载之后再冗余地重载一次。

### 9.2 解决方案：origin 字段

[processId()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/utils/node/process-id.ts) 为每个进程生成唯一标识：

```typescript
const parts = [hostname(), process.pid, new Date().getTime()];
const hash = createHash('md5').update(parts.join(''));
_cache.id = hash.digest('hex');
```

**发布消息时**，在 payload 中附带 `origin` 字段：

```typescript
// manager.ts - broadcastReloadNotification()
await this.messenger.publish(this.reloadChannel, { ...options, origin: this.processId });
```

**接收消息时**，过滤掉 `origin === this.processId` 的消息：

```typescript
// manager.ts - initialize() 中的订阅回调
this.messenger.subscribe(this.reloadChannel, (payload: Record<string, unknown>) => {
    if (isPlainObject(payload) && 'origin' in payload && payload['origin'] === this.processId) return;
    // 仅处理其他进程发出的 reload 通知
    const options: ExtensionSyncOptions = {};
    if (typeof payload['forceSync'] === 'boolean') options.forceSync = payload['forceSync'];
    if (typeof payload['partialSync'] === 'string') options.partialSync = payload['partialSync'];
    this.reload(options);
});
```

### 9.3 Bus 实现

[useBus()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/bus/lib/use-bus.ts) 返回全局消息总线：

- **Redis 可用**：使用 Redis Pub/Sub 实现跨进程通信
- **Redis 不可用**：使用本地内存总线（仅同进程通信）

### 9.4 install / uninstall 中的 Bus 通知

```
install(versionId)
  ├─ installationManager.install(versionId)
  ├─ broadcastReloadNotification({ partialSync: '.registry/<versionId>' })
  │    └─ 通知其他进程局部同步该扩展
  ├─ reload({ skipSync: true })
  │    └─ 当前进程跳过同步（因为 install 已完成文件写入）
  └─ emitter.emitAction('extensions.installed', ...)

uninstall(folder)
  ├─ installationManager.uninstall(folder)
  ├─ broadcastReloadNotification({ partialSync: '.registry/<folder>' })
  ├─ reload({ skipSync: true })
  └─ emitter.emitAction('extensions.uninstalled', ...)
```

### 9.5 reload 的防竞态机制

[reload()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L338) 使用 `PQueue({ concurrency: 1 })` 确保串行执行：

```typescript
public reload(options?: ExtensionSyncOptions): Promise<unknown> {
    if (this.reloadQueue.size > 0) {
        return Promise.resolve();  // 队列中已有任务，无需重复排队
    }
    this.reloadQueue.add(async () => {
        if (this.isLoaded) {
            await this.unload();
            await this.load(options);
            // ... 对比前后扩展列表，更新 watcher
        }
    });
    return this.reloadPromise;
}
```

- 如果队列中已有等待的 reload 任务，直接返回，避免多个 reload 竞争
- `reloadPromise` 允许外部代码等待 reload 完成（通过 `isReloading()` 方法）

---

## 10. Watcher 与热重载机制

### 10.1 Chokidar 监听

[initializeWatcher()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L474) 启动 chokidar 文件监听：

- **监听目标**：`package.json`（根目录） + 扩展目录路径
- **忽略规则**：
  - `node_modules` 目录
  - 非扩展目录下的非 `package.json` 文件
  - 非扩展入口点的文件
- **深度限制**：`depth: 1`（仅监听扩展目录顶层），但扩展的 `dist` 入口文件通过 `updateWatchedExtensions` 单独添加
- **macOS 特殊处理**：`followSymlinks: false`，避免 dotdirs 被监听

### 10.2 事件触发重载

```typescript
this.watcher
    .on('add', debounce(() => this.reload(), 500))
    .on('change', debounce(() => this.reload(), 650))
    .on('unlink', debounce(() => this.reload(), 2000));
```

三种事件分别使用不同的 debounce 延迟，避免频繁重载。

### 10.3 动态更新监听列表

[updateWatchedExtensions()](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/extensions/manager.ts#L543) 在 reload 完成后调用，添加新增扩展的入口文件、移除已删除扩展的入口文件：

```typescript
// 仅监控 local 且非 .registry 目录下的扩展
extensions.filter((extension) => extension.local && !extension.path.startsWith(registryDir))
```

---

## 总结：完整生命周期时序图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         initialize()                                │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │ initWatcher  │  │   load()     │  │ subscribe bus reload   │    │
│  │  (optional)  │  │              │  │ (filter by origin)     │    │
│  └──────────────┘  │              │  └────────────────────────┘    │
│                    │  ┌─────────┐ │                                 │
│                    │  │syncExt  │ │  (if EXTENSIONS_LOCATION)       │
│                    │  └─────────┘ │                                 │
│                    │  ┌─────────┐ │                                 │
│                    │  │getExt   │ │  → local + registry + module   │
│                    │  └─────────┘ │                                 │
│                    │  ┌─────────┐ │                                 │
│                    │  │getSets  │ │  → DB read + incremental sync  │
│                    │  └─────────┘ │                                 │
│                    │  ┌─────────────────────────────────┐          │
│                    │  │ registerInternalOperations()    │          │
│                    │  │ + registerApiExtensions()       │          │
│                    │  │   ├─ hook (sandbox/normal)      │          │
│                    │  │   ├─ endpoint (sandbox/normal)  │          │
│                    │  │   ├─ operation (sandbox/normal) │          │
│                    │  │   └─ bundle (normal only)       │          │
│                    │  └─────────────────────────────────┘          │
│                    │  ┌─────────────────────────────┐              │
│                    │  │ generateExtensionBundle()    │              │
│                    │  │ → rollup/rolldown            │              │
│                    │  │ → appExtensionChunks         │              │
│                    │  └─────────────────────────────┘              │
│                    └──────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                          reload()                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ PQueue (concurrency: 1)                                      │   │
│  │  ┌──────────┐  ┌──────────┐                                  │   │
│  │  │ unload() │→ │ load()   │                                  │   │
│  │  └──────────┘  └──────────┘                                  │   │
│  │  ┌───────────────────────────────────────────────┐           │   │
│  │  │ diff extensions → updateWatchedExtensions()   │           │   │
│  │  └───────────────────────────────────────────────┘           │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Bus Reload (跨进程)                               │
│  Process A: install(vId)                                           │
│    └─ broadcastReloadNotification({ partialSync, origin: pid_A })  │
│         │                                                          │
│         ▼  Redis Pub/Sub                                           │
│  Process B: subscribe('extensions.reload', payload)                │
│    └─ if payload.origin === pid_B → skip (自己的消息)               │
│    └─ else → reload({ forceSync, partialSync })                    │
└─────────────────────────────────────────────────────────────────────┘
```
