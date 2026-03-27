# DeerFlow 2.0 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 1 | 序言：全局视角 | ch01-overview.md | 项目定位、设计哲学、架构全景图、核心概念词典、代码库地图、极简全流程 | ✅ |
| 2 | 数据流全景 | ch02-data-flows.md | 典型场景的完整数据流：聊天请求、子代理委派、文件上传处理、IM 通道消息、内存更新 | ✅ |
| 3 | Lead Agent 与 LangGraph 运行时 | ch03-lead-agent.md | make_lead_agent 工厂、LangGraph 图结构、模型调度节点、工具调用节点、系统提示词工程 | ⏳ |
| 4 | 中间件链 | ch04-middlewares.md | 15 个中间件的职责、执行顺序、生命周期钩子、状态注入与变换 | ⏳ |
| 5 | ThreadState 与状态管理 | ch05-thread-state.md | TypedDict 字段解析、消息归并、Artifact 去重、Reducer 机制、Checkpointer | ⏳ |
| 6 | 模型抽象层 | ch06-models.md | ModelConfig、工厂函数、Thinking 模式、Vision 支持、Reasoning Effort、多 Provider 适配 | ⏳ |
| 7 | 工具体系 | ch07-tools.md | 内置工具、社区工具、MCP 工具、工具搜索(Deferred)、工具过滤策略 | ⏳ |
| 8 | Skills 系统 | ch08-skills.md | SKILL.md 规范、发现与加载、安装机制、沙箱内路径映射、渐进式加载 | ⏳ |
| 9 | 沙箱执行引擎 | ch09-sandbox.md | 抽象接口、LocalSandbox、Docker/K8s 沙箱、Provider 工厂、路径映射、中间件生命周期 | ⏳ |
| 10 | 子代理系统 | ch10-subagents.md | Registry、Executor 线程池、SubagentConfig、并发控制、工具隔离、SubagentLimitMiddleware | ⏳ |
| 11 | 长期记忆 | ch11-memory.md | 记忆数据结构、LLM 驱动提取、去抖队列、文件存储、Per-agent 隔离 | ⏳ |
| 12 | 配置系统 | ch12-config.md | AppConfig 解析链、19 个配置模块、YAML 解析、环境变量替换、ExtensionsConfig | ⏳ |
| 13 | Gateway API | ch13-gateway.md | FastAPI 应用结构、10 个 Router、Lifespan 管理、与 LangGraph Server 的分工 | ⏳ |
| 14 | 前端架构 | ch14-frontend.md | Next.js 路由、React Query 状态管理、useStream SSE 接入、组件层次、AI Elements | ⏳ |
| 15 | IM 通道集成 | ch15-channels.md | Channel 抽象、MessageBus、ChannelManager、Feishu/Slack/Telegram 适配 | ⏳ |
| 16 | 端到端追踪 | ch16-end-to-end.md | 场景 1: 用户发送消息到看到回复全链路；场景 2: 子代理分拆执行全链路；场景 3: Skill 触发文件生成全链路 | ⏳ |

## 状态说明
- ✅ 已完成  - 🔄 进行中  - ⏳ 待开始

## 术语约定

| 英文术语 | 中文解释 | 首次出现章节 |
|---------|---------|------------|
| Lead Agent | 主代理 — 系统的入口代理，接收用户请求并编排所有工作 | Ch1 |
| Sub-Agent | 子代理 — 由主代理动态创建的独立执行体，处理分拆后的子任务 | Ch1 |
| Middleware | 中间件 — 包裹 LLM 调用前后的可组合处理器链 | Ch1 |
| ThreadState | 线程状态 — 单次会话的完整状态快照 | Ch1 |
| Sandbox | 沙箱 — 隔离的代码执行环境（本地文件系统/Docker 容器/K8s Pod） | Ch1 |
| Skill | 技能 — 结构化的能力模块（SKILL.md 文件定义的工作流） | Ch1 |
| MCP | Model Context Protocol — 外部工具服务器的标准协议 | Ch1 |
| Gateway | 网关 — FastAPI REST 服务，提供模型/技能/记忆等管理端点 | Ch1 |
| Artifact | 制品 — 代理生成的可下载文件（报告、网页、幻灯片等） | Ch1 |
| Checkpointer | 检查点器 — LangGraph 的状态持久化机制 | Ch5 |
| Tool Search | 工具搜索 — 延迟加载工具的动态发现机制 | Ch7 |

## 下次续写指引

### 从哪里继续
从第 1 章（序言：全局视角）开始写作。

### 交接备忘
- 已完成对项目的全面阅读：后端 13 个子系统、前端 7 个模块、配置/部署体系
- 后端核心入口：`backend/packages/harness/deerflow/agents/lead_agent/agent.py::make_lead_agent()`
- 前端核心入口：`frontend/src/core/threads/hooks.ts::useThreadStream()`
- LangGraph 图结构是简单的 model_node ↔ tool_node 循环，复杂度在中间件链
- 中间件实际有 15 个（不是 CLAUDE.md 说的 10 个），需要源码验证确切数量

### 待验证项
- [ ] 中间件的确切数量和执行顺序（CLAUDE.md 说 10 个，探索发现 13-15 个）
- [ ] ACP (Agent Communication Protocol) 集成的具体机制
- [ ] Checkpointer 的具体实现（async_provider.py）
