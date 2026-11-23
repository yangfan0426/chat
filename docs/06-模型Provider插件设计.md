# 06 · 模型 Provider 插件设计（适配层）

> 本文档面向的是一类插件：**模型 Provider 插件**。它实现内核 `llm` 服务的具体分发目标，是"底座 + 插件"架构下的第一个插件类型。插件契约本身见 [09-插件体系设计](09-插件体系设计.md)。

## 1. 目标与职责

把不同厂商（OpenAI / Anthropic / Google / DeepSeek / Kimi / 通义 / Ollama…）的 SDK 差异收敛为**对内核统一的服务接口**，业务层（core-chat 插件）只面向 `llm` 服务编程：

1. 统一的流式对话入口（SSE 转发）
2. 统一的模型能力声明与校验
3. 配置经内核 `config` 服务（密钥加密落库）
4. API Key 只在服务端内存中可见

## 2. 核心抽象

### 2.1 统一接口（TypeScript 签名）

```ts
interface ChatProvider {
  /** 流式对话：产出增量文本，回调式消费 */
  stream(req: StreamRequest, handlers: StreamHandlers, signal: AbortSignal): Promise<StreamResult>
}

interface StreamRequest {
  model: ModelRef              // { provider: string; modelId: string }
  messages: ChatMessage[]      // [{ role, content, attachments? }]
  systemPrompt?: string
  maxTokens?: number
  temperature?: number
}

interface StreamHandlers {
  onStart(meta: { messageId: string }): void
  onDelta(delta: string): void
  onEnd(result: { usage: Usage; finishReason: FinishReason }): void
  onError(err: ProviderError): void
}

type FinishReason = 'stop' | 'aborted' | 'error' | 'length'
```

### 2.2 能力声明（Capability Registry）

每个模型在 `models` 表中声明能力，适配层据此做**请求前校验**：

| 能力 | 字段 | 校验行为 |
|---|---|---|
| 上下文窗口 | `contextWindow` | 裁剪历史时以此为上限 |
| 多模态 | `supportsVision` | 请求含图片而模型不支持 → 422 `CAPABILITY_UNSUPPORTED` |
| 输出上限 | `maxOutputTokens`（可选） | 透传给 SDK |
| streaming | 恒 true | 不支持流式的模型不接（本项目硬性要求） |

前端 `GET /api/models` 拿到能力标注后，联动 UI：不支持 vision 时隐藏/禁用图片上传；超长的会话给出"已自动裁剪历史"提示。

### 2.3 实现方式：每个厂商一个插件

Provider 插件在 apply 时向内核注册命名空间服务 `llm:provider/<name>`，并向 `models` 聚合服务贡献模型清单：

```ts
export default {
  manifest: {
    name: 'plugin-deepseek',
    using: ['config', 'events'],
    provides: ['llm:provider/deepseek'],
    config: [ /* apiKey(baseUrl, models 清单等，见 09 文档 §2 */ ],
  },
  apply(ctx) {
    // 经 config 服务读取 apiKey（服务端解密），构造 SDK 客户端
    // 注册服务：ctx.provide('llm:provider/deepseek', { stream, embed, listModels })
    return () => { /* dispose：关闭连接等 */ }
  }
}
```

- **按协议而非厂商选择实现**：绝大多数国产模型兼容 OpenAI 协议，插件内部复用 `createOpenAI({ baseURL, apiKey, compatibility: 'compatible' })`，这是多模型聚合能低成本实现的根本原因。
- 同一个 OpenAI 兼容实现可被多个厂商插件复用（共享代码包），每个厂商插件只负责自己的配置项与模型清单。

## 3. 配置管理

### 3.1 存储与加密

- 插件配置经内核 `config` 服务落库，apiKey 等 secret 类型字段**加密存储**（AES-256-GCM，主密钥来自环境变量 `APP_SECRET`）。
- 对外接口一律脱敏（掩码 `sk-****abcd`）；更新时 key 字段留空 = 不修改。
- `models` 聚合服务对所有 provider 插件的模型清单做汇总，前端选择器数据来源于此。

### 3.2 启动时缓存

- 各 provider 插件在 apply 时注册服务实现并声明模型清单；`llm` 内核服务维护 `modelId → provider 插件` 的路由表，聊天请求按会话所选模型分发。
- 插件配置变更（`plugin/settings-changed` 事件）→ 相关插件重建客户端，路由表同步更新，避免每次请求打库。

### 3.3 模型清单来源

两种模式并存：
- **手动模式**：用户在设置页自行填写模型 id/能力（M2 实现，默认）。
- **自动发现**（可选增强）：对支持 `/models` 端点的 OpenAI 兼容服务，一键拉取模型列表后人工勾选入库。

## 4. 错误处理与重试策略

| 场景 | 处理 |
|---|---|
| 401/403（key 无效） | 包装为 `PROVIDER_AUTH_ERROR`，提示检查 key；SSE 发送 `message-error` 事件 |
| 429（限流） | 不自动重试（流式场景重试语义复杂），提示稍后重试；记录日志 |
| 5xx / 网络错误 | 包装为 `UPSTREAM_ERROR`（502）；SSE 通知前端 |
| 上下文超长（400 context_length） | 识别后自动二次尝试：加大历史裁剪力度后重发一次，仍失败则提示 |
| 客户端中断 | 传播 AbortSignal 取消上游；已生成部分落库 `finishReason: aborted` |

所有上游错误统一映射为 `ProviderError { kind, status?, retryable, message }`，业务层据此决定 SSE 事件与前端提示文案。

## 5. 多模态消息映射

统一内部消息格式：

```ts
type ChatMessage = {
  role: 'user' | 'assistant'
  content: string
  attachments?: { type: 'image'; url: string }[]
}
```

- OpenAI 兼容协议：`content` 转 `[{type:'text'}, {type:'image_url', image_url:{url}}]`
- Anthropic：`content` 转 `[{type:'text'}, {type:'image', source:{type:'base64'...}}]`
- Google：`parts: [{text}, {inlineData}]`

图片统一以**服务端文件 URL** 为内部表示，各 provider 适配器负责转成自身协议（Anthropic 需要 base64 → 适配层内拉取转换）。

## 6. 演进预留

- **Embedding 走同一插件服务**：provider 插件同时实现 `embed(texts): Promise<number[][]>`，OpenAI 兼容协议同样覆盖 embedding；知识库插件经内核 `llm/embed` 调用，不感知厂商。
- **Tool calling 预留**：`StreamRequest.tools` 字段预留但阶段一不实现；`ai` SDK 协议层已支持，加功能时不动接口。
- **本地模型（Ollama）**：即一个 `openai-compatible` provider 插件，baseURL 指向 `http://localhost:11434/v1`，零代码接入——只需在文档中给出配置示例。
