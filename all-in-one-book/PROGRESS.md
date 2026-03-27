# DeerFlow 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 1 | 序言：全局视角 | ch01-overview.md | 项目定位、设计哲学、架构全景图、核心概念词典、代码库地图、一次典型交互的极简全流程 | ✅ |
| 2 | 数据流全景：一次对话的完整旅程 | ch02-data-flow.md | 用户发送消息→前端处理→Gateway→LangGraph→Agent→Tool→响应回传的完整数据流；多场景覆盖（普通对话、工具调用、Subagent 委派、文件上传） | ✅ |
| 3 | Lead Agent：智能体工厂 | ch03-lead-agent.md | make_lead_agent() 工厂函数、Agent 构建过程、模型选择、工具绑定、Prompt 构建、StateGraph 的 model_node↔tool_node 循环 | ✅ |
| 4 | Middleware Pipeline：中间件链 | ch04-middleware.md | 中间件架构设计、执行顺序、每个中间件的职责与实现、中间件如何包裹 Agent 循环 | ✅ |
| 5 | Tool 系统与 Sandbox | ch05-tools-sandbox.md | 工具发现与注册、Sandbox 抽象与本地实现、bash/read/write 等工具实现、虚拟路径映射 | ✅ |
| 6 | Subagent 系统：任务委派与并行执行 | ch06-subagent.md | task_tool、SubagentExecutor、后台任务池、结果回传与流式事件、内置 Subagent 类型 | ✅ |
| 7 | Memory 与状态持久化 | ch07-memory-state.md | ThreadState 设计、Checkpointer（SQLite/Postgres）、长期记忆系统（存储/队列/提取）、Memory Middleware | ✅ |
| 8 | MCP 与 Skills 扩展机制 | ch08-mcp-skills.md | MCP 协议集成（stdio/SSE/HTTP）、工具加载与缓存、Skills 发现与加载（SKILL.md）、Deferred Tool Registry | ✅ |
| 9 | Gateway API 与 Frontend 架构 | ch09-gateway-frontend.md | FastAPI Gateway 路由与职责、Next.js 前端架构、LangGraph SDK 流式通信、消息渲染与状态管理 | ✅ |
| 10 | 端到端追踪：三个关键场景 | ch10-e2e-trace.md | 场景1：纯对话（闪电模式）、场景2：代码执行（Sandbox）、场景3：复杂任务（Ultra 模式 + Subagent），串联全书 | ✅ |

## 章节规划说明

认知路径（先懂什么 → 才能懂什么 → 的依赖链）：

```
第1章 序言
  ↓ 读者获得全局画面，知道系统由哪些部分组成
第2章 数据流全景
  ↓ 读者知道数据怎么流动，但复杂节点还是黑盒
第3章 Lead Agent
  ↓ 打开第一个黑盒：Agent 是怎么构建的
第4章 Middleware
  ↓ 打开第二个黑盒：Agent 循环外面包了什么
第5章 Tool + Sandbox
  ↓ 打开第三个黑盒：Agent 调用工具时发生了什么
第6章 Subagent
  ↓ 打开第四个黑盒：task_tool 背后的并行执行引擎
第7章 Memory + State
  ↓ 打开横切关注点：状态怎么存、记忆怎么留
第8章 MCP + Skills
  ↓ 打开扩展机制：外部工具和技能怎么接入
第9章 Gateway + Frontend
  ↓ 补全两端：前端怎么发、后端怎么收
第10章 端到端追踪
  ↓ 融会贯通：用真实场景串联全书所有知识点
```

## 状态说明
- ✅ 已完成  - 🔄 进行中  - ⏳ 待开始

## 术语约定

### 行业标准术语（直接使用）
| 术语 | 含义 |
|------|------|
| Agent | 智能体——能自主决策并执行动作的 AI 实体 |
| LLM (Large Language Model) | 大语言模型 |
| Middleware | 中间件——在核心逻辑前后执行的可插拔处理层 |
| Pipeline | 管道——数据依次流过的处理链 |
| Sandbox | 沙箱——隔离的代码执行环境 |
| Checkpoint / Checkpointer | 检查点——保存和恢复状态的机制 |
| SSE (Server-Sent Events) | 服务器推送事件——单向流式通信协议 |
| MCP (Model Context Protocol) | 模型上下文协议——标准化的 AI 工具集成协议 |
| State Machine | 状态机——根据输入在有限状态间转换的计算模型 |
| Tool Call | 工具调用——LLM 请求执行外部工具的机制 |

### 项目特有术语
| 术语 | 说明 | 最近似的行业概念 | 差异 |
|------|------|----------------|------|
| Lead Agent | DeerFlow 的主智能体，由 `make_lead_agent()` 工厂函数创建 | Orchestrator Agent | Lead Agent 本身也执行任务，不仅仅是调度 |
| ThreadState | 每次对话的完整状态快照，LangGraph 的 `AgentState` 扩展 | Conversation State | 额外包含 sandbox、artifacts、todos 等字段 |
| Subagent | 由 Lead Agent 通过 task_tool 委派的子智能体 | Worker Agent | 在独立线程池中执行，有超时和并发限制 |
| Skills | 以 SKILL.md 文件定义的领域知识包 | Plugin / Template | 通过 Prompt 注入而非代码插件的方式生效 |
| SOUL.md | 自定义 Agent 的人格和行为指南文件 | Agent Persona / System Prompt | 文件化管理，支持热更新 |
| Deferred Tool | 延迟加载的工具，通过 tool_search 按需发现 | Lazy-loaded Tool | 有专门的 Registry 和搜索机制 |
| Plan Mode | 启用 TodoMiddleware 的任务规划模式 | Task Planning | 通过 Middleware 实现，生成可视化任务列表 |

## 下次续写指引

### 从哪里继续
全书 10 章已全部完成。

### 交接备忘
- 全书约 10 万字符（中文），覆盖后端 Agent 核心、中间件、工具、Sandbox、Subagent、Memory、MCP、Skills、Gateway、Frontend
- 三个端到端追踪场景串联全书
- 每章含质检报告和勘误建议

### 待验证项
- [x] 所有章节已完成并提交
- [ ] ThreadState 继承的 `AgentState` 的确切导入路径（`langchain.agents` vs `langgraph.prebuilt`）
- [ ] `create_agent()` 返回的 StateGraph 内部节点命名确认
- [ ] Docker Sandbox Provider（aio_sandbox）的详细实现
- [ ] Channel 系统（Feishu/Slack/Telegram）的集成细节
