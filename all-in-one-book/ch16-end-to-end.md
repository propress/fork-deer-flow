# 第 16 章 端到端追踪

> **读完本章的收获**：通过 3 个关键场景的完整代码路径追踪，验证你是否真正理解了 DeerFlow 的全流程——从用户输入的第一个字节到最终输出到达用户的每一步。

---

## 16.1 本章目标

前 15 章分别讲解了各个模块。本章是 **全书验收**——选取 3 个场景，追踪每个函数调用、每次状态变更、每次数据变换，穿越所有模块的边界。

---

## 16.2 场景 1：用户发送"写一份关于 AI 的报告"

**前置条件**：用户已打开浏览器，在 `/workspace/chats/new` 页面。

### 前端阶段

```
① 用户输入"写一份关于 AI 的报告"，点击发送
   → frontend/src/components/ai-elements/prompt-input.tsx::onSubmit()
   → frontend/src/components/workspace/input-box.tsx::handleSubmit()

② InputBox 调用 useThreadStream 的 sendMessage
   → frontend/src/core/threads/hooks.ts::sendMessage(threadId, message)
   → 乐观更新：立即在 UI 显示用户消息

③ 无文件上传，直接提交到 LangGraph
   → thread.submit({
       messages: [{ role: "human", content: "写一份关于 AI 的报告" }],
       context: { model_name: "gpt-4", thinking_enabled: true, is_plan_mode: false }
     })
   → @langchain/langgraph-sdk::useStream()
   → POST http://localhost:2026/api/langgraph/threads/{new-uuid}/runs/stream
```

### Nginx 路由

```
④ Nginx 匹配 /api/langgraph/* → 转发到 LangGraph Server :2024
   → docker/nginx/nginx.local.conf 规则
```

### LangGraph Server 阶段

```
⑤ LangGraph Server 创建新 Thread
   → 从 langgraph.json 加载图定义: deerflow.agents:make_lead_agent
   → 创建 ThreadState 初始状态
   → Checkpointer 保存初始检查点
```

### 代理构建

```
⑥ make_lead_agent(config) 被调用
   → deerflow/agents/lead_agent/agent.py::make_lead_agent()
   
   a. 模型解析:
      → config.model_name = "gpt-4"
      → get_app_config().get_model_config("gpt-4") → ModelConfig
   
   b. 工具收集:
      → get_available_tools(model_name="gpt-4", subagent_enabled=False)
        → 配置工具: [tavily_search, jina_reader, ...]
        → 内置工具: [present_files, ask_clarification]
        → view_image (如果 gpt-4 supports_vision)
        → MCP 工具 (如果有启用的 MCP 服务器)
   
   c. 中间件组装:
      → _build_middlewares(config, "gpt-4", agent_name=None)
        → [ThreadData, Uploads, Sandbox, DanglingToolCall, Guardrail,
           ToolErrorHandling, Summarization?, Title, Memory, ViewImage?,
           LoopDetection, Clarification]
   
   d. 提示词生成:
      → apply_prompt_template(subagent_enabled=False)
        → 加载全局记忆 → 技能列表 → 工作目录说明 → 响应风格
   
   e. 创建代理:
      → create_agent(model, tools, middleware, system_prompt, ThreadState)
```

### 中间件 before_agent

```
⑦ 中间件链 before_agent 阶段 (按顺序)
   
   ThreadDataMiddleware.before_agent():
     → thread_id 从 runtime.context 获取
     → 创建 data/threads/{tid}/workspace/, uploads/, outputs/
     → state.thread_data = {workspace_path, uploads_path, outputs_path}
   
   UploadsMiddleware.before_agent():
     → 无上传文件，跳过注入
   
   SandboxMiddleware.before_agent():
     → LocalSandboxProvider.acquire(thread_id) → sandbox_id
     → 创建 LocalSandbox 实例 (path_mappings 配置)
     → state.sandbox = {sandbox_id: "local-{tid}"}
   
   MemoryMiddleware.before_agent():
     → 加载 data/memory.json
     → 如果有记忆 → 注入到系统提示词
```

### LLM 推理循环

```
⑧ model_node 第一次推理
   → LLM 收到: System Prompt + "写一份关于 AI 的报告"
   → LLM 输出: tool_call("tavily_search", {query: "AI 最新发展 2026"})
   → SSE 推送 messages-tuple 事件 → 前端显示 "正在搜索..."

⑨ tool_node 执行工具
   → tavily_search_tool("AI 最新发展 2026")
   → 调用 Tavily API → 返回搜索结果
   → ToolMessage(content="搜索结果: ...")
   → Checkpointer 保存检查点

⑩ model_node 第二次推理
   → LLM 收到: 之前的上下文 + 搜索结果
   → LLM 输出: tool_call("write_file", {path: "/mnt/user-data/outputs/report.md", content: "# AI 报告..."})

⑪ tool_node 执行 write_file
   → 从 state.sandbox 获取 sandbox_id
   → LocalSandboxProvider.get(sandbox_id) → LocalSandbox 实例
   → sandbox.write_file("/mnt/user-data/outputs/report.md", content)
   → LocalSandbox.resolve("/mnt/user-data/outputs/report.md")
     → data/threads/{tid}/outputs/report.md
   → 写入文件

⑫ model_node 第三次推理
   → LLM 输出: tool_call("present_files", {filepaths: ["report.md"]})

⑬ tool_node 执行 present_files
   → state.artifacts += ["report.md"]  (merge_artifacts 去重)
   → 返回确认消息

⑭ model_node 最终推理
   → LLM 输出: "报告已完成，请查看附件中的 report.md" (无 tool_call)
```

### 中间件 after_agent

```
⑮ 中间件链 after_agent 阶段 (逆序)
   
   TitleMiddleware.after_agent():
     → 从第一条用户消息生成标题
     → LLM 调用: "为以下对话生成标题: 写一份关于 AI 的报告"
     → state.title = "AI 发展报告"
   
   MemoryMiddleware.after_agent():
     → _filter_messages_for_memory(messages)
       → 保留: HumanMessage("写一份关于AI的报告")
       → 保留: AIMessage("报告已完成...")
       → 丢弃: 所有 ToolMessage 和中间 AIMessage
     → memory_queue.add(thread_id, filtered_messages)
     → 启动 5 秒去抖定时器
   
   SandboxMiddleware.after_agent():
     → LocalSandboxProvider.release(sandbox_id)
```

### 响应回流

```
⑯ LangGraph Server 推送最终状态
   → SSE: values 事件 (完整 ThreadState)
     → messages: [...所有消息]
     → title: "AI 发展报告"
     → artifacts: ["report.md"]
   → SSE: 流结束

⑰ 前端处理
   → useStream onFinish 回调
   → MessageList 重新渲染
   → Artifact 组件显示 report.md 下载按钮
   → 用户看到完整回复 + 可下载的报告
```

### 异步后续

```
⑱ 5 秒后 MemoryUpdateQueue 触发
   → MemoryUpdater.update_memory(filtered_messages)
   → LLM: "用户请求了 AI 报告，可能对 AI 研究感兴趣"
   → data/memory.json 更新
```

---

## 16.3 场景 2：子代理并行执行

**前置条件**：用户发送"比较 Rust、Go、Zig 三种语言的性能"，开启了子代理模式。

### 关键差异路径

```
① make_lead_agent(config) 中 subagent_enabled=True
   → 工具集额外包含 task 工具
   → 中间件额外包含 SubagentLimitMiddleware
   → 提示词额外包含子代理编排指令

② LLM 第一次推理
   → 识别为可并行任务
   → 输出 3 个 tool_call:
     tool_call("task", {description: "调研 Rust 性能", prompt: "...", subagent_type: "general-purpose"})
     tool_call("task", {description: "调研 Go 性能", prompt: "...", subagent_type: "general-purpose"})
     tool_call("task", {description: "调研 Zig 性能", prompt: "...", subagent_type: "general-purpose"})

③ SubagentLimitMiddleware 检查
   → max_concurrent_subagents = 3
   → 3 个 task 调用 ≤ 3 → 全部保留

④ tool_node 并行执行 3 个 task 工具
   → task_tool("调研 Rust 性能", ...)
     → SubagentExecutor.execute_async(prompt, task_id_1)
       → _scheduler_pool 提交
       → _execution_pool 创建子代理
       → 子代理: create_agent(model, filtered_tools, middlewares)
         → 工具: [tavily_search, jina_reader, ...] (无 task)
       → 子代理推理循环: 搜索 → 分析 → 返回结果
       → result.status = COMPLETED
   
   → task_tool 轮询循环 (每 5 秒):
     → get_background_task_result(task_id_1)
     → 发现新 AI 消息 → dispatch_custom_event("task_running")
     → 前端显示子任务进度卡片
   
   [task_tool_2 和 task_tool_3 同时在其他线程中执行]

⑤ 3 个 task 工具全部返回结果
   → "Task Succeeded. Result: Rust 性能分析..."
   → "Task Succeeded. Result: Go 性能分析..."
   → "Task Succeeded. Result: Zig 性能分析..."

⑥ LLM 综合推理
   → 收到 3 个子任务结果
   → 生成综合对比报告
   → tool_call("write_file", {path: ".../comparison.md", content: "..."})
   → tool_call("present_files", {filepaths: ["comparison.md"]})
```

---

## 16.4 场景 3：Telegram 用户发送消息

**前置条件**：Telegram Bot 已配置并连接。

### 完整路径

```
① Telegram 平台传递消息到 Bot
   → TelegramChannel._on_text(update)
     → 校验 user_id 在 allowed_users 中
     → 构建 InboundMessage(chat_id="12345", user_id="67890", text="帮我总结今天的新闻")
     → bus.publish_inbound(msg)

② MessageBus 传递到 ChannelManager
   → _dispatch_loop() 取出消息
   → _handle_message(msg) → msg_type=CHAT → _handle_chat(msg)

③ 线程管理
   → store.get_thread_id("telegram", "12345", None)
   → 无现有线程 → client.threads.create() → thread_id = "new-uuid"
   → store.set_thread_id("telegram", "12345", None, "new-uuid")

④ 参数解析
   → _resolve_run_params(msg, thread_id)
   → 全局默认: assistant_id=lead_agent, thinking_enabled=true
   → 通道级覆盖: (如果 telegram.session 有配置)
   → 用户级覆盖: (如果 telegram.session.users.67890 有配置)

⑤ 同步执行代理 (Telegram 不支持流式)
   → client.runs.wait(
       thread_id="new-uuid",
       "lead_agent",
       input={"messages": [{"role": "human", "content": "帮我总结今天的新闻"}]},
       config=run_config, context=run_context
     )
   → [完整的代理执行流程，同场景 1]
   → 返回最终状态

⑥ 响应提取
   → _extract_response_text(result) → "今天的主要新闻..."
   → _extract_artifacts(result) → [] (无文件)

⑦ 出站消息
   → publish_outbound(OutboundMessage(
       channel_name="telegram",
       chat_id="12345",
       text="今天的主要新闻..."
     ))
   → bus 通知 TelegramChannel
   → TelegramChannel._on_outbound(msg)
   → context.bot.send_message(chat_id="12345", text="今天的主要新闻...")

⑧ 用户在 Telegram 中看到回复
```

---

## 16.5 全书概念交叉验证

通过这 3 个场景，我们验证了以下概念在实际执行中的交互：

| 概念 (首次出现章节) | 场景 1 | 场景 2 | 场景 3 |
|---|---|---|---|
| Lead Agent (Ch3) | ✅ 构建和运行 | ✅ 构建和运行 | ✅ 构建和运行 |
| 中间件链 (Ch4) | ✅ before/after 全链路 | ✅ + SubagentLimit | ✅ 完整链路 |
| ThreadState (Ch5) | ✅ 状态变更追踪 | ✅ artifacts 去重 | ✅ 状态管理 |
| 模型工厂 (Ch6) | ✅ create_chat_model | ✅ 子代理继承模型 | ✅ 配置覆盖 |
| 工具体系 (Ch7) | ✅ 搜索+文件+展示 | ✅ task 工具 | ✅ 搜索工具 |
| Skills (Ch8) | 间接（提示词注入） | 间接 | 间接 |
| 沙箱 (Ch9) | ✅ 路径映射+文件写入 | ✅ 子代理共享沙箱 | ✅ 沙箱获取/释放 |
| 子代理 (Ch10) | — | ✅ 并行执行全流程 | — |
| 记忆 (Ch11) | ✅ 注入+异步更新 | ✅ 注入+异步更新 | ✅ 注入+异步更新 |
| 配置 (Ch12) | ✅ 模型/工具解析 | ✅ subagent 配置 | ✅ 通道配置层级 |
| Gateway (Ch13) | 间接（Nginx 分流） | 间接 | ✅ Channel Service |
| 前端 (Ch14) | ✅ SSE 流 + 组件渲染 | ✅ 子任务卡片 | — |
| 通道 (Ch15) | — | — | ✅ 完整通道流程 |

---

### 质检报告

**完整性**
- [x] 3 个场景覆盖了系统的主要路径
- [x] 每个场景追踪到具体函数和文件
- [x] 交叉验证表覆盖所有章节概念

**准确性**
- [x] 调用路径与前序章节描述一致
- [x] 函数签名和文件路径与源码一致

**可读性**
- [x] 步骤编号清晰
- [x] 每步标注了具体文件和函数
- [x] 交叉验证表提供全书回顾

**勘误建议**
- 无
