# FlowManager 执行机制分析

## 概述

FlowManager 是 Directus 中负责流程（Flow）引擎的核心管理类，负责从数据库加载流程定义，构建可执行的操作树，并为不同类型的触发器（Trigger）注册处理器，最终驱动流程的执行。

核心文件：[flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts)

---

## 一、FlowManager.load：从 directus_flows 和 directus_operations 构造成可执行树

### 1.1 数据加载

`load()` 方法首先通过 `FlowsService` 从数据库读取所有状态为 `active` 的流程及其关联的操作：

```typescript
const flows = await flowsService.readByQuery({
    filter: { status: { _eq: 'active' } },
    fields: ['*', 'operations.*'],
    limit: -1,
});
```

- 数据源：
- `directus_flows` 表：存储流程元数据（id、name、trigger、options、operation 等）
- `directus_operations` 表：存储操作节点数据（id、key、type、options、resolve、reject、flow 等）

原始数据结构：
- `FlowRaw`：扁平结构，`operation` 字段是字符串（根操作 ID），`operations` 是操作数组
- `OperationRaw`：扁平结构，`resolve` 和 `reject` 是字符串（后继操作 ID）

### 1.2 构建可执行树

通过 [constructFlowTree](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/construct-flow-tree.ts) 函数将扁平数据转换为嵌套的可执行树结构：

```typescript
const flowTrees = flows.map((flow) => constructFlowTree(flow));
```

构建过程：

1. **找到根操作**：根据 `flow.operation`（根操作 ID，在 operations 数组中找到根操作节点
2. **递归构建子树**：调用 `constructOperationTree` 递归处理每个操作节点
   - 根据 `resolve` ID 查找 resolve 分支的后继操作
   - 根据 `reject` ID 查找 reject 分支的后继操作
   - 递归构建每个后继操作的 resolve/reject 子树
3. **生成 Flow 对象**：
   - `operation` 字段替换为嵌套的 `Operation` 树
   - `options` 默认为空对象（如果为 null）

构建后的类型定义见 [flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/flows.ts)：

```typescript
interface Operation {
    id: string;
    key: string;        // 操作键名，用于在执行上下文中存储结果
    type: string;       // 操作类型，如 condition、log、item-create 等
    options: Record<string, any>;
    resolve: Operation | null;  // 成功分支后继
    reject: Operation | null;   // 失败分支后继
}
```

### 1.3 可执行树的特点

- **二叉分支结构**：每个操作节点有 `resolve`（成功）和 `reject`（失败）两个分支
- **有向无环图**：从根操作开始，沿 resolve 或 reject 路径向下执行
- **通过 key 标识**：每个操作有唯一的 `key`，用于在执行上下文中引用该操作的结果

---

## 二、五类 Trigger 处理器注册

`load()` 方法为每类触发器注册相应的处理器，共支持五种触发类型：`event`、`schedule`、`operation`、`webhook、`manual`。

### 2.1 event（事件触发）

根据 `flow.options.type` 分为 `filter` 和 `action` 两种子类型。

**filter 类型**：
- 通过 `emitter.onFilter(event, handler)` 注册
- Handler 签名：`(payload, meta, context) => executeFlow(...)`
- 事件名生成逻辑：
  - 从 `flow.options.scope` 获取事件范围（如 items.create、items.update、items.delete）
  - 如果是 items.* 类型的事件，需要结合 `flow.options.collections` 生成具体事件名
  - 系统集合（directus_前缀）的事件名转换：`collection.substring(9) + '.' + action
  - 非系统集合：`${collection}.${scope}`

**action 类型**：
- 通过 `emitter.onAction(event, handler)` 注册
- Handler 签名：`(meta, context) => executeFlow(...)`

注册结果存储在 `triggerHandlers` 数组中，每个元素包含：
- `id`: flow id
- `events`: 事件数组，每个事件包含 `type`（filter/action）、`name`（事件名）、`handler`

### 2.2 schedule（定时触发）

- 使用 `scheduleSynchronizedJob(flow.id, cron, handler)` 注册定时任务
- Cron 表达式验证：`validateCron(flow.options.cron)`
- Handler：定时调用 `executeFlow(flow)`
- 异常捕获：定时任务内部 try-catch，错误仅记录日志，不向外传播

### 2.3 operation（操作触发）

- 存储在 `operationFlowHandlers` 对象中，键为 `flow.id`
- Handler：`(data, context) => executeFlow(flow, data, context)`
- 触发方式：通过 `flowManager.runOperationFlow(id, data, context)` 调用
- 典型场景：另一个流程的 trigger 操作节点调用（见 [operations/trigger/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/trigger/index.ts)）

### 2.4 webhook（Webhook 触发）

- 存储在 `webhookFlowHandlers` 对象中，键为 `${method}-${flow.id}`
- 方法：`flow.options.method`，默认为 `GET`
- 缓存控制：GET 方法支持 `cacheEnabled` 选项
- 异步模式：
  - 异步（`flow.options.async` 为 true）：立即返回，不等待执行结果
  - 同步：等待执行完成并返回结果
- **默认 return 值**：`$last`
- Handler 返回：`{ result, cacheEnabled }`

### 2.5 manual（手动触发）

- 同样存储在 `webhookFlowHandlers` 对象中，键为 `POST-${flow.id}`
- 手动触发有完整的权限校验逻辑（详见第三节）
- 强制 return 值：`$last`
- 支持异步模式
- Handler 返回：`{ result }`

---

## 三、manual flow 可执行性检查

手动触发的流程在执行前会进行多重检查，确保调用者有权限执行该流程。检查逻辑位于 [flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L269-L362) 的 `load()` 方法中 manual 分支的 handler 内。

### 3.1 collection 检查

```typescript
const enabledCollections = flow.options?.['collections'] ?? [];
const targetCollection = data?.['body'].collection;

if (!targetCollection) throw ForbiddenError();
if (enabledCollections.length === 0) throw ForbiddenError();
if (!enabledCollections.includes(targetCollection)) throw ForbiddenError();
```

检查项：
- 请求中必须指定 `collection`
- 流程必须配置了允许的集合列表
- 指定的集合必须在允许列表中

### 3.2 requireSelection 检查

```typescript
const requireSelection = flow.options?.['requireSelection'] ?? true;
const targetKeys = data?.['body'].keys;

if (requireSelection && (!targetKeys || !Array.isArray(targetKeys))) throw ForbiddenError();
if (requireSelection && targetKeys.length === 0) throw ForbiddenError();
```

检查项：
- 如果 `requireSelection` 为 true（默认值），必须提供 `keys` 数组
- `keys` 不能为空数组

### 3.3 authenticated 检查

```typescript
const accountability = context?.['accountability'];

if (!accountability) {
    throw ForbiddenError();
}
```

检查项：
- 必须有 `accountability`（已认证用户），即必须登录状态

### 3.4 permissions 检查（非 admin 用户）

```typescript
if (accountability.admin === false) {
    // 1. 获取用户策略
    const policies = await fetchPolicies(accountability, { schema, knex: database });
    
    // 2. 检查对目标集合的 read 权限
    const permissions = await fetchPermissions({
        policies,
        accountability,
        action: 'read',
        collections: [targetCollection],
    });
    
    if (permissions.length === 0) throw ForbiddenError();
    
    // 3. 检查对具体 keys 的访问权限
    if (Array.isArray(targetKeys) && targetKeys.length > 0) {
        const service = getService(targetCollection, { ... });
        const keys = await service.readMany(targetKeys, { fields: [primaryField] });
        const allowedKeys = keys.map((key) => String(key[primaryField]));
        
        if (targetKeys.some((key) => !allowedKeys.includes(String(key)))) {
            throw ForbiddenError();
        }
    }
}
```

检查逻辑：
1. **策略获取**：通过 `fetchPolicies` 获取用户关联的所有策略
2. **集合级权限**：通过 `fetchPermissions` 检查用户对目标集合是否有 `read` 权限
3. **记录级权限**：如果提供了 keys，尝试读取这些记录，验证用户能读取的 keys 是否都在允许范围内
   - 使用目标集合的服务读取 keys
   - 对比请求的 keys 和实际能读取的 keys
   - 如果有任何 key 无法读取，则拒绝执行

### 3.5 keys 检查

keys 检查是 permissions 检查的一部分，用于验证用户有权访问指定的所有记录。

---

## 四、executeFlow 执行流程与上下文维护

### 4.1 执行入口

`executeFlow(flow, data, context)` 是流程执行的核心方法。

参数：
- `flow`: Flow 对象（已构建的可执行树）
- `data`: 触发数据（可选，默认为 null）
- `context`: 执行上下文（可选）

### 4.2 执行上下文初始化

```typescript
const keyedData: Record<string, unknown> = {
    [TRIGGER_KEY]: data,           // '$trigger'
    [LAST_KEY]: data,               // '$last'
    [ACCOUNTABILITY_KEY]: context?.['accountability'] ?? null,  // '$accountability'
    [ENV_KEY]: this.envs,            // '$env'
};
```

四个核心变量：

| 变量 | 说明 | 来源 |
|------|------|------|
| `$trigger` | 触发时的原始数据 | 触发时传入的 data 参数 |
| `$last` | 上一个操作的结果 | 初始为 trigger 数据，每次操作后更新 |
| `$accountability` | 用户认证信息 | 上下文的 accountability |
| `$env` | 环境变量白名单 | `FLOWS_ENV_ALLOW_LIST` 配置的环境变量 |

### 4.3 循环执行操作

```typescript
let nextOperation = flow.operation;
let lastOperationStatus: 'resolve' | 'reject' | 'unknown' = 'unknown';
const steps: { operation: string; key: string; status: string; options: any }[] = [];

while (nextOperation !== null) {
    const { successor, data, status, options } = await this.executeOperation(nextOperation, keyedData, context);
    
    keyedData[nextOperation.key] = data;   // 按 key 存储结果
    keyedData[LAST_KEY] = data;           // 更新 $last
    lastOperationStatus = status;
    steps.push({ operation: nextOperation.id, key: nextOperation.key, status, options });
    
    nextOperation = successor;
}
```

执行流程：
1. 从根操作 `flow.operation` 开始
2. 调用 `executeOperation` 执行当前操作
3. 将操作结果存入 `keyedData`（按键存储和更新 $last
4. 记录步骤日志
5. 将 `successor`（后继操作作为下一个操作
6. 循环直到 `nextOperation` 为 null

### 4.4 步骤日志（steps）

每个执行步骤都被记录在 `steps` 数组中，包含：
- `operation`: 操作 ID
- `key`: 操作键名
- `status`: 执行状态（resolve/reject/unknown）
- `options`: 处理后的选项数据

这些日志会在流程配置了 `accountability` 时存入 revisions 表中。

### 4.5 活动与修订记录

```typescript
if (flow.accountability !== null) {
    // 创建活动记录
    const activity = await activityService.createOne({
        action: Action.RUN,
        user: accountability?.user ?? null,
        collection: 'directus_flows',
        item: flow.id,
        // ...
    });
    
    if (flow.accountability === 'all') {
        // 创建修订记录，包含 steps 和 data
        await revisionsService.createOne({
            activity: activity,
            collection: 'directus_flows',
            item: flow.id,
            data: {
                steps: steps.map(...),  // 脱敏处理
                data: redactObject(keyedData, ...),  // 敏感数据脱敏
            },
        });
    }
}
```

`accountability` 取值：
- `null`: 不记录
- `'activity'`: 仅记录活动
- `'all'`: 记录活动和详细修订（包含 steps 和执行数据

敏感数据脱敏：
- 环境变量值
- Authorization/Cookie 等请求头
- access_token
- password/token/tfa_secret 等敏感字段

---

## 五、resolve/reject 推进机制

### 5.1 executeOperation 执行

[executeOperation](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts#L506-L600) 方法负责执行单个操作节点。

执行流程：

1. **查找操作处理器**：从 `this.operations.get(operation.type)`
2. **选项数据应用**：`applyOptionsData(options, optionData)`，用 keyedData 中的变量替换 options 中的模板变量
3. **执行 handler**：调用操作的 handler 函数
4. **结果验证**：`JSON.stringify(result ?? null)` 验证结果可序列化
5. **undefined 处理**：对象中的 undefined 替换为 null（JSON 不支持 undefined）

### 5.2 resolve 分支（成功）

```typescript
return { successor: operation.resolve, status: 'resolve', data: result ?? null, options };
```

- 操作成功执行（无异常抛出）
- 后继操作：`operation.resolve`
- 状态：`'resolve'`
- 数据：操作返回的结果

### 5.3 reject 分支（失败）

```typescript
catch (error) {
    let data;
    if (error instanceof Error) {
        delete error.stack;  // 不暴露堆栈
        data = error;
    } else if (typeof error === 'string') {
        data = isValidJSON(error) ? parseJSON(error) : error;
    } else {
        data = error ?? null;
    }
    
    return {
        successor: operation.reject,
        status: 'reject',
        data,
        options,
    };
}
```

- 操作抛出异常
- 后继操作：`operation.reject`
- 状态：`'reject'`
- 数据：错误信息（Error 对象删除 stack，字符串尝试解析 JSON）

### 5.4 推进逻辑

在 `executeFlow` 的 while 循环中：
- 成功 → 走 `operation.resolve` 继续
- 失败 → 走 `operation.reject` 继续
- 后继为 null → 流程结束

这形成了类似 Promise 的链式调用模式。

---

## 六、return 字段取值

### 6.1 return 配置

流程的 `flow.options.return` 决定了流程执行完成后返回什么数据。

### 6.2 取值逻辑

```typescript
if (flow.options['return'] === '$all') {
    return keyedData;
} else if (flow.options['return']) {
    return get(keyedData, flow.options['return']);
}

return undefined;
```

三种情况：

1. **`$all`**：返回整个 `keyedData` 对象（包含所有操作的结果、`$trigger`、`$last`、`$accountability`、`$env`）

2. **其他值**：使用 `micromustache` 的 `get` 函数从 `keyedData` 中获取指定路径的值
   - 例如：`$last` → 获取 `$last` 的值
   - 例如：`create_item.data.id` → 获取 create_item 操作结果中的 data.id

3. **未配置**：返回 `undefined`

### 6.3 默认值

- **webhook**：默认 `$last`（`flow.options['return'] = flow.options['return'] ?? '$last'`）
- **manual**：强制 `$last`（`flow.options['return'] = '$last'`）
- **event/schedule/operation**：无默认值，返回 undefined

---

## 七、error_on_reject 错误传播

### 7.1 作用

`error_on_reject` 选项控制当流程最后一个操作以 `reject` 状态结束时，是否向外抛出错误。

### 7.2 触发条件

```typescript
if (
    (flow.trigger === 'manual' || flow.trigger === 'webhook') &&
    flow.options['async'] !== true &&
    flow.options['error_on_reject'] === true &&
    lastOperationStatus === 'reject'
) {
    throw keyedData[LAST_KEY];
}
```

触发条件（同时满足）：
1. 触发类型为 `manual` 或 `webhook`
2. 不是异步模式（`async !== true`）
3. `error_on_reject` 为 `true`
4. 最后一个操作状态为 `reject`

抛出的值：`$last`（即最后一个操作的错误数据）

### 7.3 event filter 触发器的特殊处理

```typescript
if (flow.trigger === 'event' && flow.options['type'] === 'filter' && lastOperationStatus === 'reject') {
    throw keyedData[LAST_KEY];
}
```

对于 event 类型的 filter 触发器，**无论是否配置 error_on_reject**，只要最后状态为 reject 就抛出错误。

这是因为 filter 类型的事件需要通过抛出错误来阻止操作继续执行。

### 7.4 错误传播行为对比

| 触发类型 | error_on_reject | 异步模式 | 最后状态 reject 时行为 |
|---------|----------------|---------|----------------------|
| event (filter) | - | - | 抛出错误 |
| event (action) | - | - | 不抛出（日志记录） |
| schedule | - | - | 不抛出（日志记录） |
| operation | - | - | 不抛出（返回结果） |
| webhook | true | 同步 | 抛出错误 |
| webhook | false/未设置 | 同步 | 不抛出（返回结果） |
| webhook | - | 异步 | 不抛出（立即返回） |
| manual | true | 同步 | 抛出错误 |
| manual | false/未设置 | 同步 | 不抛出（返回结果） |
| manual | - | 异步 | 不抛出（立即返回） |

---

## 八、整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                    FlowManager                       │
├─────────────────────────────────────────────────────────┤
│  flows: { [id: string]: Flow }                   │
│  operations: Map<string, OperationHandler>         │
│  triggerHandlers: TriggerHandler[]  (event/schedule │
│  operationFlowHandlers: { [id]: handler }          │
│  webhookFlowHandlers: { [method-id]: handler }     │
└───────────────────┬───────────────────────────────┘
                    │
      ┌─────────────┼─────────────┐
      │             │             │
┌─────▼─────┐ ┌───▼───┐ ┌─────▼─────┐
│   event   │ │schedule│ │  operation  │
│  trigger  │ │ trigger│ │   trigger   │
└─────┬─────┘ └───┬───┘ └─────┬─────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
              ┌─────▼─────┐
              │ executeFlow │
              └─────┬─────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────▼───┐  ┌──▼────┐  ┌─▼──────┐
   │$trigger │  │ $last  │  │$account│
   └──────────┘  └─────────┘  └──────────┘
        │           │           │
        └───────────┼───────────┘
                    │
              ┌─────▼─────┐
              │Operation Tree│
              └─────┬─────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    ┌────▼─────┐         ┌──▼──────┐
    │ resolve  │         │  reject   │
    │  (成功)  │         │  (失败)  │
    └────┬─────┘         └──┬──────┘
         │                     │
         └──────────┬──────────┘
                    │
              ┌─────▼─────┐
              │  return   │
              │ $all/$last │
              │ / custom  │
              └───────────┘
```

---

## 九、关键文件索引

| 文件 | 说明 |
|------|------|
| [flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/flows.ts) | FlowManager 核心实现 |
| [construct-flow-tree.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/construct-flow-tree.ts) | 流程树构建工具 |
| [flows.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/flows.ts) | Flow/Operation 类型定义 |
| [operations/trigger/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/trigger/index.ts) | Trigger 操作（嵌套调用 Flow |
| [operations/condition/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/operations/condition/index.ts) | Condition 操作示例 |
