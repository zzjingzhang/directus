# Flow Operation 三类 Handler 比较：item-create、request、trigger

本文档深入比较 Directus Flow 引擎中 `item-create`、`request`、`trigger` 三类 operation 的输入转换、权限选择和输出传播机制，并说明它们如何被 `executeOperation` 包装成 resolve/reject 路径。

---

## 1. 整体架构：executeOperation 包装层

所有 operation handler 都经由 [FlowManager.executeOperation](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L506-L600) 统一调度，形成标准化的 resolve / reject 双路径执行模型。

### 1.1 调用时序

```
executeFlow
  │
  ├─ 构造 keyedData ($trigger, $last, $accountability, $env)
  │
  └─ while (nextOperation)
       └─ executeOperation(operation, keyedData, context)
            │
            ├─ applyOptionsData(options, keyedData)   ← 输入模板渲染
            ├─ handler(options, context)               ← 调用具体 handler
            ├─ 成功 → status='resolve', successor=operation.resolve
            └─ 失败 → status='reject',  successor=operation.reject
```

### 1.2 输入转换：applyOptionsData

在调用 handler 之前，`executeOperation` 会先通过 [applyOptionsData](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/utils/shared/apply-options-data.ts#L14-L22) 对 operation 的 options 进行 Mustache 模板渲染，将 `keyedData`（含 `$trigger`、`$last`、各步骤 keyed 结果等）中的数据注入到 options 模板字符串中。

- 纯变量替换：`"{{ $last.id }}"` → 直接返回对应值（保持原始类型）
- 字符串内插值：`"User {{ $trigger.name }} created"` → 渲染为字符串
- 对象和数组递归处理
- 特殊操作（如 log）会对敏感字段做脱敏

### 1.3 Resolve 路径（成功）

[flows.ts#L557-L577](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L557-L577)

1. `JSON.stringify(result ?? null)` 验证结果可序列化
2. `deepMap(result)` 将所有 `undefined` 替换为 `null`（JSON 不支持 undefined，避免后续 applyOptionsData 出错）
3. 返回 `{ successor: operation.resolve, status: 'resolve', data: result ?? null, options }`
4. 下一步操作指向 `operation.resolve`

### 1.4 Reject 路径（失败）

[flows.ts#L578-L598](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L578-L598)

handler 抛出的任何异常都会被捕获并标准化：

| 错误类型 | 处理方式 | data 结果 |
|---------|---------|----------|
| `Error` 实例 | 删除 `stack` 属性，保留 error 对象 | Error 对象（无 stack） |
| 字符串 | 若为合法 JSON 则 `parseJSON`，否则原样保留 | 解析后的对象或原字符串 |
| 其他类型 / null | 直接使用（空值兜底为 null） | 原值或 null |

返回 `{ successor: operation.reject, status: 'reject', data, options }`，下一步操作指向 `operation.reject`。

---

## 2. item-create Operation

**文件**：[api/src/operations/item-create/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/item-create/index.ts)

### 2.1 选项类型

```typescript
type Options = {
    collection: string;
    payload?: Record<string, any> | string | null;
    emitEvents: boolean;
    permissions: string; // $public, $trigger, $full, or UUID of a role
};
```

### 2.2 输入转换

1. **payload 转换**：使用 `optionToObject(payload)` —— 若 payload 是 JSON 字符串则解析为对象，否则直接使用；最终 `?? null` 兜底。
2. **payload 数组化**：使用 `toArray(payloadObject)` 将单个对象包装为数组，统一走 `createMany` 路径。

### 2.3 权限选择：permissions → accountability 映射

item-create 的核心特色是通过 `permissions` 选项动态构造自定义 accountability，共 4 种模式：

| permissions 值 | 行为 | 调用函数 |
|---------------|------|---------|
| `$trigger`（或空） | 直接沿用 flow 上下文的 accountability | 无（直接赋值） |
| `$full` | 系统级管理员权限（admin: true, app: true） | `getAccountabilityForRole('system', ...)` |
| `$public` | 公开/匿名权限 | `getAccountabilityForRole(null, ...)` |
| 角色 UUID | 指定角色及其权限树 | `getAccountabilityForRole(roleId, ...)` |

**getAccountabilityForRole 内部逻辑**（[get-accountability-for-role.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-accountability-for-role.ts)）：

- `null` → `createDefaultAccountability()`（默认空权限）
- `'system'` → `createDefaultAccountability({ admin: true, app: true })`（系统管理员）
- 角色 UUID → `fetchRolesTree(role)` + `fetchGlobalAccess(...)` → 构造带 role、roles、user、权限的 accountability

### 2.4 ItemsService 调用

```typescript
const itemsService = new ItemsService(collection, {
    schema: await getSchema({ database }),
    accountability: customAccountability,
    knex: database,
});

result = await itemsService.createMany(toArray(payloadObject), { emitEvents: !!emitEvents });
```

关键点：
- 传入的是 `customAccountability` 而非原始 accountability
- `emitEvents` 控制是否触发 items.create 等事件
- `schema` 通过 `getSchema({ database })` 重新获取（确保与当前数据库一致）

### 2.5 输出传播

- **成功（resolve）**：返回 `PrimaryKey[]`（创建记录的主键数组）；若 payload 为空则返回 `null`
- **失败（reject）**：ItemsService 抛出的任何异常（如权限不足、校验失败等）直接冒泡，由 executeOperation 捕获并走 reject 路径

---

## 3. request Operation

**文件**：[api/src/operations/request/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/request/index.ts)

### 3.1 选项类型

```typescript
type Options = {
    url: string;
    method: string;
    body: Record<string, any> | string | null;
    headers?: { header: string; value: string }[] | null;
};
```

### 3.2 输入转换

**Headers 转换**：将 `{ header, value }[]` 数组结构转换为 `Record<string, string>` 对象。

```typescript
const customHeaders =
    headers?.reduce(
        (acc, { header, value }) => {
            acc[header] = value;
            return acc;
        },
        {} as Record<string, string>,
    ) ?? {};
```

**Content-Type 自动推断**：
- 若用户未显式设置 `Content-Type`
- 且 `body` 是对象 **或** `body` 是合法 JSON 字符串
- → 自动设置为 `'application/json'`

**URL 编码**：使用 `encodeUrl(url)` 对 URL 进行编码处理。

### 3.3 Axios 调用

使用 `getAxios()` 获取预配置的 axios 实例（而非直接 import axios），确保共享拦截器、超时等全局配置。

请求配置：
```typescript
await axios({
    url: encodeUrl(url),
    method,
    data: body,
    headers: customHeaders,
});
```

### 3.4 输出传播

**成功（resolve）**：
返回结构化对象 `{ status, statusText, headers, data }`，仅提取响应的核心字段，而非整个 axios response 对象。

**失败（reject）** —— **特殊的 Axios 错误格式处理**：

```typescript
if (isAxiosError(error) && error.response) {
    throw JSON.stringify({
        status: error.response.status,
        statusText: error.response.statusText,
        headers: error.response.headers,
        data: error.response.data,
    });
} else {
    throw error;
}
```

关键点：
- 有响应体的 Axios 错误（HTTP 错误，如 404、500）：**序列化为 JSON 字符串后抛出**
  - 这样做是为了让 executeOperation 的 reject 路径能以 JSON 解析方式提取结构化错误信息
  - 下游 reject 分支可通过 `{{ $last.status }}`、`{{ $last.data }}` 访问错误详情
- 非 Axios 错误或无 response 的错误（如网络超时、DNS 失败）：原样抛出
  - 由 executeOperation 按普通 Error 处理（剥掉 stack）

---

## 4. trigger Operation

**文件**：[api/src/operations/trigger/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/trigger/index.ts)

### 4.1 选项类型

```typescript
type Options = {
    flow: string;                              // 目标子 flow ID
    payload?: Record<string, any> | Record<string, any>[] | string | null;
    iterationMode?: 'serial' | 'batch' | 'parallel';
    batchSize?: number;                        // 仅 batch 模式使用，默认 10
};
```

### 4.2 输入转换

**Payload 转换**：`optionToObject(payload) ?? null` —— JSON 字符串解析为对象/数组，否则原样使用。

**Context 透传**：使用 `omit(context, 'data')` 去掉父 flow 的 `data`（keyedData）后传给子 flow，避免数据污染。

### 4.3 执行模式映射

trigger 的核心是根据 `iterationMode` 和 payload 是否为数组，调度子 operation flow 的执行方式。

#### 场景 A：payload 非数组（单个执行）

直接调用一次 `flowManager.runOperationFlow(flow, payloadObject, context)`。

#### 场景 B：payload 是数组（迭代执行）

共三种迭代模式：

| 模式 | 执行方式 | 并发特性 | 适用场景 |
|-----|---------|---------|---------|
| **serial**（串行） | `for` 循环逐个 `await` | 0 并发，逐个执行 | 有顺序依赖、避免资源竞争 |
| **batch**（分批） | 按 `batchSize`（默认 10）分批次，每批内 `Promise.all` 并行 | 有限并发（batchSize 个） | 控制并发量，防止打爆下游 |
| **parallel**（并行，默认） | 整个数组一次性 `Promise.all` | 全并发 | 数量少、无资源约束 |

**串行模式 (serial)**：
```typescript
for (const payload of payloadObject) {
    result.push(await flowManager.runOperationFlow(flow, payload, context));
}
```

**分批模式 (batch)**：
```typescript
const size = batchSize ?? 10;
for (let i = 0; i < payloadObject.length; i += size) {
    const batch = payloadObject.slice(i, i + size);
    const batchResults = await Promise.all(
        batch.map((payload) => flowManager.runOperationFlow(flow, payload, context))
    );
    result.push(...batchResults);
}
```

**并行模式 (parallel)**：
```typescript
return await Promise.all(
    payloadObject.map((payload) => flowManager.runOperationFlow(flow, payload, context))
);
```

### 4.4 runOperationFlow 调用链

`flowManager.runOperationFlow(id, data, context)` → 查找 `this.operationFlowHandlers[id]` → 执行子 flow 的 `executeFlow`。

子 flow 独立维护自己的 `keyedData` 和操作链，与父 flow 隔离。子 flow 的最终返回值作为 trigger 操作的输出。

### 4.5 输出传播

- **单个 payload**：返回子 flow 的执行结果（任意可序列化值）
- **数组 payload**：返回结果数组，顺序与输入数组一致
- **失败传播**：任何一次子 flow 执行抛出异常，都会导致整个 trigger 操作失败（`Promise.all` 或串行 `await` 的天然特性），异常沿 reject 路径向上冒泡

---

## 5. 三类 Operation 对比总表

| 维度 | item-create | request | trigger |
|-----|-------------|---------|---------|
| **核心职责** | 创建数据库记录 | 发起 HTTP 请求 | 触发子 flow 执行 |
| **输入转换** | `optionToObject(payload)` + `toArray()` | headers 数组→对象 + Content-Type 自动推断 + URL 编码 | `optionToObject(payload)` + 模式分发 |
| **权限机制** | 4 种 permissions 模式 → 自定义 accountability → ItemsService | 无（HTTP 鉴权由 headers 自理） | 继承父 flow context（omit data） |
| **服务依赖** | ItemsService、getAccountabilityForRole、getSchema | axios (getAxios)、encodeUrl | FlowManager.runOperationFlow |
| **成功输出** | `PrimaryKey[]` 或 `null` | `{ status, statusText, headers, data }` | 子 flow 结果 或 结果数组 |
| **失败输出** | Service 异常直接抛出 | Axios 响应错误 → JSON 字符串化抛出；其他错误原样抛出 | 子 flow 异常直接冒泡 |
| **Reject 数据形态** | Error 对象 | 结构化 JSON 对象（被 parseJSON 解析后） | 取决于子 flow |
| **与 executeOperation 关系** | handler 异步函数，成功 resolve / 异常 reject | 同左；但主动将 HTTP 错误转为 JSON 字符串以利下游使用 | 同左；迭代模式影响失败时机（任一失败即整体失败） |

---

## 6. 关键源码位置速查

| 组件 | 文件路径 |
|-----|---------|
| executeOperation 包装器 | [flows.ts#L506-L600](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L506-L600) |
| executeFlow 主循环 | [flows.ts#L393-L504](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L393-L504) |
| item-create handler | [operations/item-create/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/item-create/index.ts) |
| request handler | [operations/request/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/request/index.ts) |
| trigger handler | [operations/trigger/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/trigger/index.ts) |
| getAccountabilityForRole | [utils/get-accountability-for-role.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-accountability-for-role.ts) |
| applyOptionsData / optionToObject | [packages/utils/shared/apply-options-data.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/utils/shared/apply-options-data.ts) |
