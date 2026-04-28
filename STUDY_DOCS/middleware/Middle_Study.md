# 🛡️ DeerFlow 中间件防线全景解析 (Middleware Defense Matrix)

> **源码路径声明**：中间件位于 `backend/packages/harness/deerflow/agents/middlewares` 目录下。

在大型 AI Agent 架构中，大模型 (LLM) 本体只是一个缺乏自控力、没有记忆且容易触发底层异常的“狂战士”。DeerFlow 通过“洋葱模型 (Onion Architecture)”构建了 16 道中间件防线，在**大模型推理前后**以及**工具执行前后**进行静默拦截、状态劫持和物理阻断，从而将其封装成一个企业级的高可用智能体。

按照业务域，这 16 个中间件可以被划分为五大阵营：

---

## 🎯 阵营一：状态机调度与时空截断 (State & Flow Control)

此类中间件通过拦截 LangGraph 的控制流，实现“断点续传”、“人类介入(HITL)”以及上下文破损修复。

### 1. `clarification_middleware.py` (求救对讲机 / 时空冻结仪)
* **核心功能**：将同步的工具执行转换为异步的“挂起等待人类”机制。
* **内部机制**：
  - `_handle_clarification`: 拦截 `ask_clarification` 工具调用，抓取 JSON 参数。
  - `_format_clarification_message`: 根据 `missing_info`/`risk_confirmation` 等类型将参数美化为带 Emoji 的自然语言选项。
  - `_stable_message_id`: 使用 SHA-256 生成确定性 ID，保证 Checkpointer 幂等覆盖而不是追加。
  - **核心拦截点 (`wrap_tool_call`)**: 不执行原函数，直接返回 `Command(goto=END)` 强行切断状态机执行。
* **交互关系**：与前端 UI 直连（前端识别特定的问询结构并展示交互卡片）；与 LangGraph 底层 Checkpointer 配合实现状态入库。

### 2. `dangling_tool_call_middleware.py` (记忆缝合师)
* **核心功能**：解决因人类中断导致大模型丢失 `ToolMessage` 而引发的 OpenAI API 原生报错崩溃。
* **内部机制**：
  - `_message_tool_calls`: 兼容解析各家大模型（甚至残缺的 JSON string）的工具 Payload。
  - `_build_patched_messages`: 扫描历史记录，如果发现只有 AI 请求而没有 Tool 结果的“幽灵调用”，强行在其后插入 `[Tool call was interrupted...]` 的伪造错误返回。
  - **核心拦截点 (`wrap_model_call`)**: 必须在模型推理前执行，精准篡改 `request.messages`。
* **交互关系**：强行打补丁，保护下游 LLM Provider（如 OpenAI、Anthropic）严格的角色交替校验网关。

### 3. `todo_middleware.py` (冷酷项目监工)
* **核心功能**：检测到大模型的待办清单被上下文截断时，重新向记忆注入提醒；并在未完成任务时堵门不让下班。
* **内部机制**：
  - `_todos_in_messages`: 雷达扫描当前记忆流是否还有待办清单。
  - `_completion_reminder_count`: 计数器，防止大模型真的做不出来导致死循环（最多挡门 2 次）。
  - **核心拦截点 (`before_model` / `after_model`)**: 
    - 战前：用 `HumanMessage` 把丢失的 `todos` 重新默写给模型。
    - 战后：发现没用工具且没做完任务，返回 `Command(jump_to="model")` 将流程指针倒拨。
* **交互关系**：高度依赖并弥补 `SummarizationMiddleware` 的副作用（即旧的 todo 记录被当做废话烧毁）；与内部的 Todo 状态节点保持数据一致性。

---

## 🎯 阵营二：并发节流与死循环镇压 (Concurrency & Loop Prevention)

大模型经常会陷入抽风或暴力并发，这组防线旨在保护物理资源和 Token 资金盘。

### 4. `subagent_limit_middleware.py` (克隆人调度中心)
* **核心功能**：防止大模型在一次回复中并发生成几百个子智能体导致服务器 OOM 和限流。
* **内部机制**：
  - `_clamp_subagent_limit`: 防御性编程，强行将并发数卡死在 [2, 4] 区间。
  - `_truncate_task_calls`: 计算 `tool_calls` 数组中名为 `task` 的数量。如果超标，通过切片 `indices_to_drop` 剔除，并用 `model_copy(update=...)` 覆盖大模型刚生成的原数据。
* **交互关系**：保护后方的 `SubagentExecutor` （子执行器线程池）免受海量并发洪峰攻击。

### 5. `loop_detection_middleware.py` (精神病院长)
* **核心功能**：识别并强行截断大模型的死循环行为。
* **内部机制**：
  - `_stable_tool_key`: **降噪 Hash**。例如对 `read_file` 行数除以 200 分桶，将“同一区域的无效翻页”认定为同一种操作。
  - `_track_and_check`: 采用双层过滤（精确 Hash 匹配 + 同类工具频率计数）。
  - `_evict_if_needed`: 用 `OrderedDict` 维护跨线程的 LRU 记录池。
  - `_build_hard_stop_update`: 直接把大模型产生的 `tool_calls` 数组置空，实施“物理阉割”，伪装成正常的 `finish_reason="stop"`。
* **交互关系**：为了绕过 Anthropic 的 API 史诗级 Bug（不允许中途插 SystemMessage），必须将警告包装成 `HumanMessage`。

### 6. `deferred_tool_filter_middleware.py` (保密局局长)
* **核心功能**：隐藏庞大工具集的 Schema，节省 Token，按需解锁（MCP 架构必备）。
* **内部机制**：
  - `_filter_tools`: 在大模型思考前，将未解密的工具从 `request.tools` 中硬抠掉。
  - `_blocked_tool_message`: 如果大模型试图凭幻觉越权盲调隐藏工具，拦截并返回错误信：“请先调用 `tool_search` 申请解锁”。
* **交互关系**：与 `tool_search` 工具紧密配合，实现无限级别工具挂载的零 Context 负担。

---

## 🎯 阵营三：沙箱安全与容错降级 (Security & Error Handling)

保证系统绝不崩溃，且底层容器绝对安全。

### 7. `sandbox_audit_middleware.py` (海关安检局)
* **核心功能**：在将 `bash` 脚本发给底层 Docker 之前，执行词法级和正则级的拦截。
* **内部机制**：
  - `_split_compound_command`: X光拆包仪。精确处理单双引号和转义符，把 `&&`, `||`, `;` 组装的复合命令拆解。
  - `_classify_command`: 双重审问机制。先扫全句抓捕 Fork 炸弹，再拆分后抓捕 `rm -rf /`（高危击毙）和 `chmod 777`（中危放行但记黄牌警告）。
* **交互关系**：作为 `AioSandbox` 的绝对前置网关，配合 Docker 构建了“软件+物理”双重隔离。

### 8. `llm_error_handling_middleware.py` (大模型网络外交官 / 熔断器)
* **核心功能**：处理所有 LLM Provider 抛出的网络与计费异常。
* **内部机制**：
  - `_check_circuit` / `_record_failure`: 经典的基于 `threading.Lock` 的微服务**熔断器 (Circuit Breaker)**。实现 Closed -> Open -> Half-Open (放探子) 的优雅降级。
  - `_classify_error`: 基于状态码字典判断 502/504 (重试)、Quota/Auth (直接拦截)。
  - `_emit_retry_event`: 向前端 UI 发射 `llm_retry` 信号弹，防止用户干等。
* **交互关系**：独立于主业务流，保护下游厂商同时安抚前端用户。通过 `except GraphBubbleUp: raise` 保证不吞噬框架本身的挂起指令。

### 9. `tool_error_handling_middleware.py` (战地医疗兵与总装车间)
* **核心功能**：捕获 Python 工具的运行报错；并负责把所有防弹衣拼装到一起。
* **内部机制**：
  - `_build_error_message`: 将长达几千行的 Python Stack Trace 强行截断至 500 字符，包装成温柔的 `ToolMessage` 回传给大模型。
  - `_build_runtime_middlewares`: 工厂模式，根据 `include_uploads` 区分总指挥官和克隆人的中间件套装顺序。
* **交互关系**：系统最后一道屏障。包扎好异常后，让大模型（LLM）重新决策，而不是直接导致 Python 进程 `sys.exit`。

---

## 🎯 阵营四：物理基建与多模态数据融合 (Infrastructure & Multimodal)

### 10. `thread_data_middleware.py` (基建工程兵)
* **核心功能**：在开战前，给每个行动分配专属的沙箱磁盘路径。
* **内部机制**：
  - `lazy_init` (懒加载开关): 如果为 True，仅计算 `workspace`, `uploads`, `outputs` 路径字符串写入状态黑板；只有后续真有工具用时，才会触发 `os.makedirs`。
* **交互关系**：为后续的 `SandboxMiddleware` 和 `UploadsMiddleware` 提供物理坐标；懒加载模式极大降低了纯对话轮次的磁盘 I/O 损耗。

### 11. `uploads_middleware.py` (情报卷宗档案员)
* **核心功能**：拦截用户上传的文件，提取大纲后悄悄塞进 Prompt 中。
* **内部机制**：
  - `_extract_outline_for_file`: 寻找后端转化的 `.md` 文件，利用 `extract_outline` 提取标题（L10: 营收）。
  - `_create_files_message`: 生成含有特工指南的 `<uploaded_files>` XML。
  - `before_agent`: 判断如果用户发的是多模态数组（List），则将 XML 巧妙插在第一块，实现移花接木。
* **交互关系**：解决“直接塞文件导致 Token 爆炸”的问题，引导大模型通过 `bash` 和 `read_file` 去主动检索内容。

### 12. `view_image_middleware.py` (多模态视觉神经局)
* **核心功能**：打通工具与 LLM 的视觉神经，实现“主动看图”。
* **内部机制**：
  - `_all_tools_completed`: 防并发干扰机制，确认前线发起的看图指令已经得到返回。
  - `_create_image_details_message`: 抽出暗箱里庞大的 Base64 乱码，组装成带有 `image_url` 协议的格式。
  - `_inject_image_message`: 将多模态数据伪装成 `HumanMessage` 塞入上下文！
* **交互关系**：完美解决 API “工具回复 (`ToolMessage`) 不支持传输图片”的限制，通过 **状态旁路 (`state["viewed_images"]`) + 身份伪造** 机制完成图像传递。

---

## 🎯 阵营五：长程记忆与体验增强 (Memory, Compress & Audit)

### 13. `summarization_middleware.py` (火海抢救秘籍的压缩局)
* **核心功能**：防止上下文过长导致溢出的同时，无损保留“技术规范 (Skills)”。
* **内部机制**：
  - `_is_skill_tool_call`: 硬性正则校验，只有对 `/mnt/skills` 目录调用特定的 `read_file` 动作，才会被标记为“武功秘籍”。
  - `_partition_with_skill_rescue`: 利用 `tool_call_id` 追踪问题与答案。如果大模型同一句话既读秘籍又查天气，调用 `_clone_ai_message` 将消息一劈为二。
  - `_select_bundles_to_rescue`: 基于 5000 / 25000 的 Token 预算限制执行 LRU 倒序抢救。
* **交互关系**：通过 `_fire_hooks` 广播观察者事件，实现了记忆总结与全局档案库的高度解耦。

### 14. `memory_middleware.py` (战地全局档案员)
* **核心功能**：战后提取精华提问与反馈，送往外部向量库学习。
* **内部机制**：
  - `after_agent`: 拦截点发生在最终回复生成后。
  - `filter_messages_for_memory`: 红笔去噪，剔除冗长的工具日志，仅保留 `Human` 和 `AI`。
  - **防抖异步 (`queue.add`)**: 采用异步消息队列投递。
* **交互关系**：响应上一个压缩局触发的 `BeforeSummarizationHook`，赶在聊天记录被粉碎前抄走底稿。

### 15. `title_middleware.py` (起名前台大爷)
* **核心功能**：聊天界面自动生成极简话题标题。
* **内部机制**：
  - `_should_generate_title`: 严格卡在第一轮对话结束触发（`len == 1` & `len >= 1`）。
  - `_strip_think_tags`: 正则剔除 DeepSeek-R1 等模型的 `<think>...</think>` 长文本内耗，提升命名精准度。
  - **双轨制**: 异步模式 `_agenerate_title_result` 召唤无思考能力的小模型省钱；同步模式 `_generate_title_result` 为了不阻塞主线程直接掐头去尾截断。
* **交互关系**：将定名的结果持久化到 `state["title"]`，供前端直接渲染 Sidebar。

### 16. `token_usage_middleware.py` (CFO 财务核算仪)
* **核心功能**：记录算力消耗开销。
* **内部机制**：
  - 从最后一个 `AIMessage` 中提取隐藏的 `usage_metadata`。
  - 提取 `input_tokens` 与 `output_tokens` 打印至系统底层日志。
* **交互关系**：基于开闭原则 (OCP)，与业务代码解耦。只需稍微改造即可无缝接入 Kafka，实现企业商业化的实时计费中心。