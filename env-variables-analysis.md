# Directus 环境变量全链路分析

本文档追踪 Directus 环境变量从默认值定义、配置文件读取、`*_FILE` 机制、cast 类型转换、`useEnv` 缓存到业务代码使用的完整路径。

---

## 1. 整体架构概览

```
┌─────────────┐    ┌──────────────┐    ┌────────────────┐
│  DEFAULTS    │    │ process.env  │    │  配置文件       │
│  (硬编码默认) │    │ (系统环境变量)│    │ (.env/yaml/json│
│             │    │              │    │  /js)          │
└──────┬──────┘    └──────┬───────┘    └───────┬────────┘
       │                  │                     │
       │                  ▼                     ▼
       │        readConfigurationFromProcess  readConfigurationFromFile
       │                  │                     │
       │                  └──────┬──────────────┘
       │                         │ 合并 (file 覆盖 process)
       │                         ▼
       │               rawConfiguration
       │                         │
       │                         │ *_FILE → 读文件内容
       │                         │
       │                         ▼
       │                    cast() 类型转换
       │                         │
       ▼                         ▼
  默认值写入 output      用户值写入 output (覆盖默认值)
       │                         │
       └─────────┬───────────────┘
                 ▼
              Env 对象
                 │
                 ▼
           useEnv() 缓存
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
  直接访问    getConfigFromEnv()  业务代码
  env[KEY]    前缀+嵌套转换
```

---

## 2. createEnv：合并 process.env 与配置文件

核心实现位于 [create-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/create-env.ts#L14-L49)。

### 2.1 两步读取

```typescript
const baseConfiguration = readConfigurationFromProcess();   // Step 1
const fileConfiguration = readConfigurationFromFile(getConfigPath()); // Step 2
const rawConfiguration = { ...baseConfiguration, ...fileConfiguration }; // 合并
```

**Step 1** — [readConfigurationFromProcess](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/read-configuration-from-process.ts#L4-L6)：直接展开 `process.env` 获取所有系统环境变量。

**Step 2** — [readConfigurationFromFile](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/read-configuration-from-file.ts#L13-L33)：根据 [getConfigPath](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/get-config-path.ts#L7-L9) 获取的路径（默认 `cwd()/.env`，可通过 `CONFIG_PATH` 环境变量覆盖），读取配置文件并按扩展名分发：

| 扩展名 | 读取方式 | 说明 |
|--------|---------|------|
| `.js` / `.mjs` / `.cjs` | [readConfigurationFromJavaScript](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/read-configuration-from-javascript.ts#L4-L24) | 支持 `export default {}` 对象或 `export default function(env) {}` 函数（接收 process.env） |
| `.json` | [readConfigurationFromJson](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/read-configuration-from-json.ts#L4-L11) | `require()` 加载 |
| `.yaml` / `.yml` | [readConfigurationFromYaml](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/read-configuration-from-yaml.ts#L4-L11) | YAML 解析 |
| 其他（如 `.env`）| [readConfigurationFromDotEnv](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/read-configuration-from-dotenv.ts#L4-L6) | 使用 `dotenv` 库的 `parse()` |

### 2.2 合并策略

```typescript
const rawConfiguration = { ...baseConfiguration, ...fileConfiguration };
```

**配置文件优先于 process.env**。`fileConfiguration` 中的同名键会覆盖 `baseConfiguration`。

### 2.3 默认值填充 + 用户值覆盖

```typescript
// 第一轮：写入所有默认值
for (const [key, value] of Object.entries(DEFAULTS)) {
    output[key] = getDefaultType(key) ? cast(value, key) : value;
}

// 第二轮：用用户提供的值覆盖（含 *_FILE 读取）
for (let [key, value] of Object.entries(rawConfiguration)) {
    // ... *_FILE 处理（见第4节）
    output[key] = cast(value, key);
}
```

- 第一轮将 [DEFAULTS](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L6-L260) 中的所有默认值写入 `output`，有类型映射的进行 cast。
- 第二轮遍历用户配置（process.env + 文件），覆盖默认值。**用户值永远优先**。

---

## 3. cast：类型转换引擎

核心实现位于 [cast.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/cast.ts#L8-L33)。

### 3.1 类型判定优先级

```typescript
const type = castFlag ?? getDefaultType(key) ?? guessType(value);
```

三级优先级从高到低：

1. **castFlag** — 值前缀显式声明（如 `number:8055`、`boolean:true`、`array:a,b,c`）
2. **getDefaultType(key)** — 根据键名在 [TYPE_MAP](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/type-map.ts#L7-L95) 中查找预定义类型
3. **guessType(value)** — 根据值的特征自动推断类型

### 3.2 castFlag（显式类型前缀）

[getCastFlag](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/has-cast-prefix.ts#L4-L11) 检测值是否以 `type:` 格式开头。支持的类型前缀定义在 [ENV_TYPES](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/env-types.ts#L5)：

```
'string' | 'number' | 'regex' | 'array' | 'json' | 'boolean'
```

示例：
- `number:8055` → 提取前缀 `number`，值变为 `8055`，转换为数字 `8055`
- `boolean:true` → 提取前缀 `boolean`，值变为 `true`，转换为布尔值 `true`
- `array:a,b,c` → 提取前缀 `array`，值变为 `a,b,c`，转换为数组 `['a','b','c']`

### 3.3 TYPE_MAP（键名类型映射）

[TYPE_MAP](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/type-map.ts#L7-L95) 将特定键名映射到固定类型。支持正则匹配：

| 键名/模式 | 类型 | 说明 |
|-----------|------|------|
| `PORT` | `number` | 端口号 |
| `DB_PASSWORD` | `string` | 确保密码不被意外转换 |
| `DB_EXCLUDE_TABLES` | `array` | 逗号分隔转数组 |
| `CACHE_AUTO_PURGE_IGNORE_LIST` | `array` | 逗号分隔转数组 |
| `MCP_OAUTH_ENABLED` | `boolean` | 布尔开关 |
| `MCP_OAUTH_MAX_CLIENTS` | `number` | 数值限制 |
| `MCP_OAUTH_ALLOWED_REDIRECT_DOMAINS` | `array` | 域名列表 |
| `STORAGE_.+_SECRET` | `string` | 正则匹配，存储密钥 |
| `AUTH_.+_BIND_PASSWORD` | `string` | 正则匹配，LDAP 密码 |

### 3.4 guessType（自动类型推断）

[guessType](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/guess-type.ts#L3-L24) 的判定逻辑：

```
值 === true / false / "true" / "false"  →  boolean
值是 number 类型，或符合数字字符串条件     →  number
  (不以0开头、NaN检查、安全整数范围内)
值是 Array，或字符串包含逗号              →  array
其他所有情况                              →  json
```

**关键细节**：
- 以 `0` 开头的数字字符串（如 `"0383312312"`）不被识别为 number，而是 json
- 超出 `Number.MIN_SAFE_INTEGER` / `Number.MAX_SAFE_INTEGER` 的也降级为 json
- 纯字符串（如 `"hello"`）默认推断为 json 而非 string

### 3.5 各类型转换行为

| 类型 | 转换函数 | 行为 |
|------|---------|------|
| `string` | lodash `toString()` | 转为字符串 |
| `number` | lodash `toNumber()` | 转为数字 |
| `boolean` | `@directus/utils` `toBoolean()` | 转为布尔值 |
| `regex` | `new RegExp(String(value))` | 构造正则表达式 |
| `array` | `@directus/utils` `toArray()` | 逗号分隔转数组，每个元素递归 `cast()`，过滤空串 |
| `json` | [tryJson](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/try-json.ts#L3-L8) → `parseJSON()` | 尝试 JSON 解析，失败则保留原值 |

**array 类型的递归 cast**：数组中的每个元素会再次调用 `cast(v)`（不传 key），此时会走 `guessType` 推断。这意味着 `array:string:hey,number:1` 可以正确产出 `['hey', 1]`。

---

## 4. `*_FILE` 机制：从文件读取敏感值

实现位于 [create-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/create-env.ts#L27-L43)。

### 4.1 判定逻辑

三个条件必须同时满足：

```typescript
if (isFileKey(key) && isDirectusVariable(key) && typeof value === 'string')
```

1. **[isFileKey](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/is-file-key.ts#L4)**：键名长度 > 5 且以 `_FILE` 结尾
2. **[isDirectusVariable](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/is-directus-variable.ts#L3-L8)**：去掉 `_FILE` 后缀的键名匹配 [DIRECTUS_VARIABLES](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/directus-variables.ts#L4-L323) 列表中的某个模式
3. **值类型为 string**：因为文件路径必须是字符串

### 4.2 读取流程

```typescript
// 1. 提取 cast 前缀（如果有）
const castFlag = getCastFlag(value);         // 如 "string:/path/to/secret"
const castPrefix = castFlag ? castFlag + ':' : '';
const filePath = castFlag ? value.replace(castPrefix, '') : value;

// 2. 读取文件内容
const fileContent = readFileSync(filePath, { encoding: 'utf8' });

// 3. 恢复键名（去掉 _FILE 后缀）并重组值
key = removeFileSuffix(key);                 // DB_PASSWORD_FILE → DB_PASSWORD
value = castPrefix + fileContent;            // 保留 cast 前缀用于后续 cast()
```

### 4.3 使用示例

```bash
# 通过文件提供数据库密码
DB_PASSWORD_FILE=/run/secrets/db_password

# 通过文件提供密钥，并指定类型前缀
KEY_FILE=string:/run/secrets/directus_key
```

`DB_PASSWORD_FILE` 的值被读取为文件内容，键名变为 `DB_PASSWORD`，后续经过 `cast()` 时因 TYPE_MAP 中 `DB_PASSWORD` 映射为 `string`，确保密码不会被错误转换。

---

## 5. useEnv：单例缓存

实现位于 [use-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/use-env.ts#L4-L16)。

```typescript
export const _cache: { env: Env | undefined } = { env: undefined };

export const useEnv = (): Env => {
    if (_cache.env) {
        return _cache.env;    // 命中缓存直接返回
    }
    _cache.env = createEnv();  // 首次调用时执行完整初始化
    return _cache.env;
};
```

- **懒初始化**：`createEnv()` 仅在首次调用 `useEnv()` 时执行
- **全局单例**：`_cache.env` 一旦赋值，后续所有调用返回同一对象引用
- **对外导出**：[packages/env/src/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/index.ts#L1) 仅导出 `useEnv`
- **无重置机制**：运行期间没有提供清除缓存的方法，确保配置一致性

---

## 6. getConfigFromEnv：前缀提取与嵌套对象转换

实现位于 [get-config-from-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-config-from-env.ts#L12-L59)。

### 6.1 前缀匹配

```typescript
const lowerCasePrefix = prefix.toLowerCase();
for (const [key, value] of Object.entries(env)) {
    const lowerCaseKey = key.toLowerCase();
    if (lowerCaseKey.startsWith(lowerCasePrefix) === false) continue;
    // ...
}
```

匹配不区分大小写，如 `STORAGE_LOCAL_DRIVER` 匹配前缀 `STORAGE_LOCAL_`。

### 6.2 双下划线 → 嵌套对象

```typescript
if (key.includes('__')) {
    const path = key
        .split('__')
        .map((key, index) =>
            index === 0
                ? transform(transform(key.slice(prefix.length)))  // 首段去前缀并转换
                : transform(key)                                  // 后续段直接转换
        );
    set(config, path.join('.'), value);  // lodash set 创建嵌套结构
}
```

**转换规则**（由 `transform` 函数决定）：
- `type: 'camelcase'`（默认）：使用 `camelcase()` 转为驼峰
- `type: 'underscore'`：使用 `toLowerCase()` 转为下划线小写

### 6.3 转换示例

```
环境变量                              →  前缀 "STORAGE_LOCAL_"  →  结果
───────────────────────────────────────────────────────────────────────
STORAGE_LOCAL_DRIVER                  →  { driver: 'local' }
STORAGE_LOCAL_ROOT                    →  { driver: 'local', root: './uploads' }
STORAGE_LOCAL_S3__REGION              →  { s3: { region: 'us-east-1' } }
STORAGE_LOCAL_S3__ACCESS_KEY_ID       →  { s3: { region: '...', accessKeyId: '...' } }

环境变量                              →  前缀 "EMAIL_SES_CREDENTIALS__"  →  结果
───────────────────────────────────────────────────────────────────────
EMAIL_SES_CREDENTIALS__ACCESS_KEY_ID  →  { accessKeyId: '...' }
EMAIL_SES_CREDENTIALS__SECRET_ACCESS_KEY → { secretAccessKey: '...' }
```

注意 `DB_SSL__CA_FILE` 中的 `__` 会被转换为嵌套对象 `{ ssl: { caFile: '...' } }`。

### 6.4 omitKey 与 omitPrefix

- `omitKey`：排除完全匹配（不区分大小写）的键
- `omitPrefix`：排除以指定前缀开头的键

---

## 7. 业务代码使用示例

### 7.1 QUERYSTRING_MAX_PARSE_DEPTH → 中间件

**默认值**：`10`（[defaults.ts#L14](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L14)）

**使用位置**：[app.ts#L155-L160](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L155-L160)

```typescript
app.set('query parser', (str: string) =>
    qs.parse(str, {
        depth: Number(env['QUERYSTRING_MAX_PARSE_DEPTH']),
        arrayLimit: Number(env['QUERYSTRING_ARRAY_LIMIT']),
    }),
);
```

**影响链路**：

```
QUERYSTRING_MAX_PARSE_DEPTH=10
    → useEnv() 返回 { QUERYSTRING_MAX_PARSE_DEPTH: 10 }
    → app.set('query parser', qs.parse depth=10)
    → 所有 HTTP 请求的 query string 解析最大嵌套深度为 10 层
    → 超过深度的嵌套参数将被忽略，防止原型污染攻击
```

这是 Express 应用的全局 query parser 配置，在所有中间件之前生效，属于最底层的安全防护之一。

### 7.2 FILES_MAX_UPLOAD_SIZE → 文件上传

**默认值**：无默认值（未在 DEFAULTS 中定义），`bytes.parse(undefined)` 返回 `null`

**使用位置**：

1. [controllers/files.ts#L46](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L46)：Busboy 解析器的文件大小限制

    ```typescript
    const busboy = Busboy({
        headers,
        defParamCharset: 'utf8',
        limits: {
            fileSize: bytes.parse(env['FILES_MAX_UPLOAD_SIZE'] as string) ?? undefined,
        },
    });
    ```

2. [constants.ts#L99](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts#L99)：全局常量

    ```typescript
    export const FILE_UPLOADS = {
        MAX_SIZE: bytes.parse(env['FILES_MAX_UPLOAD_SIZE'] as string),
        MAX_CONCURRENCY: Number(env['FILES_MAX_UPLOAD_CONCURRENCY']),
    };
    ```

**影响链路**：

```
FILES_MAX_UPLOAD_SIZE="100mb"
    → useEnv() 返回 { FILES_MAX_UPLOAD_SIZE: "100mb" }
        （注意：无 TYPE_MAP 映射，guessType("100mb") = 'json'，
         tryJson("100mb") 解析失败保留原字符串 "100mb"）
    → Busboy limits.fileSize = bytes.parse("100mb") = 104857600 (bytes)
    → 上传文件超过 100MB 时 Busboy 自动截断并触发 error 事件
    → 若未设置，bytes.parse(undefined) = null → ?? undefined → 无大小限制
```

### 7.3 WEBSOCKETS_ENABLED → WebSocket 行为

**默认值**：`false`（[defaults.ts#L176](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L176)）

**影响位置**（多处使用 `toBoolean()` 二次确认）：

1. **[server.ts#L104-L110](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/server.ts#L104-L110)**：服务启动时决定是否初始化 WebSocket 控制器

    ```typescript
    if (toBoolean(env['WEBSOCKETS_ENABLED']) === true) {
        createSubscriptionController(server);
        createWebSocketController(server);
        createLogsController(server);
        startWebSocketHandlers();
    }
    ```

2. **[services/websocket.ts#L15-L16](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/websocket.ts#L15-L16)**：服务层守卫，禁用时抛出 503

    ```typescript
    if (!toBoolean(env['WEBSOCKETS_ENABLED']) || !toBoolean(env['WEBSOCKETS_REST_ENABLED'])) {
        throw new ServiceUnavailableError({ service: 'ws', reason: 'WebSocket server is disabled' });
    }
    ```

3. **[services/server.ts#L136-L167](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/server.ts#L136-L167)**：`/server/info` 端点返回 WebSocket 配置信息

4. **[graphql/resolvers/system.ts#L112](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/graphql/resolvers/system.ts#L112)**：GraphQL 系统解析器暴露 WebSocket 状态

**影响链路**：

```
WEBSOCKETS_ENABLED=false (默认)
    → useEnv() 返回 { WEBSOCKETS_ENABLED: false }
        （DEFAULTS 中默认值为 false，cast(false, 'WEBSOCKETS_ENABLED')
         → getDefaultType 无匹配 → guessType(false) = 'boolean' → toBoolean(false) = false）
    → server.ts: 不创建 WS 控制器，HTTP 服务器无 WS 升级处理
    → WebSocketService: 抛出 ServiceUnavailableError
    → /server/info: websocket = false

WEBSOCKETS_ENABLED=true
    → useEnv() 返回 { WEBSOCKETS_ENABLED: true }
    → server.ts: 创建 REST/GraphQL/Logs/Collab 四个 WS 控制器
    → /server/info: websocket = { rest: {...}, graphql: {...}, heartbeat: 30, ... }
```

### 7.4 STORAGE_LOCATIONS → 存储注册

**默认值**：`'local'`（[defaults.ts#L25](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L25)）

**核心使用**：[register-locations.ts#L7-L22](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/storage/register-locations.ts#L7-L22)

```typescript
export const registerLocations = async (storage: StorageManager) => {
    const env = useEnv();
    const locations = toArray(env['STORAGE_LOCATIONS'] as string);

    locations.forEach((location: string) => {
        location = location.trim();
        const driverConfig = getConfigFromEnv(`STORAGE_${location.toUpperCase()}_`);
        const { driver, ...options } = driverConfig;
        storage.registerLocation(location, { driver, options: { ...options, tus } });
    });
};
```

**影响链路**：

```
STORAGE_LOCATIONS="local,s3"
    → cast("local,s3", 'STORAGE_LOCATIONS')
        → getDefaultType('STORAGE_LOCATIONS') 无匹配
        → guessType("local,s3") → 包含逗号 → 'array'
        → toArray("local,s3") → ['local', 's3']
    → toArray(env['STORAGE_LOCATIONS']) = ['local', 's3']

对每个 location（如 "s3"）：
    → getConfigFromEnv('STORAGE_S3_')
    → 匹配所有 STORAGE_S3_ 开头的环境变量：

    STORAGE_S3_DRIVER=s3
    STORAGE_S3_KEY=AKIAIOSFODNN7EXAMPLE
    STORAGE_S3_SECRET=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    STORAGE_S3_BUCKET=my-bucket
    STORAGE_S3_REGION=us-east-1
    STORAGE_S3_ENDPOINT=https://s3.amazonaws.com
    STORAGE_S3_SERVER_SIDE_ENCRYPTION=true

    → 转换为:
    {
        driver: 's3',
        key: 'AKIAIOSFODNN7EXAMPLE',
        secret: 'wJalrXUtnFEMI/...',
        bucket: 'my-bucket',
        region: 'us-east-1',
        endpoint: 'https://s3.amazonaws.com',
        serverSideEncryption: true   // camelcase 转换
    }

    → storage.registerLocation('s3', { driver: 's3', options: {...} })
```

**嵌套配置示例**：

```
STORAGE_S3_CREDENTIALS__ACCESS_KEY_ID=AKIA...
STORAGE_S3_CREDENTIALS__SECRET_ACCESS_KEY=wJalrX...
    → getConfigFromEnv('STORAGE_S3_') 产出:
    {
        credentials: {
            accessKeyId: 'AKIA...',
            secretAccessKey: 'wJalrX...'
        }
    }
```

### 7.5 MCP_OAUTH_* → OAuth 行为

**默认值**（[defaults.ts#L232-L246](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L232-L246)）：

| 变量 | 默认值 | TYPE_MAP 类型 |
|------|--------|-------------|
| `MCP_OAUTH_ENABLED` | `false` | `boolean` |
| `MCP_OAUTH_AUTH_CODE_TTL` | `'60s'` | `string` |
| `MCP_OAUTH_MAX_CLIENTS` | `10000` | `number` |
| `MCP_OAUTH_CLIENT_UNUSED_TTL` | `'24h'` | `string` |
| `MCP_OAUTH_CLIENT_IDLE_TTL` | `'0'` | `string` |
| `MCP_OAUTH_REQUIRE_RESOURCE` | `false` | `boolean` |
| `MCP_OAUTH_CLEANUP_SCHEDULE` | `'*/15 * * * *'` | `string` |
| `MCP_OAUTH_ALLOWED_REDIRECT_DOMAINS` | `''` | `array` |
| `MCP_OAUTH_ALLOWED_CUSTOM_REDIRECTS` | `'raycast://...,cursor://...'` | `array` |
| `MCP_OAUTH_DCR_ENABLED` | `false` | `boolean` |
| `MCP_OAUTH_CIMD_ENABLED` | `false` | `boolean` |
| `MCP_OAUTH_CIMD_ALLOW_HTTP` | `false` | `boolean` |
| `MCP_OAUTH_CIMD_ALLOWED_DOMAINS` | `''` | `array` |
| `MCP_OAUTH_CIMD_BLOCKED_TLDS` | `'test,localhost,...'` | `array` |

**影响位置**：

1. **[app.ts#L127-L131](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L127-L131)**：前置依赖检查

    ```typescript
    if (env['MCP_OAUTH_ENABLED'] === true) {
        if (toBoolean(env['MCP_ENABLED']) !== true) {
            logger.warn('MCP_OAUTH_ENABLED requires MCP_ENABLED=true. OAuth disabled.');
            env['MCP_OAUTH_ENABLED'] = false;
        }
    }
    ```

2. **[app.ts#L324-L333](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L324-L333)**：注册 OAuth 路由

    ```typescript
    if (env['MCP_OAUTH_ENABLED'] === true) {
        app.use(mcpOAuthPublicRouter);     // 公开端点（/authorize, /token 等）
    }
    app.use(authenticate);
    app.use(mcpOAuthGuard);
    if (env['MCP_OAUTH_ENABLED'] === true) {
        app.use(mcpOAuthProtectedRouter);  // 受保护端点
    }
    ```

3. **[app.ts#L397-L399](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L397-L399)**：注册客户端管理路由

    ```typescript
    if (toBoolean(env['MCP_OAUTH_ENABLED']) === true) {
        app.use('/mcp-oauth/clients', mcpOAuthClientsRouter);
    }
    ```

4. **[app.ts#L426-L428](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L426-L428)**：启动定时清理任务

    ```typescript
    if (env['MCP_OAUTH_ENABLED'] === true) {
        await scheduleOAuthCleanup();
    }
    ```

5. **[controllers/mcp/index.ts#L43-L48](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/mcp/index.ts#L43-L48)**：MCP 端点守卫

    ```typescript
    if (req.accountability?.oauth &&
        (toBoolean(env['MCP_OAUTH_ENABLED']) !== true || toBoolean(mcp_oauth_enabled) !== true)) {
        throw new ForbiddenError({ reason: 'MCP OAuth must be enabled' });
    }
    ```

6. **[middleware/error-handler.ts#L42](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/middleware/error-handler.ts#L42)**：OAuth 感知的错误处理

    ```typescript
    if (isMcpPath(req.path) && env['MCP_OAUTH_ENABLED'] === true) {
        res.status(401)
           .set('WWW-Authenticate', buildMcpChallengeHeader(metadataUrl, 'invalid_token'))
           .json({ error: 'invalid_token', ... });
    }
    ```

**影响链路**：

```
MCP_OAUTH_ENABLED=true, MCP_ENABLED=true
    → cast(true, 'MCP_OAUTH_ENABLED')
        → getDefaultType('MCP_OAUTH_ENABLED') = 'boolean' (TYPE_MAP)
        → toBoolean(true) = true
    → app.ts: 依赖检查通过（MCP_ENABLED=true）
    → app.ts: 挂载 mcpOAuthPublicRouter（/authorize, /token, /consent 等）
    → app.ts: 挂载 mcpOAuthProtectedRouter
    → app.ts: 挂载 /mcp-oauth/clients 客户端管理路由
    → app.ts: 启动 scheduleOAuthCleanup() 定时清理过期客户端和授权码
    → error-handler: MCP 路径 401 错误返回 WWW-Authenticate 头（RFC 6750）
    → mcp/index.ts: 允许 OAuth 认证的请求访问 MCP 端点

MCP_OAUTH_ENABLED=true, MCP_ENABLED=false
    → app.ts: 依赖检查失败，强制设 env['MCP_OAUTH_ENABLED'] = false
    → 所有 OAuth 路由均不挂载

MCP_OAUTH_ALLOWED_REDIRECT_DOMAINS="example.com,api.example.com"
    → cast("example.com,api.example.com", 'MCP_OAUTH_ALLOWED_REDIRECT_DOMAINS')
        → getDefaultType → TYPE_MAP = 'array'
        → toArray("example.com,api.example.com") → ['example.com', 'api.example.com']
    → OAuth 授权回调仅允许重定向到这两个域名
```

---

## 8. 完整数据流示意

以 `STORAGE_S3_SECRET_FILE=/run/secrets/s3_secret` 为例：

```
1. readConfigurationFromProcess()
   → { ..., STORAGE_S3_SECRET_FILE: '/run/secrets/s3_secret', ... }

2. readConfigurationFromFile('.env') → { ... }

3. rawConfiguration = { ...baseConfiguration, ...fileConfiguration }

4. createEnv() 第二轮遍历:
   key = 'STORAGE_S3_SECRET_FILE'
   value = '/run/secrets/s3_secret'

   isFileKey('STORAGE_S3_SECRET_FILE')     → true  (以 _FILE 结尾)
   isDirectusVariable('STORAGE_S3_SECRET') → true  (匹配 'STORAGE_.+_SECRET' 正则)
   typeof value === 'string'               → true

   → 读取文件: fileContent = readFileSync('/run/secrets/s3_secret')
   → key = 'STORAGE_S3_SECRET' (去掉 _FILE)
   → value = fileContent (文件内容)

5. cast(fileContent, 'STORAGE_S3_SECRET')
   → getCastFlag(fileContent) → null (无类型前缀)
   → getDefaultType('STORAGE_S3_SECRET') → TYPE_MAP: 'STORAGE_.+_SECRET' 匹配 → 'string'
   → toString(fileContent) → 密钥字符串

6. output['STORAGE_S3_SECRET'] = 'wJalrXUtnFEMI/K7MDENG/...'

7. useEnv() 缓存

8. registerLocations() 调用 getConfigFromEnv('STORAGE_S3_')
   → 遍历 env，匹配 STORAGE_S3_ 开头的键
   → STORAGE_S3_SECRET: 'wJalrXUtnFEMI/K7MDENG/...'
   → 产出 { secret: 'wJalrXUtnFEMI/K7MDENG/...', ... }
   → storage.registerLocation('s3', { driver: 's3', options: { secret: '...', ... } })
```

---

## 9. 关键源文件索引

| 文件 | 职责 |
|------|------|
| [defaults.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts) | 所有环境变量默认值定义 |
| [type-map.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/type-map.ts) | 键名到类型的显式映射（支持正则） |
| [env-types.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/env-types.ts) | 合法的 cast 类型前缀列表 |
| [directus-variables.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/directus-variables.ts) | 合法的 Directus 变量名列表（用于 *_FILE 校验） |
| [create-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/create-env.ts) | createEnv 主流程：合并 + *_FILE + cast |
| [cast.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/cast.ts) | 类型转换引擎 |
| [use-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/use-env.ts) | 单例缓存 |
| [guess-type.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/guess-type.ts) | 自动类型推断 |
| [has-cast-prefix.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/has-cast-prefix.ts) | 检测显式类型前缀 |
| [get-default-type.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/get-default-type.ts) | 从 TYPE_MAP 查找键名类型 |
| [is-file-key.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/is-file-key.ts) | 判断是否为 *_FILE 键 |
| [is-directus-variable.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/utils/is-directus-variable.ts) | 判断是否为合法 Directus 变量名 |
| [read-configuration-from-file.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/lib/read-configuration-from-file.ts) | 配置文件读取分发 |
| [get-config-from-env.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-config-from-env.ts) | 前缀提取 + 嵌套对象转换 |
