# Python AI Agent 开源框架调研报告

> 数据来源：GitHub API（2025-07-31 实时查询）+ 官方文档 + 架构分析

---

## 一、总览对比

| 项目 | GitHub Stars | Forks | 语言 | License | 定位 |
|------|-------------|-------|------|---------|------|
| **AutoGPT** | 185,746 | 46,058 | Python | Other | 自主任务执行 Agent 平台 |
| **LangChain** | 143,100 | 23,831 | Python/TS | MIT | Agent 工程化平台（全栈框架） |
| **CrewAI** | 56,432 | 8,027 | Python | MIT | 多 Agent 角色协作框架 |
| **OpenAI Swarm** | 21,870 | 2,334 | Python | MIT | 轻量多 Agent 编排（教学性质） |
| **BabyAGI** | ~20,000+ (已归档) | - | Python | MIT | 自主循环 Agent 原型（已停更） |

---

## 二、LangChain —— Agent 工程化平台

### 基本信息
- **仓库**: `langchain-ai/langchain`
- **Stars**: 143,100 | **Forks**: 23,831
- **描述**: "The agent engineering platform."
- **Topics**: agents, ai-agents, langgraph, multiagent, rag, pydantic, python, typescript
- **创建时间**: 2022-10-17

### 核心架构模式

LangChain 经历了从 v0.1 到 v0.3 的架构演进，当前核心架构包含两大体系：

#### 1. AgentExecutor（经典 Agent 执行器，已逐步被 LangGraph 替代）
- **设计模式**: ReAct (Reasoning + Acting) 循环
- **流程**: `LLM 思考 → 选择 Tool → 执行 Tool → 观察结果 → 继续思考 → ... → 最终回答`
- **关键组件**:
  - `Agent`: 决策核心，由 LLM + Prompt + 解析器组成，决定下一步动作
  - `Tools`: 工具集合，每个工具是 `name + description + func` 的三元组
  - `AgentExecutor`: 循环控制器，管理 `intermediate_steps` 列表，控制 `max_iterations`、`early_stopping`
  - `Callbacks`: 全生命周期回调钩子（on_llm_start, on_tool_start, on_tool_end, on_agent_action, on_agent_finish）
- **局限**: 线性循环，难以表达复杂控制流（条件分支、并行、循环回退）

#### 2. LCEL (LangChain Expression Language) —— 声明式链式编排
- **设计模式**: 管道组合（Pipe Composition），类似 Unix 管道
- **核心抽象**: `Runnable` 协议
  - 统一接口: `invoke()`, `batch()`, `stream()`, `ainvoke()`, `abatch()`, `astream()`
  - 可组合: `prompt | model | parser` 用 `|` 管道符连接
  - 内置能力: 自动批处理、流式输出、异步支持、回退（with_fallbacks）、重试（with_retry）
- **关键设计**: 每个 Runnable 都有 `input_schema` 和 `output_schema`（基于 Pydantic），链式组合时自动做类型校验
- **适用场景**: RAG 管道、简单链式流程

#### 3. LangGraph —— 状态图 Agent 编排（当前推荐方案）
- **设计模式**: 有向有环图（State Graph）
- **核心概念**:
  - `State`: 全局状态对象（TypedDict / Pydantic Model），在各节点间传递和合并
  - `Node`: 图节点，每个节点是一个函数 `(State) → State`，负责修改状态
  - `Edge`: 图边，可以是固定边或条件边（conditional_edges）
  - `Reducer`: 状态合并策略，定义如何合并各节点对状态的修改（如 `add_messages`）
- **AgentLoop 实现**: 
  - `agent` 节点调用 LLM 决策
  - `tools` 节点执行工具
  - 条件边 `should_continue` 判断是继续循环还是结束
  - 支持循环、并行节点、子图、人在回路（human-in-the-loop）、持久化检查点（checkpointer）
- **优势**: 可以精确表达任意控制流，包括多 Agent 间复杂交互

### 工具系统设计
```python
# 工具定义方式演进
# 1. 传统方式
@tool
def search(query: str) -> str:
    """Search the web."""
    ...

# 2. StructuredTool + Pydantic Schema
class SearchInput(BaseModel):
    query: str
    max_results: int = 5

search_tool = StructuredTool.from_function(
    func=search,
    name="search",
    description="Search the web",
    args_schema=SearchInput
)

# 3. LangGraph 工具节点
tool_node = ToolNode([search_tool, calculator_tool])
```
- **特点**: 工具基于 Pydantic 做参数校验；支持同步/异步；可通过 `toolkit` 封装工具集合；ToolNode 可并行执行多个工具调用

### 记忆系统设计
- **历史方案**: `ConversationBufferMemory`（全量保存）、`ConversationSummaryMemory`（LLM 摘要）、`ConversationBufferWindowMemory`（滑动窗口）、`ConversationTokenBufferMemory`（Token 限制）
- **当前方案**: LangGraph 中通过 State + Checkpointer 实现
  - 短期记忆: State 中的 `messages` 列表，每轮对话自动累积
  - 长期记忆: 通过 `Store` API（基于 namespace 的键值存储），支持语义搜索
  - 跨会话持久化: `MemorySaver` / `SqliteSaver` / `PostgresSaver` 等 Checkpointer 实现
- **Retriever 设计**: 
  - `BaseRetriever` 接口: `get_relevant_documents(query)` 
  - 支持向量检索（FAISS, Chroma, Pinecone 等）、关键词检索、混合检索
  - 可与 LCEL 管道无缝组合: `retriever | prompt | model | parser`

### 最值得 minibox 借鉴的点
1. **Runnable 统一协议**: 所有组件实现统一接口（invoke/stream/batch/async），这是极好的抽象——minibox 可以定义自己的 `Invocable` 协议
2. **LangGraph 的 State + Reducer 模式**: 状态在节点间传递与合并的设计非常适合多步 Agent 循环
3. **Pydantic 驱动的工具 Schema**: 用 Pydantic Model 自动生成工具描述和参数校验，简洁且类型安全
4. **Checkpointer 持久化**: 通过检查点实现 Agent 状态的持久化和恢复，支持中断续做
5. **Callback 系统**: 全生命周期回调钩子，便于监控、日志、追踪

---

## 三、AutoGPT / BabyAGI —— 自主循环架构

### AutoGPT 基本信息
- **仓库**: `Significant-Gravitas/AutoGPT`
- **Stars**: 185,746 | **Forks**: 46,058
- **描述**: "AutoGPT is the vision of accessible AI for everyone"
- **Topics**: agentic-ai, autonomous-agents, agents, claude, gpt, llm, openai
- **创建时间**: 2023-03-16
- **主页**: https://agpt.co

### BabyAGI 基本信息
- **仓库**: `yoheinakajimi/babyagi`（已归档/迁移）
- **Stars**: ~20,000+（历史数据）
- **描述**: 任务驱动的自主 Agent 原型
- **状态**: 已停止维护，但其架构理念影响深远

### AutoGPT 核心架构模式

#### 自主循环架构（Autonomous Loop）
```
用户目标 → Agent 循环:
  ┌──────────────────────────────────┐
  │ 1. 思考(Reason): LLM 分析当前状态 │
  │ 2. 计划(Plan): 生成/更新任务列表  │
  │ 3. 行动(Act): 选择并执行工具      │
  │ 4. 观察(Observe): 收集执行结果     │
  │ 5. 反思(Reflect): 评估进展        │
  └──────────────────────────────────┘
  循环直到目标完成 或 用户中断 或 资源耗尽
```

#### AutoGPT 架构演进
- **v1 (Classic)**: 单一自主循环，GPT-4 驱动，文件系统 + Web 搜索
  - `Agent` → `Thought` → `Reasoning` → `Plan` → `Command` → `Result`
  - 命令系统: 预定义命令集（web_search, write_file, read_file, execute_python 等）
  - 工作空间: 本地文件系统作为持久化存储
  
- **v2 (Forge)**: 模块化重构
  - `AgentProtocol`: 标准化 Agent HTTP API（创建任务、执行步骤、获取状态）
  - `PromptEngine`: 模板化 Prompt 管理
  - 可插拔的能力系统

- **当前版本 (AutoGPT Platform)**: 
  - 可视化 Agent 构建器 + 服务器架构
  - `graph` 执行引擎: 基于有向图的 Block 编排
  - `Block`: 最小执行单元，有输入/输出 schema
  - 支持 Agent 市场和自定义 Block

#### BabyAGI 核心架构（极简自主循环）
```
1. 任务创建: 根据目标生成初始任务列表
2. 任务优先级排序: LLM 对任务列表重新排序
3. 任务执行: 取第一个任务，使用可用工具执行
4. 结果存储: 将执行结果存入向量数据库
5. 创建新任务: 根据执行结果和目标，LLM 生成新任务
→ 回到步骤 2，循环
```
- **核心创新**: 任务列表 + 向量记忆 的组合，让 Agent 能动态规划
- **极简实现**: 整个核心逻辑仅约 140 行 Python 代码

### 工具系统设计
- **AutoGPT**: 
  - `Command` 注册制: 用 `@command` 装饰器注册，自动生成工具描述
  - 命令约束: 定义 `DISABLED_COMMANDS` 和 `ALLOWLIST_COMMANDS`
  - Agent Protocol: 标准化的 `POST /ap/v1/agent/tasks` 和 `POST /ap/v1/agent/tasks/{id}/steps` API
- **BabyAGI**: 极简——直接在 Prompt 中列举可用工具，LLM 输出 JSON 格式的工具调用

### 记忆系统设计
- **AutoGPT**: 
  - 工作记忆: 当前对话上下文（Thought/Action/Observation 序列）
  - 长期记忆: 文件系统持久化 + 向量数据库（Chroma/Pinecone）做语义检索
  - `Memory` 抽象: `add()`, `get_relevant()`, `get_stats()`, `clear()`
- **BabyAGI**: 
  - 向量数据库（Pinecone/Chroma）存储任务执行结果
  - 每次创建新任务前，检索相关历史结果作为上下文

### 最值得 minibox 借鉴的点
1. **BabyAGI 的极简任务循环**: 5 步循环（创建→排序→执行→存储→生成新任务）是自主 Agent 的最小可行架构
2. **任务列表作为工作记忆**: 用动态任务列表驱动 Agent 行为，而非固定流程
3. **向量记忆 + 任务生成的组合**: 执行结果存入向量库，新任务创建时检索相关历史——这是"经验积累"的雏形
4. **Agent Protocol 标准化 API**: AutoGPT Forge 定义的标准化 Agent HTTP API（task/step 模型）值得参考
5. **AutoGPT Platform 的 Block 图引擎**: 可视化 Block 编排 + 有向图执行，适合复杂工作流

---

## 四、CrewAI —— 多 Agent 角色协作框架

### 基本信息
- **仓库**: `crewAIInc/crewAI`
- **Stars**: 56,432 | **Forks**: 8,027
- **描述**: "Framework for orchestrating role-playing, autonomous AI agents."
- **Topics**: agents, ai-agents, aiagentframework, llms
- **创建时间**: 2023-10-27
- **主页**: https://crewai.com

### 核心架构模式

#### 角色 → 任务 → 编排 三层模型
```
Crew (编排层)
├── Agents (角色定义)
│   ├── Agent(role="Researcher", goal="...", backstory="...", tools=[...])
│   ├── Agent(role="Writer", goal="...", backstory="...", tools=[...])
│   └── Agent(role="Editor", goal="...", backstory="...", tools=[...])
├── Tasks (任务定义)
│   ├── Task(description="...", agent=researcher, expected_output="...")
│   ├── Task(description="...", agent=writer, expected_output="...", context=[task1])
│   └── Task(description="...", agent=editor, expected_output="...")
└── Process (编排策略)
    └── sequential | hierarchical
```

#### 核心概念
- **Agent**: 有角色、目标、背景故事、工具集的自主实体
  - `role`: 定义 Agent 身份（如 "Senior Data Analyst"）
  - `goal`: Agent 的具体目标
  - `backstory`: 给 LLM 提供角色上下文，影响行为风格
  - `llm`: 可为不同 Agent 配置不同模型
  - `tools`: Agent 可用的工具列表
  - `allow_delegation`: 是否允许将任务委托给其他 Agent
  - `max_iter`: 单任务最大迭代次数
  
- **Task**: 可执行的工作单元
  - `description`: 任务描述
  - `agent`: 负责任的 Agent
  - `expected_output`: 期望输出格式描述
  - `context`: 依赖的其他任务（其输出作为本任务上下文）
  - `output_file`: 可选输出文件路径
  - `async_execution`: 是否异步执行

- **Crew**: 编排器
  - `agents`: Agent 列表
  - `tasks`: 任务列表
  - `process`: 编排策略
    - `sequential`: 按任务列表顺序执行，前一个的输出作为后一个的输入
    - `hierarchical`: 有管理 Agent 负责任务分配和协调
  - `memory`: 是否启用记忆
  - `verbose`: 详细输出

- **Flow** (新功能): 类似工作流引擎
  - `@start()`: 定义流程入口
  - `@listen()`: 定义步骤间依赖
  - `@router()`: 条件路由
  - 支持状态管理和多 Crew 编排

### AgentLoop 设计
```python
# CrewAI 的 Agent 内部循环（ReAct 变体）
class AgentExecutor:
    def _execute(self, task, context):
        for iteration in range(self.max_iter):
            # 1. LLM 思考：给定任务 + 上下文 + 可用工具
            result = self.llm.call(task, context, self.tools)
            
            # 2. 解析输出：是最终答案还是工具调用？
            if result.is_final():
                return result.output
            elif result.is_delegation():
                # 委托给其他 Agent
                delegate_result = self.delegate(result.delegate_to, result.task)
                context += delegate_result
            else:
                # 执行工具
                tool_result = self.execute_tool(result.tool_call)
                context += tool_result
                
        # 达到最大迭代，强制返回
        return self.force_answer(context)
```

### 工具系统设计
- **BaseTool**: 继承自 Pydantic BaseModel
  ```python
  class BaseTool(ABC, BaseModel):
      name: str
      description: str
      args_schema: Type[BaseModel]
      
      def _run(self, **kwargs) -> str: ...
      async def _arun(self, **kwargs) -> str: ...
  ```
- **LangChain 工具兼容**: 直接支持 LangChain 的 `BaseTool`，生态复用
- **工具集合**: 支持 `Toolkit` 概念，一组相关工具的封装
- **结构化输出**: 工具的 `args_schema` 使用 Pydantic，自动注入到 LLM 的 function calling schema

### 记忆系统设计
- **短期记忆 (Short-term Memory)**: 当前 Crew 执行期间的任务输出和 Agent 间交互记录
- **长期记忆 (Long-term Memory)**: 跨 Crew 执行的记忆，通过向量数据库存储
  - 使用 `embedchain` 或自定义后端
  - 存储格式: `(task_description, output, metadata)`
  - 检索: 语义相似度检索相关历史执行结果
- **Entity Memory**: 实体记忆，自动从对话中提取和更新实体信息
- **User Memory**: 用户偏好记忆，跨会话保留

### 最值得 minibox 借鉴的点
1. **角色 + 背景 + 目标 的 Agent 定义模式**: 用 `role/goal/backstory` 三元组定义 Agent 身份，比纯 Prompt 更结构化
2. **任务依赖图 (context 链)**: Task 的 `context` 参数自然地构建任务间依赖关系，前驱任务输出自动作为后继任务上下文
3. **sequential vs hierarchical 编排**: 两种编排策略覆盖了大多数多 Agent 协作场景
4. **delegation 机制**: Agent 可以将子任务委托给其他 Agent，实现动态分工
5. **Flow 工作流引擎**: `@start/@listen/@router` 装饰器模式定义工作流，简洁且表达力强
6. **Entity Memory**: 自动提取和管理实体信息的记忆设计，适合需要跟踪多实体状态的场景

---

## 五、OpenAI Swarm —— 轻量多 Agent Handoff 设计

### 基本信息
- **仓库**: `openai/swarm`
- **Stars**: 21,870 | **Forks**: 2,334
- **描述**: "Educational framework exploring ergonomic, lightweight multi-agent orchestration."
- **创建时间**: 2024-02-22
- **状态**: 教学性质（experimental），非生产级
- **代码量**: 极小，核心实现仅约 300 行

### 核心架构模式

#### Agent + Handoff 极简模型
```python
# Swarm 的核心概念只有两个
# 1. Agent: 有指令和工具的实体
# 2. Handoff: 将控制权从一个 Agent 转移到另一个 Agent

# 定义 Agent
triage_agent = Agent(
    name="Triage Agent",
    instructions="Determine which agent to route to.",
    functions=[transfer_to_billing, transfer_to_support]
)

billing_agent = Agent(
    name="Billing Agent", 
    instructions="Handle billing questions.",
    functions=[issue_refund, check_balance]
)

# Handoff 函数：返回一个 Agent 就完成转移
def transfer_to_billing():
    return billing_agent  # 返回目标 Agent = handoff
```

#### 运行时核心
```python
class Swarm:
    def run(self, agent, messages, context_variables=None):
        # 核心循环
        while True:
            # 1. 调用 LLM，传入 agent.instructions + agent.functions + messages
            response = self.client.chat.completions.create(
                model=agent.model,
                messages=messages,
                tools=[func_to_tool(f) for f in agent.functions],
                tool_choice="auto"
            )
            
            # 2. 处理工具调用
            messages.append(response.message)
            
            # 3. 执行工具
            for tool_call in response.tool_calls:
                result = self.execute_function(tool_call, agent, context_variables)
                
                # 4. 检查是否是 handoff（工具返回了 Agent）
                if isinstance(result, Agent):
                    agent = result  # 切换当前 Agent
                    break  # 跳出工具执行循环，用新 Agent 继续
                else:
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": str(result)
                    })
            
            # 5. 如果没有更多工具调用，返回最终结果
            if not response.tool_calls:
                break
        
        return Response(messages=messages, agent=agent, context_variables=context_variables)
```

#### 核心设计理念
- **Agent = instructions + functions**: 一个 Agent 就是一段系统提示 + 一组函数
- **Handoff = 返回 Agent 的函数**: 最优雅的设计——任何返回 `Agent` 实例的函数自动成为 handoff
- **Context Variables**: 全局上下文字典，在 Agent 间共享，函数可读写
- **无状态**: Swarm 本身无状态，状态通过 messages 和 context_variables 传递

### AgentLoop 设计
- **极简 ReAct 循环**: `LLM 调用 → 处理 tool_calls → 执行函数 → 检查 handoff → 继续或结束`
- **Handoff 即工具调用**: Agent 切换不是特殊语法，而是普通函数返回值
- **自动循环**: LLM 返回 tool_calls 就继续循环，不返回就结束——简洁到极致
- **无最大迭代限制**: 默认无限制（教学性质，实际使用需自行加限）

### 工具系统设计
- **函数即工具**: 普通 Python 函数，自动从 docstring + type hints 生成 OpenAI function schema
  ```python
  def issue_refund(amount: float, reason: str) -> str:
      """Issue a refund for the given amount and reason."""
      ...
  ```
- **无装饰器、无基类**: 不需要 `@tool` 或继承 `BaseTool`
- **函数签名即 Schema**: 参数类型注解 + docstring 自动转换为 function calling schema
- **context_variables 注入**: 如果函数签名包含 `context_variables` 参数，自动注入

### 记忆系统设计
- **无内置记忆**: Swarm 是无状态的
- **记忆 = messages 列表**: 对话历史就是 messages 数组，由调用者管理
- **Context Variables**: 可作为简单的工作记忆，在 Agent 间共享状态
- **持久化由调用者负责**: 调用者需要自己保存和恢复 messages 和 context_variables

### 最值得 minibox 借鉴的点
1. **Handoff 即函数返回值**: 这是最高雅的 Agent 切换设计——返回一个 Agent 实例就完成切换，零额外语法
2. **函数签名自动生成工具 Schema**: 利用 Python type hints + docstring，无需任何装饰器或基类
3. **极简核心循环**: 整个框架核心约 300 行，证明了 Agent 框架可以非常简洁
4. **Context Variables 全局共享**: 简单的字典在 Agent 间传递状态，适合轻量场景
5. **Agent = instructions + functions 的极简定义**: 两个字段就定义一个 Agent，降低使用门槛
6. **"教学性质"的定位本身值得借鉴**: 用最小实现展示核心概念，不追求大而全

---

## 六、横向对比与 minibox 借鉴总结

### 架构模式对比

| 维度 | LangChain/LangGraph | AutoGPT | CrewAI | OpenAI Swarm |
|------|-------------------|---------|--------|-------------|
| **核心抽象** | Runnable / StateGraph | Autonomous Loop + Task | Agent + Task + Crew | Agent + Handoff |
| **AgentLoop** | LangGraph 条件边循环 | 思考→计划→行动→观察 | ReAct 循环 + delegation | 极简 tool_calls 循环 |
| **多 Agent** | 图节点 = Agent | 单 Agent 为主 | 角色协作编排 | Handoff 切换 |
| **工具系统** | @tool + Pydantic | @command 注册 | BaseTool + LangChain 兼容 | 函数签名自动推导 |
| **记忆** | State + Checkpointer + Store | 文件系统 + 向量库 | 短期/长期/Entity Memory | 无（messages 即记忆） |
| **复杂度** | 高（全栈框架） | 中高（平台化） | 中（聚焦多 Agent） | 极低（300 行核心） |
| **生产就绪** | ✅ 成熟 | ✅ 平台化 | ✅ 较成熟 | ❌ 教学性质 |

### 对 minibox 的综合建议

#### 1. AgentLoop 设计 —— 借鉴 Swarm + LangGraph
- 采用 **Swarm 的极简 tool_calls 循环** 作为基础: `LLM → tool_calls → 执行 → 检查切换 → 继续/结束`
- 加入 **LangGraph 的 State 传递** 机制，让循环中携带结构化状态
- 支持 **CrewAI 的 delegation** 概念，允许 Agent 间动态委托

#### 2. 工具系统设计 —— 借鉴 Swarm + LangChain
- 采用 **Swarm 的函数签名自动推导** 作为主要工具定义方式（零装饰器，type hints → schema）
- 提供 **LangChain 的 @tool 装饰器** 作为进阶选项（支持更丰富的元数据）
- 使用 **Pydantic** 做参数校验和 schema 生成

#### 3. 记忆系统设计 —— 借鉴 LangGraph + CrewAI
- **短期记忆**: messages 列表 + 滑动窗口/摘要策略
- **工作记忆**: 类似 Swarm 的 context_variables + LangGraph 的 State
- **长期记忆**: 向量检索 + 键值存储（参考 LangGraph Store API）
- **Entity Memory**: 借鉴 CrewAI 的实体记忆，自动提取和跟踪关键实体

#### 4. 多 Agent 协作 —— 借鉴 Swarm + CrewAI
- **Handoff 模式**: 借鉴 Swarm 的"返回 Agent 即切换"设计
- **角色协作**: 借鉴 CrewAI 的 role/goal/backstory 三元组定义
- **任务编排**: 支持 sequential 和条件路由两种模式

#### 5. 架构哲学 —— 借鉴 Swarm 的极简主义
- **核心极简**: 像 Swarm 一样保持核心循环简洁（<500 行），复杂功能作为可选扩展
- **渐进增强**: 基础是单 Agent + 工具循环；扩展为多 Agent handoff；再扩展为图编排
- **协议优先**: 像 LangChain 的 Runnable 一样定义统一的 Invocable 协议

---

*报告生成时间: 2025-07-31*
*数据来源: GitHub API 实时查询 + 官方文档 + 源码架构分析*
