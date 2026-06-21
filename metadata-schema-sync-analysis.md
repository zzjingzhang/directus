# Directus 元数据与数据库 DDL 同步机制分析

## 概述

Directus 通过三层架构维护元数据与真实数据库 DDL 的一致性：

1. **数据库层** - 实际的 SQL 表、列、索引、外键约束
2. **元数据层** - `directus_collections`、`directus_fields`、`directus_relations` 等系统表
3. **服务层** - CollectionsService、FieldsService、RelationsService、SchemaService 等业务服务

本文档深入分析各服务如何协同工作，确保 Directus 元数据与真实数据库 DDL 保持一致。

---

## 1. Snapshot / Diff / Apply 的 Hash 与 Force 校验机制

### 1.1 核心组件

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| SchemaService | [services/schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/schema.ts) | 对外提供 snapshot、diff、apply 三大核心能力 |
| getSnapshot | [utils/get-snapshot.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-snapshot.ts) | 生成当前数据库的完整 schema 快照 |
| getSnapshotDiff | [utils/get-snapshot-diff.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-snapshot-diff.ts) | 计算两个快照之间的差异 |
| applyDiff | [utils/apply-diff.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/apply-diff.ts) | 将差异应用到当前数据库 |
| validateApplyDiff | [utils/validate-diff.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/validate-diff.ts) | 校验 diff 的合法性与 hash 一致性 |
| getVersionedHash | [utils/get-versioned-hash.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-versioned-hash.ts) | 生成带版本号的对象 hash |

### 1.2 Hash 生成机制

Hash 由 `getVersionedHash` 函数生成，使用 `object-hash` 库：

```typescript
// 来源: get-versioned-hash.ts
export function getVersionedHash(item: Record<string, any>): string {
    return hash({ item, version });
}
```

**关键特性**：
- 基于 `object-hash` 库对对象进行哈希计算
- 嵌入 Directus 版本号，确保不同版本间快照不兼容
- 快照生成前会对所有对象进行深度排序（`sortDeep`），保证相同内容的快照 hash 一致

### 1.3 Snapshot 工作流

```
调用 /schema/snapshot
    ↓
SchemaService.snapshot()
    ↓
getSnapshot()
    ├─ CollectionsService.readByQuery()  // 读取所有集合
    ├─ FieldsService.readAll()          // 读取所有字段
    ├─ RelationsService.readAll()       // 读取所有关系
    ├─ 过滤系统项与未跟踪项
    ├─ 深度排序（sortDeep）确保一致性
    └─ sanitize 处理后返回
```

快照包含以下数据结构（定义于 [snapshot.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/snapshot.ts)）：

```typescript
type Snapshot = {
    version: number;        // 快照版本号（当前为 1）
    directus: string;       // Directus 版本号
    vendor?: DatabaseClient;// 数据库厂商
    collections: SnapshotCollection[];
    fields: SnapshotField[];
    systemFields: SnapshotSystemField[];  // 仅含索引变更的系统字段
    relations: SnapshotRelation[];
};
```

### 1.4 Diff 工作流与 Force 参数

```
调用 /schema/diff?force=...
    ↓
SchemaService.diff(snapshot, { force })
    ↓
validateSnapshot(snapshot, force)  // 校验快照格式
    ↓
getSnapshotDiff(currentSnapshot, targetSnapshot)
    ├─ collections diff（新增/删除/编辑）
    ├─ fields diff（新增/删除/编辑，含 alias 转换特殊处理）
    ├─ systemFields diff（仅 schema.is_indexed）
    └─ relations diff
```

**Force 参数在 Diff 阶段的作用**：
- `force: true` 时，`validateSnapshot` 会跳过部分严格校验
- 用于在某些边界情况下允许 diff 计算继续进行

### 1.5 Apply 阶段的 Hash 校验

这是最关键的安全校验机制，位于 `validateApplyDiff` 函数中：

**校验流程**：

1. **Joi Schema 校验** - 验证 diff 结构完整性
2. **空 diff 检查** - 无变更时直接返回 false
3. **Hash 匹配检查** - 核心校验逻辑

```typescript
// 来源: validate-diff.ts 第 90-91 行
if (applyDiff.hash === currentSnapshotWithHash.hash || force) return true;
```

**Hash 不匹配时的降级检查**（当 `force: false` 且 hash 不匹配时）：

- **集合检查**：新增集合不能已存在；删除集合必须存在
- **字段检查**：新增字段不能已存在；删除字段必须存在
- **系统字段检查**：仅允许 `schema.is_indexed` 的 EDIT 操作
- **关系检查**：新增关系不能已存在；删除关系必须存在

如果以上所有检查都通过但 hash 不匹配，最终仍会抛出错误：

```
"Provided hash does not match the current instance's schema hash, 
indicating the schema has changed after this diff was generated. 
Please generate a new diff and try again or use the "force" query 
parameter to bypass this check"
```

### 1.6 Force 参数在 Apply 阶段的作用

| force 值 | 行为 | 适用场景 |
|----------|------|----------|
| `false`（默认） | 严格校验 hash，hash 不匹配则报错 | 生产环境、自动化部署 |
| `true` | 跳过 hash 校验，直接应用 diff | 手动干预、紧急修复、已知 schema 已变更但确认 diff 仍可应用 |

> **安全提示**：Force 模式下仍会执行 Joi schema 结构校验和空 diff 检查，只是跳过了 hash 一致性验证。

---

## 2. CollectionsService - 创建表时的主键与字段 Meta 注入

### 2.1 核心文件

[services/collections.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/collections.ts)

### 2.2 createOne 方法主流程

```
createOne(payload)
    ↓
校验 collection 名称合法性
    ├─ 不能为空
    ├─ 不能以 "directus_" 开头
    └─ 不能包含 "/"
    ↓
事务内执行
    ├─ 注入主键字段（如果没有）
    ├─ 补全字段 meta 的 collection/field 属性
    ├─ 创建数据库表 (knex createTable)
    ├─ 写入 directus_fields 记录
    └─ 写入 directus_collections 记录
    ↓
事务外：并发索引创建（如果启用）
    ↓
清理缓存与触发事件
```

### 2.3 主键注入机制

Directus 强制要求每个集合必须有主键，创建时自动注入逻辑位于 [collections.ts 第 114-140 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/collections.ts#L114-L140)：

**注入的默认主键字段**：
```typescript
const injectedPrimaryKeyField: RawField = {
    field: 'id',
    type: 'integer',
    meta: {
        hidden: true,
        interface: 'numeric',
        readonly: true,
    },
    schema: {
        is_primary_key: true,
        has_auto_increment: true,
    },
};
```

**注入条件**：
- 没有 `fields` 数组或数组为空 → 注入主键
- 有 `fields` 但没有任何字段设置 `is_primary_key: true` 或 `has_auto_increment: true` → 在数组开头注入

### 2.4 字段 Meta 补全

创建表时，每个字段的 meta 会被自动补全：

```typescript
// 来源: collections.ts 第 143-160 行
payload.fields = payload.fields.map((field) => {
    if (field.meta) {
        field.meta = {
            ...field.meta,
            field: field.field,        // 注入字段名
            collection: payload.collection!,  // 注入集合名
        };
    }
    // 日期类型特殊标记
    const flagToAdd = this.helpers.date.fieldFlagForField(field.type);
    if (flagToAdd) {
        addFieldFlag(field, flagToAdd);
    }
    return field;
});
```

### 2.5 排序处理

字段 meta 写入前会进行排序处理：
- 无分组字段：按顺序从 1 开始编号
- 有分组字段：每个分组内从 1 开始独立编号
- 使用 `lodash.merge` 保证用户自定义的 `sort` 值可以覆盖默认值

### 2.6 并发索引（Concurrent Index）

PostgreSQL 的 `CREATE INDEX CONCURRENTLY` 不能在事务内执行，因此采用两阶段策略：

1. **事务内**：创建表和列，但不创建索引（跳过 index/unique）
2. **事务外**：遍历字段，单独调用 `addColumnIndex` 创建并发索引

相关代码位于 [collections.ts 第 234-245 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/collections.ts#L234-L245)。

---

## 3. FieldsService - 字段 CRUD 的详细处理逻辑

### 3.1 核心文件

[services/fields.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts)

### 3.2 核心方法概览

| 方法 | 主要职责 |
|------|----------|
| `createField()` | 创建新字段（数据库列 + meta 记录） |
| `updateField()` | 更新字段（列类型/属性变更 + meta 更新） |
| `deleteField()` | 删除字段（列删除 + meta 清理 + 关系清理） |
| `addColumnToTable()` | 在 Knex table builder 上添加列定义 |
| `addColumnIndex()` | 创建索引（支持并发索引） |

### 3.3 Alias 字段处理

Alias 字段是 Directus 中的虚拟字段，在数据库中没有对应列。

**识别方式**：
- 字段的 `type` 在 `ALIAS_TYPES` 列表中
- 或通过 `getLocalType()` 推断出类型为 `alias`

**处理差异**：

| 操作 | 普通字段 | Alias 字段 |
|------|----------|------------|
| 创建 | 添加数据库列 + 写入 meta | 仅写入 meta |
| 更新 | 可能修改列属性 + 更新 meta | 仅更新 meta |
| 删除 | 删除数据库列 + 删除 meta + 清理关系 | 仅删除 meta + 清理关系 |
| 类型转换 | 不允许与 alias 互转 | 不允许与普通字段互转 |

**类型转换保护**（[fields.ts 第 539-546 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts#L539-L546)）：
```typescript
if (
    hookAdjustedField.type &&
    (hookAdjustedField.type === 'alias' ||
        this.schema.collections[collection]!.fields[field.field]?.type === 'alias') &&
    hookAdjustedField.type !== (this.schema.collections[collection]!.fields[field.field]?.type ?? 'alias')
) {
    throw new InvalidPayloadError({ reason: 'Alias type cannot be changed' });
}
```

### 3.4 System Field 处理

系统字段由 Directus 内置，不能随意修改。

**识别方式**：`isSystemField(collection, field)` 函数，来自 `@directus/system-data`

**更新限制**：
- 系统字段的 meta 不能创建或修改（[fields.ts 第 590 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts#L590)）
- 只有 `schema.is_indexed` 允许被修改（通过 systemFields 快照 diff 机制）
- 由 `systemFieldUpdateSchema`（zod 校验）严格限制可修改的属性

```typescript
// 来源: fields.ts 第 56-66 行
export const systemFieldUpdateSchema = z
    .object({
        collection: z.string().optional(),
        field: z.string().optional(),
        schema: z
            .object({
                is_indexed: z.boolean().optional(),
            })
            .strict(),
    })
    .strict();
```

### 3.5 Concurrent Index 处理

并发索引是 PostgreSQL 的特性，允许在不锁表的情况下创建索引。

**实现策略**：

1. **`addColumnToTable` 中跳过**：当 `attemptConcurrentIndex: true` 时，不在 `table.column.index()` 中创建索引
2. **`addColumnIndex` 中创建**：事务提交后，单独调用创建索引

**`addColumnToTable` 中的跳过逻辑**（[fields.ts 第 1008-1014 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts#L1008-L1014)）：
```typescript
if (field.schema?.is_indexed === true) {
    if ((!existing || existing.is_indexed === false) && !options?.attemptConcurrentIndex) {
        column.index(this.helpers.schema.generateIndexName('index', collection, field.field));
    }
}
```

**`addColumnIndex` 方法**（[fields.ts 第 1022-1052 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts#L1022-L1052)）：
- 通过 `helpers.schema.createIndex()` 调用具体数据库方言的实现
- 支持 `unique` 和普通索引两种类型
- 主键字段跳过（已有索引约束）

### 3.6 列类型变更处理

列类型变更在 `addColumnToTable` 方法中统一处理，支持的类型映射如下：

| Directus 类型 | Knex / 数据库类型 | 特殊处理 |
|--------------|-------------------|----------|
| `integer` + auto_increment | `increments()` | 自增主键 |
| `bigInteger` + auto_increment | `bigIncrements()` | 大整数自增 |
| `string` | `string(length)` | MSSQL 下 null length 转为 text |
| `float` / `decimal` | `float()` / `decimal()` | 支持 precision/scale |
| `csv` | `text()` | 以文本存储 |
| `hash` | `string(255)` | 固定长度 |
| `dateTime` | `dateTime(useTz: false)` | 无时区 |
| `timestamp` | `timestamp(useTz: true)` | 有时区 |
| `geometry*` | 空间类型 | 由 helpers.st.createColumn 处理 |
| 其他 KNEX_TYPES | 直接映射 | 如 boolean、text、json 等 |

**Alter 模式注意事项**：
- Knex 的 `column.alter()` 不是增量的，每次都需要完整定义
- 可空性、默认值必须每次都设置，否则会被重置
- 通过 `existing` 参数获取当前列的信息作为默认值

**列属性处理细节**：

1. **可空性**（`setNullable`）：每次 alter 都必须显式设置
2. **默认值**：
   - `now()` / `CURRENT_TIMESTAMP` → `knex.fn.now()`
   - `CURRENT_TIMESTAMP(n)` → 带精度的 now
   - 其他允许的 SQL 函数 → `knex.raw()`
   - 普通值 → 直接 `defaultTo()`
3. **主键**：设置 `primary().notNullable()`
4. **唯一约束**：添加/删除 unique 索引
5. **普通索引**：添加/删除普通索引

### 3.7 字段删除的级联处理

删除字段时的清理工作非常丰富（[fields.ts 第 721-860 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts#L721-L860)）：

```
deleteField(collection, field)
    ↓
 触发 delete filter 钩子
    ↓
事务内执行
    ├─ 清理相关关系
    │   ├─ M2O 字段：删除 FK 约束 + 删除 o2m 别名字段 + 删除 relation meta
    │   └─ O2M 字段：仅移除 relation 中的 one_field
    ├─ 删除数据库列（仅非 alias 字段）
    ├─ 清理集合 meta 中的引用
    │   ├─ archive_field
    │   ├─ sort_field
    │   └─ item_duplication_fields
    ├─ 处理字段分组（group 字段置空）
    ├─ 删除 directus_fields 记录
    └─ 清理权限表中的字段引用
    ↓
缓存清理与事件触发
```

---

## 4. RelationsService - 外键与关系元数据管理

### 4.1 核心文件

[services/relations.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/relations.ts)

### 4.2 关系数据模型

Directus 的关系由两部分组成：

1. **Schema 层**：数据库中的 `FOREIGN KEY` 约束
2. **Meta 层**：`directus_relations` 表中的关系元数据

**stitchRelations 方法**（[relations.ts 第 526-563 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/relations.ts#L526-L563)）将两部分数据合并：

- 有 FK 约束 + 有 meta → 完整关系
- 有 FK 约束 + 无 meta → schema 关系（meta 为 null）
- 无 FK 约束 + 有 meta → 逻辑关系（schema 为 null，如 a2o）

### 4.3 创建关系（createOne）

```
createOne(relation)
    ↓
校验
    ├─ collection 存在
    ├─ field 存在
    ├─ field 不是主键
    ├─ related_collection 存在
    └─ 该字段尚无关系
    ↓
preColumnChange + preRelationChange
    ↓
事务内执行
    ├─ 如有 related_collection：
    │   ├─ alterType（MySQL int 转 unsigned）
    │   └─ 创建外键约束
    └─ 写入 directus_relations 记录
    ↓
postColumnChange + 缓存清理 + 事件
```

**外键创建细节**（[relations.ts 第 254-273 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/relations.ts#L254-L273)）：

```typescript
const constraintName: string = getDefaultIndexName('foreign', relation.collection!, relation.field!);

const builder = table
    .foreign(relation.field!, constraintName)
    .references(`${relation.related_collection!}.${primaryKey}`);

if (relation.schema?.on_delete) builder.onDelete(relation.schema.on_delete);
if (relation.schema?.on_update) builder.onUpdate(relation.schema.on_update);
```

**MySQL 特殊处理 - alterType**（[relations.ts 第 635-653 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/relations.ts#L635-L653)）：

MySQL 的 `increments()` 默认是 `int unsigned`，但普通 `int` 字段是 `int signed`，导致外键无法创建。`alterType` 方法会自动将 m2o 字段转换为 `int unsigned` 以匹配主键类型。

### 4.4 更新关系（updateOne）

```
updateOne(collection, field, relation)
    ↓
校验：关系必须已存在
    ↓
事务内执行
    ├─ 如有 schema 变更：
    │   ├─ 删除旧 FK 约束
    │   ├─ alterType（MySQL）
    │   └─ 创建新 FK 约束
    └─ 更新/创建 meta 记录
```

**更新的限制**：
- Meta 部分可以自由更新
- Schema 部分仅支持更新 `on_delete` 和 `on_update`
- 如需更改关联的集合或字段，需先删除再创建

### 4.5 删除关系（deleteOne）

```
deleteOne(collection, field)
    ↓
校验：关系必须存在
    ↓
事务内执行
    ├─ 检查 FK 约束是否存在
    │   └─ 存在则 dropForeign
    └─ 删除 directus_relations 记录（如有 meta）
```

**安全检查**：删除 FK 前会先查询当前数据库中实际存在的外键约束名，避免尝试删除不存在的约束导致错误。

---

## 5. PostgresSchemaInspector - 数据库结构信息提供

### 5.1 核心文件

[packages/schema/src/dialects/postgres.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/schema/src/dialects/postgres.ts)

### 5.2 类结构与初始化

```typescript
export default class Postgres implements SchemaInspector {
    knex: Knex;
    schema: string;           // 默认 'public' 或 searchPath[0]
    explodedSchema: string[]; // searchPath 数组
}
```

**Schema 解析**：
- 无 `searchPath` 配置 → 使用 `public`
- `searchPath` 为字符串 → 单 schema
- `searchPath` 为数组 → 多 schema，取第一个为主

### 5.3 表信息（Tables）

**`tables()`** - 列出所有表名
- 查询 `pg_class` 系统表
- 过滤条件：`relkind = 'r'`（普通表）
- 按表名排序

**`tableInfo()`** - 获取表详细信息
- 关联 `pg_description` 获取表注释
- 支持单表查询或全量查询

**`hasTable()`** - 检查表是否存在

### 5.4 列信息（Columns）

**`columnInfo()`** 是最核心的方法，返回完整的列定义：

```typescript
type Column = {
    name: string;
    table: string;
    data_type: string;
    default_value: string | null;
    generation_expression: string | null;
    max_length: number | null;
    numeric_precision: number | null;
    numeric_scale: number | null;
    is_generated: boolean;
    is_nullable: boolean;
    is_unique: boolean;
    is_indexed: boolean;
    is_primary_key: boolean;
    has_auto_increment: boolean;
    foreign_key_schema: string | null;
    foreign_key_table: string | null;
    foreign_key_column: string | null;
    comment: string | null;
};
```

**实现方式**：两次并行查询 + 合并

1. **主查询**（`pg_attribute` + `pg_class` + `pg_attrdef` + `pg_description`）：
   - 列名、表名、数据类型、默认值、是否可空
   - 最大长度（char/varchar/bit/varbit）
   - 数值精度与小数位（int2/int4/int8/numeric/float4/float8）
   - 生成列信息（PostgreSQL 12+）
   - 单列普通索引（通过 LATERAL 子查询）

2. **约束查询**（`pg_constraint`）：
   - 主键约束（`contype = 'p'`）
   - 唯一约束（`contype = 'u'`）
   - 外键约束（`contype = 'f'`）
   - 自增检测（`pg_get_serial_sequence`）

**默认值解析**（`parseDefaultValue` 函数）：
- `nextval(...)` → 保留原值（自增序列）
- `'value'::type` → 剥离类型转换，提取值
- `'null'` → 返回 null
- 其他 → 去除引号

### 5.5 索引信息

索引信息通过两个渠道获取：

1. **`columnInfo` 中的 LATERAL 子查询**：检测单列普通索引
2. **约束查询**：检测主键、唯一约束（也算一种索引）

**索引命名约定**（在 helpers 中实现）：
- 唯一索引：`unique_<table>_<column>`
- 普通索引：`index_<table>_<column>`
- 外键：`foreign_<table>_<column>`

### 5.6 Geometry 类型支持

PostgreSQL 通过 PostGIS 扩展支持空间数据类型。

**检测 PostGIS**：
```sql
SELECT oid FROM pg_proc WHERE proname = 'postgis_version'
```

**Geometry 列获取**：
- 查询 `geometry_columns` 和 `geography_columns` 视图
- UNION 合并结果
- 关联 `information_schema.tables` 确保只包含基表

**数据类型修正逻辑**（[postgres.ts 第 507-560 行](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/schema/src/dialects/postgres.ts#L507-L560)）：

由于 PostgreSQL 原生的 `point`、`polygon` 等类型与 PostGIS 的几何类型不同，系统采用以下策略：

1. **原生几何类型（point/polygon）** → 标记为 `'unknown'`（不支持）
2. **PostGIS 几何类型** → 用 PostGIS 提供的 `type` 覆盖 `data_type`

**overview() 方法中的处理**：
- 先检查 PostGIS 是否存在
- 存在则额外查询 geometry/geography 列
- 将 geometry 列的 data_type 覆盖到 overview 对象中

### 5.7 外键信息（foreignKeys）

独立的 `foreignKeys()` 方法查询完整的外键约束信息：

```typescript
type ForeignKey = {
    constraint_name: string;
    table: string;
    column: string;
    foreign_key_schema: string | null;
    foreign_key_table: string | null;
    foreign_key_column: string | null;
    on_update: string | null;  // RESTRICT / CASCADE / SET NULL / SET DEFAULT / NO ACTION
    on_delete: string | null;  // RESTRICT / CASCADE / SET NULL / SET DEFAULT / NO ACTION
};
```

**操作码映射**：
| 代码 | 含义 |
|------|------|
| `r` | RESTRICT |
| `c` | CASCADE |
| `n` | SET NULL |
| `d` | SET DEFAULT |
| `a` | NO ACTION |

### 5.8 主键查询（primary）

单独的 `primary(table)` 方法用于快速查询单个表的主键列名。

---

## 6. 整体同步机制总结

### 6.1 数据一致性保证

Directus 通过以下机制保证元数据与数据库 DDL 的一致性：

1. **事务封装**：大部分 schema 操作都在事务中执行，失败时自动回滚
2. **双写机制**：每次操作同时更新数据库 DDL 和 Directus 元数据表
3. **Hash 校验**：snapshot/apply 流程通过 hash 确保 schema 未被意外修改
4. **缓存失效**：schema 变更后立即清除缓存，确保读取最新状态
5. **Schema 快照**：可随时生成快照，作为 schema 的可信版本

### 6.2 关键约束

- **MySQL 限制**：DDL 语句不支持事务，部分失败可能导致不一致
- **并发索引**：PostgreSQL 的 CONCURRENTLY 索引不能在事务内创建，需特殊两阶段处理
- **系统字段**：只能修改索引属性，其他属性不可修改
- **Alias 字段**：不能与普通字段互相转换
- **主键约束**：每个集合必须有主键，创建时自动注入

### 6.3 调用链总览

```
SchemaService
    ├─ snapshot() → getSnapshot()
    │   ├─ CollectionsService.readByQuery()
    │   ├─ FieldsService.readAll()
    │   └─ RelationsService.readAll()
    │       └─ SchemaInspector (Postgres)
    │           ├─ tableInfo()
    │           ├─ columnInfo()
    │           └─ foreignKeys()
    │
    ├─ diff() → getSnapshotDiff() → deep-diff 库
    │
    └─ apply() → validateApplyDiff() → applyDiff()
        ├─ CollectionsService.createOne()/updateOne()/deleteOne()
        │   └─ FieldsService.createField()
        │       └─ addColumnToTable()
        ├─ FieldsService.createField()/updateField()/deleteField()
        │   └─ addColumnIndex()
        └─ RelationsService.createOne()/updateOne()/deleteOne()
            └─ 外键约束操作
```

---

## 附录：关键文件索引

| 模块 | 文件路径 |
|------|----------|
| Schema 控制器 | [controllers/schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/schema.ts) |
| Schema 服务 | [services/schema.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/schema.ts) |
| Collections 服务 | [services/collections.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/collections.ts) |
| Fields 服务 | [services/fields.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/fields.ts) |
| Relations 服务 | [services/relations.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/relations.ts) |
| 获取快照 | [utils/get-snapshot.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-snapshot.ts) |
| 快照差异 | [utils/get-snapshot-diff.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-snapshot-diff.ts) |
| 应用差异 | [utils/apply-diff.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/apply-diff.ts) |
| 验证 Diff | [utils/validate-diff.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/validate-diff.ts) |
| 版本化 Hash | [utils/get-versioned-hash.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/utils/get-versioned-hash.ts) |
| Postgres Inspector | [packages/schema/src/dialects/postgres.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/schema/src/dialects/postgres.ts) |
| 快照类型定义 | [packages/types/src/snapshot.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/packages/types/src/snapshot.ts) |
