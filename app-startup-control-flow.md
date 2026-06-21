# Directus App 启动控制流分析

## 一、启动初始化顺序（main.ts → mount）

从 [main.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/main.ts) 的 `init()` 函数开始，整个应用的初始化严格按照以下顺序执行：

### 初始化流程图

```
createApp(App)
    │
    ├─ app.use(i18n)                     // 1. 国际化插件
    ├─ app.use(createPinia())            // 2. Pinia 状态管理
    ├─ app.use(createHead())             // 3. Unhead 文档头部管理
    │
    ├─ registerDirectives(app)           // 4. 注册自定义指令
    ├─ registerComponents(app)           // 5. 注册全局组件
    ├─ registerViews(app)                // 6. 注册视图组件
    │
    ├─ await loadExtensions()            // 7. 异步加载扩展
    ├─ registerExtensions(app)           // 8. 注册扩展（接口/显示/布局/模块等）
    │
    ├─ app.use(router)                   // 9. 路由（扩展加载后注册，确保扩展路由可用）
    │
    ├─ useSystem(app)                    // 10. 系统级 provide（stores/api/sdk/extensions）
    │
    └─ app.mount('#app')                 // 11. 挂载到 DOM
```

### 各阶段详细说明

#### 1. i18n 初始化
- 位置：[lang/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/lang/index.ts)
- 使用 `vue-i18n` 的 Composition API 模式（`legacy: false`）
- 默认语言 `en-US`，包含日期格式和数字格式配置
- 其他语言包按需动态加载（`set-language.ts`）

#### 2. Pinia 初始化
- 位置：[packages/stores/src/app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/stores/src/app.ts)
- 创建全局 Pinia 实例，供所有 store 使用
- 核心全局 store：`useAppStore`（管理 hydrated/hydrating/authenticated 等状态）

#### 3. Unhead 初始化
- 使用 `@unhead/vue` 管理文档头部（title、meta、link、style 等）
- **注意**：main.ts 第3行注释强调 Vue 必须先于 Unhead 导入，否则会报错

#### 4. 指令注册
- 位置：[directives/register.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/directives/register.ts)
- 注册 7 个自定义指令：
  - `click-outside` - 点击外部检测
  - `context-menu` - 右键菜单
  - `focus` - 自动聚焦
  - `prevent-focusout` - 阻止失焦
  - `input-auto-width` - 输入框自适应宽度
  - `md` - Markdown 渲染
  - `tooltip` - 工具提示

#### 5. 全局组件注册
- 位置：[components/register.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/components/register.ts)
- 注册约 60+ 全局组件（VButton、VCard、VDialog 等 UI 基础组件）
- 以及私有视图中的通用组件（DrawerItem、SidebarDetail 等）

#### 6. 视图注册
- 位置：[views/register.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/views/register.ts)
- 注册三个顶层视图组件：
  - `PublicView` - 公开页面布局（登录、注册等）
  - `PrivateView` - 私有页面布局（异步组件，登录后加载）
  - `SharedView` - 共享页面布局

#### 7. 扩展加载
- 位置：[extensions.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/extensions.ts)
- `loadExtensions()`：异步动态导入扩展入口文件
  - 开发环境：`@directus-extensions`
  - 生产环境：`/extensions/sources/index.js`
- 扩展失败不阻断应用启动，仅控制台警告

#### 8. 扩展注册
- 注册 7 类扩展：interfaces、displays、layouts、modules、panels、operations、themes
- 内置扩展 + 自定义扩展合并后统一注册
- 模块扩展通过 `registerModules` 注册，其路由在 hydrate 阶段才动态添加
- 监听 i18n locale 变化，自动翻译扩展名称

#### 9. 路由注册
- 位置：[router.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/router.ts)
- **必须在扩展加载之后注册**，确保扩展的路由能被正确添加
- 基础路由（默认路由）在创建时即注册
- 模块路由通过 `router.addRoute()` 在 hydrate 阶段动态添加

#### 10. System Provide
- 位置：[composables/use-system.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/composables/use-system.ts)
- 通过 `app.provide` 注入全局依赖：
  - `STORES_INJECT` - 所有 store 的工厂函数集合
  - `API_INJECT` - API 请求实例
  - `SDK_INJECT` - SDK 实例
  - `EXTENSIONS_INJECT` - 扩展引用
- 解决 #26411 问题：pinia store 中的 `inject` 需要 Vue 实例级别的 `provide`

#### 11. 挂载
- `app.mount('#app')` 挂载到 DOM
- 挂载后触发路由解析，进入 `router.beforeEach` 守卫

---

## 二、router.beforeEach 守卫控制流

### 守卫概览

位置：[router.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/router.ts) 第 141-233 行

`onBeforeEach` 守卫在每次路由跳转前执行，包含以下检查层级：

```
路由跳转
    │
    ├─ 首次加载检查 (firstLoad)
    │   └─ refresh({ navigate: false })   // 尝试刷新 access token
    │
    ├─ Server Info 检查
    │   └─ 若 project 为 null → serverStore.hydrate()
    │
    ├─ Setup 完成检查
    │   └─ 未完成 → 重定向到 /setup
    │
    └─ 非公开路由检查 (to.meta.public !== true)
        │
        ├─ 未 hydrate 检查
        │   ├─ 已认证 → hydrate() → 放行/重定向
        │   └─ 未认证 → 重定向到 /login?redirect=...
        │
        ├─ TFA 强制检查
        │   ├─ 角色强制 TFA 且未设置 → /tfa-setup
        │   └─ 用户主动要求 TFA → /tfa-setup
        │
        └─ License 锁定检查
            └─ 管理员 + 锁定 + 非白名单路径 → /license-recovery
```

### 首次进入私有路由的详细条件

#### 1. refresh（Token 刷新）

**触发条件**：
- `firstLoad === true`（页面首次加载）
- 在守卫最开始执行，无论目标路由是否公开

**执行逻辑**：
- 位置：[auth.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/auth.ts) 第 103-126 行
- 首次刷新（`firstRefresh === true`）时，即使 `appStore.authenticated` 为 false 也会尝试刷新
- 非首次刷新时，仅在已认证状态下执行
- 如果 access token 尚未过期（距离过期还有 `SDK_AUTH_REFRESH_BEFORE_EXPIRES` 毫秒以上），则只调用 `readMe` 验证会话
- 刷新失败 → 调用 `logout({ navigate: false, reason: SESSION_EXPIRED })`

**关键点**：
- `navigate: false` 表示刷新失败时不主动跳转，由后续守卫逻辑处理
- 刷新成功后设置 `appStore.authenticated = true` 和 `appStore.accessTokenExpiry`

#### 2. serverHydrate（服务端信息获取）

**触发条件**：
- `serverStore.info.project === null`（服务端信息未加载）

**执行逻辑**：
- 位置：[stores/server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/stores/server.ts) 第 145-182 行
- 并行请求两个接口：
  - `/server/info` - 获取项目信息、配置、license 状态等
  - `/auth?sessionOnly` - 获取认证提供者列表
- 根据返回的 rate limit 配置调整请求队列

**关键判断**：
- `setupCompleted`：通过 `info.setup === null` 判断。若 setup 对象存在说明未完成

#### 3. setupRedirect（安装引导重定向）

**触发条件**：
- `serverStore.info.setupCompleted === false`（系统未完成初始化设置）

**重定向逻辑**：
- 若目标路径不是 `/setup` → 重定向到 `/setup`
- 若已在 `/setup` → 放行（正常渲染 Setup 页面）
- 根路径 `/` 的 redirect 函数也会根据 setupCompleted 决定跳 `/login` 还是 `/setup`

#### 4. authenticated 检查

**触发条件**：
- 目标路由 `meta.public !== true`（私有路由）
- `appStore.hydrated === false`（应用尚未完成数据 hydration）

**执行逻辑**：
```typescript
if (appStore.authenticated === true) {
    await hydrate();         // 执行全量数据加载
    // TFA 设置页面特殊处理...
    return to.fullPath;      // 放行到目标路由
} else {
    // 未认证 → 重定向到登录页，携带 redirect 参数
    return '/login?redirect=' + encodeURIComponent(to.fullPath);
}
```

**关键点**：
- `hydrated` 是「所有 store 数据已加载完成」的标志
- `authenticated` 是「用户已登录」的标志
- 已认证但未 hydrate → 触发 hydrate 流程后放行
- 未认证 → 跳登录页，保存目标路径以便登录后返回

#### 5. TFA 强制检查

**触发条件**：
- 用户已登录（`userStore.currentUser` 存在）
- 不是共享用户（`!('share' in userStore.currentUser)`）
- 目标路径不是 `/tfa-setup`

**两种触发场景**：

1. **角色强制 TFA**：
   - 条件：`userStore.currentUser.enforce_tfa === true` 且 `tfa_secret === null`
   - 行为：重定向到 `/tfa-setup`，可携带 redirect 参数

2. **用户主动要求 TFA**：
   - 条件：localStorage 中 `directus-require_tfa_setup` 等于当前用户 ID 且 `tfa_secret === null`
   - 行为：同上，重定向到 TFA 设置页

**反向检查**：
- 如果用户已在 `/tfa-setup` 但 `tfa_secret !== null`（已设置）
- 则重定向到 `userStore.currentUser.last_page` 或 `/login`

#### 6. License 锁定检查

**触发条件**：
- 目标路径不在 `ALLOWED_WHILE_LOCKED` 白名单中
  - 白名单：`['/license-recovery', '/logout', '/settings/license']`
- 当前用户是管理员（`userStore.isAdmin`）
- License 处于锁定状态（`licenseStore.isLocked`）

**行为**：
- 重定向到 `/license-recovery` 页面
- 管理员必须先解决 license 问题才能继续使用

**补充**：
- 在 [app.vue](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/app.vue) 中还有一个 watch 监听 `isLocked` 变化
- 因为路由守卫只在导航时触发，所以需要响应式 watch 来处理运行时 license 变锁的情况

#### 7. last_page 追踪

**触发位置**：`router.afterEach` 钩子
- 位置：[router.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/router.ts) 第 237-253 行

**执行逻辑**：
- 非公开页面 + `meta.track !== false` → 追踪
- 使用 500ms 防抖（`setTimeout`），避免页面加载期间频繁调用
- 调用 `userStore.trackPage(to)` 发送 PATCH 请求到 `/users/me/track/page`
- 同时更新本地 `currentUser.last_page`

**特殊情况**：
- 全屏预览路径（以 `/preview` 结尾）不追踪，避免登录后跳转到无法返回的全屏页
- TFA 设置页（`meta.track: false`）不追踪

---

## 三、Hydrate 机制与 Store 初始化顺序

### hydrate 函数总览

位置：[hydrate.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/hydrate.ts) 第 49-92 行

`hydrate()` 是登录后应用数据初始化的核心函数，负责按正确顺序加载所有 store 的数据。

### Store 列表

`useStores()` 返回所有参与 hydrate 的 store，顺序如下：

| 序号 | Store 名称 | $id | 说明 |
|------|-----------|-----|------|
| 1 | useCollectionsStore | collectionsStore | 集合数据 |
| 2 | useFieldsStore | fieldsStore | 字段数据 |
| 3 | useUserStore | userStore | 当前用户信息 |
| 4 | useRequestsStore | requestsStore | 请求队列 |
| 5 | usePresetsStore | presetsStore | 预设/书签 |
| 6 | useSettingsStore | settingsStore | 系统设置 |
| 7 | useServerStore | serverStore | 服务器信息 |
| 8 | useRelationsStore | relationsStore | 关系数据 |
| 9 | usePermissionsStore | permissionsStore | 权限数据 |
| 10 | useInsightsStore | insightsStore | 洞察/仪表盘 |
| 11 | useFlowsStore | flowsStore | 工作流 |
| 12 | useNotificationsStore | notificationsStore | 通知 |
| 13 | useAiStore | useAiStore | AI 功能 |
| 14 | useLicenseStore | licenseStore | License 信息 |

### Hydration 三阶段控制

```
hydrate()
    │
    ├─ 前置检查
    │   ├─ 已 hydrated → 直接返回
    │   └─ 正在 hydrating → 直接返回（防并发）
    │
    ├─ 第一阶段：User Store（串行）
    │   └─ await userStore.hydrate()       // 必须最先完成
    │
    ├─ 第二阶段：Permissions + Fields（并行）
    │   └─ await Promise.all([
    │          permissionsStore.hydrate(),
    │          fieldsStore.hydrate({ skipTranslation: true })
    │        ])
    │
    ├─ 第三阶段：其余所有 Store（并行）
    │   └─ await Promise.all(
    │          stores.filter(...).map(store => store.hydrate?.())
    │        )
    │
    ├─ 扩展 Hydration
    │   └─ await onHydrateExtensions()     // 模块扩展注册路由等
    │
    └─ 语言设置
        └─ await setLanguage(userStore.language)
```

### 为什么 UserStore 必须先完成？

代码注释明确说明：
> Multiple stores rely on the userStore to be set, so they can fetch user specific data.

具体依赖关系：

1. **permissionsStore** - 需要用户身份来获取 `/permissions/me` 的权限数据
2. **fieldsStore** - 字段可能受权限控制，且翻译依赖用户语言
3. **presetsStore** - 用户的个人预设/书签
4. **settingsStore** - 用户级别的设置
5. **collectionsStore** - 集合列表受权限影响
6. **licenseStore** - 仅管理员可访问，需要判断 `isAdmin`
7. **modules 扩展** - `preRegisterCheck` 需要用户和权限数据来决定是否显示模块

### Permissions + Fields 第二阶段

这两个 store 在 userStore 之后、其他 store 之前并行加载，因为：

- **permissionsStore**：几乎所有其他 store 都依赖权限数据来过滤/判断可见性
- **fieldsStore**：字段是数据模型的核心，collections、relations、layouts 等都依赖
- `fieldsStore.hydrate({ skipTranslation: true })` 跳过翻译是因为此时用户语言可能尚未设置，翻译会在 `setLanguage` 中统一处理

### 其余 Store 并行加载

排除已 hydration 的 4 个 store（userStore、permissionsStore、fieldsStore、serverStore），剩余 store 全部 `Promise.all` 并行加载。

**注意**：虽然 `serverStore` 在 `useStores` 列表中，但它在 router 守卫中更早被 hydration（在 hydrate() 调用之前），所以在 hydrate 函数中被跳过。

### 扩展 Hydration

`onHydrateExtensions()` 执行所有扩展的 hydrate 回调，目前主要是模块扩展：

- 位置：[modules/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/modules/index.ts) 第 22-50 行
- 根据用户权限过滤模块（`preRegisterCheck`）
- 通过 `router.addRoute()` 动态添加模块路由
- 这就是为什么 router 必须在扩展之后注册，但模块路由在 hydrate 之后才可用

### 语言设置

`setLanguage(userStore.language)` 在所有 store 数据加载完成后执行：

- 位置：[lang/set-language.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/app/src/lang/set-language.ts)
- 动态加载语言包（如果未加载过）
- 加载系统翻译（translationsStore）
- 翻译集合名称（collectionsStore.translateCollections）
- 翻译字段名称（fieldsStore.translateFields）
- 加载 date-fns locale

### Hydration 状态管理

通过 `appStore` 的三个状态变量控制：

| 状态 | 类型 | 说明 |
|------|------|------|
| `hydrating` | boolean | 是否正在进行 hydration，防止并发调用 |
| `hydrated` | boolean | 是否已完成 hydration，完成后置为 true |
| `error` | any | hydration 过程中发生的错误 |

**防并发机制**：
```typescript
if (appStore.hydrated) return;    // 已完成 → 跳过
if (appStore.hydrating) return;   // 进行中 → 跳过
appStore.hydrating = true;        // 标记开始
```

**错误处理**：
- hydrate 过程中的任何错误会被捕获并设置到 `appStore.error`
- 但 `hydrated` 仍会设为 true（在 finally 之后）
- app.vue 中检测到 error 会显示错误页面，而不是正常渲染

---

## 四、完整时序图（首次进入私有路由）

```
用户访问 /admin/collections/articles
    │
    ├─ main.ts 初始化完成 → app.mount()
    │
    ├─ router 解析路由 → beforeEach 守卫触发
    │   │
    │   ├─ firstLoad = true
    │   │   └─ refresh({ navigate: false })
    │   │       ├─ 有有效 session → 成功，设置 authenticated=true
    │   │       └─ 无有效 session → 失败，authenticated=false（静默）
    │   │
    │   ├─ serverStore.info.project === null
    │   │   └─ serverStore.hydrate()
    │   │       ├─ GET /server/info
    │   │       └─ GET /auth?sessionOnly
    │   │
    │   ├─ setupCompleted === true → 继续
    │   │
    │   └─ to.meta.public !== true
    │       │
    │       ├─ appStore.hydrated === false
    │       │   │
    │       │   ├─ appStore.authenticated === true
    │       │   │   │
    │       │   │   └─ hydrate()
    │       │   │       ├─ await userStore.hydrate()        [阶段1]
    │       │   │       │   ├─ GET /users/me
    │       │   │       │   ├─ GET /policies/me/globals
    │       │   │       │   └─ GET /roles/me
    │       │   │       │
    │       │   │       ├─ currentUser.app_access === true
    │       │   │       │   │
    │       │   │       │   ├─ Promise.all([                [阶段2]
    │       │   │       │   │   permissionsStore.hydrate(),
    │       │   │       │   │   fieldsStore.hydrate()
    │       │   │       │   │ ])
    │       │   │       │   │
    │       │   │       │   ├─ Promise.all(其余 stores)     [阶段3]
    │       │   │       │   │
    │       │   │       │   └─ onHydrateExtensions()
    │       │   │       │       └─ 模块路由动态注册
    │       │   │       │
    │       │   │       ├─ setLanguage(userStore.language)
    │       │   │       │
    │       │   │       └─ appStore.hydrated = true
    │       │   │
    │       │   └─ authenticated === false
    │       │       └─ 重定向 → /login?redirect=/collections/articles
    │       │
    │       ├─ TFA 强制检查
    │       │   ├─ enforce_tfa && tfa_secret === null → /tfa-setup
    │       │   └─ requireTfaSetup === userId → /tfa-setup
    │       │
    │       └─ License 锁定检查
    │           └─ isAdmin && isLocked → /license-recovery
    │
    ├─ 守卫放行 → afterEach 钩子
    │   └─ 500ms 防抖后 trackPage(last_page)
    │
    └─ 页面组件渲染
```

---

## 五、关键设计要点总结

### 1. 扩展驱动的路由架构
- 基础路由（登录、注册等）静态定义
- 功能模块路由通过扩展机制在 hydrate 时动态注册
- 保证了扩展可以灵活添加/移除功能模块

### 2. 分层 Hydration 策略
- **串行第一阶段**：用户信息（其他所有数据的基础）
- **并行第二阶段**：权限和字段（数据模型和访问控制的核心）
- **并行第三阶段**：所有其他业务数据
- 既保证了依赖顺序，又最大化了并发加载效率

### 3. 多维度的认证检查
- Token 刷新（refresh）
- 服务器信息获取（判断系统是否已初始化）
- Hydration 状态检查
- TFA 双因素认证强制
- License 合规性检查
- 每层检查都可能中断导航流程

### 4. 响应式与守卫双重保障
- License 锁定：既在路由守卫中检查，也在 app.vue 中通过 watch 监听
- 确保运行时状态变化也能及时响应

### 5. 防抖优化
- last_page 追踪使用 500ms 防抖
- 避免页面初始加载期间不必要的 API 调用
- 优先保证核心数据加载性能
