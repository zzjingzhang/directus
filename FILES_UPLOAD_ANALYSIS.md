# Directus `/files` Multipart 上传与文件替换流程分析

## 1. 总体架构

文件上传涉及两个核心模块：

| 模块 | 文件 | 职责 |
|------|------|------|
| Controller | [files.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts) | HTTP 层：Busboy 解析 multipart、字段/文件事件处理、路由分发 |
| Service | [files.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts) | 业务层：`uploadOne` 区分新建/替换、Storage 写入、DB 事务一致性、metadata 补全 |
| Storage | [index.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/storage/src/index.ts) | 存储抽象层：`write`/`move`/`delete`/`list`/`stat` 原语 |
| Metadata | [extract-metadata.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files/lib/extract-metadata.ts) + [get-metadata.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files/utils/get-metadata.ts) | 从图像文件中提取 EXIF/IPTC/XMP/ICC 等元数据 |

---

## 2. Busboy 字段顺序为何影响 payload/storage

### 2.1 关键代码位置

Controller 层的 `multipartHandler` 函数在 [files.ts#L28-L135](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L28-L135)。

### 2.2 顺序要求

文件开头有一段**非常重要的注释** [files.ts#L55-L59](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L55-L59)：

> The order of the fields in multipart/form-data is important. We require that all fields are provided _before_ the files. This allows us to set the storage location, and create the row in directus_files async during the upload of the actual file.

翻译：multipart/form-data 中**字段必须全部出现在文件之前**。这样做的目的是：

1. 在文件流到达之前，`payload` 对象已经包含了用户传入的所有元数据字段（如 `storage`、`folder`、`title`、`description`、`filename_download` 等）。
2. 当 `busboy.on('file')` 事件触发时，可以**立刻**把 `payload` 传给 `FilesService.uploadOne`，让它在上传文件流的同时**异步创建 DB 记录**（见下文 §4），从而实现流式上传 + 预占 PK。

### 2.3 字段/文件事件的处理

```
Busboy 事件流（正确顺序）：
  field 'storage'   → payload.storage = 'local'
  field 'folder'    → payload.folder = 'abc-123'
  field 'title'     → payload.title = 'My Photo'
  ... （所有字段）
  file  'file'      → 读取 payload，调用 uploadOne(stream, payload, pk)
                      → 处理完后清空 payload = {}
  [可选：file 'file2' → 复用已被清空的 payload（空）+ 自动生成 title]
```

关键代码：
- 字段收集：[files.ts#L64-L72](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L64-L72)
- 到达文件事件时使用 payload：[files.ts#L74-L102](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L74-L102)
- 每处理完一个文件即 `payload = {}` 清空，为下一个文件（多文件上传场景）做准备。

### 2.4 顺序颠倒的后果

如果客户端把 `file` 放在 `storage` 等字段**之前**，会导致：

| 场景 | 后果 |
|------|------|
| 指定 `storage: 's3'` 字段放在文件后 | `payload.storage` 为空，回落到环境变量 `STORAGE_LOCATIONS[0]`（默认 `local`），文件被存到错误位置 |
| 指定 `folder` 字段放在文件后 | `payload.folder` 为空，回落到 `directus_settings.storage_default_folder`（若存在），文件归属错误文件夹 |
| 多文件上传时第二个文件想带自定义字段 | 第一个文件事件清空了 payload，第二个文件之前的字段**必须重新再发一遍**，否则使用自动推断值 |

### 2.5 文件大小限制

Busboy 实例化时通过 `FILES_MAX_UPLOAD_SIZE` 环境变量设置 `limits.fileSize`：[files.ts#L42-L48](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L42-L48)。超出大小后 Busboy 会截断流，Service 层通过 `stream.truncated` 标志检测并抛出 `ContentTooLargeError`（见 §5.2）。

---

## 3. 文件名与 MIME Allowlist 校验

### 3.1 文件名校验

在 Controller 的 `file` 事件中第一步就检查：

```ts
if (!filename) {
  return busboy.emit('error', new InvalidPayloadError({ reason: `File is missing filename` }));
}
```

位置：[files.ts#L75-L77](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L75-L77)

注意：此 `filename` 是 multipart part 的 `Content-Disposition` 中的 `filename=` 参数，对应前端表单中选择文件时的**原始文件名**。

### 3.2 MIME Allowlist 校验

#### 3.2.1 校验位置与流程

Controller 层（multipart 上传路径）：[files.ts#L79-L84](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L79-L84)

```ts
const allowedPatterns = toArray(env['FILES_MIME_TYPE_ALLOW_LIST'] as string | string[]);
const mimeTypeAllowed = allowedPatterns.some((pattern) => minimatch(mimeType, pattern));

if (mimeTypeAllowed === false) {
  return busboy.emit('error', new InvalidPayloadError({ reason: `File is of invalid content type` }));
}
```

Service 层（importOne 远程导入路径）：[files.ts#L304-L325](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L304-L325) 做了**双重校验**：
1. 全局 `FILES_MIME_TYPE_ALLOW_LIST` 校验
2. 可选的接口级 `filterMimeType` 限制（字段配置中的文件类型过滤）

#### 3.2.2 匹配算法

使用 [`minimatch`](https://www.npmjs.com/package/minimatch) 做 glob 风格匹配：

| 配置示例 | 匹配结果 |
|----------|----------|
| `*/*`（默认值，见 [defaults.ts#L219](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/env/src/constants/defaults.ts#L219)） | 允许所有 MIME |
| `image/*` | `image/jpeg`、`image/png` 匹配；`application/pdf` 不匹配 |
| `image/png,application/pdf` | 两个具体类型匹配 |
| `video/*,audio/*` | 音视频类型匹配 |

#### 3.2.3 `importOne` 的 Content-Type 处理

从 URL 导入文件时，会先 `split(';')[0].trim()` 剥离 `charset` 等参数：

```ts
const mimeType = fileResponse.headers['content-type']?.split(';')[0]?.trim() || 'application/octet-stream';
```

位置：[files.ts#L302](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L302)。这样可以避免 `image/jpeg; charset=binary` 等带参数的类型被误判。

### 3.3 `filename_disk` 唯一性校验

这是**落盘文件名**（默认 = `primaryKey + extension`）的校验，在 `createOne` / `updateMany` 中进行：

- `createOne` 路径：[files.ts#L345-L354](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L345-L354)
- `updateMany` 路径：[files.ts#L368-L377](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L368-L377)

```ts
private async checkUniqueFilename(filename: string, excludeId?: PrimaryKey) {
  const query = this.knex.select('filename_disk').from('directus_files').where({ filename_disk: filename });
  if (excludeId) query.whereNot('id', excludeId);
  const existingFile = await query.first();
  if (existingFile) throw new ForbiddenError();
}
```

注意：此处用的是 `ForbiddenError` 而不是 `ConflictError`，与权限错误同类，是因为要把错误**延迟到权限校验之后**再抛出（通过 `opts.preMutationError` 机制），避免暴露存在性信息。

---

## 4. `FilesService.uploadOne` 怎样区分新建和替换

### 4.1 核心判断逻辑

`uploadOne` 签名：

```ts
async uploadOne(
  stream: BusboyFileStream | Readable,
  data: Partial<File>,
  primaryKey?: PrimaryKey,   // ← 关键参数
  opts?: MutationOptions,
): Promise<PrimaryKey>
```

位置：[files.ts#L81-L252](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L81-L252)

区分逻辑分两步：

**Step 1：是否传了 `primaryKey`？**
- `POST /files`（新建）：Controller 调用时不传第三个参数，`primaryKey = undefined`
- `PATCH /files/:pk`（替换）：Controller 从 `req.params['pk']` 取到，传入第三个参数

对应 Controller 代码：
- 新建：[files.ts#L38-L149](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L138-L149)（`multipartHandler` 中 `existingPrimaryKey = req.params['pk']`，POST 路由无 `:pk`，所以为 `undefined`）
- 替换：[files.ts#L301-L326](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L301-L326)（`PATCH /:pk` 路由，`:pk` 被 `multipartHandler` 读取）

**Step 2：DB 中是否存在对应记录？**

```ts
if (primaryKey !== undefined) {
  existingFile = (await this.knex
    .select('folder', 'filename_download', 'filename_disk', 'title', 'description', 'metadata', 'storage')
    .from('directus_files')
    .where({ id: primaryKey })
    .first()) ?? null;
}
```

位置：[files.ts#L92-L100](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L92-L100)

**最终判定：**

```ts
const isReplacement = existingFile !== null && primaryKey !== undefined;
```

位置：[files.ts#L121](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L121)

### 4.2 新建 vs 替换：执行路径对比

| 步骤 | 新建 (`isReplacement = false`) | 替换 (`isReplacement = true`) |
|------|-------------------------------|--------------------------------|
| ① payload 合并 | `{ storage: 默认, ...data }` | `{ storage: 默认, ...existingFile, ...data }` （旧值为底，新数据覆盖） |
| ② DB 记录 | 调用 `this.createOne(payload)` 立即插入，生成新 PK | **不**新建，使用已有 PK |
| ③ 计算 `filename_disk` | `primaryKey + extension` | 同左，但如果扩展名变化会**强制更新**（§4.3） |
| ④ 写入 Storage | 直接写 `payload.filename_disk` | 先写 `temp_ + filename_disk` |
| ⑤ 后处理 | 无 | `updateOne(payload)` → 删除旧文件/缩略图 → `disk.move(temp, final)` |
| ⑥ 失败清理 | `deleteMany([pk])` + `disk.delete(final)` | `disk.delete(temp)` |

### 4.3 payload 合并策略（替换场景的细节）

替换时的 merge 顺序非常关键：

```ts
const payload = {
  storage: toArray(env['STORAGE_LOCATIONS'] as string)[0]!,   // 第1优先级：环境默认
  ...(existingFile ?? {}),                                    // 第2优先级：DB 中旧值
  ...clone(data),                                             // 第3优先级：此次上传传入的 data
};
```

位置：[files.ts#L103-L107](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L103-L107)

这意味着：
- 若用户本次上传**未指定** `folder`，则沿用上一次的 `folder`
- 若用户本次上传**未指定** `filename_download`，则沿用上一次的
- 若用户本次上传**未指定** `storage`，则沿用上一次的（而**不是**默认的第一个 storage location，因为 `existingFile` 覆盖了默认值）
- 若扩展名变化（例：旧文件是 `.jpg`，新上传是 `.png`），会强制重置 `filename_disk`：[files.ts#L137-L139](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L137-L139)

### 4.4 `importOne` 的特殊情况

远程导入时，用户可以在 `body` 中传 `id` 来**用远程文件替换一个已有的记录**：

```ts
return await this.uploadOne(decompressResponse(...), payload, payload.id);
```

位置：[files.ts#L334](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L334)

---

## 5. 临时文件的移动与清理、metadata 补全

### 5.1 临时文件策略（仅替换场景）

替换流程中不直接覆盖目标文件，而是先写一个临时文件：

```ts
const tempFilenameDisk = 'temp_' + filenameDisk;

if (isReplacement === true) {
  await disk.write(tempFilenameDisk, stream, payload.type);
} else {
  await disk.write(payload.filename_disk, stream, payload.type);
}
```

位置：[files.ts#L142](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L142) + [files.ts#L173-L180](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L173-L180)

**临时文件升级**（写入成功且 DB 更新成功后）：[files.ts#L201-L217](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L201-L217)

```ts
if (isReplacement === true) {
  await this.updateOne(primaryKey, payload, { emitEvents: false });

  // 删除旧文件 + 旧缩略图
  for await (const filepath of disk.list(String(primaryKey))) {
    await disk.delete(filepath);
  }

  // 原子性 move（如果 storage 支持）
  await disk.move(tempFilenameDisk, payload.filename_disk);
}
```

### 5.2 流截断检测

`disk.write` 完成后立刻检查：

```ts
if ('truncated' in stream && stream.truncated === true) {
  throw new ContentTooLargeError();
}
```

位置：[files.ts#L183-L185](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L183-L185)

Busboy 的 `limits.fileSize` 只截断**不抛错**，必须通过读取流对象上的 `truncated` 标志确认。

### 5.3 异常时的清理函数

`cleanUp` 定义于 [files.ts#L149-L171](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L149-L171)，在**两个 catch 分支**中被调用：
1. storage 写入阶段失败：[files.ts#L186-L199](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L186-L199)
2. 替换升级阶段失败：[files.ts#L213-L216](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L213-L216)

清理行为：

| 场景 | 清理动作 |
|------|----------|
| 新建失败 | `super.deleteMany([primaryKey])` 删除 DB 记录 + `disk.delete(filename_disk)` 删除已写部分 |
| 替换失败 | `disk.delete(tempFilenameDisk)` 删除临时文件（**保留**旧文件和旧 DB 记录） |

清理失败仅记录 `logger.warn`，**不重新抛出**——因为主要错误已经被抛出，清理是「尽力而为」的 best-effort 操作。

### 5.4 metadata 补全流程

文件写入完成后，依次补全：

**Step 1：补 `filesize`**

```ts
const { size } = await storage.location(payload.storage).stat(payload.filename_disk);
payload.filesize = size;
```

位置：[files.ts#L219-L220](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L219-L220)

**Step 2：提取图片元数据（仅支持的格式）**

`extractMetadata` 见 [extract-metadata.ts#L6-L46](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files/lib/extract-metadata.ts#L6-L46)：

```ts
if (data.type && SUPPORTED_IMAGE_METADATA_FORMATS.includes(data.type)) {
  const stream = await storage.location(storageLocation).read(data.filename_disk);
  const { height, width, description, title, tags, metadata } = await getMetadata(stream);
  ...
}
```

支持格式定义于 [constants.ts#L88-L95](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/constants.ts#L88-L95)：
- `image/jpeg`, `image/png`, `image/webp`, `image/gif`, `image/tiff`, `image/avif`

`getMetadata`（见 [get-metadata.ts#L17-L142](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files/utils/get-metadata.ts#L17-L142)）使用 sharp 提取：
- 基础尺寸：`width` / `height`（注意 orientation ≥ 5 时宽高互换）
- EXIF：`exif-reader` 解析为 `ifd0` / `ifd1` / `exif` / `gps` / `interop`
- ICC Profile：`icc` 解析
- IPTC：`parseIptc` → 提取 `caption`（写入 description）、`headline`（写入 title）、`keywords`（写入 tags）
- XMP：`parseXmp`

然后按 `FILE_METADATA_ALLOW_LIST` 过滤嵌套字段（默认保留全部）。

**Step 3：补 `uploaded_on`**

```ts
payload.uploaded_on = new Date().toISOString();
```

**Step 4：Sudo 更新 DB**

```ts
const sudoFilesItemsService = new ItemsService('directus_files', {
  knex: this.knex,
  schema: this.schema,
});

await sudoFilesItemsService.updateOne(primaryKey, { ...payload, ...metadata }, { emitEvents: false });
```

位置：[files.ts#L228-L233](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L228-L233)

关键点：
- **不传入 accountability** = 以「系统权限」运行。因为即使当前用户对该文件没有 `update` 权限，也需要把提取到的 filesize/metadata 等系统字段写回去。
- `emitEvents: false`：因为 `createOne` 阶段没发事件，这里也暂不发，统一在最后 `files.upload` action 时发。

---

## 6. 旧文件和缩略图何时删除

缩略图（thumbnails）/ 转换产物（transformations）的文件名约定是 `{prefix}-*`，所以通过 `disk.list(prefix)` 可以全部列出。prefix 就是 filename_disk 的「无扩展名基名」，对于替换场景等于 `String(primaryKey)`。

### 6.1 文件替换时的删除

```ts
for await (const filepath of disk.list(String(primaryKey))) {
  await disk.delete(filepath);
}
```

位置：[files.ts#L206-L209](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L206-L209)

**执行时机**：
- 临时文件写入成功 ✓
- DB `updateOne` 成功 ✓
- **在** `disk.move(temp, final)` **之前**

顺序合理性：先清掉所有基于旧 primaryKey 的产物，再把新主文件移入，避免 list 同时扫到新旧两种主文件。但注意极端情况下 move 失败会导致旧主文件被删、新文件还在 temp 位置——这种情况由 `cleanUp` 负责删除 temp，DB 记录还是指向旧 filename_disk，下次请求会 404，但不会有脏数据（§7 详述）。

### 6.2 删除文件时的删除

```ts
override async deleteMany(keys: PrimaryKey[]): Promise<PrimaryKey[]> {
  ...
  const files = await sudoFilesItemsService.readMany(keys, { fields: ['id', 'storage', 'filename_disk'], limit: -1 });
  await super.deleteMany(keys);   // 先删 DB

  for (const file of files) {
    const disk = storage.location(file['storage']);
    const filePrefix = path.parse(file['filename_disk']).name;
    for await (const filepath of disk.list(filePrefix)) {
      await disk.delete(filepath);
    }
  }
}
```

位置：[files.ts#L469-L492](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L469-L492)

**执行顺序**：先删 DB（事务内），再删 Storage。若 DB 删除失败则事务回滚，Storage 完全不动。若 DB 删除成功但 Storage 删除失败——DB 记录已删，前端看不到了，但 Storage 上有**孤儿文件**（orphan files）。这是有意的权衡：宁可留几个占空间的垃圾文件，也不要 DB 里有指向已删文件的死链接（dangling pointer）。

### 6.3 `updateMany`（重命名 filename_disk）时的删除

在 [files.ts#L363-L452](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L363-L452)，通过 `disk.list(filePrefixPath)` 遍历所有以旧前缀命名的文件：

| 文件类型 | 处理方式 |
|----------|----------|
| 主文件（路径匹配 old filepath） | 若新路径已存在则只**删原**（需 `FILES_DELETE_ORIGINAL_ON_MOVE=true`，默认 false）；否则 `disk.move` |
| 缩略图/转换产物（其他所有匹配） | **总是删除**（下次访问时按需重新生成） |

---

## 7. 一致性维护：DB 记录失败 / Storage 写入失败

### 7.1 新建文件场景的一致性

新建的时序：

```
    createOne(DB)  ──①──►  OK  →  disk.write(Storage) ──②──►  OK  →  updateOne(补全字段)
              │                                  │
              └ FAIL: 直接抛错                   └ FAIL → cleanUp()：
                (rollback 事务)                         deleteMany(DB) + disk.delete()
```

- **① DB create 失败**：直接抛异常，Storage 还没写入，**完全一致**。
- **② Storage write 失败**：DB 已插入，调用 `cleanUp` 把 DB 记录和已写的存储文件都删。若 `disk.delete` 失败（best-effort，只记 warn），则出现「孤儿文件」——DB 已干净，无死链接，风险较低。
- **②之后 updateOne(metadata) 失败**：此阶段没有 try/catch，异常直接向上冒泡。此时 DB 中记录缺 `filesize`、`uploaded_on`、`width/height` 等字段，但主文件已写入 Storage——属于**半完成状态**，后续可手动修复。实际部署中建议在此阶段外再包一层补偿。

### 7.2 替换文件场景的一致性

替换的时序：

```
disk.write(temp)  ─①→ OK → updateOne(DB) ─②→ OK → list+delete 旧文件 ─③→ OK → disk.move(temp→final)
           │                     │                        │                      │
           └ FAIL                └ FAIL                   └ FAIL                 └ FAIL
          cleanUp:           cleanUp:                 cleanUp:              cleanUp:
          disk.delete(temp)  disk.delete(temp)        disk.delete(temp)     disk.delete(temp)
          (保留DB原记录)     (保留DB原记录)            (保留DB原记录)        (保留DB原记录)
```

关键点：
- **替换场景的 cleanUp 永远不删 DB 记录**——因为记录本来就存在，用户有权利保留旧版本文件和记录。`cleanUp` 只删临时文件，DB 和旧主文件保持原状。
- **① 失败**：没动任何生产数据，最安全。
- **② 失败**：temp 已写但 DB 未动，cleanUp 删 temp 即可。
- **③ 失败（list/delete 中途）**：DB 的 `updateOne` 已经完成（metadata 已是新值），但旧主文件和部分缩略图还没删，接着 cleanUp 会删 temp——此时 DB 指向新的 filename_disk，但真正的文件还在 temp 位置。**这是最危险的时刻**：后续请求该文件会 404（因为 filename_disk 不存在），但 temp 文件仍在（可手动 rename 恢复）。实际使用时 storage driver 的 `move` 最好是原子 rename（local filesystem、S3 copy+delete 等均已实现）。
- **④ move 失败**：同③的尾端状态——temp 仍在、旧文件已删、DB 已指向新 filename_disk。cleanUp 只删 temp，然后抛出异常让调用方知道替换失败。

### 7.3 `updateMany`（重命名）场景的事务

在 [files.ts#L395-L448](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/files.ts#L395-L448)，**每个 key 包一个事务**：

```ts
await transaction(this.knex, async (trx) => {
  const filesItemService = new ItemsService(this.collection, { knex: trx, ... });
  await filesItemService.updateMany([key], data, opts);

  // storage 操作在事务外
  for await (const filePath of disk.list(filePrefixPath)) { ... }
});
```

但注意：**Storage 操作不是事务性的**。DB 事务只保护 `updateMany` 本身。若 DB update 成功但 storage move/delete 失败，则事务不会回滚（knex 事务不管理外部系统），系统处于「DB 说文件在 X，但实际文件还在旧位置」的不一致状态——需要运维/定时任务补偿。

### 7.4 `deleteMany` 场景的一致性

```
先 super.deleteMany(DB事务内) → 再遍历 disk.list 删 Storage 文件
```

- DB 失败：事务回滚，Storage 完全未动 ✅
- DB 成功 + Storage 部分/全部失败：DB 记录已删，前端不可见，Storage 留孤儿文件。可以通过定期扫描 `disk.list()` 对比 DB 的方式做 GC。

### 7.5 Controller 层的 Busboy 错误传播

Controller 中任何 `busboy.on('file')` 回调里的异常，都是通过 `busboy.emit('error', error)` 抛到 `busboy.on('error')` → `next(error)`：

```ts
busboy.on('error', (error: Error) => {
  next(error);
});
```

位置：[files.ts#L115-L117](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/files.ts#L115-L117)

这样 Express 的错误处理中间件可以正确捕获并返回 HTTP 错误响应（含正确的错误码如 `INVALID_PAYLOAD`、`CONTENT_TOO_LARGE` 等）。注意：**Busboy 出错后不会自动消费剩余的 request body**，需要依赖上游（Express/Node.js）关闭 socket 或 drain 流——实际行为取决于 Node.js 版本。

### 7.6 一致性机制总览

| 失败点 | DB 状态 | Storage 状态 | 一致性评价 | 补偿措施 |
|--------|---------|--------------|-----------|----------|
| createOne（新建） | 回滚 | 未写入 | ✅ 完全一致 | - |
| disk.write（新建中途） | 已插入 | 部分写入 | ⚠️ 需清理 | cleanUp() 双删 |
| metadata updateOne（新建末尾） | 缺字段记录 | 完整主文件 | ⚠️ 半完成 | 手动补 metadata |
| disk.write(temp)（替换） | 不变（旧记录） | 部分 temp | ⚠️ 需清理 | cleanUp() 删 temp |
| updateOne + list/delete（替换中间） | 已指向新文件 | 旧文件已删，temp 还在 | ❌ 不一致（404 风险） | 手动 move temp → final |
| disk.move(temp→final)（替换末尾） | 已指向新文件 | temp 存在，final 缺失 | ❌ 不一致（404 风险） | 手动 move |
| DB delete（删除） | 回滚 | 未动 | ✅ 完全一致 | - |
| Storage delete（删除） | 已删 | 部分/全未删 | ⚠️ 孤儿文件 | 定期 GC |
| filename_disk 更新 | 已指向新路径 | 旧位置未变 | ❌ 不一致（404 风险） | 手动 move |

总体而言，Directus 的一致性策略是 **「DB 优先」**：宁可 Storage 上残留孤儿文件，也不让 DB 出现死链接（因为 DB 是对外提供查询的真源，404 比返回损坏数据更容易被感知和修复）。替换流程中引入 `temp_` 前缀的临时文件是为了在最终 move 前保留旧文件，降低了替换失败导致的 404 窗口，但并未完全消除（极端情况下 `move` 本身失败）。
