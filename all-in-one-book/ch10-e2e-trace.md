# 第十章 端到端追踪：三个关键场景

> **一句话收获**：本章用三个完整场景串联全书所有知识点——从用户键盘到 AI 回复，每一个函数调用、每一次数据变形、每一个组件参与，全程可追踪。

---

## 10.1 为什么需要端到端追踪

前九章，我们逐个拆解了 DeerFlow 的每个子系统。但真正的理解来自于**把它们串起来**——看一个完整请求如何穿越所有层次。

本章选择三个递进复杂度的场景：

| 场景 | 模式 | 涉及的子系统 | 复杂度 |
|------|------|-------------|--------|
| A：Flash 模式纯对话 | Flash | Frontend → LangGraph → Agent → LLM | ⭐ |
| B：代码执行 + 文件产出 | Pro | + Sandbox + Middleware + Artifacts | ⭐⭐ |
| C：多 Subagent 研究任务 | Ultra | + Subagent Pool + Skills + Memory | ⭐⭐⭐ |

每个场景都标注了**涉及的源码文件和函数**，你可以在阅读过程中对照代码。

---

## 10.2 场景 A：Flash 模式 —— "什么是 React Server Components？"

**用户操作**：在 Flash 模式下输入问题，不附加文件。

### 完整调用链

```
[1] InputBox.handleSubmit()                    ← frontend/src/components/workspace/input-box.tsx
  └─[2] sendMessage()                         ← frontend/src/core/threads/hooks.ts
      ├─[3] 乐观更新 setOptimisticMessages()
      └─[4] thread.submit({messages, config})  ← @langchain/langgraph-sdk
          └─[5] SSE → Nginx → LangGraph Server
              └─[6] make_lead_agent(config)    ← backend/.../agents/lead_agent/agent.py
                  ├─[7] create_chat_model()    ← backend/.../models/factory.py
                  ├─[8] get_available_tools()  ← backend/.../tools/tools.py
                  ├─[9] _build_middlewares()   ← backend/.../agents/lead_agent/agent.py
                  └─[10] create_agent()        ← langchain.agents
              └─[11] 中间件前置
                  ├─ ThreadDataMiddleware.before_agent()
                  ├─ UploadsMiddleware.before_agent()
                  └─ SandboxMiddleware.before_agent()
              └─[12] model_node → LLM
                  └─ AIMessage(content="React Server Components 是...") — 无 tool_calls
              └─[13] 循环终止（无 tool_calls）
              └─[14] 中间件后置
                  ├─ TitleMiddleware.after_model() → 生成标题
                  ├─ MemoryMiddleware.after_agent() → 排入记忆队列
                  └─ TokenUsageMiddleware.after_model() → 记录 Token
              └─[15] SSE 流式推送
          └─[16] useStream.onUpdateEvent → 更新标题
          └─[17] useStream.onFinish → 清理乐观消息
  └─[18] groupMessages() → [human, assistant]  ← frontend/.../messages/utils.ts
  └─[19] MessageList 渲染                      ← frontend/.../messages/message-list.tsx
```

### 关键数据变化

| 步骤 | 数据形态 |
|------|---------|
| [1] | `{ text: "什么是 React Server Components？", files: [] }` |
| [4] | `{ messages: [{ type: "human", content: [{type: "text", text: "..."}] }], context: { thinking_enabled: false, subagent_enabled: false } }` |
| [6] | `cfg = { thinking_enabled: False, is_plan_mode: False, subagent_enabled: False }` |
| [12] | `ThreadState.messages = [SystemMessage, HumanMessage] → + AIMessage(无 tool_calls)` |
| [14] | `ThreadState.title = "React Server Components 简介"` |
| [18] | `groups = [{type: "human", msgs: [...]}, {type: "assistant", msgs: [...]}]` |

---

## 10.3 场景 B：Pro 模式 —— "帮我写一个 Python 爬虫脚本"

**用户操作**：在 Pro 模式下请求创建代码文件。涉及 Sandbox 文件写入和 artifact 追踪。

### 完整调用链

```
[1] sendMessage()
  └─[2] thread.submit({context: {thinking_enabled: true, is_plan_mode: true}})
      └─[3] make_lead_agent(config)
          ├─ thinking_enabled = True
          ├─ tools = [...所有工具, 不含 task_tool]
          └─ middlewares = [..., TodoMiddleware, ...]  ← plan_mode 启用
      └─[4] 中间件前置
          ├─ ThreadDataMiddleware → 初始化 workspace/uploads/outputs 路径
          └─ SandboxMiddleware → 获取 LocalSandbox 实例
      
      ┌─[5] Agent 循环 — 第 1 轮
      │  └─ model_node → LLM（带扩展思考）
      │     └─ AIMessage(
      │          reasoning="我需要先创建一个爬虫脚本...",
      │          content="好的，让我来创建...",
      │          tool_calls=[{name: "write_file", args: {path: "/mnt/user-data/workspace/crawler.py", content: "..."}}]
      │        )
      │  └─ tool_node
      │     └─[6] write_file_tool(path, content, runtime=ToolRuntime)
      │         ├─ runtime.state.sandbox → sandbox_id
      │         ├─ get_sandbox_provider().get(sandbox_id) → LocalSandbox
      │         ├─ _resolve_path("/mnt/user-data/workspace/crawler.py")
      │         │   → "~/.deerflow/threads/{id}/user-data/workspace/crawler.py"
      │         ├─ sandbox.write_file(resolved_path, content)
      │         └─ 更新 state.artifacts = ["crawler.py"]
      │     └─ ToolMessage(content="文件已创建", tool_call_id="call_xxx")
      │
      ├─[7] Agent 循环 — 第 2 轮
      │  └─ model_node → LLM
      │     └─ AIMessage(
      │          tool_calls=[{name: "bash", args: {command: "cd /mnt/user-data/workspace && python crawler.py"}}]
      │        )
      │  └─ tool_node
      │     └─[8] bash_tool(command, runtime)
      │         ├─ 正向路径映射：/mnt/... → ~/.deerflow/...
      │         ├─ sandbox.execute_command(mapped_command)
      │         ├─ 反向路径映射：输出中的真实路径 → 虚拟路径
      │         └─ ToolMessage(content="爬取完成：获取了 50 条数据\n保存到 /mnt/user-data/workspace/data.json")
      │
      └─[9] Agent 循环 — 第 3 轮
         └─ model_node → LLM
            └─ AIMessage(content="爬虫脚本已创建并执行成功...") — 无 tool_calls → 循环终止

      └─[10] 中间件后置
          ├─ TodoMiddleware → (如果创建了 todo 列表，检查是否完成)
          ├─ TitleMiddleware → "Python 爬虫脚本"
          ├─ MemoryMiddleware → 排队
          └─ TokenUsageMiddleware → 记录

      └─[11] SSE 流推送给前端
          └─ values 事件包含 artifacts: ["crawler.py", "data.json"]

[12] Frontend
  └─ groupMessages() → [
       {type: "human", msgs: [用户消息]},
       {type: "assistant:processing", msgs: [AI+tool_call, ToolMessage, AI+tool_call, ToolMessage]},
       {type: "assistant", msgs: [最终回复]}
     ]
  └─ ArtifactPanel 显示 crawler.py 和 data.json 预览
```

### 路径映射追踪

```
Agent 视角:     /mnt/user-data/workspace/crawler.py
    ↓ _resolve_path()
宿主机实际:     ~/.deerflow/threads/abc-123/user-data/workspace/crawler.py
    ↓ write_file()
文件系统:       写入成功
    ↓ Artifact URL
前端访问:       GET /api/threads/abc-123/artifacts/workspace/crawler.py
    ↓ Gateway
返回文件内容
```

---

## 10.4 场景 C：Ultra 模式 —— "比较 React、Vue、Svelte 的性能"

**用户操作**：Ultra 模式下请求复杂研究任务。涉及 Subagent 并行执行、Skills 加载、Memory 更新。

### 完整调用链

```
[1] thread.submit({context: {thinking_enabled: true, is_plan_mode: true, subagent_enabled: true}})

[2] make_lead_agent(config)
    ├─ tools = [...所有工具, task_tool]          ← subagent_enabled=True
    ├─ middlewares = [..., SubagentLimitMiddleware(max=3), ...]
    └─ system_prompt 包含 Subagent 编排指令     ← apply_prompt_template()
        └─ "🚀 SUBAGENT MODE ACTIVE - DECOMPOSE, DELEGATE, SYNTHESIZE"
        └─ "⛔ HARD CONCURRENCY LIMIT: MAXIMUM 3 `task` CALLS PER RESPONSE"
        └─ Skills 列表（deep-research 等）

[3] 中间件前置（同场景 B）

[4] Agent 循环 — 第 1 轮
    └─ model_node → LLM（带编排指令）
       └─ AIMessage(
            reasoning="这个任务适合拆分为三个子任务并行研究...",
            content="我来分别研究三个框架的性能特性。",
            tool_calls=[
                {id: "c1", name: "task", args: {description: "研究 React", prompt: "深入研究 React 的性能...", subagent_type: "general-purpose"}},
                {id: "c2", name: "task", args: {description: "研究 Vue", prompt: "深入研究 Vue 的性能...", subagent_type: "general-purpose"}},
                {id: "c3", name: "task", args: {description: "研究 Svelte", prompt: "深入研究 Svelte 的性能...", subagent_type: "general-purpose"}},
            ]
          )

[5] SubagentLimitMiddleware.after_model()
    └─ 3 个 task 调用 ≤ max_concurrent(3) → 通过，不截断

[6] tool_node — 并行执行 3 个 task_tool
    ├─[6a] task_tool("研究 React", ...)
    │   ├─ get_subagent_config("general-purpose")  ← registry.py
    │   │   └─ SubagentConfig(disallowed_tools=["task"], timeout=900s)
    │   ├─ _filter_tools(parent_tools, disallowed=["task","ask_clarification","present_files"])
    │   ├─ SubagentExecutor(config, filtered_tools, parent_model, sandbox, thread_data)
    │   ├─ executor.execute_async(task_id_1)       ← 提交到 _scheduler_pool
    │   │   └─ _execution_pool.submit(_aexecute)   ← 后台线程
    │   │       └─ create_agent(model, tools, middleware, system_prompt="研究 React...")
    │   │       └─ agent.astream(initial_state)    ← 独立的 Agent 循环
    │   │           ├─ 搜索 React 性能文章
    │   │           ├─ 读取搜索结果
    │   │           └─ 生成研究报告
    │   ├─ writer({"type": "task_started", "task_id": "...", "description": "研究 React"})
    │   └─ 每 5s 轮询:
    │       └─ writer({"type": "task_running", "task_id": "...", "message": ...})
    │
    ├─[6b] task_tool("研究 Vue", ...)  ← 同上，并行
    └─[6c] task_tool("研究 Svelte", ...) ← 同上，并行

[7] 前端实时接收
    └─ onCustomEvent: task_started × 3 → 显示 3 个子任务卡片
    └─ onCustomEvent: task_running × N → 更新进度
    └─ onCustomEvent: task_completed × 3 → 标记完成

[8] 所有 Subagent 完成
    └─ ToolMessage(tool_call_id="c1", content="React 性能分析：...")
    └─ ToolMessage(tool_call_id="c2", content="Vue 性能分析：...")
    └─ ToolMessage(tool_call_id="c3", content="Svelte 性能分析：...")

[9] Agent 循环 — 第 2 轮
    └─ model_node → LLM（看到三个研究结果）
       └─ AIMessage(content="## 三大框架性能对比\n\n### React\n...\n### Vue\n...\n### Svelte\n...")
       └─ 无 tool_calls → 循环终止

[10] 中间件后置
    ├─ TitleMiddleware → "React/Vue/Svelte 性能对比"
    ├─ MemoryMiddleware → 排入队列
    │   └─ 30s 后: MemoryUpdater.update_memory()
    │       ├─ 过滤消息（保留 human + final AI）
    │       ├─ LLM 提取: "用户关注前端框架性能比较"
    │       └─ 新事实: {content: "关注 React/Vue/Svelte 性能", category: "knowledge", confidence: 0.8}
    └─ Checkpointer 保存完整 ThreadState

[11] 前端渲染
    └─ groupMessages() → [
         {type: "human"},
         {type: "assistant:subagent", msgs: [AI with 3 task calls]},  ← 子任务卡片
         {type: "assistant:processing", msgs: [3 ToolMessages]},
         {type: "assistant", msgs: [综合分析回复]}
       ]
```

---

## 10.5 三个场景的知识点覆盖

| 知识点 | 场景 A | 场景 B | 场景 C |
|--------|--------|--------|--------|
| **第 1 章 全局架构** | ✅ 四服务协作 | ✅ | ✅ |
| **第 2 章 数据流** | ✅ 最简路径 | ✅ 工具调用循环 | ✅ 扇出-汇聚 |
| **第 3 章 Lead Agent** | ✅ 工厂函数 | ✅ 模式参数 | ✅ Subagent 启用 |
| **第 4 章 Middleware** | ✅ Title + Memory | ✅ + Todo | ✅ + SubagentLimit |
| **第 5 章 Tool + Sandbox** | — | ✅ write_file + bash | ✅ 继承 |
| **第 6 章 Subagent** | — | — | ✅ 完整流程 |
| **第 7 章 Memory** | ✅ 排队 | ✅ 排队 | ✅ 完整提取 |
| **第 8 章 MCP + Skills** | — | — | ✅ Skills 注入 |
| **第 9 章 Gateway + Frontend** | ✅ SSE 流 | ✅ + Artifacts | ✅ + Subtask UI |

---

## 10.6 全书总结

恭喜你走完了 DeerFlow 的全部实现原理。让我们用一段话概括全书的核心：

**DeerFlow 的本质是一个精心编排的 Agent 运行框架。** 它的核心是一个极其简洁的循环——LLM 决策 + 工具执行，无限交替直到 LLM 认为任务完成。所有的复杂性都被推到了循环之外：

- **Middleware** 注入了横切关注点，让核心循环保持纯净
- **Sandbox** 提供了安全的执行环境，让 Agent 能"动手"而不伤害系统
- **Subagent** 实现了并行执行，让复杂任务可以分而治之
- **Memory** 跨越对话边界，让 Agent 不再是"金鱼记忆"
- **MCP + Skills** 构建了开放的扩展生态，让能力和知识可以持续增长
- **Gateway + Frontend** 提供了流畅的人机界面，让一切对用户透明

理解了这些，你就掌握了 DeerFlow 的全部架构精髓。接下来，不妨亲自 `make dev` 启动它，带着本书的知识去阅读源码——你会发现一切都已 make sense。

---

### 质检报告

**讲解节奏**
- [x] 三个场景递进复杂度
- [x] 每个场景都有完整调用链 + 数据变化追踪

**讲透了吗**
- [x] 场景 A 覆盖最简路径的每一步
- [x] 场景 B 深入路径映射和 Artifact 追踪
- [x] 场景 C 完整展示 Subagent 并行执行 + Memory 更新
- [x] 知识点覆盖表格串联全书

**准确吗**
- [x] 调用链中的函数名和文件路径基于实际源码
- [x] 数据变化追踪基于前几章的分析

**读得下去吗**
- [x] 调用链格式清晰，缩进表示调用层次
- [x] 关键步骤有注释标注来源文件
- [x] 全书总结简洁有力

**勘误建议**
- 场景 C 中 `deep-research` Skill 是否在该场景中被实际加载取决于 Prompt 模板的实现
