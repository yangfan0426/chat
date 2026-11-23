# 04 · API 接口设计

> 基础约定：REST + JSON；流式接口用 **SSE**（`text/event-stream`）。所有类型定义在 `packages/shared` 中前后端共享。

## 0. 通用约定

- Base URL：`/api`
- 时间格式：ISO 8601 字符串
- 分页：游标式，`?cursor=<messageId>&limit=50`（聊天历史倒序加载用）
- 错误响应统一结构：

```json
{
  "error": {
    "code": "MODEL_UNAVAILABLE",
    "message": "人类可读的描述",
    "details": {}
  }
}
```

| HTTP 状态 | 场景 |
|---|---|
| 400 | 参数校验失败（`INVALID_REQUEST`） |
| 404 | 资源不存在（`NOT_FOUND`） |
| 413 | 文件超限（`FILE_TOO_LARGE`） |
| 422 | 模型/能力不支持当前请求（如非 vision 模型传图片）（`CAPABILITY_UNSUPPORTED`） |
| 500 | 未捕获异常（`INTERNAL`） |
| 502 | 上游模型 API 调用失败（`UPSTREAM_ERROR`） |

## 1. 会话与消息

### `GET /api/conversations`
会话列表（按更新时间倒序）。

**响应 200**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Docker 网络问题排查",
      "modelId": "deepseek-chat",
      "kbIds": ["kb-uuid"],
      "createdAt": "2026-11-23T10:00:00Z",
      "updatedAt": "2026-11-23T10:05:00Z"
    }
  ]
}
```

### `POST /api/conversations`
创建会话。

**请求**
```json
{ "title": "可选，默认取第一条用户消息前 20 字", "modelId": "deepseek-chat", "kbIds": [] }
```

### `PATCH /api/conversations/:id`
重命名 / 切换关联知识库 / 切换模型。

**请求**
```json
{ "title": "新标题", "kbIds": ["kb-uuid"], "modelId": "kimi-k2" }
```

### `DELETE /api/conversations/:id`
删除会话及其全部消息。返回 `204`。

### `GET /api/conversations/:id/messages`
分页加载历史消息（旧→新返回当前页）。

**查询参数**：`cursor`（早于该消息）、`limit`（默认 50，最大 100）

**响应 200**
```json
{
  "data": [
    {
      "id": "uuid",
      "role": "user" | "assistant",
      "content": "消息正文（Markdown）",
      "attachments": [
        { "type": "image", "url": "/api/files/xxx.png", "name": "报错截图.png" }
      ],
      "modelId": "deepseek-chat",
      "finishReason": "stop" | "aborted" | "error" | null,
      "usage": { "promptTokens": 1234, "completionTokens": 56 },
      "createdAt": "..."
    }
  ],
  "nextCursor": "messageId | null"
}
```

## 2. 聊天（流式核心）

### `POST /api/chat`
发送消息，SSE 流式返回。SSE 事件序列：`message-start` → N 个 `message-delta` → `message-end` / `message-error`。

**请求**
```json
{
  "conversationId": "uuid",
  "content": "用户输入",
  "attachments": [
    { "type": "image", "fileId": "uploaded-file-uuid" }
  ]
}
```

**响应头**：`Content-Type: text/event-stream`、`Cache-Control: no-cache`、`X-Accel-Buffering: no`（反代防缓冲）

**SSE 事件**（事件名 + JSON data）：
```
event: message-start
data: {"messageId": "uuid", "createdAt": "..."}

event: message-delta
data: {"delta": "这一段"}

event: message-end
data: {"usage": {"promptTokens": 1, "completionTokens": 2}, "finishReason": "stop"}
```
- 中断/出错时 `finishReason` 为 `aborted` / `error`；仍保证 assistant 消息完整落库，前端只需渲染。
- 客户端断开 → 服务端取消上游流，部分正文落库。

**错误**：会话不存在 `404`；会话绑定的模型已删除 `422 MODEL_UNAVAILABLE`。

## 3. 模型与服务商配置

### `GET /api/models`
当前可用模型列表（供前端选择器；**不含任何密钥信息**）。

**响应 200**
```json
{
  "data": [
    {
      "id": "deepseek-chat",
      "label": "DeepSeek Chat",
      "provider": "deepseek",
      "capabilities": { "vision": false, "contextWindow": 64000 },
      "isDefault": true
    }
  ]
}
```

### `GET /api/providers`
服务商列表（脱敏，key 仅返回掩码 `sk-****abcd`）。

### `POST /api/providers`
新增服务商。

**请求**
```json
{
  "name": "deepseek",
  "baseUrl": "https://api.deepseek.com/v1",
  "apiKey": "sk-...",
  "models": [
    { "id": "deepseek-chat", "label": "DeepSeek Chat", "vision": false, "contextWindow": 64000 }
  ],
  "embeddingModel": { "id": "...", "dimension": 1536 }
}
```

### `PATCH /api/providers/:id` / `DELETE /api/providers/:id`
修改（apiKey 留空表示不修改）/ 删除（校验无会话引用该 provider 下模型）。

## 4. 知识库

### `POST /api/kb/documents`（multipart）
上传文档。字段：`file`（pdf/docx/txt/md，≤20MB）、`kbId` 可选（默认进 default 库）。

**响应 201**
```json
{ "id": "uuid", "filename": "产品手册.pdf", "status": "pending", "size": 102400 }
```

### `GET /api/kb/documents`
文档列表，含解析状态。

**响应 200**
```json
{
  "data": [
    {
      "id": "uuid", "filename": "产品手册.pdf", "status": "parsing",
      "chunkCount": 12, "error": null, "createdAt": "..."
    }
  ]
}
```

### `DELETE /api/kb/documents/:id`
删除文档及其 chunks 与向量。返回 `204`。

### `POST /api/kb/retry/:id`
对 `failed` 状态文档重新入队解析。返回 `202`。

### `POST /api/kb/search`（调试接口）
检索测试：输入 query 返回命中的 chunks，用于 RAG 调优。

**请求**：`{ "query": "退款政策", "kbIds": ["..."], "topK": 5 }`

**响应**
```json
{
  "data": [
    { "documentId": "uuid", "filename": "产品手册.pdf", "score": 0.87,
      "content": "命中分块文本...", "chunkIndex": 3 }
  ]
}
```

## 5. 文件

### `POST /api/files`（multipart）
通用文件上传（聊天图片等）。返回 `{ id, url, name, size }`。

### `GET /api/files/:id`
按 id 读取文件流（带 content-type）。聊天中的附件经由此接口展示。

## 6. 健康检查

### `GET /healthz`
`{ "status": "ok", "db": "ok" }`，供 docker 探活。

## 7. 版本策略

- 路径不含版本号，阶段一为唯一版本；破坏性变更时以 `Accept` 头或新路径演进。
- SSE 事件结构增加字段视为兼容变更（客户端忽略未知字段）。
