# 后端技术研究

Go 后端相关的技术调研报告（独立文档，可直接消费，不依赖项目上下文）。

| 文档 | 主题 | 对应路线图落点 |
|---|---|---|
| [go-sqlite-fts5-vector-search-research.md](./go-sqlite-fts5-vector-search-research.md) | SQLite + FTS5 全文 + 向量搜索（含中文分词分析、RRF 融合） | Phase 1 存储 ★核心 |
| [go-multi-provider-llm-research.md](./go-multi-provider-llm-research.md) | 多供应商 LLM：统一抽象、Key 轮询、断路器、错误矩阵 | Phase 2 LLM 层 |
| [go-agent-frameworks-research.md](./go-agent-frameworks-research.md) | Go Agent 框架：Eino / LangChainGo / ADK 等 | Phase 2 Agent 引擎 |
| [go-sse-http-api-best-practices.md](./go-sse-http-api-best-practices.md) | SSE 流式 + chi HTTP API 最佳实践 | Phase 1-2 API 层 |
| [go-self-update-research.md](./go-self-update-research.md) | Go 二进制自升级方案对比 | Phase 5 自升级 |
| [agent_loop_research.md](./agent_loop_research.md) | AgentLoop 设计模式、缺陷与上下文管理 | Phase 2 AgentLoop |
| [ai-agent-frameworks-research.md](./ai-agent-frameworks-research.md) | Python 生态框架：LangChain/AutoGPT/CrewAI/Swarm | Phase 2 借鉴 |
| [ai-agent-memory-systems-research.md](./ai-agent-memory-systems-research.md) | 记忆系统：Mem0/Letta/Graphiti/RAG 最佳实践 | Phase 1 记忆分层 |

> 全部为互联网调研沉淀，建议结合 `../02_开发参考.md` 使用。
