# Directus `/assets/:pk` 链路深度分析

## 1. 整体架构概览

`/assets/:pk` 链路由两层构成：

| 层级 | 文件 | 职责 |
|------|------|------|
| **Controller** | [assets.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts) | 查询参数校验、transform_mode 策略决策、格式协商、Range 解析、ETag/If-Modified-Since 重验证、HTTP 响应头设置、流式传输 |
| **Service** | [assets.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts) | 权限校验（含 public system key 绕过）、存储读取、sharp 转换、生成资产缓存命中/写入、Range 边界校验 |

请求处理流程：

```
HTTP GET /assets/:pk/:filename?
  │
  ├─ middleware: useCollection('directus_files') + checkIsLocked('assets')
  │
  ├─ handler 1: 查询参数校验 + transform_mode 策略决策
  │
  ├─ handler 2: Helmet CSP 安全头
  │
  └─ handler 3: 完整的资产获取与响应
       ├─ 格式协商 (Accept header → format)
       ├─ Range 解析（仅无转换时）
       ├─ ETag/If-Modified-Since 重验证（需 ASSETS_CACHE_REVALIDATE=true）
       ├─ AssetsService.getAsset()
       │    ├─ 权限校验（public system key 绕过 or validateItemAccess）
       │    ├─ 缓存命中检查（已存在的生成资产）
       │    ├─ sharp 并发/超时保护
       │    ├─ sharp 转换 + 写入生成资产
       │    └─ 返回 stream + file + stat
       └─ HTTP 响应（Content-Length, ETag, Cache-Control, Vary, 流传输）
```

---

## 2. 七种下载场景的差异分析

### 2.1 原始文件下载

**触发条件**：请求无任何 transform 参数（`key`、`width`、`height`、`format`、`transforms` 等均为空）。

**代码路径**：

- Controller 层：[assets.ts#L232-L237](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L232-L237) —— `Object.keys(transformation).length === 0` 直接 `next()`，跳过所有 transform_mode 校验。
- Service 层：[assets.ts#L401-L409](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L401-L409) —— `transforms.length === 0`，走 `else` 分支，直接读原始文件 `filename_disk`。

**特征**：
- 无 sharp 处理，零延迟
- 支持 Range 请求
- Content-Type 为原始 MIME 类型
- 不写入任何生成资产文件

### 2.2 预设转换（Preset）

**触发条件**：请求带 `?key=<preset_key>`，key 匹配系统预设或用户自定义预设。

**代码路径**：

- Controller 层：[assets.ts#L219-L230](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L219-L230) —— 将系统预设 `SYSTEM_ASSET_ALLOW_LIST` 与用户自定义预设 `storage_asset_presets` 合并到 `res.locals.shortcuts`。
- Controller 层：[assets.ts#L293-L298](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L293-L298) —— 以 key 查找 shortcuts 数组，展开为完整 `transformationParams`。
- Service 层：[assets.ts#L298](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L298) —— `TransformationUtils.resolvePreset()` 将预设参数解析为 sharp transforms 数组。

**系统预设列表**（[constants.ts#L10-L41](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts#L10-L41)）：

| Key | 尺寸 | Fit | 格式 |
|-----|------|-----|------|
| `system-small-cover` | 64×64 | cover | auto |
| `system-small-contain` | 64宽 | contain | auto |
| `system-medium-cover` | 300×300 | cover | auto |
| `system-medium-contain` | 300宽 | contain | auto |
| `system-large-cover` | 800×800 | cover | auto |
| `system-large-contain` | 800宽 | contain | auto |

**特征**：
- 预设中 `format: 'auto'` 触发格式协商
- 首次请求触发 sharp 转换 + 生成资产写入，后续请求命中缓存
- 不支持 Range 请求

### 2.3 动态 Transforms

**触发条件**：请求带 `?transforms=[["resize",{"width":200}]]` 或 `?width=300&height=200&fit=cover` 等动态参数。

**代码路径**：

- Controller 层：[assets.ts#L179-L217](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L179-L217) —— 解析 JSON transforms 数组，校验操作数量（≤ `ASSETS_TRANSFORM_MAX_OPERATIONS`，默认 5），逐项校验方法名是否在 `TransformationMethods` 白名单中。
- Service 层：[assets.ts#L359-L369](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L359-L369) —— 遍历 transforms 数组，依次调用 sharp 实例的对应方法。

**允许的 TransformationMethods**（[assets.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/assets.ts#L6-L57)）：

| 类别 | 方法 |
|------|------|
| 输出格式 | `toFormat`, `jpeg`, `png`, `tiff`, `webp`, `avif` |
| 缩放 | `resize`, `extend`, `extract`, `trim` |
| 图像操作 | `rotate`, `flip`, `flop`, `sharpen`, `median`, `blur`, `flatten`, `gamma`, `negate`, `normalise`/`normalize`, `clahe`, `convolve`, `threshold`, `linear`, `recomb`, `modulate` |
| 颜色 | `tint`, `greyscale`/`grayscale`, `toColorspace`/`toColourspace` |
| 通道 | `removeAlpha`, `ensureAlpha`, `extractChannel`, `bandbool` |

**快捷参数映射**（[transformations.ts#L5-L94](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/transformations.ts#L5-L94)）：

`?width=300&height=200` 等快捷参数由 `resolvePreset()` 内部转换为 `resize` transform；`?format=auto&quality=80` 转换为 `toFormat` transform。focal point 参数触发 `resize` + `extract` 两步操作。

**特征**：
- 受 `transform_mode` 策略约束（见第 4 节）
- 同样走 sharp 管线 + 生成资产缓存
- 不支持 Range 请求

### 2.4 格式协商

**触发条件**：transformation 参数中 `format=auto`。

**代码路径**：

- Controller 层：[assets.ts#L300-L310](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L300-L310) —— 检查 `Accept` 请求头，优先 avif → webp → 不协商。
- Controller 层：[assets.ts#L309](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L309) —— 将 `Accept` 加入 `Vary` 头，确保 CDN 按客户端能力缓存不同版本。
- Service 层：[transformations.ts#L96-L118](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/transformations.ts#L96-L118) —— `getFormat()` 的决策逻辑：

```
format=auto + Accept含image/avif → avif
format=auto + Accept含image/webp → webp
format=auto + 原图是avif/webp/tiff → png（降级为无损格式）
format=auto + 其他 → 保持原格式
format=jpg/png/webp/... → 直接使用指定格式
```

**特征**：
- 同一文件可能因客户端能力不同而返回不同格式
- `Vary: Accept` 确保中间缓存正确区分
- 格式变化影响生成资产的文件扩展名（见第 6 节）

### 2.5 ETag / If-Modified-Since 重验证

**触发条件**：环境变量 `ASSETS_CACHE_REVALIDATE=true`（默认 `false`），且请求携带 `If-None-Match` 或 `If-Modified-Since` 头。

**代码路径**：[assets.ts#L335-L389](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L335-L389)

**流程**：

1. **先验权限**：即使做重验证也必须先调用 `validateAccess()` 确认用户有读取权限（[L343-L356](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L343-L356)）。
2. **读取 modified_on**：使用 `FilesService.readOne(id, { fields: ['modified_on'] })` 获取文件修改时间。
3. **ETag 计算**：`"${Math.floor(modifiedOnTime / 1000)}"` —— 基于秒级时间戳的弱 ETag。
4. **If-None-Match 匹配**：精确字符串比较，匹配则 304。
5. **If-Modified-Since 比较**：秒级精度比较 `modified_on ≤ ifModifiedSince`，满足则 304。
6. **304 响应头**：`Cache-Control: max-age=0, must-revalidate` + `ETag` + `Last-Modified`。

**非重验证模式**（默认 `ASSETS_CACHE_REVALIDATE=false`）：

- 使用 `getCacheControlHeader()` 生成带 `max-age` 的 Cache-Control（TTL 来自 `ASSETS_CACHE_TTL`，默认 30 天）。
- 根据用户角色决定 `public`/`private`。
- 仍然设置 ETag 和 Last-Modified，但不做服务端重验证逻辑。

**关键差异**：

| 特性 | ASSETS_CACHE_REVALIDATE=true | ASSETS_CACHE_REVALIDATE=false（默认） |
|------|-----|------|
| Cache-Control | `max-age=0, must-revalidate` | `max-age=<TTL秒>, public/private` |
| 每次请求是否查库 | 是（需查 modified_on） | 否（仅在 getAsset 内查文件元数据） |
| 304 短路 | 是 | 否 |
| 适合场景 | 内容频繁变更 | 内容相对静态 + CDN 缓存 |

### 2.6 Range 请求

**触发条件**：请求携带 `Range` 头 **且** 无任何 transform 参数。

**代码路径**：

- Controller 层：[assets.ts#L312-L333](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L312-L333) —— `req.headers.range && Object.keys(transformationParams).length === 0` 才解析 Range。
- Service 层：[assets.ts#L259-L295](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L259-L295) —— Range 边界校验与修正。
- Controller 层响应：[assets.ts#L414-L420](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L414-L420) —— 设置 `Content-Range`、`Content-Length`、206 状态码。

**Range 边界校验逻辑**（[assets.ts#L259-L295](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L259-L295)）：

| 异常条件 | 行为 |
|----------|------|
| start 和 end 均为 undefined | 抛出 `RangeNotSatisfiableError` |
| end ≤ start | 抛出 `RangeNotSatisfiableError` |
| start ≥ file.filesize | 抛出 `RangeNotSatisfiableError` |
| end ≤ 0 | 抛出 `RangeNotSatisfiableError` |
| `bytes=-500`（仅 end） | 从尾部取 500 字节：`start = filesize - 500, end = lastByte` |
| `bytes=100-`（仅 start） | 从 100 到文件末尾：`end = lastByte` |
| start < 0 | 修正为 0 |
| end ≥ filesize | 修正为 `lastByte` |

**为什么 Range 仅在无转换时可用**（详见第 7 节）。

### 2.7 ZIP 下载

**触发条件**：`POST /assets/folder/:pk` 或 `POST /assets/files/`。

**代码路径**：[assets.ts#L33-L155](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L33-L155)

**两种 ZIP 端点**：

| 端点 | 方法 | 输入 | Service |
|------|------|------|---------|
| `/assets/folder/:pk` | POST | 文件夹 ID | `AssetsService.zipFolder()` |
| `/assets/files/` | POST | `{ ids: [...] }` | `AssetsService.zipFiles()` |

**ZIP 处理流程**：

1. 使用 `FilesService`（带 accountability）查询文件列表（权限控制）。
2. 创建 `archiver('zip')` 实例。
3. 使用 [NameDeduper](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets/name-deduper.ts) 处理文件名去重（同名文件自动追加 `(1)`, `(2)` 等后缀）。
4. 使用 `sudoFilesService`（无 accountability）逐个读取文件元数据和存储流。
5. 通过 `archive.append(stream, { name, prefix })` 写入 ZIP，支持按文件夹结构组织。
6. 空文件夹也会被加入 ZIP。
7. 客户端断开时自动 `archive.destroy()` 清理。

**关键特征**：
- ZIP 内的文件**不做任何 transform**，始终是原始文件。
- 使用 `sudoFilesService`（`accountability: null`）读取文件，但文件列表查询使用带权限的 `FilesService`。
- 流式传输，`archive.pipe(res)`。
- 文件夹 ZIP 的 Content-Disposition 文件名包含文件夹名 + 时间戳。

---

## 3. Public System Key 如何绕过普通 Item Access

**核心代码**：[assets.ts#L214-L251](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L214-L251)

```typescript
const publicSettings = await this.knex
    .select('project_logo', 'public_background', 'public_foreground', 'public_favicon')
    .from('directus_settings')
    .first();

const systemPublicKeys: string[] = Object.values(publicSettings || {});

if (!systemPublicKeys.includes(id) && this.accountability && this.accountability.admin !== true) {
    // 需要执行 validateItemAccess 权限校验
}

// 跳过权限校验 → allowedFields 保持 ['*']
```

**绕过逻辑**：

1. 从 `directus_settings` 表读取 4 个公共资产字段：`project_logo`、`public_background`、`public_foreground`、`public_favicon`，这些字段存储的是 `directus_files` 的主键 ID。
2. 如果请求的文件 ID 在这 4 个值中（即它是项目公开页面所使用的图片），则**完全跳过权限校验**。
3. 跳过后 `allowedFields` 保持为 `['*']`，返回所有字段。

**设计意图**：Directus 的公开登录页面、注册页面等需要在用户未认证的情况下展示 logo、背景图、前景图和 favicon。如果这些资源也走正常权限，未认证用户将无法看到这些图片，公开页面将无法正常渲染。

**三种免权路径**：

| 条件 | 权限行为 |
|------|---------|
| 文件 ID ∈ systemPublicKeys | 完全跳过 validateItemAccess，allowedFields = `['*']` |
| `this.accountability` 为 null | 跳过 validateItemAccess（内部/系统调用） |
| `this.accountability.admin === true` | 跳过 validateItemAccess（管理员全权访问） |

**注意**：即使跳过了权限校验，文件记录仍通过 `sudoFilesService.readOne()` 读取，且存储层仍校验文件物理存在性（`storage.exists()`）。

**`sanitizeFields` 的作用**：当权限校验执行后，返回给客户端的 file 对象会经过 `sanitizeFields(file, allowedFields)` 过滤，只保留权限允许的字段。`type` 和 `filesize` 作为 `bypassFields` 始终保留，因为 HTTP 响应头需要它们。

---

## 4. Transform Mode 决策逻辑

**代码位置**：[assets.ts#L232-L261](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L232-L261)

`storage_asset_transform` 存储在 `directus_settings` 表中，默认值为 `'all'`。

### 4.1 `all` — 允许一切转换

```
如果 transformation 参数为空 → next()（原始文件，不拦截）
如果 transformation 含 key → 校验 key 是否在 systemKeys + userPresets 中，不在则拒绝
如果 transformation 含动态参数（无 key）→ 直接放行
```

**特点**：预设 key 必须是已配置的（防止任意 key），但动态参数组合完全自由。

### 4.2 `presets` — 仅允许预设

```
如果 transformation 参数为空 → next()
如果 transformation 只有 key 且 key ∈ allKeys → next()
其他任何情况（含动态参数或 key + 其他参数）→ 抛出 InvalidQueryError
```

**特点**：完全禁止动态转换，只能使用系统预设和用户自定义预设。`Object.keys(transformation).length === 1` 确保不能附加其他参数。

### 4.3 `disabled` — 完全禁止转换

```
如果 transformation 参数为空 → next()
如果 transformation 只有 key 且 key ∈ systemKeys 且 Object.keys(transformation).length === 1 → next()
其他任何情况 → 抛出 InvalidQueryError
```

**特点**：只保留 6 个系统内置预设（system-small-cover 等），用户自定义预设也被禁止。这是最严格的模式。

### 4.4 决策矩阵

| 模式 | 原始文件 | 系统预设 key | 用户预设 key | 动态 transforms/width/height | key + 额外参数 |
|------|---------|-------------|-------------|------------------------------|---------------|
| `all` | ✅ | ✅（key 须已配置） | ✅（key 须已配置） | ✅ | ✅ |
| `presets` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `disabled` | ✅ | ✅ | ❌ | ❌ | ❌ |

> **注意**：所有模式都允许原始文件下载（无 transform 参数），因为 `Object.keys(transformation).length === 0` 时直接 `next()`，不进入任何模式判断。

---

## 5. Sharp 并发与 Timeout 保护

**代码位置**：[assets.ts#L330-L393](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L330-L393)

### 5.1 图像尺寸限制

```typescript
if (!width || !height
    || width > ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION
    || height > ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION) {
    throw new IllegalAssetTransformationError(...);
}
```

- 默认值：`ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION = 6000`（像素）
- 同时在 sharp 实例中设置 `limitInputPixels: Math.trunc(6000^2) = 35999999`（[get-sharp-instance.ts#L8](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files/lib/get-sharp-instance.ts#L8)）
- 双重保护：先检查数据库记录中的宽高，再由 sharp 内部限制输入像素总数

### 5.2 并发控制

```typescript
const { queue, process } = sharp.counters();

if (queue + process > ASSETS_TRANSFORM_MAX_CONCURRENT) {
    throw new ServiceUnavailableError({ service: 'files', reason: 'Server too busy' });
}
```

- 默认值：`ASSETS_TRANSFORM_MAX_CONCURRENT = 25`
- `sharp.counters()` 返回当前排队（queue）和正在处理（process）的任务数
- 超限时返回 **503 Service Unavailable**，而非排队等待
- 这是"快速失败"策略，避免内存耗尽导致整个 API 崩溃

### 5.3 超时保护

```typescript
transformer.timeout({
    seconds: clamp(Math.round(getMilliseconds(ASSETS_TRANSFORM_TIMEOUT, 0) / 1000), 1, 3600),
});
```

- 默认值：`ASSETS_TRANSFORM_TIMEOUT = '7500ms'` → 7.5 秒 → clamp 到 8 秒
- 范围：1 秒 ≤ timeout ≤ 3600 秒
- 超时触发后 sharp 抛出 `timeout` 错误
- Service 层捕获并清理：先尝试删除未完成的生成资产文件，再抛出 `ServiceUnavailableError`

```typescript
try {
    await storage.location(file.storage).write(assetFilename, readStream.pipe(transformer), type);
} catch (error) {
    try { await storage.location(file.storage).delete(assetFilename); } catch { /* ignored */ }

    if (error?.message?.includes('timeout')) {
        throw new ServiceUnavailableError({ service: 'assets', reason: 'Transformation timed out' });
    } else {
        throw error;
    }
}
```

### 5.4 其他 Sharp 配置

| 配置 | 值 | 说明 |
|------|----|------|
| `sequentialRead` | `true` | 顺序读取优化，适合流式处理 |
| `failOn` | `'warning'`（默认） | 无效图片灵敏度级别，warning 级别会在严重错误时失败 |
| 自动旋转 | 默认添加 `.rotate()` | 如果没有显式 `rotate` transform，自动根据 EXIF 旋转 |

### 5.5 转换操作数量限制

Controller 层校验 `transforms` 数组长度：

```typescript
if (transforms.length > ASSETS_TRANSFORM_MAX_OPERATIONS) {
    throw new InvalidQueryError({ reason: `...only allowed ${ASSETS_TRANSFORM_MAX_OPERATIONS} transformations` });
}
```

- 默认值：`ASSETS_TRANSFORM_MAX_OPERATIONS = 5`

---

## 6. 生成资产命名与缓存命中逻辑

### 6.1 命名规则

**代码位置**：[assets.ts#L306-L309](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L306-L309)

```typescript
const assetFilename =
    path.basename(file.filename_disk, path.extname(file.filename_disk))  // 去掉扩展名
    + getAssetSuffix(transforms)                                           // __<hash>
    + (maybeNewFormat ? `.${maybeNewFormat}` : path.extname(file.filename_disk));  // 新扩展名 or 原扩展名
```

**`getAssetSuffix`**（[assets.ts#L413-L416](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/assets.ts#L413-L416)）：

```typescript
const getAssetSuffix = (transforms: Transformation[]) => {
    if (Object.keys(transforms).length === 0) return '';
    return `__${hash(transforms)}`;
};
```

使用 `object-hash` 对 transforms 数组做确定性哈希，确保相同参数 → 相同后缀。

**命名示例**：

| 原始文件 | Transform | 生成文件名 |
|----------|-----------|-----------|
| `abc123.jpg` | `resize(300,300,cover) + toFormat(webp,80)` | `abc123__a1b2c3d4.webp` |
| `abc123.jpg` | `resize(64,64,cover)` | `abc123__e5f6g7h8.jpg` |
| `abc123.png` | `resize(300,300,contain) + toFormat(avif)` | `abc123__i9j0k1l2.avif` |

### 6.2 格式扩展名替换

当存在 `toFormat` transform 时，`maybeExtractFormat()` 提取最终格式作为扩展名：

```typescript
const maybeNewFormat = TransformationUtils.maybeExtractFormat(transforms);
```

如果格式发生变更，`file.type` 也会被更新：

```typescript
if (maybeNewFormat) {
    file.type = contentType(assetFilename) || null;
}
```

这确保了 HTTP `Content-Type` 头与生成资产的实际格式一致。

### 6.3 缓存命中流程

```
getAsset(id, transformation)
  │
  ├─ 解析 transforms = resolvePreset(transformation, file)
  │
  ├─ 检查: type ∈ SUPPORTED_IMAGE_TRANSFORM_FORMATS && transforms.length > 0 ?
  │    │
  │    ├─ 是 → 构造 assetFilename
  │    │    │
  │    │    ├─ storage.exists(assetFilename) ?
  │    │    │    ├─ 是 → 缓存命中！直接读取生成资产，返回 stream + stat
  │    │    │    └─ 否 → 执行 sharp 转换 → 写入生成资产 → 返回 stream + stat
  │    │
  │    └─ 否 → 直接读原始文件
```

### 6.4 缓存命中条件

生成资产能命中缓存需同时满足：

1. **文件类型受支持**：`file.type ∈ ['image/jpeg', 'image/png', 'image/webp', 'image/tiff', 'image/avif']`
2. **存在 transform**：`transforms.length > 0`（原始文件不走缓存机制）
3. **存储中存在生成文件**：`storage.exists(assetFilename) === true`
4. **哈希一致**：相同 transforms 参数 → 相同 hash → 相同文件名 → 命中

### 6.5 缓存失效

生成资产**没有显式的失效机制**。失效依赖于：

- 原始文件被更新时，`modified_on` 变更，版本号 `version` 变更，某些存储驱动会据此刷新
- 文件被删除时，关联的生成资产不会被自动删除（存储层积留）
- 手动清理存储是唯一的"主动失效"方式

---

## 7. 为什么 Range 仅在无转换时可用

**代码位置**：[assets.ts#L314](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/assets.ts#L314)

```typescript
if (req.headers.range && Object.keys(transformationParams).length === 0) {
    // 解析 Range...
}
```

### 7.1 技术原因

1. **转换后文件大小不可预知**：Range 请求的核心是客户端必须知道文件的完整大小才能正确请求字节范围。在 sharp 转换之前，无法确定输出文件的确切大小（resize、format 转换、quality 压缩都会改变大小）。`Content-Range: bytes start-end/total` 中的 `total` 在转换前不可知。

2. **缓存生成资产的 Range 语义问题**：即使生成资产已缓存，其文件大小与原始文件不同。如果允许 Range，客户端可能用原始文件的大小来请求转换后的资产，导致 Range Not Satisfiable。

3. **流式管道不兼容**：sharp 转换是流式管道 `readStream.pipe(transformer)`，整个管道必须完整执行才能产生有效输出。不能对管道中间的某个字节范围进行截取。

4. **ETag/Last-Modified 的双重含义**：如果同时有 Range 和 transform，304 重验证会变得复杂——原始文件的 modified_on 可能没变，但转换参数变了，ETag 应该也不同。

### 7.2 设计权衡

| 方案 | 优点 | 缺点 |
|------|------|------|
| 仅原始文件支持 Range | 简单可靠，语义清晰 | 大图转换后无法断点续传 |
| 转换后也支持 Range | 支持大图渐进加载 | 需要先完成转换再响应，首字节延迟大；缓存前无法确定大小 |
| 生成资产缓存后支持 Range | 兼顾缓存和 Range | 缓存命中/未命中行为不一致，实现复杂 |

当前设计选择了最简单可靠的方案：**原始文件 = 确定性字节流 = 支持 Range；转换后 = 动态生成 = 不支持 Range**。

### 7.3 补偿机制

对于需要渐进加载的场景，Directus 提供了预设转换作为替代方案——客户端请求小尺寸缩略图而非原始大图的 Range，既节省带宽又避免复杂的 Range 语义。

---

## 8. 完整请求处理时序图

```
Client                     Controller                    Service                       Storage
  │                           │                            │                             │
  │── GET /assets/:pk ──────→ │                            │                             │
  │                           │── 查询 directus_settings ─→│                             │
  │                           │←─ presets + transform_mode─│                             │
  │                           │                            │                             │
  │                           │── 校验 transform_mode ────→│                             │
  │                           │   (all/presets/disabled)   │                             │
  │                           │                            │                             │
  │                           │── 格式协商 (Accept) ──────→│                             │
  │                           │                            │                             │
  │                           │── Range 解析 ─────────────→│                             │
  │                           │   (仅无 transform 时)      │                             │
  │                           │                            │                             │
  │                           │── ETag 重验证 ────────────→│                             │
  │                           │   (ASSETS_CACHE_REVALIDATE)│                             │
  │                           │                            │                             │
  │                           │── getAsset() ─────────────→│                             │
  │                           │                            │── 查 systemPublicKeys ──────→│
  │                           │                            │── validateItemAccess ───────→│
  │                           │                            │── readOne (sudo) ───────────→│
  │                           │                            │── exists(filename_disk) ────→│
  │                           │                            │                             │
  │                           │                            │  [有 transform?]             │
  │                           │                            │  ├─ 是: 构造 assetFilename   │
  │                           │                            │  │  exists(assetFilename)──→ │
  │                           │                            │  │  ├─ 命中: 直接读 ───────→ │
  │                           │                            │  │  └─ 未命中:               │
  │                           │                            │  │    sharp.counters()       │
  │                           │                            │  │    timeout()              │
  │                           │                            │  │    read+pipe+write ────→  │
  │                           │                            │  │    读生成资产 ──────────→ │
  │                           │                            │  └─ 否: 直接读原始文件 ────→│
  │                           │                            │                             │
  │                           │←─ { stream, file, stat } ─│                             │
  │                           │                            │                             │
  │                           │── 设置响应头 ─────────────│                             │
  │                           │   Content-Type             │                             │
  │                           │   Content-Length            │                             │
  │                           │   ETag / Last-Modified     │                             │
  │                           │   Cache-Control            │                             │
  │                           │   Vary                     │                             │
  │                           │   Content-Range (206)      │                             │
  │                           │                            │                             │
  │                           │── stream.pipe(res) ───────→│                             │
  │←────────── 数据流 ────────│                            │                             │
```

---

## 9. 环境变量速查表

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `ASSETS_CACHE_TTL` | `30d` | 非重验证模式下的缓存 TTL |
| `ASSETS_CACHE_REVALIDATE` | `false` | 是否启用服务端 ETag/If-Modified-Since 重验证 |
| `ASSETS_TRANSFORM_MAX_CONCURRENT` | `25` | sharp 最大并发转换数（queue + process） |
| `ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION` | `6000` | 可转换图片的最大宽/高像素 |
| `ASSETS_TRANSFORM_MAX_OPERATIONS` | `5` | `transforms[]` 数组最大操作数 |
| `ASSETS_TRANSFORM_TIMEOUT` | `7500ms` | 单次 sharp 转换超时时间 |
| `ASSETS_INVALID_IMAGE_SENSITIVITY_LEVEL` | `warning` | sharp failOn 选项 |
| `ASSETS_CONTENT_SECURITY_POLICY` | - | 自定义 CSP 指令 |
