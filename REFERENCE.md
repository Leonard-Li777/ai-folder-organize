# AI Folder Organize — API 参考

## 基础信息

- **Base URL**: 由 `discover.js` 发现，格式为 `http://127.0.0.1:{port}`
- **响应格式**: `{ "success": true, ... }` 或 `{ "success": false, "error": "..." }`
- **数据编码**: 所有请求与响应必须使用 **UTF-8** 编码。包含中文字符的 POST 请求必须以 UTF-8 字节流发送（Header 指定 `Content-Type: application/json; charset=utf-8`）
- **GET** 用于数据查询；**POST** 用于触发客户端操作

---

## 数据查询端点

### GET /api/workspaces

获取所有已注册工作区。

**返回**:

```json
{
  "success": true,
  "data": [{ "id": 1, "path": "D:\\Workspace\\Downloads", "name": "Downloads", "type": "PRIVATE" }]
}
```

---

## GET /api/analysis/queue-status

查询分析队列积压状态。

**返回**:

```json
{
  "success": true,
  "systemIdle": false,
  "queueLength": 45,
  "currentProcessingFile": "D:\\Workspace\\Downloads\\IMG_102.jpg"
}
```

---

## GET /api/analysis/progress

查询分析进度。

**返回**:

```json
{
  "success": true,
  "isIdle": false,
  "analysis": { "status": "processing", "progressPercentage": 75.0 },
  "organizePage": { "status": "idle", "progressPercentage": 0 }
}
```

---

## GET /api/files/analysis-data

查询文件分析数据。

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `fileId` | number | 否 | 文件 ID，提供时查单个文件；不提供时分页返回列表 |
| `fields` | string | 否 | 逗号分隔，可选值: `description`, `smartName`, `tags`, `metadata`, `qualityScore`, `content` |
| `limit` | number | 否 | 分页大小（默认 10） |
| `offset` | number | 否 | 分页偏移（默认 0） |

**返回（单个文件）**:

```json
{
  "success": true,
  "fileFingerprint": "abc123",
  "smartName": "项目计划_v3",
  "description": "这是一份项目计划文档...",
  "size": 1024000,
  "type": "document",
  "mimeType": "application/pdf",
  "author": "张三",
  "language": "zh-CN",
  "path": "D:\\Workspace\\Downloads\\项目计划.pdf",
  "name": "项目计划.pdf",
  "parentArchive": null,
  "unitId": "unit_1",
  "thumbnailPath": null
}
```

**返回（列表）**:

```json
{
  "success": true,
  "data": [ ... ]
}
```

---

## GET /api/files/search

V4 混合全文与语义检索文件（支持两阶段渐进式返回与视频切片高光定位）。

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `keyword` | string | 是 | 搜索关键词（支持多语言、拼音缩写及自然语言短语） |
| `mode` | string | 否 | 响应模式：`final`（默认，8s 超时返回最终融合全量结果）、`stream`（SSE 分阶段推送） |
| `scope` | string | 否 | 范围：`real`（默认真实工作区）, `virtual`（虚拟目录）, `all`（全域） |
| `workspaceId` | number | 否 | 工作区 ID（仅 `scope=real` 时有效） |
| `virtualDirectoryId` | number | 否 | 虚拟目录 ID（仅 `scope=virtual` 时有效） |
| `limit` | number | 否 | 分页大小（默认 20，上限 50） |
| `offset` | number | 否 | 分页偏移（默认 0） |

**返回 (mode=final 单次 JSON 响应)**:

```json
{
  "success": true,
  "data": {
    "total": 42,
    "isSettled": true,
    "searchDurationMs": 348,
    "degraded": false,
    "degradedReasons": [],
    "clipPending": false,
    "items": [
      {
        "id": "105",
        "name": "产品发布会录像.mp4",
        "path": "D:\\videos\\2026_launch.mp4",
        "score": 0.94,
        "matchType": "video_chunk",
        "mediaType": "video",
        "hasVideoSegments": true,
        "snippet": "...新一代文件智能整理架构正式亮相...",
        "videoSegments": [
          {
            "chunkIndices": [4],
            "startSec": 132.0,
            "endSec": 156.5,
            "repPtsSec": 140.0,
            "seekSec": 132.0,
            "score": 0.94,
            "textSnippet": "这里是发布会重点介绍的智能流式搜索与视频定位功能"
          }
        ],
        "videoSegmentsTotal": 1,
        "videoSegmentsTruncated": false
      }
    ]
  }
}
```

**返回 (mode=stream SSE 流式推送)**:

响应头为 `Content-Type: text/event-stream; charset=utf-8`。

- `event: phase1`：字面与缓存快速召回（首屏快速响应，通常无视频切片）
- `event: phase2`：语义多路召回、融合重排与视频切片结果（全量置换 phase1 列表）
- `event: done`：搜索完成通知，客户端收到后请关闭 EventSource 连接
- `event: error`：异常通知，如限流或引擎错误

---

## GET /api/organize/templates

获取整理方案提示词（**不调用本地 AI**）。

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `workspaceId` | number | 是 | 工作区 ID |
| `userInstruction` | string | 否 | 用户视角要求，如"大学生视角" |

**返回**:

```json
{
  "success": true,
  "workspaceId": 1,
  "workspacePath": "D:\\Workspace\\Downloads",
  "fileCount": 150,
  "systemPrompt": "你是一名资深内容归类专家...",
  "userPrompt": "### 文件数据\n..."
}
```

### 使用方式

接口返回的 `systemPrompt` 和 `userPrompt` **不是最终结果**，而是需要提交给 AI 模型的提示词：

1. 将 `systemPrompt` 作为 system 消息
2. 将 `userPrompt` 作为 user 消息
3. 提交给当前 AI 模型推理
4. 模型输出 3 份整理方案，每份包含 `name`, `perspective`, `strategy`

---

## GET /api/virtual-directories

查询虚拟目录列表。

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `workspaceId` | number | 是 | 工作区 ID |
| `depth` | number | 否 | 限制返回层级 |

**返回**:

```json
{
  "success": true,
  "data": [{ "id": 1, "name": "我的虚拟目录", "workspaceId": 1 }]
}
```

---

## POST /api/organize/apply-plan

将 AI 生成的整理方案提交到客户端的自定义虚拟目录弹窗中。客户端将自动切换到整理页面、弹窗并预填方案数据，等待用户确认。

**请求体**:

```json
{
  "name": "encodeURIComponent(\"学生作业整理方案\")",
  "perspective": "encodeURIComponent(\"从大学生的视角，按学期和课程分类\")",
  "strategy": "encodeURIComponent(\"根目录：按学期（2024秋/2025春）→ 子目录：按课程名称 → 文件：按类型（作业/实验/报告）\")"
}
```

> **重要编码说明**：
>
> 1. 所有 POST 请求必须以 **UTF-8** 编码传输（Header 设置 `Content-Type: application/json; charset=utf-8`）。
> 2. 在 **Windows PowerShell (Invoke-RestMethod)** 环境中，由于默认 ANSI 编码会将中文损毁为问号 `?`，须使用 `[System.Text.Encoding]::UTF8.GetBytes($jsonString)` 将字符串转为 UTF-8 字节数组后通过 `-Body` 发送。
> 3. 或者使用 `encodeURIComponent()` 对中文/Unicode 字段进行 URI 编码后再发送，服务端会自动解包并还原。

| 字段          | 类型   | 必填 | 说明                                      |
| ------------- | ------ | ---- | ----------------------------------------- |
| `name`        | string | 是   | 虚拟目录名称（方案标题），须 URI 编码     |
| `strategy`    | string | 是   | 整理策略描述（树形结构说明），须 URI 编码 |
| `perspective` | string | 否   | 用户视角说明，须 URI 编码                 |

**返回**:

```json
{
  "success": true,
  "message": "整理方案已发送到整理页面"
}
```

**错误**:

```json
{
  "success": false,
  "error": "缺少必填字段: name"
}
```

---

## 文件处理工具端点 (Skill Tool APIs)

以下端点由统一微服务（UnifiedWorkerServer）提供，支持直接调用文件处理工具。

### GET /api/health

健康检查端点，返回微服务状态及各服务可用性。

**返回**:

```json
{
  "success": true,
  "status": "ok",
  "uptime": 3600,
  "memoryUsage": {
    "rss": 52428800,
    "heapTotal": 20971520,
    "heapUsed": 15728640
  },
  "activeServices": {
    "ocr": true,
    "magika": true,
    "exiftool": true,
    "libreoffice": true,
    "ffmpeg": true
  }
}
```

---

### POST /api/extract/ocr

OCR 文字识别，支持从图片或文档中提取文字内容。

**请求体**:

| 参数                | 类型   | 必填 | 说明                                                |
| ------------------- | ------ | ---- | --------------------------------------------------- |
| `filePath`          | string | 否   | 文件绝对路径（与 imageBufferBase64 二选一）         |
| `imageBufferBase64` | string | 否   | Base64 编码的图片数据（与 filePath 二选一）         |
| `modelType`         | string | 否   | OCR 模型类型，可选值: `default`, `fast`, `accurate` |

**请求示例**:

```json
{
  "filePath": "D:\\photos\\receipt.jpg"
}
```

或

```json
{
  "imageBufferBase64": "iVBORw0KGgoAAAANSUhEUgAA...",
  "modelType": "accurate"
}
```

**返回**:

```json
{
  "success": true,
  "text": "识别出的文字内容...",
  "confidence": 0.95,
  "durationMs": 1200
}
```

| 字段         | 类型   | 说明                 |
| ------------ | ------ | -------------------- |
| `text`       | string | OCR 识别出的文本内容 |
| `confidence` | number | 识别置信度 (0-1)     |
| `durationMs` | number | 处理耗时（毫秒）     |

---

### POST /api/extract/identify

Magika 智能文件类型识别，基于 Google Magika 深度学习模型判断文件真实类型。

**请求体**:

| 参数       | 类型   | 必填 | 说明         |
| ---------- | ------ | ---- | ------------ |
| `filePath` | string | 是   | 文件绝对路径 |

\*\*请求示例:

```json
{
  "filePath": "D:\\docs\\mystery_file"
}
```

**返回**:

```json
{
  "success": true,
  "label": "pdf",
  "group": "document",
  "mime_type": "application/pdf",
  "extensions": [".pdf"]
}
```

| 字段         | 类型     | 说明                                        |
| ------------ | -------- | ------------------------------------------- |
| `label`      | string   | 文件类型标签（如 `pdf`, `jpeg`, `mp3`）     |
| `group`      | string   | 文件组别（如 `document`, `image`, `audio`） |
| `mime_type`  | string   | MIME 类型                                   |
| `extensions` | string[] | 推荐的文件扩展名列表                        |

---

### POST /api/extract/metadata

ExifTool 元数据提取，支持从图片、视频、音频、文档中提取详细元数据。

**请求体**:

| 参数       | 类型   | 必填 | 说明         |
| ---------- | ------ | ---- | ------------ |
| `filePath` | string | 是   | 文件绝对路径 |

\*\*请求示例:

```json
{
  "filePath": "D:\\photos\\vacation.jpg"
}
```

**返回**:

```json
{
  "success": true,
  "Make": "Canon",
  "Model": "EOS R5",
  "DateTimeOriginal": "2024:08:15 14:30:00",
  "ImageWidth": 8192,
  "ImageHeight": 5464,
  "FileSize": 4567890,
  "FileType": "JPEG",
  "MIMEType": "image/jpeg"
}
```

返回的元数据字段因文件类型而异：

- **图片**: 相机信息、拍摄参数、GPS 坐标、尺寸等
- **视频**: 编码格式、分辨率、时长、码率等
- **音频**: 艺术家、专辑、时长、采样率等
- **文档**: 作者、创建时间、页数、标题等

---

### POST /api/extract/text

文档内容提取，基于 Anydoc 引擎从 PDF、Word、PPT、Excel、HTML 等格式中提取 Markdown 格式文本内容和嵌入图片资源。

**请求体**:

| 参数       | 类型   | 必填 | 说明         |
| ---------- | ------ | ---- | ------------ |
| `filePath` | string | 是   | 文件绝对路径 |
| `options`  | object | 否   | 提取配置选项 |

**options 参数**:

| 参数        | 类型   | 必填 | 说明                         |
| ----------- | ------ | ---- | ---------------------------- |
| `timeoutMs` | number | 否   | 超时时间（毫秒，默认 60000） |

**请求示例**:

```json
{
  "filePath": "D:\\docs\\report.pdf",
  "options": { "timeoutMs": 30000 }
}
```

**返回**:

```json
{
  "success": true,
  "content": "# 报告标题\n\n## 摘要\n\n本文档介绍了...",
  "assets": [
    {
      "path": "images/chart1.png",
      "name": "chart1.png",
      "width": 800,
      "height": 600,
      "size": 125000
    }
  ]
}
```

| 字段            | 类型   | 说明                           |
| --------------- | ------ | ------------------------------ |
| `content`       | string | 提取出的 Markdown 格式文本内容 |
| `assets`        | array  | 文档中嵌入的图片资源列表       |
| `assets.path`   | string | 图片相对路径                   |
| `assets.name`   | string | 图片文件名                     |
| `assets.width`  | number | 图片宽度（像素）               |
| `assets.height` | number | 图片高度（像素）               |
| `assets.size`   | number | 图片文件大小（字节）           |

**支持的文件格式**:

- **Office 文档**: DOCX, DOC, PPTX, PPT, XLSX, XLS
- **OpenDocument**: ODT, ODP, ODS
- **其他文档**: RTF, EPUB, CSV, PDF
- **标记语言**: HTML, XML, Markdown

**错误处理**:

- `unsupported`: 不支持的文件格式（如 .lnk 快捷方式）
- `encrypted`: 加密的文档
- 超时：默认 60 秒超时

---

### POST /api/extract/cover-image

文档/图像封面与转码预览，生成文档或视频的缩略图封面。

**请求体**:

| 参数              | 类型   | 必填 | 说明                                |
| ----------------- | ------ | ---- | ----------------------------------- |
| `filePath`        | string | 是   | 源文件绝对路径                      |
| `outputCoverPath` | string | 是   | 输出封面图片路径（推荐 .webp 格式） |
| `options`         | object | 否   | 配置选项                            |

**options 参数（全部可选）**:

| 参数        | 类型   | 默认值 | 说明              |
| ----------- | ------ | ------ | ----------------- |
| `maxWidth`  | number | 800    | 最大宽度（像素）  |
| `maxHeight` | number | 800    | 最大高度（像素）  |
| `quality`   | number | 80     | WebP 质量 (1-100) |

**请求示例**:

```json
{
  "filePath": "D:\\docs\\report.pdf",
  "outputCoverPath": "D:\\temp\\report_cover.webp"
}
```

或自定义尺寸：

```json
{
  "filePath": "D:\\docs\\report.pdf",
  "outputCoverPath": "D:\\temp\\report_cover.webp",
  "options": { "maxWidth": 1200, "maxHeight": 900, "quality": 90 }
}
```

**返回**:

```json
{
  "success": true,
  "coverPath": "D:\\temp\\report_cover.webp",
  "durationMs": 2500
}
```

| 字段         | 类型   | 说明               |
| ------------ | ------ | ------------------ |
| `coverPath`  | string | 生成的封面图片路径 |
| `durationMs` | number | 处理耗时（毫秒）   |

**支持的文件格式**:

- **文档**: PDF, DOCX, PPTX, XLSX, ODT, ODP, ODS
- **视频**: MP4, AVI, MKV, MOV, WMV, FLV, WebM
- **图像**: JPG, PNG, TIFF, BMP, WebP

---

### POST /api/files/analysis-data-smart

智能获取文件分析数据，自动检查文件所属工作目录，如果未分析则优先插入分析队列。

**请求体**:

| 参数        | 类型    | 必填 | 默认值 | 说明                             |
| ----------- | ------- | ---- | ------ | -------------------------------- |
| `filePath`  | string  | 是   | -      | 文件绝对路径                     |
| `autoQueue` | boolean | 否   | true   | 未分析时是否自动加入队列         |
| `priority`  | number  | 否   | 100    | 队列优先级（数值越大优先级越高） |

**请求示例**:

```json
{
  "filePath": "D:\\docs\\report.pdf",
  "autoQueue": true,
  "priority": 100
}
```

**返回示例（已分析）**:

```json
{
  "success": true,
  "belongsToWorkspace": true,
  "workspace": {
    "id": 1,
    "name": "工作目录",
    "path": "D:\\workspace",
    "type": "SPEEDY"
  },
  "isAnalyzed": true,
  "result": {
    "id": 123,
    "fileFingerprint": "abc123...",
    "path": "D:\\docs\\report.pdf",
    "name": "report.pdf",
    "smartName": "2024年度报告",
    "description": "公司年度财务报告...",
    "category": "document",
    "qualityScore": 8.5,
    "multimodalContent": "...",
    "lastAnalyzedAt": "2024-08-15T10:30:00Z",
    "thumbnailPath": ".VirtualDirectory/.thumbnail/abc123.webp"
  }
}
```

**返回示例（未分析，已入队）**:

```json
{
  "success": true,
  "belongsToWorkspace": true,
  "workspace": {
    "id": 1,
    "name": "工作目录",
    "path": "D:\\workspace",
    "type": "SPEEDY"
  },
  "isAnalyzed": false,
  "queued": true,
  "queueTaskId": 456,
  "message": "文件已加入分析队列，正在优先分析"
}
```

**返回示例（不属于工作区）**:

```json
{
  "success": true,
  "belongsToWorkspace": false,
  "isAnalyzed": false,
  "message": "该文件不属于任何工作目录，请将其所在目录添加到萤核智能文件夹工作目录中"
}
```

**返回字段说明**:

| 字段                 | 类型    | 说明                               |
| -------------------- | ------- | ---------------------------------- |
| `belongsToWorkspace` | boolean | 是否属于已知工作目录               |
| `workspace`          | object  | 工作区信息（id, name, path, type） |
| `isAnalyzed`         | boolean | 文件是否已完成分析                 |
| `queued`             | boolean | 是否已加入分析队列                 |
| `queueTaskId`        | number  | 队列任务 ID（如果已入队）          |
| `result`             | object  | 分析结果（如果已分析）             |
| `message`            | string  | 提示信息                           |
