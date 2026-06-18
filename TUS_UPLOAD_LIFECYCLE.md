# Directus TUS 文件上传完整生命周期

本文档详细追踪了 `/files/tus` 端点从创建、上传、查询、取消到完成的完整生命周期。

## 目录

1. [整体架构概览](#整体架构概览)
2. [权限映射：checkFileAccess](#权限映射checkfileaccess)
3. [POST 创建上传](#post-创建上传)
4. [PATCH 写入 Chunk](#patch-写入-chunk)
5. [HEAD 查询上传状态](#head-查询上传状态)
6. [DELETE 取消上传](#delete-取消上传)
7. [upload-metadata 与 replace_id 处理](#upload-metadata-与-replace_id-处理)
8. [directus_files 临时行记录](#directus_files-临时行记录)
9. [Storage 驱动 TUS 方法](#storage-驱动-tus-方法)
10. [onUploadFinish：新上传 vs 替换上传](#onuploafinish新上传-vs-替换上传)
11. [定时清理过期上传](#定时清理过期上传)

---

## 整体架构概览

Directus 的 TUS（Resumable Uploads）实现基于 `@tus/server` 库，通过自定义 `TusDataStore` 将上传状态持久化到 `directus_files` 表中。

### 核心文件

| 文件 | 作用 |
|------|------|
| [controllers/tus.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/controllers/tus.ts) | TUS 路由与权限中间件 |
| [services/tus/server.ts](file:///Users/zhangjing/Desktop/so-coders/0619-under/directus/api/src/services/tus/server.ts) | TUS 服务器创建与 onUploadFinish 回调 |
| [services/tus/data-store.ts](file:///Users/zhangjing/Desktop/so-coders/061