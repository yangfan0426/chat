# 07 · RAG 知识库设计（knowledge-base 插件）

> 本文档描述 **knowledge-base 插件**的内部设计。对底座而言它只是一个普通插件：订阅 `chat/request` 事件、注册自己的路由与表、经内核 `llm/embed` 服务获取向量化能力。插件契约见 [09-插件体系设计](09-插件体系设计.md)。

## 1. 设计目标

让用户上传自有文档（PDF/Word/TXT/Markdown），在指定会话中提问时自动基于文档内容回答，并标注信息来源。核心质量指标：**召回准确（Top-K 命中相关段落）** 与 **回答可溯源**。

## 2. 插件对底座的挂接方式

| 挂接点 | 用途 |
|---|---|
| `using: ['config', 'http', 'database', 'llm']` | 注入所需内核服务 |
| 订阅 `chat/request` 事件 | 检索命中 → 改写 systemPrompt 注入资料段 |
| `http.registerRoute` | 文档上传/列表/删除、检索调试等 `/api/kb/*` 路由 |
| `database` | 建表 `kb_documents` / `kb_chunks`（命名空间归属本插件） |

## 3. 流水线总览

```
上传 → 存储原始文件 → 解析为纯文本 → 清洗 → 语义分块
    → 批量 Embedding → 写入 chunks(向量) → ready
                                  │
提问时：query → embed → Top-K 检索 → 注入 prompt → LLM 流式回答（带来源标注）
```

## 4. 文档解析

| 格式 | 库 | 处理要点 |
|---|---|---|
| PDF | `unpdf` | 按页提取文本；记录页码作为 source 定位信息 |
| DOCX | `mammoth` | 转纯文本，保留标题/段落层级（`mammoth.extractRawText` + styleMap） |
| TXT / MD | 原生读取 | MD 保留标题结构有助于分块 |

**清洗规则**：
- 合并因 PDF 换行产生的断行（行尾非句读字符 + 下一行小写开头 → 合并）
- 去除页眉页脚等重复模式（连续 N 页出现相同行则剔除）
- 压缩 3 个以上连续空白行
- 首期**不支持扫描件**（无文本层的 PDF 解析结果为空 → 状态置 failed，error 提示"不支持扫描件"）

## 5. 分块策略（Chunking）

采用**结构感知的递归分块**，而非固定长度硬切：

1. 优先按 Markdown/文档标题、段落边界切分
2. 块目标大小 **512 tokens**，重叠 **50 tokens**（重叠保证跨块语义不被切断）
3. 单段落超过目标大小 → 按句子递归切；句子仍超长 → 按固定 token 窗口硬切
4. 每块携带元数据：`chunk_index`、`token_count`、所属标题路径（如 "第3章 > 安装部署"）

> 中文 token 估算：1 token ≈ 0.6 个汉字（粗估即可，无需精确 tokenizer）。

## 6. 向量化与存储

- **Embedding 模型来源**：经内核 `llm/embed` 服务调用，由已挂载的 provider 插件提供（如 `text-embedding-3-small` 1536 维）。同一知识库内维度必须一致——建库时选定 embedding 提供方并记录维度。
- **写入**：分块后批量 `embed()`（每批 ≤ 100 条，受 API 限流约束），结果写入 `kb_chunks.embedding`。
- **索引**：HNSW（`vector_cosine_ops`），建表后执行 `analyze`。
- **事务**：同一文档的分块写入在单事务内完成；失败整文档回滚并置 `failed`，避免半入库状态。

## 7. 检索（Retriever）

```ts
interface RetrieveOptions {
  query: string
  kbIds: string[]        // 会话关联的知识库
  topK?: number          // 默认 5
  minScore?: number      // 默认 0.35（余弦相似度阈值，可调）
}

async function retrieve(opts): Promise<RetrievedChunk[]>
```

SQL 形态（Drizzle `sql` 片段）：

```sql
SELECT document_id, chunk_index, content, filename,
       1 - (embedding <=> :queryEmbedding) AS score
FROM chunks JOIN documents ON ...
WHERE kb_id = ANY(:kbIds)
ORDER BY embedding <=> :queryEmbedding
LIMIT :topK;
```

**设计决策**：
- 阶段一不做混合检索（向量 + BM25），万级数据下纯向量足够；留 `fulltext` 索引位给阶段二。
- 多知识库检索直接 `kb_id = ANY(...)` 一次查询，无需按库合并。
- 结果按文档去重展示：同一文档命中多块时在 prompt 中标注为多段引用。

## 8. Prompt 注入模板

系统 prompt 动态组装（不落库）：

```
你是知识库问答助手。请严格根据以下【参考资料】回答用户问题。
要求：
1. 仅使用资料中的信息作答，资料未覆盖时明确说明"知识库中未找到相关内容"；
2. 回答中引用信息时标注来源，格式如 〔产品手册.pdf · 第3块〕；
3. 保持简洁准确，可用 Markdown 排版。

【参考资料】
〔产品手册.pdf · 第3块〕
<chunk 内容>
...（其余 chunks）...
```

- 检索为空（或全部低于阈值）→ **不注入**资料段，降级为普通对话，由模型自行回答（前端可提示"未命中知识库"）。
- 上下文预算：参考资料总量占用控制在模型上下文窗口的 30% 以内，超出则截断低分块。

## 9. 引用与溯源展示

- `POST /api/chat` 的 SSE `message-end` 事件扩展 `citations` 字段：

```json
{ "usage": {...}, "finishReason": "stop",
  "citations": [{ "documentId": "uuid", "filename": "产品手册.pdf", "chunkIndex": 3, "score": 0.87 }] }
```

- 前端在 assistant 消息底部渲染引用卡片列表；全文检索接口 `/api/kb/search` 返回同样结构，便于调试。

## 10. 检索质量调优（为什么这么设计）

RAG 效果差的排查顺序：**召回 → 注入 → 生成**。本项目把第一环做成可调试：

| 手段 | 说明 |
|---|---|
| `/api/kb/search` 调试接口 | 直接看 query 命中的 chunks 与分数 |
| 分块可视化 | 文档详情页可查看每块内容，检查切分是否合理 |
| 可调参数 | topK、minScore、分块大小做成知识库级配置 |
| 典型调优路径 | 召回差 → 调分块大小/重叠 → 换 embedding 模型 → （阶段二）上混合检索 |

## 11. 已知限制与后续演进

| 限制 | 演进方向 |
|---|---|
| 扫描件/图片型 PDF 不支持 | 接 `tesseract.js` 或云端 OCR |
| 纯向量检索，对精确术语/编号类 query 弱 | 混合检索（pgvector + tsvector BM25，RRF 融合） |
| 表格类内容解析质量一般 | 表格转 Markdown 的专用解析 |
| 无重排序（rerank） | 接 bge-reranker 类模型做精排 |
| embedding 模型更换需重建向量 | 提供"重建索引"后台任务（重新 embed 全部 chunks） |
