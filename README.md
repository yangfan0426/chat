# Chat — 底座 + 插件架构的 AI 聊天平台

一个 **"底座 + 插件"架构**的 AI 聊天平台：底座只提供插件内核（Cordis）、会话管理、聊天主循环与事件总线等最小核心；模型接入（Provider 插件）、RAG 知识库（knowledge-base 插件）、多模态等能力全部以插件形式挂载，通过声明式配置组合。**替换能力 = 换插件，而不是改代码。**

架构范式参考 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh，"一切皆插件"），聚焦聊天场景，形态更轻。

> 当前状态：技术设计阶段，尚未开始编码。所有设计决策记录在 `docs/` 目录下。

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/01-需求分析与产品定位.md](docs/01-需求分析与产品定位.md) | 产品定位（底座+插件）、插件粒度、功能需求与优先级 |
| [docs/02-技术选型报告.md](docs/02-技术选型报告.md) | 各层技术选型对比：Cordis 插件内核、Fastify、pgvector、Drizzle |
| [docs/03-系统架构设计.md](docs/03-系统架构设计.md) | 内核+插件总体架构、公共服务、管线、关键流程 |
| [docs/04-API-接口设计.md](docs/04-API-接口设计.md) | REST + SSE 接口契约、错误码、约定 |
| [docs/05-数据模型设计.md](docs/05-数据模型设计.md) | 表结构（按插件命名空间归属）、HNSW 索引、状态机 |
| [docs/06-模型Provider插件设计.md](docs/06-模型Provider插件设计.md) | Provider 插件：统一服务接口、能力声明、错误映射、多模态 |
| [docs/07-RAG知识库设计.md](docs/07-RAG-知识库设计.md) | knowledge-base 插件：解析、分块、检索、Prompt 注入与调优 |
| [docs/08-开发路线图与验收标准.md](docs/08-开发路线图与验收标准.md) | M0~M4 里程碑任务、验收清单、风险登记 |
| [docs/09-插件体系设计.md](docs/09-插件体系设计.md) | 插件契约：manifest 规范、生命周期、Service/Event、管线扩展点 |

## 技术栈速览

- **架构范式**：底座 + 插件（一切皆插件，参考 dsh）
- **插件内核**：Cordis（依赖注入 + 生命周期管理）
- **前端**：React 18 + Vite + TypeScript + Tailwind CSS + shadcn/ui
- **后端**：Node.js 22+ / Fastify（HTTP 适配层）+ Vercel AI SDK（Provider 插件内部使用）
- **数据库**：PostgreSQL + pgvector
- **ORM**：Drizzle
- **文件存储**：本地磁盘（预留 S3/MinIO 驱动插件）

## 仓库结构（规划）

```
apps/web          # 前端 SPA
apps/server       # Fastify + Cordis 启动入口与插件组合配置
packages/kernel   # 内核契约：Service/Event 接口、manifest 类型
packages/plugin-sdk  # 插件开发 SDK
packages/shared   # DTO 类型
plugins/          # core-chat、core-storage-pg、plugin-openai、plugin-knowledge-base ...
```

## 快速开始

（待 M1 完成后补充）
