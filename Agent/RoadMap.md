# RoadMap

### 先建立一个正确的 Agent 运行模型

从工程角度看，一个 Agent 本质上更接近：

```
                ┌─────────────────────┐
User Request ──▶│   Agent Runtime     │
                └──────────┬──────────┘
                           │
                    Build Context
                           │
                           ▼
                ┌─────────────────────┐
                │        LLM          │
                │  Decide next action │
                └──────────┬──────────┘
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
        Final Answer   Tool Call       Handoff
                          │                │
                          ▼                ▼
                       Tool/MCP         Sub Agent
                          │                │
                          └───────┬────────┘
                                  │
                             Observation
                                  │
                                  ▼
                         Update Context/State
                                  │
                                  └───────▶ LLM
```

也就是：

```
while (!finished) {

    context = buildContext(
        systemPrompt,
        userInput,
        history,
        memory,
        toolResults,
        workspaceState
    )

    response = llm(context, availableTools)

    switch (response) {
        case TOOL_CALL:
            result = executeTool()
            context += result
            break

        case HANDOFF:
            currentAgent = anotherAgent
            break

        case FINAL:
            return response
    }
}
```

OpenAI Agents SDK 对这个循环甚至给出了非常明确的官方定义：`Runner` 调用当前 Agent 的模型；如果模型返回 final output 就结束，如果是 tool call 就执行工具并把结果追加回上下文继续循环，如果是 handoff 就切换 Agent 继续执行。

这其实是理解几乎所有现代 Agent 框架的“主干”。

------

### 我建议按这个顺序看

| 阶段 | 资料                                              | 重点                                |
| ---- | ------------------------------------------------- | ----------------------------------- |
| 1    | **Anthropic – Building Effective Agents**         | Agent 的基本思想和架构模式          |
| 2    | **OpenAI – A Practical Guide to Building Agents** | Agent 工程模型                      |
| 3    | **OpenAI Agents SDK – Running Agents**            | 真正理解 Agent Loop                 |
| 4    | **ReAct Paper**                                   | Agent 为什么能“思考→行动→观察→继续” |
| 5    | **OpenAI Agent Orchestration**                    | 单 Agent / Multi-Agent / Handoff    |
| 6    | **MCP Architecture**                              | Agent 如何连接工具和外部系统        |
| 7    | **LangGraph**                                     | Agent Runtime / State Machine       |
| 8    | **AutoGen**                                       | Multi-Agent runtime                 |
| 9    | **LLM Agent Survey**                              | 系统性理论总结                      |

其中我认为最值得看的具体资料是：

1. **Anthropic：Building Effective Agents**。这是目前讲 Agent 架构非常清晰的一篇。它首先区分了 **Workflow** 和 **Agent**：Workflow 是代码决定路径，而 Agent 是模型动态决定下一步操作。随后从 augmented LLM 开始逐渐扩展到 prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer 和 autonomous agent，非常适合建立总体架构。
    [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents?utm_source=chatgpt.com)
2. **OpenAI：A Practical Guide to Building Agents**。这份资料比 Anthropic 更偏工程化，覆盖 Agent 定义、模型、Tools、Instructions、Orchestration、Guardrails 等核心组件，适合在建立概念以后继续往实现走。
    [A Practical Guide to Building Agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/?utm_source=chatgpt.com)
3. **OpenAI Agents SDK：Running Agents**。如果你的问题是“Agent 到底是怎么跑起来的”，这一篇是我最推荐精读的。官方直接把 runtime loop 描述成 `Model → Tool/Handoff/Final → Model` 循环，而不是只讲抽象概念。
    [Running Agents](https://openai.github.io/openai-agents-js/guides/running-agents/?utm_source=chatgpt.com)
4. **OpenAI Agents SDK：Agent Orchestration**。这一篇解决“多个 Agent 到底怎么协同”。官方明确区分了两大类：**LLM-driven orchestration** 和 **code-driven orchestration**；在多 Agent 中又有 `Agent as Tool` 和 `Handoff` 两种很重要的模式。
    [Agent Orchestration](https://openai.github.io/openai-agents-python/multi_agent/?utm_source=chatgpt.com)
5. **ReAct: Synergizing Reasoning and Acting in Language Models**。这是理解 Agent 思想很关键的一篇论文。核心就是交替执行：

```
Reason
  ↓
Action
  ↓
Observation
  ↓
Reason
  ↓
Action
...
```

ReAct 把 reasoning 与 environment action 结合起来，让模型根据工具返回的 Observation 更新后续行动。今天大量 Agent Loop 都能看到这个思想的影子。
 [ReAct Paper](https://arxiv.org/abs/2210.03629?utm_source=chatgpt.com)

1. **MCP Architecture**。理解 Agent 后，下一步必须理解 Tool。MCP 很值得系统学习，因为它把 Agent 外部能力标准化成了 `Tools / Resources / Prompts` 等 primitive，并定义 Host / Client / Server 边界。官方架构里 Host 负责协调和安全边界，Server 暴露能力。
    [MCP Architecture](https://modelcontextprotocol.io/specification/2025-06-18/architecture?utm_source=chatgpt.com)
2. **LangGraph 官方 Learn**。前面的东西理解后再看 LangGraph，会突然发现它其实没有那么复杂：本质上就是把 Agent Runtime 显式建模成 **State + Node + Edge + Loop**。官方教程现在也分别覆盖 Memory、Context Engineering、Graph API、Subagents、Handoffs、Router、Skills 等内容。
    [LangChain / LangGraph Learn](https://docs.langchain.com/oss/python/learn?utm_source=chatgpt.com)
3. **Microsoft AutoGen AgentChat**。如果重点研究 Multi-Agent，这套资料很好，因为 Agent、Message、Team、Termination、State、Human-in-the-loop 都拆得比较明确；再继续下沉到 `autogen-core`，可以看到更接近 Event-driven Agent Runtime 的设计。
    [AutoGen AgentChat Tutorial](https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tutorial/index.html?utm_source=chatgpt.com)
4. 最后再看 **A Survey on Large Language Model based Autonomous Agents**。这篇适合作为“教材式总结”，它试图用统一框架描述 LLM Agent 的构建、应用和评估。不要第一篇就看它，先把 Runtime 理解了再看，会容易很多。
    [LLM Autonomous Agents Survey](https://arxiv.org/abs/2308.11432?utm_source=chatgpt.com)

------

## 如果只让我选 4 份

按照这个顺序：

```
① Anthropic Building Effective Agents
        ↓
理解 Agent / Workflow / Orchestrator

② OpenAI Practical Guide
        ↓
理解 Agent / Tool / Instruction / Guardrail

③ OpenAI Agents SDK - Running Agents
        ↓
理解真正的 Agent Runtime Loop

④ MCP Architecture
        ↓
理解 Agent 如何接入外部世界
```

看完这四个以后，再去看 LangGraph / AutoGen / CrewAI / Google ADK，会很容易看出：

```
Agent Framework
│
├── Agent Definition
│    ├── Instructions
│    ├── Model
│    ├── Tools
│    └── Skills
│
├── Runtime
│    ├── Agent Loop
│    ├── Tool Dispatcher
│    ├── Context Builder
│    ├── State Manager
│    └── Termination
│
├── Context
│    ├── Conversation
│    ├── Working Memory
│    ├── Long-term Memory
│    ├── RAG
│    └── Workspace
│
├── Tool System
│    ├── Function Calling
│    ├── MCP
│    ├── Shell
│    ├── Browser
│    └── Code Execution
│
├── Planning
│    ├── ReAct
│    ├── Planner
│    ├── Reflection
│    └── Evaluator
│
├── Multi-Agent
│    ├── Router
│    ├── Handoff
│    ├── Agent-as-Tool
│    ├── Supervisor
│    └── Parallel Workers
│
└── Production
     ├── Guardrails
     ├── Human Approval
     ├── Tracing
     ├── Evals
     ├── Retry
     └── Cost/Token Control
```

### 对程序员，我更推荐“读源码”的方式

如果你的重点是**搞懂 Claude Code / Codex / Cursor 类 Agent 为什么能够读代码、执行 shell、修改文件、加载 Skill、调用 MCP、多轮自行工作**，那么光看理论还不够。

最值得研究的是这几个模块的实现关系：

```
                    Agent Runtime
                         │
          ┌──────────────┼───────────────┐
          │              │               │
          ▼              ▼               ▼
     Context Engine   Tool Registry   State Store
          │              │               │
          │         ┌────┼────┐          │
          │         ▼    ▼    ▼          │
          │       MCP  Skill Shell       │
          │                              │
          └──────────────┬───────────────┘
                         ▼
                        LLM
                         │
                 tool_calls / final
                         │
                         ▼
                    Tool Executor
                         │
                    Observation
                         │
                         └──────→ Loop
```

**这里才是 Agent 真正的核心。**

例如 OpenAI Agents SDK 官方当前把 Agent 描述为由 instructions、tools、handoffs、guardrails、structured outputs 等组成，而 `Runner` 管理 turns、tool execution、handoffs 和 sessions；这非常适合作为一个“小而完整”的 Agent Runtime 源码参考。

如果你希望系统深入，我建议下一步直接按 **“Agent Runtime 源码精读”** 的方式学习。我可以给你整理一套类似你平时看 AOSP 源码的课程结构，例如 **30 章：Agent Loop → Context → Tool Calling → MCP → Skill → Memory → Planning → SubAgent → Multi-Agent → Human-in-the-loop → Tracing/Eval → Claude Code/Codex 架构拆解**，每章都配架构图、时序图、伪代码以及推荐开源源码。这会比单纯整理一批链接更适合真正掌握 Agent 的实现。