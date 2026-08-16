# AI Agent 记忆系统与知识库方案调研报告

> 调研日期: 2025-07-11
> 调研目标: 为 minibox 记忆系统设计提供参考，分析当前最受关注的 AI Agent 记忆方案

---

## 一、Mem0 (mem0ai/mem0)

### 基本信息
| 项目 | 数据 |
|------|------|
| GitHub | https://github.com/mem0ai/mem0 |
| Stars | **62.2k** |
| Forks | 7.3k |
| 语言 | Python 49.4%, TypeScript 48% |
| License | Apache 2.0 |
| 论文 | arXiv:2504.19413 (2025) |

### 记忆分层设计

Mem0 将记忆分为 **四层 + 短期/长期两个维度**：

#### 四层存储架构
1. **Conversation Memory（对话记忆）**: 单轮对话中的即时消息，类似"便签纸"，工具调用输出和中间计算。生命周期：单次响应。
2. **Session Memory（会话记忆）**: 当前任务或频道相关的短期事实。生命周期：分钟到小时。用 `run_id` 标识，可自动过期。
3. **User Memory（用户记忆）**: 与用户/账户绑定的长期知识。生命周期：周到永久。用 `user_id` 标识。
4. **Organizational Memory（组织记忆）**: 跨 agent/团队共享的上下文。生命周期：全局配置。

#### 短期 vs 长期记忆映射
| 类别 | 包含内容 |
|------|----------|
| **短期记忆** | 对话历史（近期轮次有序排列）、工作记忆（工具输出/中间计算）、注意力上下文（即时焦点） |
| **长期记忆** | 事实记忆（用户偏好/账户详情/领域事实）、情景记忆（过去交互摘要）、语义记忆（概念间关系） |

#### 工作流程
1. **Capture（捕获）**: 消息进入对话层
2. **Promote（提升）**: 相关细节基于 `user_id`、`run_id`、metadata 持久化到 session 或 user 层
3. **Retrieve（检索）**: 搜索管道从所有层拉取，**优先排序：user memories → session notes → raw history**

### 记忆算法（2026年4月新版）

核心创新点：
1. **Single-pass ADD-only extraction**: 单次 LLM 调用完成记忆提取，无 UPDATE/DELETE 操作。记忆只增不改。
2. **Agent-generated facts as first-class**: Agent 确认动作时产生的信息与用户信息同等权重存储。
3. **Entity linking（实体链接）**: 提取实体 → 嵌入 → 跨记忆链接，用于检索增强。
4. **Multi-signal retrieval（多信号检索）**: 语义搜索 + BM25 关键词匹配 + 实体匹配，**三路并行打分后融合**。
5. **Temporal Reasoning（时序推理）**: 时间感知检索，对"当前状态"、"过去事件"、"未来计划"查询能排序到正确的时间实例。

### 向量化策略
- 默认嵌入模型: `text-embedding-3-small` (OpenAI)
- 推荐: Qwen 600M 或同等嵌入模型（用于 hybrid search 最佳效果）
- 向量数据库: 默认使用 **Qdrant**
- 实体也单独嵌入并链接

### 搜索算法
- **Hybrid Search**: 语义向量搜索 + BM25 关键词匹配 + 实体匹配
- 三路并行打分，结果融合排序
- 单次检索（single-pass），top_200 检索预算
- 时序感知排序

### Benchmark 表现
| Benchmark | 旧算法 | 新算法 | Token 消耗 | 延迟 p50 |
|-----------|--------|--------|-----------|----------|
| LoCoMo | 71.4 | **92.5** | 7.0K | 0.88s |
| LongMemEval | 67.8 | **94.4** | 6.8K | 1.09s |
| BEAM (1M) | - | 64.1 | 6.7K | 1.00s |
| BEAM (10M) | - | 48.6 | 6.9K | 1.05s |

### ⭐ 最值得 minibox 借鉴的点
1. **分层记忆 + 自动过期**: conversation/session/user/org 四层模型，run_id 自动过期机制轻量且实用
2. **Single-pass 提取**: 单次 LLM 调用提取记忆，避免多轮 agentic loop，显著降低延迟和成本
3. **三路并行检索融合**: semantic + BM25 + entity 三路并行打分后融合，比纯向量搜索更准确
4. **实体链接增强检索**: 提取并嵌入实体，跨记忆链接，检索时 boost 关联记忆
5. **时序推理**: 时间感知排序，处理"现在是什么"vs"过去是什么"的查询

---

## 二、Letta（前 MemGPT）

### 基本信息
| 项目 | 数据 |
|------|------|
| GitHub | https://github.com/letta-ai/letta |
| Stars | **24.0k** |
| Forks | 2.6k |
| 语言 | Python 99.5% |
| License | Apache 2.0 |
| 文档 | docs.letta.com |

### 记忆分层设计

Letta 的记忆系统分为 **五个层级**，按数据规模和重要性递增：

#### 1. Core Memory（核心记忆 / Memory Blocks）
- **性质**: 始终在 context window 中，无需检索
- **实现**: 以 XML 格式 prepend 到 agent prompt
- **结构**: 每个 block 有 label、description、value、limit
- **默认 blocks**: persona（agent 人格）、human（用户信息）
- **关键特性**:
  - Agent-managed: Agent 自主读写（通过 `memory_rethink`、`memory_replace`、`memory_insert` 工具）
  - Flexible: 可用于知识、指南、状态追踪、草稿空间
  - Shareable: 多个 agent 可共享同一 block，更新一次到处可见
  - Always visible: 始终在上下文中
  - Read-only 支持: 可设 `read_only=true` 防止 agent 修改
- **限制**: 单 block 建议 <50k 字符，每 agent 建议 <20 blocks

#### 2. Archival Memory（归档记忆）
- **性质**: 语义可搜索的向量数据库，按需查询
- **关键特性**:
  - Agent-immutable: Agent 不能轻易修改/删除（开发者可通过 SDK 操作）
  - Unlimited storage: 无实际大小限制
  - Semantic search: 按语义而非精确关键词搜索
  - Tagged organization: Agent 可用 tags 分类
- **工具**: `archival_memory_insert`（存储）、`archival_memory_search`（查询）
- **每条记忆**: ~300 tokens
- **适用**: 文档库、对话日志超出 context window、客户交互历史、研究报告

#### 3. Recall Memory（回忆记忆 / Conversation Search）
- **性质**: 搜索实际的历史对话消息
- **特点**: 自动生成，无需 agent 主动策划
- **vs Archival**: Archival 是 agent 主动选择值得长期记住的事实；Recall 是自动的历史消息检索

#### 4. Files（文件系统）
- **性质**: 只读文件，agent 可打开/关闭/搜索
- **工具**: `open`、`close`、`semantic_search`、`grep`
- **限制**: 单文件 5MB，每 agent 建议 <100 文件

#### 5. External RAG
- **性质**: 外部数据库（向量 DB、RAG DB），通过自定义工具或 MCP 访问
- **限制**: 无限

### Context Hierarchy（上下文层级）

| 层级 | 访问方式 | 在上下文中? | 工具 | 大小限制 | 数量限制 |
|------|----------|-------------|------|----------|----------|
| Memory Blocks | 可编辑(可只读) | ✅ 是 | memory_rethink/replace/insert | <50k chars | <20/agent |
| Files | 只读 | 部分 | open/close/semantic_search/grep | 5MB | <100/agent |
| Archival Memory | 读写 | ❌ 否 | archival_memory_insert/search | 300 tokens | 无限 |
| External RAG | 读写 | ❌ 否 | 自定义工具/MCP | 无限 | 无限 |

**选择原则**: 数据量小且重要 → Memory Blocks；数据量大 → 外部存储 + 检索

### 向量化策略
- Archival Memory 使用向量嵌入进行语义搜索
- Files 支持语义搜索 (`semantic_search`) 和精确搜索 (`grep`)
- 具体嵌入模型和向量数据库可配置

### 搜索算法
- **Core Memory**: 无需搜索，始终在上下文
- **Archival Memory**: 语义向量搜索，支持 tag 过滤和分页
- **Recall Memory**: 对话消息搜索
- **Files**: 语义搜索 + grep 精确搜索

### ⭐ 最值得 minibox 借鉴的点
1. **Memory Blocks 的 XML 格式 prepend**: 简单但有效——核心记忆以结构化 XML 直接注入 prompt，无需检索，agent 可自主编辑
2. **五级 context hierarchy**: 按数据规模和重要性选择存储方式，从 in-context 到 external RAG 的清晰递进
3. **Agent 自主记忆管理工具**: `memory_rethink`（重写整个 block）、`memory_replace`（替换片段）、`memory_insert`（插入），让 LLM 自己决定如何维护记忆
4. **Read-only blocks**: 防止 agent 修改重要信息（如公司政策），但保持可见
5. **Shared memory blocks**: 多 agent 共享同一 block，实现跨 agent 协调
6. **Archival vs Recall 区分**: 主动策展的事实 vs 自动历史消息，两者用途不同

---

## 三、Zep / Graphiti

### 基本信息
| 项目 | 数据 |
|------|------|
| Zep GitHub | https://github.com/getzep/zep (4.8k stars, 示例/集成仓库) |
| Graphiti GitHub | https://github.com/getzep/graphiti (**29.4k stars**, 核心引擎) |
| Graphiti Forks | 3.0k |
| 语言 | Python 71.4%, TypeScript 14%, Go 13.1% |
| License | Apache 2.0 |
| 论文 | "Zep: A Temporal Knowledge Graph Architecture for Agent Memory" |

> **注意**: Zep Community Edition 已废弃（移至 legacy/），核心开源引擎为 **Graphiti**。Zep 转型为基于 Graphiti 的托管平台。

### 记忆分层设计

Graphiti 采用 **时态上下文图（Temporal Context Graph）** 而非传统的平坦文档块或原始聊天历史：

#### 上下文图的四个组件
| 组件 | 存储内容 |
|------|----------|
| **Entities（实体/节点）** | 人、产品、策略、概念——摘要随时间演化 |
| **Facts/Relationships（事实/关系/边）** | 三元组 (Entity → Relationship → Entity)，带有**时间有效期窗口** |
| **Episodes（来源/溯源）** | 原始数据流——每个推导出的事实都可溯源到此处 |
| **Custom Types（本体）** | 通过 Pydantic 模型定义的实体和边类型 |

#### 核心创新：时态有效期窗口
- 每个事实都有有效期：何时变为真、何时被取代
- 例: "Kendra 喜欢 Adidas 鞋（截至 2026 年 3 月）"
- 支持增量更新，无需完整图重算
- 可查询"现在什么是真的"和"过去什么是真的"

### 工作原理
1. **持续集成**: 将用户交互、结构化/非结构化企业数据、外部信息持续集成到统一的可查询图中
2. **自主构建**: 从非结构化和结构化数据自主构建上下文图
3. **增量更新**: 处理变化的关系同时保留完整时态历史
4. **精确历史查询**: 不需要完整图重计算

### 向量化策略
- 实体和关系均进行向量化嵌入
- 结合语义嵌入 + 关键词索引 + 图结构

### 搜索算法
- **Hybrid Retrieval（混合检索）**: 语义搜索 + 关键词搜索 + **图遍历（graph traversal）**
- 三种检索维度：时间、语义、关系
- 图遍历可发现跨实体的关联信息

### Zep vs Graphiti
| 维度 | Zep | Graphiti |
|------|-----|----------|
| 定位 | 托管平台，大规模生产部署 | 开源时态上下文图引擎 |
| 数据库 | 专有图数据库 (Context Graph Engine) | 支持 Neo4j 等 |
| 规模 | 百万级上下文图，低延迟 | 单实例 |
| 适用 | 生产环境 | 自建/研究 |

### ⭐ 最值得 minibox 借鉴的点
1. **时态知识图谱**: 事实带有效期窗口，自动处理"事实变更"——比简单的记忆覆盖更优雅
2. **三元组 + 溯源**: Entity → Relationship → Entity，每条事实可追溯到原始数据（Episodes）
3. **图遍历检索**: 除了语义和关键词搜索，还能通过图遍历发现关联信息，这是纯向量搜索做不到的
4. **增量更新无需重算**: 新数据到来时增量更新图，而非重建
5. **Pydantic 本体定义**: 开发者可定义实体和边的类型，约束图谱结构

---

## 四、RAG 最佳实践

### RAG Pipeline 四步流程
1. **Document Preparation & Chunking（文档准备与分块）**
2. **Vector Indexing（向量索引）**
3. **Retrieval（检索）**
4. **Prompt Augmentation（提示增强）**

### Chunking 策略最佳实践

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| **Fixed-size chunking** | 固定 token/字符数分块 | 简单文档，快速原型 |
| **Sentence-aware chunking** | 按句子边界分块 | 保持语义完整性 |
| **Semantic chunking** | 按语义相似度变化点分块 | 高质量检索 |
| **Recursive chunking** | 递归分层分块（如 LangChain RecursiveCharacterTextSplitter） | 通用场景 |
| **Overlap chunking** | 块间保留重叠区域 | 避免边界信息丢失 |
| **Document-aware chunking** | 按文档结构（标题/段落/列表）分块 | 结构化文档 |

**关键原则**:
- Chunk size 需平衡：太小丢失上下文，太大稀释相关性
- 重叠（overlap）通常设 chunk size 的 10-20%
- Mem0 的做法：将对话提取为独立事实条目（~300 tokens/条，参考 Letta archival），而非传统文档分块

### Re-ranking（重排序）最佳实践

| 方法 | 描述 |
|------|------|
| **Cross-encoder re-ranking** | 用 cross-encoder 模型对 query-chunk 对精细打分，精度高但慢 |
| **LLM re-ranking** | 用 LLM 对候选结果重新排序 |
| **Cohere Rerank / BGE Reranker** | 专用重排序模型 API |
| **Multi-signal fusion** | 多路检索结果融合排序（Mem0 的做法：semantic + BM25 + entity） |

**关键原则**:
- 先用快速检索（向量/BM25）获取 top-K 候选（如 top-200）
- 再用重排序模型精排到 top-N（如 top-3~10）
- Mem0: 单次检索 top_200 预算，三路并行融合

### Hybrid Search（混合搜索）最佳实践

| 组件 | 作用 |
|------|------|
| **Dense vector search** | 语义相似性，理解概念和意图 |
| **Sparse/BM25 keyword search** | 精确关键词匹配，专有名词/ID |
| **Entity matching** | 实体级别匹配和链接（Mem0 创新） |
| **Graph traversal** | 关系遍历发现关联信息（Graphiti 创新） |

**融合方法**:
- **RRF (Reciprocal Rank Fusion)**: 各路结果按排名取倒数加权融合
- **Weighted score fusion**: 各路分数加权求和
- **Mem0 方案**: semantic + BM25 + entity 三路并行打分后融合

### ⭐ RAG 最值得 minibox 借鉴的点
1. **多路并行检索 + 融合排序**: 语义 + 关键词 + 实体三路并行，覆盖不同匹配场景
2. **两阶段检索**: 先粗排（top-200）再精排（重排序），平衡速度和精度
3. **记忆粒度优化**: 将对话提取为独立事实条目而非文档分块，更适合 agent 记忆场景
4. **实体增强检索**: 提取实体并嵌入，检索时 boost 关联记忆

---

## 五、综合对比与 minibox 借鉴建议

### 综合对比表

| 维度 | Mem0 | Letta | Zep/Graphiti |
|------|------|-------|-------------|
| **GitHub Stars** | 62.2k | 24.0k | 4.8k / 29.4k |
| **记忆模型** | 四层分层 + 短期/长期 | 五级 context hierarchy | 时态知识图谱 |
| **核心创新** | Single-pass 提取 + 三路融合检索 | Memory Blocks XML + Agent 自主管理 | 时态有效期窗口 + 图遍历 |
| **向量化** | text-embedding-3-small + Qdrant | 可配置向量 DB | 实体+关系嵌入 |
| **搜索算法** | Semantic + BM25 + Entity 融合 | 语义搜索 + grep | Semantic + Keyword + Graph traversal |
| **时序处理** | Temporal Reasoning | 无（block 是全量替换） | 时态有效期窗口 |
| **Agent 自主性** | 自动提取（LLM 驱动） | Agent 自主读写 memory blocks | 自主构建图谱 |
| **多 agent 共享** | Org memory 层 | Shared memory blocks | 图谱共享 |
| **延迟 p50** | ~0.88-1.09s | - | - |
| **Token 效率** | 6.7-7.0K tokens | 取决于 block 大小 | - |

### 对 minibox 记忆系统的核心借鉴建议

#### 1. 记忆分层架构（融合 Mem0 + Letta）
```
Layer 0: Context Memory（上下文记忆）
  - 始终在 prompt 中，XML 格式注入（借鉴 Letta Memory Blocks）
  - persona / human / task_state / scratchpad blocks
  - Agent 可自主编辑，支持 read-only
  - 限制: <50k chars, <20 blocks

Layer 1: Session Memory（会话记忆）
  - 当前任务相关的短期事实（借鉴 Mem0 Session Memory）
  - run_id 标识，可自动过期
  - 生命周期: 分钟到小时

Layer 2: Long-term Memory（长期记忆）
  - 用户偏好、事实、情景记忆（借鉴 Mem0 User Memory）
  - user_id 标识，持久化
  - 向量存储 + 语义搜索

Layer 3: Knowledge Graph（知识图谱，可选）
  - 实体-关系-事实三元组 + 时态窗口（借鉴 Graphiti）
  - 图遍历检索发现关联
  - 增量更新
```

#### 2. 记忆提取策略（借鉴 Mem0）
- **Single-pass ADD-only**: 单次 LLM 调用提取记忆，避免多轮 loop
- **实体链接**: 提取实体 → 嵌入 → 跨记忆链接
- **Agent 事实一等公民**: Agent 确认的动作信息同等存储

#### 3. 检索策略（融合三者 + RAG 最佳实践）
```
查询 → 三路并行检索:
  ├─ Semantic Search（向量语义搜索）
  ├─ BM25 Keyword Search（关键词精确匹配）
  └─ Entity Matching（实体匹配 + 链接 boost）
      ↓
  多路融合排序（RRF 或加权融合）
      ↓
  Top-K 候选（如 top-50）
      ↓
  时序感知重排序（借鉴 Mem0 Temporal Reasoning）
      ↓
  最终 Top-N 结果（如 top-3~5）
```

#### 4. 时序处理（借鉴 Graphiti）
- 为每条记忆添加时间戳和有效期
- 支持查询"当前状态"vs"历史状态"
- 事实变更时标记旧事实为"已过期"而非删除

#### 5. Agent 自主记忆管理（借鉴 Letta）
- 提供 `memory_insert`、`memory_replace`、`memory_rethink` 工具
- Agent 自主决定记忆的写入、更新、重组
- Block description 引导 Agent 正确使用记忆区域

#### 6. 实现优先级建议
| 优先级 | 功能 | 借鉴来源 | 复杂度 |
|--------|------|----------|--------|
| P0 | 分层记忆（context + session + long-term） | Mem0 + Letta | 中 |
| P0 | 向量语义搜索 | 通用 RAG | 低 |
| P0 | LLM 驱动的记忆提取 | Mem0 | 中 |
| P1 | BM25 + 语义混合搜索 | Mem0 + RAG | 中 |
| P1 | 实体提取与链接 | Mem0 | 中 |
| P1 | Agent 自主记忆工具 | Letta | 中 |
| P2 | 时序感知检索 | Mem0 + Graphiti | 高 |
| P2 | 多路融合排序 + 重排序 | RAG | 中 |
| P3 | 知识图谱 + 图遍历 | Graphiti | 高 |
| P3 | 多 agent 共享记忆 | Letta + Mem0 | 中 |

---

## 六、关键参考链接

| 资源 | URL |
|------|-----|
| Mem0 GitHub | https://github.com/mem0ai/mem0 |
| Mem0 文档 | https://docs.mem0.ai |
| Mem0 论文 | https://arxiv.org/abs/2504.19413 |
| Letta GitHub | https://github.com/letta-ai/letta |
| Letta 文档 | https://docs.letta.com |
| Letta Memory Blocks | https://docs.letta.com/v1-sdk/memory/memory-blocks |
| Letta Archival Memory | https://docs.letta.com/v1-sdk/memory/archival-memory |
| Letta Context Hierarchy | https://docs.letta.com/v1-sdk/memory/context-hierarchy |
| Zep GitHub | https://github.com/getzep/zep |
| Graphiti GitHub | https://github.com/getzep/graphiti |
| Zep 文档 | https://help.getzep.com |
