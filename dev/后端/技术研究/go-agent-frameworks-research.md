# Go 语言 Agent 框架与智能体架构调研报告

> 调研时间：2025年7月
> 来源：GitHub 仓库 + 技术博客综合整理

---

## 一、LangChainGo（Go 版 LangChain）

### 基本信息
| 项目 | 值 |
|------|------|
| GitHub | https://github.com/tmc/langchaingo |
| Stars | 9.6k |
| Forks | 1.1k |
| License | MIT |
| 最新版本 | v0.1.14 |
| Go 版本 | 1.18+ |

### 核心架构模式

**1. Chain（链）组合模式** — 核心设计理念
- 将 LLM 调用、提示词模板、输出解析等组件像 LEGO 积木一样链式组合
- 支持 Sequential Chains（顺序链）、Router Chains（路由链/条件分支）、Map-Reduce Chains（并行处理）
- 类型安全，可测试，可扩展

**2. Agent Executor（代理执行器）— AgentLoop 模式**
```
agents/
├── agents.go              # 核心 Agent 接口定义
├── executor.go            # AgentExecutor — AgentLoop 核心循环
├── conversational.go      # 对话型 Agent
├── mrkl.go                # ReAct (MRKL) Agent — 推理+行动模式
├── mrkl_prompt.go         # ReAct 提示词模板
├── openai_functions_agent.go  # OpenAI Function Calling Agent
├── options.go             # 配置选项（MaxIterations 等）
├── errors.go              # 错误定义
├── initialize.go          # 初始化逻辑
└── prompts/               # 提示词模板目录
```
- **AgentLoop**: `Executor` 内部循环执行：思考 → 选择工具 → 执行工具 → 观察结果 → 继续思考，直到得出最终答案或达到最大迭代次数
- **ReAct 模式**: Reasoning + Acting，通过 Prompt 引导 LLM 输出 Thought/Action/Observation
- **多种 Agent 类型**: ConversationalAgent（对话型）、MRKL/ReAct Agent（推理行动型）、OpenAI Functions Agent（函数调用型）

**3. Callback 机制**
```
callbacks/
├── callbacks.go    # 回调处理器接口定义
├── log.go          # 日志回调实现
└── simple.go       # 简单回调实现
```
- 在 Chain/LLM/Agent 的关键节点触发回调
- 支持自定义回调处理器用于日志、追踪、指标收集

**4. 工具（Tool）注册**
- 通过 `tools.Tool` 接口定义工具
- Agent 创建时传入工具列表：`agents.NewExecutor(agent, tools, agents.WithMaxIterations(5))`
- 支持 Search、Calculator、自定义 API 工具等

### 代码组织方式
```
langchaingo/
├── agents/           # Agent 实现（ReAct, Conversational, OpenAI Functions）
├── callbacks/        # 回调处理器
├── chains/           # Chain 组合（LLMChain, SequentialChain, MapReduceChain 等）
├── documentloaders/  # 文档加载器（PDF, CSV, Web 等）
├── embeddings/       # 向量嵌入（OpenAI, HuggingFace, Ollama 等）
├── llms/             # LLM 提供者（OpenAI, Anthropic, Ollama, Cohere, Mistral 等 10+）
├── memory/           # 对话记忆（Buffer, Summary, Entity Memory）
├── outputparsers/    # 输出解析器
├── schema/           # 核心类型定义（ChatMessage, PromptTemplate 等）
├── textsplitter/     # 文本分割
├── tools/            # 工具集（Search, Calculator 等）
├── vectorstores/     # 向量存储（Pinecone, Chroma, Weaviate, Qdrant 等）
├── exp/              # 实验性功能
└── httputil/         # HTTP 工具
```

### 值得借鉴的设计
1. **Chain 组合模式**: 将复杂 AI 流程拆解为可复用的链式组件，类型安全且可测试
2. **Agent Executor 循环**: 通过 Executor 封装 AgentLoop，支持 MaxIterations 限制、错误处理
3. **多 LLM 提供者抽象**: 统一接口，切换模型只需改配置
4. **丰富的生态**: 向量存储、文档加载器、记忆管理等模块化设计

---

## 二、CloudWeGo Eino（字节跳动 Go AI 框架）

### 基本信息
| 项目 | 值 |
|------|------|
| GitHub | https://github.com/cloudwego/eino |
| Stars | 12.6k |
| Forks | 1.0k |
| License | Apache-2.0 |
| 最新版本 | v0.9.13 |
| Go 版本 | 1.18+ |
| 所属组织 | ByteDance / CloudWeGo |

### 核心架构模式

**1. 组件抽象（Component Abstraction）— 核心设计理念**
- 定义可复用的组件接口：`ChatModel`, `Tool`, `Retriever`, `Embedding`, `ChatTemplate`
- 官方实现支持 OpenAI, Claude, Gemini, Ark, Ollama, Elasticsearch 等
- 组件实现在独立的 `eino-ext` 仓库中

**2. ADK（Agent Development Kit）— Agent 架构**
```
adk/
├── ChatModelAgent    # 基础 Agent — 内置 ReAct 循环
├── DeepAgent         # 深度 Agent — 多子代理协调
├── Runner            # 执行器 — 运行 Agent 并返回事件流
└── ToolsConfig       # 工具配置
```

- **ChatModelAgent**: 配置 ChatModel + Tools 即可获得工作 Agent，内部自动处理 ReAct 循环
```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model: chatModel,
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{weatherTool, calculatorTool},
        },
    },
})
runner := adk.NewRunner(ctx, adk.RunnerConfig{Agent: agent})
iter := runner.Query(ctx, "Hello, who are you?")
// 通过迭代器模式获取事件流
for {
    event, ok := iter.Next()
    if !ok { break }
    fmt.Println(event.Message.Content)
}
```

- **DeepAgent**: 复杂任务分解，委托子代理，追踪进度
```go
deepAgent, _ := deep.New(ctx, &deep.Config{
    ChatModel: chatModel,
    SubAgents: []adk.Agent{researchAgent, codeAgent},
    ToolsConfig: ...,
})
```

**3. Callback Aspects（回调切面）— 独特设计**
```
callbacks/
├── callbacks.go    # 回调接口定义
├── handler.go      # 回调处理器
└── internal/       # 内部实现
```
- 在固定切点注入日志、追踪、指标：
  - `OnStart` — 组件/图/Agent 开始执行
  - `OnEnd` — 正常结束
  - `OnError` — 发生错误
  - `OnStartWithStreamInput` — 流式输入开始
  - `OnEndWithStreamOutput` — 流式输出结束
- **切面（Aspect）模式**: 跨组件、图、Agent 统一注入横切关注点，无需修改业务逻辑

**4. Graph 编排（Composition）— DAG 工作流**
```go
graph := compose.NewGraph[*Input, *Output]()
graph.AddLambdaNode("validate", validateFn)
graph.AddChatModelNode("generate", chatModel)
graph.AddLambdaNode("format", formatFn)
graph.AddEdge(compose.START, "validate")
graph.AddEdge("validate", "generate")
graph.AddEdge("generate", "format")
graph.AddEdge("format", compose.END)
runnable, _ := graph.Compile(ctx)
result, _ := runnable.Invoke(ctx, input)
```
- 支持将 Graph 编译为可执行单元
- **Graph 可暴露为 Tool**: 桥接确定性工作流与自主 Agent 行为
  ```go
  tool, _ := graphtool.NewInvokableGraphTool(graph, "data_pipeline", "Process and validate data")
  ```

**5. 流处理（Stream Processing）— 自动流编排**
- 在编排过程中自动处理流的拼接（concatenate）、装箱（box）、合并（merge）、复制（copy）
- 组件只需实现适合自身的流式范式，框架处理其余部分

**6. Interrupt/Resume（中断/恢复）— Human-in-the-Loop**
- 任何 Agent 或 Tool 可暂停执行等待人工输入
- 从检查点（checkpoint）恢复执行
- 框架处理状态持久化和路由

### 代码组织方式
```
eino/
├── adk/            # Agent Development Kit (ChatModelAgent, DeepAgent, Runner)
├── callbacks/      # 回调切面 (Callback Aspects)
├── components/     # 组件抽象 (ChatModel, Tool, Retriever, Embedding 等)
├── compose/        # 图编排 (Graph, Chain, Lambda, ToolsNode 等)
├── ext/            # 扩展 (GraphTool 等)
├── flow/           # 流程控制 (ReActAgent 等)
├── schema/         # 核心类型定义
├── internal/       # 内部实现
└── examples/       # 示例代码

# 配套仓库：
# eino-ext      — 组件实现、回调处理器、示例、评估器、提示词优化器
# eino-devops   — 可视化开发和调试
# eino-examples — 示例应用和最佳实践
```

### 值得借鉴的设计
1. **Callback Aspects 切面模式**: 通过固定切点统一注入横切关注点，比传统回调更优雅
2. **Graph → Tool 桥接**: 确定性工作流可暴露为 Agent 可调用的工具，实现确定性+自主性的融合
3. **自动流处理**: 框架自动处理流式数据的拼接/合并/复制，降低开发者心智负担
4. **Interrupt/Resume + Checkpoint**: 原生支持 Human-in-the-Loop，状态持久化由框架处理
5. **Runner + Iterator 模式**: Agent 执行返回事件迭代器，天然支持流式输出
6. **DeepAgent 多代理协调**: 内置多子代理任务分解和协调能力
7. **企业级可靠性**: 内置断路器、指数退避重试、超时、批量隔离

---

## 三、其他流行 Go Agent 框架

### 3.1 Google ADK Go（Google Agent Development Kit）

| 项目 | 值 |
|------|------|
| GitHub | https://github.com/google/adk-go |
| Stars | 8.6k |
| Forks | 872 |
| License | Apache-2.0 |
| 包路径 | `google.golang.org/adk/v2` |

**核心架构模式:**
- **Code-First**: 用 Go 代码直接定义 Agent 逻辑、工具、编排
- **多代理编排**: 父代理可委托任务给专门的子代理，自动处理协调、状态管理、消息路由
- **A2A (Agent2Agent) 协议**: 标准化通信协议，支持代理发现、协商、协作；同步 RPC + 异步消息传递，内置重试和断路器
- **MCP Toolbox 集成**: 连接 30+ 数据库（PostgreSQL, MySQL, MongoDB, Redis, BigQuery 等）

**代码组织:**
```
adk-go/
├── agent/          # Agent 核心定义
├── agentregistry/  # Agent 注册与发现
├── artifact/       # 制品管理
├── auth/           # 认证
├── cmd/            # 命令行工具
├── memory/         # 记忆管理
├── model/          # 模型抽象
├── internal/       # 内部实现
└── examples/       # 示例
```

**值得借鉴:**
- A2A 协议实现多代理标准化通信
- MCP Toolbox 连接丰富数据源
- 与 Google Cloud 深度集成（Vertex AI, Cloud Run）

### 3.2 Agent SDK Go（by Ingenimax）

| 项目 | 值 |
|------|------|
| GitHub | https://github.com/Ingenimax/agent-sdk-go |
| Stars | ~1k+（估算） |
| License | 开源 |

**核心架构模式:**
- **声明式 YAML 配置**: Agent 定义、安全护栏、成本限制作为代码管理
- **多租户架构**: 三级隔离（namespace 逻辑分离 / database 独立数据存储 / cluster 专用基础设施）
- **Guardrails 系统**: 内容过滤（PII 检测、脏话拦截）、行为护栏（防死循环、幻觉检测）、成本控制、合规规则
- **OpenTelemetry 可观测性**: 分布式追踪、Prometheus 指标、结构化日志

**值得借鉴:**
- 声明式 YAML Agent 定义 + GitOps 工作流
- 多租户隔离设计（namespace/database/cluster 三级）
- 内置 Guardrails 安全护栏系统
- 生产级基础设施：缓存层、请求队列、背压管理、K8s 健康检查

### 3.3 Firebase Genkit Go

| 项目 | 值 |
|------|------|
| GitHub | https://github.com/firebase/genkit-go |
| License | 开源 |

**核心架构模式:**
- **Flow（流程）模式**: 将 AI 功能定义为服务器函数（flows），调用模型、工具、检索步骤
- **统一生成 API**: 单一接口跨所有 LLM 提供者
- **结构化输出**: 使用 Go 结构体自动生成 JSON Schema，类型安全
- **函数调用**: Agent 可调用带参数验证的 Go 函数

**值得借鉴:**
- Flow 抽象：将 AI 功能封装为可调试、可追踪的服务器函数
- 开发体验优化：热重载、内置 UI 测试、自动追踪
- 一键部署到 Cloud Run / Firebase Functions

### 3.4 Jetify AI SDK

| 项目 | 值 |
|------|------|
| GitHub | https://github.com/jetify-com/ai |
| License | 开源 |

**核心架构模式:**
- **Provider 抽象层**: 单一 API 契约跨所有 LLM 提供者
- **自动故障转移**: 可配置的故障转移链 + 健康监测 + 断路器
- **延迟路由**: 自动路由到最快的可用提供者
- **类型安全**: 编译时防止无效 API 调用

**值得借鉴:**
- 多 Provider 自动故障转移 + 健康检查 + 延迟路由
- 惯用 Go 设计：context 感知 API、错误包装、接口驱动

### 3.5 Anyi

| 项目 | 值 |
|------|------|
| GitHub | https://github.com/jieliu2000/anyi |
| License | 开源 |

**核心架构模式:**
- **工作流优先架构**: 声明式工作流定义，可视化 DAG 表示
- **业务规则验证**: 字段级、跨字段、业务逻辑规则验证
- **Human-in-the-Loop**: 自动将边缘案例和异常路由给人工审核，审批门控
- **企业集成**: 预置连接器（邮件、ERP、CRM、会计软件）

**值得借鉴:**
- 声明式工作流 + DAG 可视化
- 验证检查点 + 人工审核路由
- 工作流版本管理和回滚

### 3.6 go-openai（OpenAI Go 客户端）

| 项目 | 值 |
|------|------|
| GitHub | https://github.com/sashabaranov/go-openai |
| Stars | 10.7k |
| Forks | 1.7k |
| License | Apache-2.0 |

**说明:** 非完整 Agent 框架，但是最流行的 Go LLM 客户端库，支持 ChatGPT、GPT-4、DALL·E、Whisper、Function Calling、Structured Outputs。被许多 Go Agent 框架作为底层依赖。

---

## 四、框架对比总览

| 框架 | Stars | 核心模式 | Agent Loop | Callback | 多代理 | 流式 | Human-in-Loop | 适用场景 |
|------|-------|---------|------------|----------|--------|------|---------------|---------|
| **Eino** | 12.6k | Component+Graph+ADK | 内置 ReAct | Aspect 切面 | DeepAgent | 自动编排 | ✅ Interrupt/Resume | 大规模生产 |
| **LangChainGo** | 9.6k | Chain 组合 | Executor 循环 | 回调处理器 | Agent 链 | 支持 | ❌ | LLM 应用开发 |
| **Google ADK** | 8.6k | Code-First+A2A | 内置 | — | ✅ 层级编排 | — | — | 企业多代理 |
| **Agent SDK Go** | ~1k+ | 声明式 YAML | 内置 | OpenTelemetry | ✅ 多租户 | — | ✅ 审批门控 | 企业 SaaS |
| **Genkit Go** | — | Flow 流程 | — | 追踪 | 有限 | ✅ | — | 快速原型 |
| **Jetify AI** | — | Provider 抽象 | — | — | 有限 | ✅ | — | 多模型切换 |
| **Anyi** | — | 工作流 DAG | — | — | ✅ | — | ✅ 人工审核 | RPA 自动化 |
| **go-openai** | 10.7k | API 客户端 | N/A | N/A | N/A | ✅ | N/A | LLM 客户端 |

---

## 五、核心设计模式总结与借鉴建议

### 1. AgentLoop（代理循环）
- **LangChainGo**: `Executor` 封装循环，支持 MaxIterations、错误处理
- **Eino**: Runner + Iterator 模式，Agent 内部自动处理 ReAct 循环，返回事件流
- **借鉴**: 循环应有最大迭代限制、错误恢复、以及流式事件输出

### 2. Callback/Aspect（回调/切面）
- **LangChainGo**: 传统回调处理器接口
- **Eino**: Aspect 切面模式 — 固定切点（OnStart/OnEnd/OnError/OnStartWithStreamInput/OnEndWithStreamOutput）
- **借鉴**: Aspect 模式比传统回调更优雅，统一注入横切关注点，适合 Go 接口设计

### 3. Tool 注册与调用
- **LangChainGo**: `tools.Tool` 接口 + 创建 Agent 时传入工具列表
- **Eino**: `tool.BaseTool` 接口 + `ToolsNodeConfig` 配置
- **Eino 独特**: Graph 可暴露为 Tool（GraphTool），桥接确定性工作流和自主 Agent
- **借鉴**: 工具应基于接口定义，支持将复杂工作流封装为工具

### 4. 编排（Orchestration）
- **LangChainGo**: Chain 链式组合（Sequential/Router/MapReduce）
- **Eino**: Graph DAG 编排 + Compile 编译为 Runnable
- **借鉴**: Graph DAG 编排比线性 Chain 更灵活，编译模式支持优化和验证

### 5. 流处理（Streaming）
- **Eino 独特**: 框架自动处理流的拼接/装箱/合并/复制，组件只需实现自身流式范式
- **借鉴**: 将流处理下沉到框架层，降低开发者心智负担

### 6. 多代理协调
- **Eino**: DeepAgent — 任务分解 + 子代理委托 + 进度追踪
- **Google ADK**: 父子代理层级 + A2A 协议标准化通信
- **借鉴**: 多代理需要任务分解、状态管理、消息路由的框架级支持

### 7. 状态持久化与恢复
- **Eino**: Interrupt/Resume + Checkpoint — 任何 Agent/Tool 可暂停等待人工输入
- **Agent SDK Go**: 声明式配置 + 热重载
- **借鉴**: 检查点机制支持 Human-in-the-Loop 和故障恢复

### 8. 生产级可靠性
- **Eino**: 断路器、指数退避、超时、批量隔离、死信队列
- **Agent SDK Go**: 缓存层、请求队列、背压管理、K8s 健康检查
- **Jetify**: 自动故障转移 + 健康检查 + 延迟路由
- **借鉴**: 生产级 Agent 需要全面的可靠性模式，而非简单的重试

---

## 六、推荐架构方向

基于以上调研，一个优秀的 Go Agent 框架应融合：

1. **组件抽象层** (借鉴 Eino): ChatModel/Tool/Retriever/Embedding 统一接口
2. **Graph DAG 编排** (借鉴 Eino compose): 支持条件分支、并行、循环
3. **Callback Aspect 切面** (借鉴 Eino callbacks): 固定切点注入横切关注点
4. **AgentLoop + 流式事件** (借鉴 Eino Runner/Iterator): Agent 执行返回事件迭代器
5. **Graph → Tool 桥接** (借鉴 Eino GraphTool): 确定性工作流可暴露为 Agent 工具
6. **多代理协调** (借鉴 Eino DeepAgent + Google ADK A2A): 任务分解 + 子代理委托
7. **Interrupt/Resume** (借鉴 Eino): Checkpoint 支持人工介入和故障恢复
8. **声明式配置** (借鉴 Agent SDK Go): YAML 定义 Agent 行为和安全护栏
9. **多 Provider 故障转移** (借鉴 Jetify): 自动故障转移 + 健康检查 + 延迟路由
10. **生产级可靠性** (借鉴 Eino + Agent SDK Go): 断路器、背压、缓存、K8s 就绪
