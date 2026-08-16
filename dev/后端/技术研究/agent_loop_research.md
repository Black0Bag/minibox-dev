# AgentLoop 设计模式：最佳实践与已知缺陷深度调研

---

## 1. ReAct（Reasoning + Acting）模式：优缺点、失败场景、生产环境问题

### 来源
- **原始论文**：Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022, arXiv)
- **生产实战文章**：TheCodeForge.io, "ReAct Agent Pattern - Why Your Agent Loops Forever at 3am" (2026-05-22) — 作者为 20 年生产 ML 系统经验工程师，含 $4k token overrun 真实事故
- **延迟优化案例**：Egnyte 工程团队, "Latency Lessons From Building a ReAct AI Agent for Agentic Search" (2026-07-10)
- **Agent Patterns 文档**：agent-patterns.readthedocs.io, Falconer.com Agent Loops Guide

### 优点
- **动态决策**：Agent 自主决定调用哪个工具、什么顺序，适合步骤不可预测的任务
- **防幻觉**：每步推理都基于真实工具返回数据，强制 ground reasoning
- **简单易懂**：Thought → Action → Observation 循环，上手门槛低
- **"软"执行**：LLM 能自然调整对话流程，不像硬编码状态机那样卡在死胡同

### 缺点与失败场景
| 问题 | 描述 | 实际案例 |
|------|------|---------|
| **无限循环** | 无 max_iterations 时，Agent 永远不停止 | TheCodeForge 报告：推荐引擎 agent 在凌晨循环 47 次，一晚烧掉 $4,000 token |
| **Token 爆炸** | 每轮迭代携带完整历史，上下文线性增长 | 旅行预订 agent 5 次迭代后超出 8K context，每次搜索结果 2K tokens |
| **工具格式错误** | 模型输出畸形 JSON 导致循环崩溃或静默重试 | 天气 agent 输出 `get_weather(city='')` → 工具报错 → 相同输入无限重试 |
| **观察截断** | 工具返回大量数据撑爆 context window | 搜索工具返回 10KB 文本，3 次后 context overflow，LLM 开始生成不连贯响应 |
| **错误处理缺失** | 工具异常被吞而非传回模型 | 数据库查询失败返回 "Error: database error"，模型不理解原因重复相同查询 |
| **延迟不可控** | 迭代次数决定延迟，2s 或 20s 不可预测 | 金融 chatbot 对简单查询也用 ReAct，增加 2s 延迟和 4x 成本 |
| **47% Token 浪费** | 超出推理预算后重复调用工具 | 生产 agent 5 步预算后 47% token 浪费在重复调用，加入 entropy <0.15 早停后失败率降 92% |

### 生产环境已知问题总结
1. **"loops forever at 3am" 问题**：Agent 无内置终止条件，畸形工具输出或缺失 API key 触发无限推理螺旋
2. **上下文退化**：随着历史增长，早期迭代 crisp，后期因 context window 填满变得重复
3. **多 action 输出**：模型一次输出多个 action，naive 实现只解析第一个丢失其余
4. **成本失控**：一个 bug 导致 50 次迭代 = 50x 基础调用成本

### 不应使用 ReAct 的场景
- 纯推理任务（如"解释量子纠缠"）→ 用单次 LLM 调用
- 确定性工作流（已知步骤序列）→ 用 Plan & Solve
- 成本敏感场景 → 用 REWOO（ReAct Without Observation）
- 需要从失败中学习 → 用 Reflexion
- 实时系统（需 bounded latency）→ 用 fixed-step pattern

---

## 2. OpenAI Swarm 的 tool_calls 循环设计：为何是好的极简方案，有何局限

### 来源
- **OpenAI 官方 Cookbook**：developers.openai.com/cookbook/examples/orchestrating_agents — 原始 Routines & Handoffs 设计文档
- **GitHub**：github.com/openai/swarm — 已被 OpenAI Agents SDK 取代
- **Galileo AI**：galileo.ai/blog/openai-swarm-framework-multi-agents — 生产环境评价
- **Respan**：respan.ai/articles/openai-agents-sdk-vs-swarm — 迁移指南
- **AgentBrisk**：agentbrisk.com/frameworks/openai-swarm/ — 功能与优劣分析

### 核心设计：为什么被认为"极简好方案"

Swarm 的核心是一个极简的 `run_full_turn` 循环：

```python
def run_full_turn(agent, messages):
    while True:
        response = client.chat.completions.create(model, [system] + messages, tools)
        message = response.choices[0].message
        messages.append(message)
        if not message.tool_calls:
            break  # 无更多工具调用时退出
        for tool_call in message.tool_calls:
            result = execute_tool_call(tool_call, tools_map)
            if type(result) is Agent:  # handoff 检测
                current_agent = result
            messages.append({"role": "tool", "content": result})
```

**设计精髓**：
1. **Agent = (instructions, tools) 元组**：极简抽象，一个 system prompt + 一组 Python 函数
2. **Handoff 即 tool_call**：`transfer_to_XXX()` 函数返回 Agent 对象，循环自动切换 agent — 模型"smart enough"知道何时调用
3. **无状态管理框架**：不存储状态，完全 client-side，像 Chat Completions API 一样简单
4. **"软"路由**：LLM 自主决定何时 handoff，无硬编码路由逻辑
5. **完整上下文传递**：handoff 时保留全部对话历史，新 agent 拥有完整上下文

**为什么好**：
- **透明性**：开发者完全控制 context、steps、tool calls
- **可调试性**：几乎全部在客户端运行，日志完全可见
- **最小概念**：只有 Agent + handoff 两个概念，无 State、Graph、Reducer 等抽象
- **原型快速**：适合 POC 和教学，理解 agent loop 本质

### 已知局限

| 局限 | 详情 | 来源 |
|------|------|------|
| **非生产级** | OpenAI 明确标注 "meant as an example only, should not be directly used in production" | GitHub README |
| **无状态管理** | 不存储调用间状态，每次都是全新调用 | transitive-bullshit/openai-swarm |
| **无 tracing/guardrails** | 缺少生产必需的追踪、护栏、会话管理 | Respan 迁移指南 |
| **无监控/评估** | "basic logging can't provide" tool call、handoff、model response 级别的可见性 | Galileo AI |
| **上下文无限增长** | handoff 时保留全部历史，无压缩机制 — 上下文随 agent 数量和轮次线性增长 | 架构分析 |
| **无错误恢复** | 工具失败无自动重试或回退机制 | 代码分析 |
| **已被取代** | 2025 年初被 OpenAI Agents SDK 取代（同核心设计 + tracing/guardrails/sessions） | GitHub/Respan |
| **无并行执行** | 工具调用串行处理，无并行编排能力 | 代码分析 |
| **Handoff 粒度粗** | 一次只能 handoff 到一个 agent，无条件路由 | 架构分析 |

---

## 3. LangGraph 的 State + Reducer 模式：相比简单循环的实际优势与复杂度代价

### 来源
- **LangGraph 官方文档**：docs.langchain.com/oss/python/langgraph/graph-api — Graph API overview
- **AgentPatterns**：agentpatterns.ai — 设计模式对比
- **PromptGenius**：promptgenius.net/blog/openai-agents-sdk-architecture — LangGraph vs AutoGen vs CrewAI 架构对比

### State + Reducer 模式的实际优势

LangGraph 将 agent 工作流建模为图（Graph），核心三要素：
- **State**：共享数据结构，代表应用当前快照（TypedDict 或 Pydantic 模型）
- **Nodes**：执行逻辑的函数，接收当前 state，返回更新
- **Edges**：决定下一步执行哪个 Node 的函数（条件分支或固定转移）

**Reducer 机制**：每个 state 字段可指定 reducer 函数，定义如何应用更新。例如 `add_messages` reducer 将新消息追加到列表而非覆盖。

| 优势 | vs 简单循环 | 实际价值 |
|------|-----------|---------|
| **状态显式管理** | 简单循环用 message list 隐式管理状态；LangGraph 用 typed schema 显式定义 | 状态可审计、可序列化、可 checkpoint |
| **Checkpointer** | 简单循环中断即丢失；LangGraph 支持 checkpoint 恢复 | 长时间任务可断点续传、可回溯调试 |
| **并行执行** | 简单循环串行；LangGraph 支持 super-step 内并行节点 | 多工具调用可并行，大幅降低延迟 |
| **条件路由** | 简单循环靠 LLM 决定；LangGraph 支持代码级条件边 | 确定性路由 + LLM 路由混合，更可控 |
| **Human-in-the-loop** | 简单循环难以暂停；LangGraph 支持 breakpoint + interrupt | 生产环境需要人工审批的场景 |
| **状态持久化** | 简单循环无内置持久化；LangGraph 内置 checkpointer | 跨会话状态恢复 |
| **可观测性** | 简单循环日志自定义；LangGraph 集成 LangSmith tracing | 生产级 trace 和调试 |
| **多 schema 支持** | 简单循环单 schema；LangGraph 支持不同输入/输出 schema | 复杂工作流的接口隔离 |

### 复杂度代价

| 代价 | 描述 |
|------|------|
| **学习曲线** | 需理解 State、Node、Edge、Reducer、Super-step、Channel 等概念，远比 while 循环复杂 |
| **样板代码多** | 定义 schema → 添加 nodes → 添加 edges → compile，即使简单任务也需大量代码 |
| **过度工程风险** | 简单任务用图编排是杀鸡用牛刀，增加维护成本 |
| **调试复杂** | 图中多路径、条件边、并行节点使调试轨迹复杂 |
| **框架锁定** | 深度依赖 LangChain 生态系统，迁移成本高 |
| **性能开销** | 图编译、state 序列化/deserialization、checkpoint 持久化带来额外开销 |
| **Compile 步骤** | 必须先 compile 才能使用，增加了初始化复杂度 |

### 实际建议
- **简单 agent（1-3 步）**：用简单循环或 Swarm 式设计
- **复杂工作流（多分支、需 checkpoint、需人工审批）**：用 LangGraph
- **需要并行执行多工具**：LangGraph 的 super-step 模型有明显优势

---

## 4. 多 Agent Handoff（Swarm）vs 多 Agent 图编排（LangGraph/CrewAI）：生产经验对比

### 来源
- **PromptGenius**：promptgenius.net/blog/openai-agents-sdk-architecture — 架构深度对比
- **Galileo AI**：galileo.ai/blog/openai-swarm-framework-multi-agents
- **Respan**：respan.ai/articles/openai-agents-sdk-vs-swarm
- **AgentBrisk**：agentbrisk.com/frameworks/openai-swarm/
- **Anthropic**：anthropic.com/engineering/effective-context-engineering-for-ai-agents — Sub-agent 架构
- **TheCodeForge**：thecodeforge.io/ml-ai/react-agent-pattern/

### 模式对比

| 维度 | Handoff 模式（Swarm/Agents SDK） | 图编排模式（LangGraph/CrewAI） |
|------|--------------------------------|------------------------------|
| **抽象层级** | Agent + handoff function | State + Node + Edge + Reducer |
| **路由方式** | LLM 自主决定 handoff | 代码定义条件边 + LLM 路由混合 |
| **状态管理** | 隐式（共享 message history） | 显式（typed state schema + reducer） |
| **并行能力** | 无（串行 handoff） | 有（LangGraph super-step 并行节点） |
| **可预测性** | 低（完全依赖 LLM 判断） | 高（代码控制流程 + LLM 局部决策） |
| **上下文传递** | 全量传递（可能膨胀） | 可选择性传递 state 子集 |
| **调试难度** | 中等（线性 trace） | 高（图多路径 trace） |
| **上手难度** | 低（1-2 小时） | 高（1-2 天） |
| **适用规模** | 2-5 个 agent | 5+ 个 agent，复杂依赖 |

### 生产经验要点

**Handoff 模式生产经验**：
- Galileo AI 报告：Swarm 的简单设计使原型快速，但生产需要评估和监控，核心系统不包含
- 上下文膨胀是最大生产问题：每次 handoff 保留全部历史，agent 多时 context 快速膨胀
- OpenAI Agents SDK 在 Swarm 基础上增加了 guardrails（输入/输出验证）、tracing、sessions — 表明纯 handoff 不足以生产
- 适合客服路由、简单多步骤任务

**图编排模式生产经验**：
- LangGraph 的 checkpointer 是生产关键：长任务可断点续传
- CrewAI 更高层抽象，适合快速搭建角色化多 agent 系统，但灵活性受限
- 图编排的调试复杂度是主要痛点：多路径 + 并行节点 + 条件边
- 适合复杂研究、代码迁移、数据分析 pipeline

### 小型项目建议

**结论：小型项目优先使用 Handoff 模式**

理由：
1. **概念最小**：只需理解 Agent + transfer function，无需学 State/Graph/Reducer
2. **代码量少**：Swarm 核心实现 <100 行 Python
3. **迭代快**：加 agent 只需定义新 Agent + transfer function
4. **Anthropic 也推荐**："do the simplest thing that works" — 先用简单循环，需要时再迁移到图编排
5. **迁移路径清晰**：Swarm → OpenAI Agents SDK（同核心设计 + 生产功能）

**何时迁移到图编排**：
- 需要 checkpoint 和断点续传
- 需要并行执行多个工具/agent
- 需要 human-in-the-loop 审批
- agent 数量超过 5 个，handoff 链过长
- 需要确定性路由（不依赖 LLM 决定路径）

---

## 5. Agent Loop 中的上下文窗口管理策略：何时压缩、怎么压缩、压缩后丢失什么

### 来源
- **Anthropic 官方**：anthropic.com/engineering/effective-context-engineering-for-ai-agents — "Effective context engineering for AI agents" (2025-09-29)
- **学术综述**：preprints.org/manuscript/202605.2065 — "Context Compression for LLM Agents: A Survey of Methods, Failure Modes"
- **ACON 论文**：arxiv.org/abs/2510.00615 — "Optimizing Context Compression for Long-horizon LLM Agents" (2025-10)
- **Active Context Compression**：arxiv.org/html/2601.07190v1 (2026-01-12)
- **Context Engineering Guide**：langcopilot.com/posts/2026-03-23-context-engineering-ai-agents
- **Context Compaction Guide**：morphllm.com/context-compaction (2026-03-13)
- **JetBrains 研究**：blog.jetbrains.com/research/2025/12/efficient-context-management/
- **GitHub 综述**：github.com/YerbaPage/Awesome-Agent-Context-Compression
- **TheCodeForge**：thecodeforge.io/ml-ai/react-agent-pattern/ — 生产压缩实践

### 核心问题：Context Rot

Anthropic 正式定义了 **"Context Rot"（上下文腐烂）**：
- 随着上下文窗口 token 数量增加，模型准确回忆信息的能力下降
- 根因：Transformer 架构中每个 token attend 到所有其他 token（n² 关系），context 越长注意力越分散
- 这是**性能梯度下降**而非悬崖式断崖 — 模型仍能工作但精度下降
- LLM 的"注意力预算"是有限资源，每个新 token 都在消耗

### 何时压缩

| 触发条件 | 来源 | 具体阈值 |
|----------|------|---------|
| **Token 数达到 context window 的 60-70%** | TheCodeForge 生产实践 | GPT-4 8K window → 5-6K tokens 时触发 |
| **迭代后历史超出预算** | TheCodeForge 旅行 agent 案例 | 每次搜索结果 2K tokens，5 次后超 8K |
| **模型开始忽略旧观察** | TheCodeForge 生产洞察 | "LLM started ignoring old observations" |
| **Context rot 影响推理质量** | Anthropic | 不等撞墙，在性能梯度下降明显时触发 |
| **按固定频率** | TheCodeForge | "每 3 次迭代后 summarize" |
| **Agent 间 handoff 时** | 架构分析 | Swarm handoff 时上下文可能膨胀 |
| **Long-horizon 任务** | Anthropic | 跨数十分钟到数小时的持续工作 |

### 怎么压缩：五种主要策略

#### 策略 1：Compaction（压缩/摘要）
- **机制**：将接近 context window 上限的对话传给模型，让它总结关键信息，然后用摘要重启新 context
- **Anthropic Claude Code 实现**：保留架构决策、未解决 bug、实现细节；丢弃冗余工具输出
- **调优建议**：先最大化 recall（确保摘要捕获所有相关信息），再迭代提高 precision（消除冗余）
- **来源**：Anthropic 官方

#### 策略 2：Tool Result Clearing（工具结果清理）
- **机制**：删除旧的工具调用和返回结果 — "一旦工具在历史深处被调用，agent 为什么还需要看原始结果？"
- **Anthropic 评价**："最安全的轻量级压缩形式"
- **已产品化**：Claude Developer Platform 已推出此功能
- **来源**：Anthropic 官方

#### 策略 3：Sliding Window（滑动窗口）
- **机制**：保留 system prompt + 最近 N 条消息 + 对旧消息的摘要
- **TheCodeForge 实现**：
  ```python
  def prune_history(history, max_tokens=6000):
      system = history[0]  # 保留 system prompt
      recent = history[-3:]  # 保留最近 3 条
      older = history[1:-3]  # 旧消息可被裁剪
      # 按需移除最旧消息直到低于限制
  ```
- **来源**：TheCodeForge 生产代码

#### 策略 4：Structured Note-taking（结构化笔记）
- **机制**：Agent 定期将笔记写入 context window 外部（如 NOTES.md 文件），之后按需拉回
- **Claude Code 实践**：创建 to-do list，跨工具调用跟踪进度
- **Claude 玩 Pokémon 案例**：维护精确计数（"过去 1,234 步在 Route 1 训练，Pikachu 升了 8 级/目标 10 级"），context reset 后读取笔记继续
- **来源**：Anthropic 官方

#### 策略 5：Sub-agent Architecture（子 Agent 架构）
- **机制**：主 agent 协调，子 agent 用干净 context 执行聚焦任务，返回精简摘要（通常 1,000-2,000 tokens）
- **效果**：Anthropic 多 agent 研究系统在复杂研究任务上"显著优于单 agent 系统"
- **来源**：Anthropic 官方

### 压缩后丢失什么

| 丢失内容 | 影响 | 缓解策略 |
|----------|------|---------|
| **工具原始返回数据** | 无法回溯具体 API 响应内容 | 保留关键数据的引用（文件路径、query ID） |
| **推理过程的细节** | 丢失中间推理步骤的细微逻辑 | 摘要中保留关键决策点 |
| **错误调试信息** | 丢失失败原因和重试过程 | 保留 error summary 而非完整 trace |
| **"微妙但关键"的上下文** | Anthropic 警告："过度激进压缩会导致丢失后来才显现重要性的细微上下文" | 先最大化 recall 再优化 precision |
| **多轮对话的语用信息** | 丢失语气、隐含意图等语用层面信息 | 难以完全恢复，需在摘要中显式保留 |
| **失败重试历史** | SWE-agent 用 observation masking 跳过失败重试；OpenHands 用 LLM 总结包含全部 — 两者策略不同导致行为差异 | JetBrains 研究指出此差异 |
| **时间顺序信息** | 摘要可能打乱事件顺序 | 在摘要中保留时间戳或顺序标记 |

### 压缩策略选择指南（基于 Anthropic 官方建议）

| 场景 | 推荐策略 |
|------|---------|
| 需要来回对话的连续性 | Compaction |
| 有明确里程碑的迭代开发 | Structured Note-taking |
| 复杂研究和分析（并行探索有价值） | Sub-agent Architecture |
| 轻量级首选 | Tool Result Clearing |
| 成本敏感 | Sliding Window + 摘要 |
| 长时任务（数小时） | 组合：Note-taking + Compaction + Sub-agent |

### 关键洞察（来自 Anthropic）
> "Context engineering is the art and science of curating what will go into the limited context window from that constantly evolving universe of possible information."
> 
> "Find the smallest set of high-signal tokens that maximize the likelihood of your desired outcome."

**核心原则**：Context 是有限资源，有边际递减效应。不是 context window 越大越好 — 即使有 200K window，context rot 依然存在。主动压缩和管理比被动等待窗口溢出更有效。

---

## 综合结论与建议

### 设计模式选择决策树

```
任务需要工具调用吗？
├─ 否 → 单次 LLM 调用（CoT prompting）
└─ 是 → 步骤可预测吗？
    ├─ 是 → Plan & Solve 或 固定 pipeline
    └─ 否 → 需要多 agent 协作吗？
        ├─ 否 → ReAct 循环（加 max_iterations + 错误处理 + observation 截断）
        └─ 是 → agent 数量 ≤ 5 且无需并行？
            ├─ 是 → Handoff 模式（Swarm/Agents SDK 风格）
            └─ 否 → 图编排（LangGraph）
```

### 通用生产最佳实践
1. **永远设置 max_iterations** — 这一个参数就能防止 90% 的生产事故
2. **工具错误是观察不是异常** — catch → format as string → pass back as observation
3. **截断/摘要 observation** — 大结果截断到 1000 字符或摘要
4. **监控每轮 token 使用** — 设硬预算（如 $0.10/query），超限终止
5. **用 Redis/外部存储管理 session state** — 不在内存中堆积
6. **Just-in-time context retrieval** — 用文件路径/引用代替全量加载
7. **先做最简单的能工作的方案** — Anthropic 和实践者一致推荐
