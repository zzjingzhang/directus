# Directus AI 接口深度分析：/ai/chat vs /ai/object

## 目录

- [1. 路由注册与中间件链](#1-路由注册与中间件链)
- [2. loadSettings — Settings 读取机制](#2-loadsettings--settings-读取机制)
- [3. 鉴权机制](#3-鉴权机制)
- [4. 模型白名单校验](#4-模型白名单校验)
- [5. Provider 配置与 Registry 构建](#5-provider-配置与-registry-构建)
- [6. Telemetry 遥测配置](#6-telemetry-遥测配置)
- [7. /ai/chat 请求全流程](#7-aichat-请求全流程)
  - [7.1 ChatRequest 校验](#71-chatrequest-校验)
  - [7.2 空 messages 校验](#72-空-messages-校验)
  - [7.3 fixErrorToolCalls](#73-fixerrortoolcalls)
  - [7.4 safeValidateUIMessages](#74-safevalidateuimessages)
  - [7.5 toolApprovals 与客户端工具](#75-toolapprovals-与客户端工具)
  - [7.6 chatRequestToolToAiSdkTool — ALL_TOOLS 查找与 needsApproval 决策](#76-chatrequesttooltoaisdktool--all_tools-查找与-needsapproval-决策)
  - [7.7 createUiStream 完整构建流程](#77-createuistream-完整构建流程)
- [8. /ai/object 请求全流程](#8-aiobject-请求全流程)
  - [8.1 ObjectRequest 校验](#81-objectrequest-校验)
  - [8.2 JSON Schema 处理](#82-json-schema-处理)
  - [8.3 maxOutputTokens](#83-maxoutputtokens)
  - [8.4 streamObject 返回文本流](#84-streamobject-返回文本流)
- [9. 差异对比总表](#9-差异对比总表)

---

## 1. 路由注册与中间件链

两个接口共用同一个 [router.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/router.ts)，注册在 `/ai` 路径下：

```typescript
export const aiRouter = Router()
    .post('/chat', asyncHandler(loadSettings), asyncHandler(aiChatPostHandler))
    .post('/object', asyncHandler(loadSettings), asyncHandler(aiObjectPostHandler));
```

在 [app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L375-L380) 中，仅当 `AI_ENABLED=true` 时挂载：

```typescript
if (toBoolean(env['AI_ENABLED']) === true) {
    await initAIDevTools();
    await initAITelemetry();
    app.use('/ai', aiRouter);
    app.use('/ai/files', aiFilesRouter);
}
```

**全局中间件**：`authenticate` 中间件在所有路由之前全局注册（[app.ts#L328](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L328)），因此两个接口都经过身份认证。

**路由级中间件**：两个接口共享 `loadSettings` 中间件，负责从数据库读取 AI 相关配置。

---

## 2. loadSettings — Settings 读取机制

[load-settings.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/middleware/load-settings.ts) 通过 `SettingsService` 从 `directus_settings` 表读取以下 12 个字段：

| 数据库字段 | AISettings 映射键 | 用途 |
|---|---|---|
| `ai_openai_api_key` | `openaiApiKey` | OpenAI API 密钥 |
| `ai_anthropic_api_key` | `anthropicApiKey` | Anthropic API 密钥 |
| `ai_google_api_key` | `googleApiKey` | Google AI API 密钥 |
| `ai_openai_compatible_api_key` | `openaiCompatibleApiKey` | OpenAI 兼容 API 密钥 |
| `ai_openai_compatible_base_url` | `openaiCompatibleBaseUrl` | OpenAI 兼容 Base URL |
| `ai_openai_compatible_name` | `openaiCompatibleName` | OpenAI 兼容 Provider 名称 |
| `ai_openai_compatible_models` | `openaiCompatibleModels` | OpenAI 兼容自定义模型列表 |
| `ai_openai_compatible_headers` | `openaiCompatibleHeaders` | OpenAI 兼容自定义请求头 |
| `ai_openai_allowed_models` | `openaiAllowedModels` | OpenAI 模型白名单 |
| `ai_anthropic_allowed_models` | `anthropicAllowedModels` | Anthropic 模型白名单 |
| `ai_google_allowed_models` | `googleAllowedModels` | Google 模型白名单 |
| `ai_system_prompt` | `systemPrompt` | 自定义系统提示词 |

读取结果存入 `res.locals['ai']`，供后续 handler 使用：

```typescript
res.locals['ai'] = {
    settings: aiSettings,
    systemPrompt: settings['ai_system_prompt'],
};
```

---

## 3. 鉴权机制

两个接口的鉴权逻辑**完全一致**：

```typescript
if (!req.accountability?.app) {
    throw new ForbiddenError();
}
```

- **全局 authenticate 中间件**：验证 Bearer Token / Session，填充 `req.accountability`
- **app 标志检查**：`req.accountability.app` 必须为 truthy，确保请求来自 Directus App 客户端（而非纯 API 调用）
- 未通过则抛出 `ForbiddenError`

**关键区别**：两者都**不做角色/权限细粒度校验**——只要是从 App 发起的已认证请求即可访问。

---

## 4. 模型白名单校验

两个接口的模型白名单逻辑**完全一致**：

```typescript
const allowedModelsMap: Record<StandardProviderType, string[] | null> = {
    openai: aiSettings.openaiAllowedModels,
    anthropic: aiSettings.anthropicAllowedModels,
    google: aiSettings.googleAllowedModels,
};

if (provider !== 'openai-compatible') {
    const allowedModels = allowedModelsMap[provider];
    if (!allowedModels || allowedModels.length === 0 || !allowedModels.includes(model)) {
        throw new ForbiddenError({ reason: 'Model not allowed for this provider' });
    }
}
```

| Provider | 白名单来源 | 校验规则 |
|---|---|---|
| `openai` | `ai_openai_allowed_models` | null/空数组/不包含 → 拒绝 |
| `anthropic` | `ai_anthropic_allowed_models` | null/空数组/不包含 → 拒绝 |
| `google` | `ai_google_allowed_models` | null/空数组/不包含 → 拒绝 |
| `openai-compatible` | **无白名单** | **跳过校验**，直接放行 |

---

## 5. Provider 配置与 Registry 构建

### 5.1 buildProviderConfigs

[registry.ts#L10-L43](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/providers/registry.ts#L10-L43)：根据 AISettings 中非空的 API Key 构建 `ProviderConfig[]`：

| 条件 | 产出 Config |
|---|---|
| `openaiApiKey` 非空 | `{ type: 'openai', apiKey }` |
| `anthropicApiKey` 非空 | `{ type: 'anthropic', apiKey }` |
| `googleApiKey` 非空 | `{ type: 'google', apiKey }` |
| `openaiCompatibleApiKey` + `openaiCompatibleBaseUrl` 非空 | `{ type: 'openai-compatible', apiKey, baseUrl }` |

### 5.2 createAIProviderRegistry

[registry.ts#L45-L78](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/providers/registry.ts#L45-L78)：基于 configs 创建 AI SDK 的 `ProviderRegistry`：

| Provider | SDK 创建方式 |
|---|---|
| openai | `createOpenAI({ apiKey })` |
| anthropic | `createAnthropicWithFileSupport(apiKey)` (支持文件) |
| google | `createGoogleGenerativeAI({ apiKey })` |
| openai-compatible | `createOpenAICompatible({ name, apiKey, baseURL, headers })` |

openai-compatible 的 `headers` 来自 `openaiCompatibleHeaders`（`{ header, value }[]`），`name` 来自 `openaiCompatibleName`（默认 `'openai-compatible'`）。

### 5.3 getProviderOptions

[options.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/providers/options.ts)：根据 provider + model 返回 provider 特定的选项：

- **OpenAI 推理模型**（`modelDef.reasoning === true`）：返回 `reasoningSummary: 'auto'` + `store: false`
- **OpenAI-Compatible**：如果自定义模型定义了 `providerOptions`，返回 `{ [providerName]: customModel.providerOptions }`
- **其他**：返回空对象 `{}`

### 5.4 两个接口的 Provider 构建差异

| 步骤 | /ai/chat | /ai/object |
|---|---|---|
| buildProviderConfigs | ✅ 在 createUiStream 内部 | ✅ 在 handler 内部 |
| createAIProviderRegistry | ✅ 在 createUiStream 内部 | ✅ 在 handler 内部 |
| getProviderOptions | ✅ 在 createUiStream 内部 | ✅ 在 handler 内部 |
| Provider Config 不存在时 | 抛 `ServiceUnavailableError` | 抛 `ServiceUnavailableError` |

逻辑相同，但 /ai/chat 封装在 `createUiStream` 内，/ai/object 直接在 handler 中执行。

---

## 6. Telemetry 遥测配置

[telemetry/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/telemetry/index.ts)

### 6.1 初始化

在 [app.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/app.ts#L377) 中启动时调用 `initAITelemetry()`：

- 前提：环境变量 `AI_TELEMETRY_ENABLED === true`
- Provider 选择：`AI_TELEMETRY_PROVIDER`（默认 `'langfuse'`，支持 `'braintrust'`）
- 所需密钥：
  - langfuse: `LANGFUSE_SECRET_KEY` + `LANGFUSE_PUBLIC_KEY`
  - braintrust: `BRAINTRUST_API_KEY`
- 初始化结果存入全局 `telemetryState: { recordIO, tracerProvider }`

### 6.2 getAITelemetryConfig

| 参数 | /ai/chat | /ai/object |
|---|---|---|
| `functionId` | 默认 `'directus-ai-chat'` | **显式传入 `'directus-ai-object'`** |
| metadata | `{ provider, model, userId, role }` | `{ provider, model, userId, role }` |

返回结构：
```typescript
{
    isEnabled: true,
    tracer: telemetryState.tracerProvider.getTracer('directus-ai'),
    functionId,
    recordInputs: telemetryState.recordIO,
    recordOutputs: telemetryState.recordIO,
    metadata: { provider, model, userId?, role? },
}
```

未启用遥测时返回 `undefined`，不传给 AI SDK。

---

## 7. /ai/chat 请求全流程

### 7.1 ChatRequest 校验

[chat-request.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/models/chat-request.ts) 定义了请求体的 Zod schema：

```
ChatRequest = Provider DiscriminatedUnion ⋂ {
    tools: ChatRequestTool[],
    messages: looseObject{}[],
    toolApprovals?: Record<string, ToolApprovalMode>,
    context?: ChatContext,
}
```

**Provider 部分**（discriminatedUnion on `provider`）：

| Provider | Schema |
|---|---|
| openai | `{ provider: 'openai', model: string }` |
| anthropic | `{ provider: 'anthropic', model: string }` |
| google | `{ provider: 'google', model: string }` |
| openai-compatible | `{ provider: 'openai-compatible', model: string }` |

**ChatRequestTool**（两种形态）：

```typescript
z.union([
    z.string(),                           // 服务端工具名（如 "items", "schema"）
    z.object({                            // 客户端工具（自定义 schema）
        name: z.string(),
        description: z.string(),
        inputSchema: z.custom<JSONSchema7>(zodJsonSchema7Parser),
    }),
])
```

**ToolApprovalMode**：`z.enum(['always', 'ask', 'disabled'])`

**ChatContext**：
```typescript
{
    attachments?: ContextAttachment[] (max 10),  // item | visual-element | prompt
    page?: { path, collection?, item?, module? },
}
```

校验失败抛 `InvalidPayloadError`。

### 7.2 空 messages 校验

[chat.post.ts#L42-L44](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/controllers/chat.post.ts#L42-L44)：

```typescript
if (rawMessages.length === 0) {
    throw new InvalidPayloadError({ reason: `"messages" must not be empty` });
}
```

这是 ChatRequest Zod schema 之外的**额外校验**——因为 `messages` 在 schema 中定义为 `z.array(z.looseObject({}))`，Zod 允许空数组通过，但业务逻辑要求至少一条消息。

### 7.3 fixErrorToolCalls

[fix-error-tool-calls.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/utils/fix-error-tool-calls.ts)

在 `safeValidateUIMessages` 之前执行，修复前端发来的错误工具调用：

- 遍历 `role === 'assistant'` 消息的 `parts`
- 对 `type` 以 `'tool-'` 开头 + `state === 'output-error'` + 有 `rawInput` 但无 `input` 的 part
- 将 `rawInput` 复制到 `input` 字段

**原因**：前端错误工具调用使用 `rawInput` 存储参数，而 AI SDK 的 `convertToModelMessages` 依赖 `input` 字段生成 API 参数。

### 7.4 safeValidateUIMessages

[chat.post.ts#L62-L66](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/controllers/chat.post.ts#L62-L66)

```typescript
const validationResult = await safeValidateUIMessages({ messages: fixedMessages });
if (validationResult.success === false) {
    throw new InvalidPayloadError({ reason: validationResult.error.message });
}
```

使用 AI SDK 的 `safeValidateUIMessages` 对修复后的消息进行结构验证：
- 验证消息格式是否符合 `UIMessage` 规范
- 验证消息序列是否构成合法的对话流（如 tool-result 必须跟在 tool-call 后）
- 返回验证后的 `UIMessage[]`，用于后续 `convertToModelMessages` 转换

### 7.5 toolApprovals 与客户端工具

[chat.post.ts#L46-L59](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/controllers/chat.post.ts#L46-L59) 中，遍历请求的 `tools` 数组：

```typescript
const tools = requestedTools.reduce<{ [x: string]: Tool }>((acc, t) => {
    const name = typeof t === 'string' ? t : t.name;
    const aiTool = chatRequestToolToAiSdkTool({
        chatRequestTool: t,
        accountability: req.accountability!,
        schema: req.schema,
        ...(toolApprovals && { toolApprovals }),
    }) as Tool;
    acc[name] = aiTool;
    return acc;
}, {});
```

- `toolApprovals` 仅在请求中提供时传入（`Record<string, ToolApprovalMode>`）
- 每个工具（无论是字符串还是对象）都被转换为 AI SDK 的 `Tool` 实例
- **客户端工具**（对象形式）只有 `description` + `inputSchema`，**没有 `execute` 函数**——执行权在客户端

### 7.6 chatRequestToolToAiSdkTool — ALL_TOOLS 查找与 needsApproval 决策

[chat-request-tool-to-ai-sdk-tool.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/utils/chat-request-tool-to-ai-sdk-tool.ts)

#### ALL_TOOLS 定义

[tools/index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/tools/index.ts)：

```typescript
export const ALL_TOOLS: ToolConfig<any>[] = [
    system, items, files, folders, assets,
    flows, triggerFlow, operations, schema,
    collections, fields, relations,
];
```

共 12 个服务端工具，每个 `ToolConfig` 包含：`name`, `description`, `inputSchema`(ZodType), `handler`, 可选 `validateSchema`, `admin`, `endpoint`, `annotations`。

#### 转换逻辑

**分支 1 — 字符串形式（服务端工具）**：

1. 在 `ALL_TOOLS` 中按 `name` 查找，找不到抛 `InvalidPayloadError`
2. 确定 `needsApproval`：
   - 从 `toolApprovals[toolName]` 读取审批模式
   - 默认值为 `'ask'`（未在 toolApprovals 中指定时）
   - `needsApproval = (approvalMode !== 'always')` → 即 `'ask'` 和 `'disabled'` 都需要审批
3. 构建 AI SDK Tool：
   ```typescript
   tool({
       description: directusTool.description,
       inputSchema: zodSchema(directusTool.inputSchema),
       needsApproval,                              // 控制是否需要客户端确认
       execute: async (rawArgs) => {               // 服务端执行
           const coercedArgs = coerceJsonFields(rawArgs);
           // validateSchema 校验 → handler 执行
           return directusTool.handler({ args, accountability, schema });
       },
   })
   ```

**分支 2 — 对象形式（客户端工具）**：

```typescript
tool({
    description: chatRequestTool.description,
    inputSchema: jsonSchema(chatRequestTool.inputSchema),
    // 无 execute 函数 — 客户端执行
    // 无 needsApproval — 客户端自行控制
});
```

#### ToolApprovalMode 语义

| Mode | needsApproval | 含义 |
|---|---|---|
| `'always'` | `false` | 自动执行，无需审批 |
| `'ask'`（默认） | `true` | 每次执行需用户确认 |
| `'disabled'` | `true` | 需审批（虽然名称暗示禁用，但仍设 needsApproval=true） |

---

### 7.7 createUiStream 完整构建流程

[create-ui-stream.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/lib/create-ui-stream.ts)

#### 输入参数

```typescript
interface CreateUiStreamOptions {
    provider: ProviderType;
    model: string;
    tools: { [x: string]: Tool };
    aiSettings: AISettings;
    systemPrompt?: string;
    userId?: string | null;
    role?: string | null;
    context?: ChatContext;
    onUsage?: (usage: PromptCachingUsage) => void;
}
```

#### 步骤 1：Provider Registry 构建

```typescript
const configs = buildProviderConfigs(aiSettings);
const providerConfig = configs.find((c) => c.type === provider);
if (!providerConfig) throw new ServiceUnavailableError(...);
const registry = createAIProviderRegistry(configs, aiSettings);
```

#### 步骤 2：System Prompt 组装

```typescript
const baseSystemPrompt = systemPrompt || SYSTEM_PROMPT;
const contextBlock = context ? formatContextForSystemPrompt(context) : null;
```

- 如果有自定义 `systemPrompt`（来自 Settings），使用自定义；否则使用 [SYSTEM_PROMPT](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/constants/system-prompt.ts) 默认模板
- `formatContextForSystemPrompt`（[format-context.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/utils/format-context.ts)）将 `ChatContext` 转为结构化文本，按优先级排列：
  1. `<custom_instructions>` — prompt 类型附件
  2. `<user_context>` — 页面信息 + item 附件 + 当前日期
  3. `<visual_editing>` — visual-element 附件
  4. 附件规则说明

#### 步骤 3：Anthropic 缓存感知 System Prompt

```typescript
const systemPromptText =
    provider === 'anthropic' || !contextBlock
        ? baseSystemPrompt
        : baseSystemPrompt + contextBlock;

const streamSystemPrompt = buildCacheAwareSystemPrompt(provider, systemPromptText);
```

- **Anthropic**：System prompt 只包含 `baseSystemPrompt`（不含 context），因为 context 会作为独立 user message 注入（见步骤 6）
- **其他 Provider**：System prompt = `baseSystemPrompt + contextBlock`

`buildCacheAwareSystemPrompt` 对 Anthropic 添加 `cacheControl: { type: 'ephemeral' }`：

```typescript
// Anthropic 返回 SystemModelMessage
{
    role: 'system',
    content: systemPromptText,
    providerOptions: { anthropic: { cacheControl: { type: 'ephemeral' } } }
}
// 其他 Provider 返回纯 string
```

#### 步骤 4：Language Model 获取 + DevTools 中间件

```typescript
let languageModel = registry.languageModel(`${provider}:${model}`);
const devToolsMiddleware = getDevToolsMiddleware();
if (devToolsMiddleware) {
    languageModel = wrapLanguageModel({ model: languageModel, middleware: devToolsMiddleware });
}
```

- `getDevToolsMiddleware`：仅当 `AI_DEVTOOLS_ENABLED=true` 且非生产环境时激活
- 使用 AI SDK 的 `wrapLanguageModel` 注入 DevTools 中间件

#### 步骤 5：Anthropic Tool Search

```typescript
const finalTools = sortToolsByName(applyAnthropicToolSearch(provider, model, tools));
```

[anthropic-tool-search.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/providers/anthropic-tool-search.ts)：

- **仅 Anthropic + 非 Haiku 模型 + 有工具时**生效
- 所有工具添加 `providerOptions.anthropic.deferLoading: true`（延迟加载）
- 额外注入 `toolSearch` 工具（`anthropic.tools.toolSearchBm25_20251119`）
- `sortToolsByName`：按工具名字母排序（确保缓存稳定性）

#### 步骤 6：消息转换 + Anthropic 对话缓存

```typescript
const modelMessages = await convertToModelMessages(transformFilePartsForProvider(messages));
const streamMessages = applyAnthropicConversationCaching(provider, modelMessages, contextBlock);
```

- `transformFilePartsForProvider`：将文件 part 的 `url` 替换为 `providerMetadata.directus.fileId`（provider 原生文件 ID）
- `convertToModelMessages`：AI SDK 的 `UIMessage[]` → `ModelMessage[]` 转换
- `applyAnthropicConversationCaching`：
  - **Anthropic**：在最后一条消息添加 `cacheControl: { type: 'ephemeral' }`；如果最后一条是 user 消息且存在 contextBlock，在消息末尾追加 `{ role: 'user', content: contextBlock }`
  - **其他 Provider**：直接返回原始消息

#### 步骤 7：streamText 调用

```typescript
const stream = streamText({
    system: streamSystemPrompt,
    model: languageModel,
    messages: streamMessages,
    stopWhen: [stepCountIs(10)],        // 最多 10 步
    providerOptions,
    tools: finalTools,
    ...(telemetryConfig ? { experimental_telemetry: telemetryConfig } : {}),
    onFinish(result) {
        if (onUsage) {
            onUsage(formatUsageWithCacheTokens(result));
        }
    },
});
```

- `stepCountIs(10)`：最多 10 个多步骤循环（工具调用 → 结果 → 继续调用）
- `onFinish`：流结束后通过 `onUsage` 回调发送用量数据（含 cache token 统计）
- `formatUsageWithCacheTokens`：从 `totalUsage` 和 `providerMetadata` 中提取 `cacheReadTokens` / `cacheCreationTokens`

#### 步骤 8：流式响应

```typescript
stream.pipeUIMessageStreamToResponse(res);
```

使用 AI SDK 的 `pipeUIMessageStreamToResponse`，以 SSE 格式输出 `UIMessageStream`。

同时，handler 中通过 `onUsage` 回调额外写入用量数据：

```typescript
onUsage: (usage) => {
    res.write(`data: ${JSON.stringify({ type: 'data-usage', data: usage })}\n\n`);
},
```

---

## 8. /ai/object 请求全流程

### 8.1 ObjectRequest 校验

[object-request.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/models/object-request.ts)：

```typescript
const ObjectRequest = z.intersection(
    z.discriminatedUnion('provider', [ProviderOpenAi, ProviderAnthropic, ProviderGoogle, ProviderOpenAiCompatible]),
    z.object({
        prompt: z.string(),
        outputSchema: z.custom<JSONSchema7>(zodJsonSchema7Parser, { message: 'Invalid JSON schema' }),
        maxOutputTokens: z.number().int().min(256).optional(),
    }),
);
```

与 ChatRequest 的区别：

| 字段 | ChatRequest | ObjectRequest |
|---|---|---|
| `messages` | ✅ 必需 | ❌ 无 |
| `tools` | ✅ 必需 | ❌ 无 |
| `toolApprovals` | 可选 | ❌ 无 |
| `context` | 可选 | ❌ 无 |
| `prompt` | ❌ 无 | ✅ 必需 (string) |
| `outputSchema` | ❌ 无 | ✅ 必需 (JSONSchema7) |
| `maxOutputTokens` | ❌ 无 | ✅ 可选 (int ≥ 256) |

### 8.2 JSON Schema 处理

[object.post.ts#L68-L72](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/controllers/object.post.ts#L68-L72)：

```typescript
schema: jsonSchema(addAdditionalPropertiesToJsonSchema(outputSchema)),
```

两步处理：

1. **addAdditionalPropertiesToJsonSchema**（[add-additional-properties-to-json-schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/utils/add-additional-properties-to-json-schema.ts)）：
   - 递归遍历 schema，对所有 `type: 'object'` 节点设置 `additionalProperties: false`
   - 处理 `properties`, `items`, `anyOf/allOf/oneOf`, `definitions`, `$defs`
   - OpenAI 和 Anthropic 的结构化输出要求 `additionalProperties: false`

2. **jsonSchema()**：AI SDK 的 `jsonSchema()` 将 JSON Schema 7 转为 AI SDK 可识别的 schema 格式

### 8.3 maxOutputTokens

[object.post.ts#L73](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/controllers/object.post.ts#L73)：

```typescript
...(typeof maxOutputTokens === 'number' ? { maxOutputTokens } : {}),
```

- 仅当请求中提供了 `maxOutputTokens` 且为 number 类型时传入 `streamObject`
- 最小值 256（由 Zod schema 的 `.min(256)` 保证）
- 未提供时不传此参数，使用模型默认值

### 8.4 streamObject 返回文本流

[object.post.ts#L68-L77](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/ai/chat/controllers/object.post.ts#L68-L77)：

```typescript
const result = streamObject({
    model: languageModel,
    prompt,
    schema: jsonSchema(addAdditionalPropertiesToJsonSchema(outputSchema)),
    providerOptions,
    ...(typeof maxOutputTokens === 'number' ? { maxOutputTokens } : {}),
    ...(telemetryConfig ? { experimental_telemetry: telemetryConfig } : {}),
});

result.pipeTextStreamToResponse(res);
```

与 /ai/chat 的 `streamText` + `pipeUIMessageStreamToResponse` 不同，/ai/object 使用：

- **streamObject**：AI SDK 的结构化输出流式生成
- **pipeTextStreamToResponse**：以**纯文本流**输出（而非 SSE UIMessageStream）

Object 接口的完整流程（不含工具/消息/context）：

1. 校验 `req.accountability.app`
2. Zod 校验 `ObjectRequest`
3. 模型白名单校验
4. `buildProviderConfigs` → 查找 provider config
5. `createAIProviderRegistry` → 获取 `languageModel`
6. 应用 DevTools 中间件
7. `getAITelemetryConfig`（functionId = `'directus-ai-object'`）
8. `streamObject` → `pipeTextStreamToResponse`

---

## 9. 差异对比总表

| 维度 | /ai/chat | /ai/object |
|---|---|---|
| **鉴权** | `req.accountability?.app` | `req.accountability?.app`（一致） |
| **loadSettings** | 共享中间件，读 12 个字段 | 共享中间件，读 12 个字段（一致） |
| **模型白名单** | 三个标准 provider 各自白名单，openai-compatible 跳过 | 同左（一致） |
| **Provider Registry** | 在 `createUiStream` 内构建 | 在 handler 内直接构建 |
| **请求体** | `{ provider, model, messages, tools, toolApprovals?, context? }` | `{ provider, model, prompt, outputSchema, maxOutputTokens? }` |
| **消息校验** | 空 messages → InvalidPayloadError；fixErrorToolCalls → safeValidateUIMessages | 无消息（使用 prompt 字符串） |
| **工具调用** | ✅ 支持（ALL_TOOLS + 客户端工具 + needsApproval + Tool Search） | ❌ 不支持 |
| **toolApprovals** | ✅ 支持（always/ask/disabled → needsApproval） | ❌ 不支持 |
| **客户端工具** | ✅ 支持（对象形式，无 execute） | ❌ 不支持 |
| **Context** | ✅ 支持（attachments + page，注入 system prompt 或 user message） | ❌ 不支持 |
| **System Prompt** | ✅ 自定义或默认 + context + Anthropic 缓存标记 | ❌ 无 |
| **Anthropic 缓存** | ✅ System prompt ephemeral cache + 对话消息 cache + context 作为 user message | ❌ 无 |
| **Anthropic Tool Search** | ✅ deferLoading + toolSearchBm25 | ❌ 无 |
| **Step Count** | `stepCountIs(10)`（多步工具调用循环） | 无（单次生成） |
| **DevTools 中间件** | ✅ wrapLanguageModel | ✅ wrapLanguageModel（一致） |
| **Telemetry functionId** | `'directus-ai-chat'` | `'directus-ai-object'` |
| **Telemetry metadata** | `{ provider, model, userId, role }` | `{ provider, model, userId, role }`（一致） |
| **JSON Schema 处理** | 工具的 inputSchema 用 `jsonSchema()` 包装 | outputSchema 递归设 `additionalProperties: false` + `jsonSchema()` |
| **maxOutputTokens** | ❌ 不支持 | ✅ 可选（int ≥ 256） |
| **AI SDK 函数** | `streamText` | `streamObject` |
| **流式输出格式** | SSE `UIMessageStream`（`pipeUIMessageStreamToResponse`）+ 额外 usage SSE 事件 | 纯文本流（`pipeTextStreamToResponse`） |
| **Usage 回调** | `onFinish` → `onUsage` → 写 SSE `data-usage` 事件 | 无 |
| **文件处理** | `transformFilePartsForProvider`（fileId 替换） | 无 |
