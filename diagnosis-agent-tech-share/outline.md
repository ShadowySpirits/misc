# MQTT 诊断 Agent 技术分享大纲

主题：从诊断工具到可协作 Agent：MQTT 智能诊断平台的使用、架构与开发经验

原则：这份大纲直接对应 PPT 页目录。每一项就是一页；每页的“布局”描述可以直接转成 slide 版式；每页的“内容”只保留上屏信息，详细解释放 speaker notes。

代码核对范围：

- 对话执行与事件流：`frontend/src/hooks/useConversationManager.ts`、`frontend/src/lib/agent-client.ts`
- 前端特殊块渲染：`frontend/src/components/chat/ChatArea.tsx`、`frontend/src/components/ai-elements/subagent-activity-group.tsx`
- 评价与追问：`frontend/src/lib/evaluation-utils.ts`、`frontend/src/lib/extract-messages.ts`
- 分享页/Admin 页 summary：`frontend/src/pages/SharePage.tsx`、`frontend/src/pages/AdminConversationPage.tsx`、`frontend/src/components/admin/ConversationSummaryPanel.tsx`
- Summary 后端：`backend/src/diagnosis_agent/summary_generator.py`、`backend/src/diagnosis_agent/api/admin_routes.py`、`backend/src/diagnosis_agent/conversation_summaries_db.py`
- Agent 路由与编排：`backend/src/diagnosis_agent/workflows/diagnosis_workflow.py`、`backend/src/diagnosis_agent/agents/triage.py`、`backend/src/diagnosis_agent/agents/orchestrator.py`

## PPT 页级目录

### 1. 开场：从诊断工具到可协作 Agent

布局：大标题 + 一句主张 + 三个标签块。

内容：

- 标题：从诊断工具到可协作 Agent
- 主张：一次诊断不是一次问答，而是一段可观察、可追问、可沉淀的协作过程
- 标签：MQTT 诊断 / 多 Agent 编排 / 流式前端 / 反馈闭环

### 2. 分享路线：先看系统怎么工作，再拆为什么这样做

布局：三段式目录。每段包含一个短标题、一句判断、一个关键词组。

上屏内容：

- 使用演示：从真实问题到诊断结论，再到可复用 summary
  - 关键词：原样输入 / 上下文补全 / 工具调度 / Good Case & Bad Case
- 整体架构：Triage 作为入口，把多 Agent、工具、数据源组织成诊断流程
  - 关键词：Triage first / Evidence by tools / Observable UI
- 设计经验：让 Agent 持续变可靠，而不是只让模型“更会说”
  - 关键词：有效上下文 / 原子工具 / Spec 输出 / 评估进化

讲述重点：

- 这不是一次“功能介绍”，而是从真实使用倒推系统设计。
- 第一章让观众看到系统表面行为，第二章解释这些行为背后的架构边界，第三章抽象出可迁移的 Agent 开发经验。

### 3. 使用演示 1：完整流程演示

布局：左侧放用户原始问题，右侧放 4 个流程卡片；底部放一条“用户反馈进入下一轮”的细线闭环。右侧每张卡只保留一句结论，详细证据放 notes。

上屏内容：

- 原样复制用户问题：
  `帮我诊断这个集群mqtt-a822z7oq下的客户端LE4LG4GB1RZ020010，为什么 3 月 12 日服务端没有给客户端发送 keep alive response，导致客户端掉线`
- 1. 输入适配：把“服务端没回 keep alive”视为待验证假设，而不是直接采信
- 2. 上下文补全：Triage 解析时间，发现短 clientId，要求确认完整 ID `LE4LG4GB1RZ020010-CGW2`
- 3. 证据调度：Orchestrator 先查相似历史，再让 LogAgent 验证当前日志；单客户端问题不把 Metrics 当主证据
- 4. 诊断结论：不是服务端漏发 PINGRESP，而是客户端停止 PINGREQ；服务端按 90s 阈值触发 `keepalive_timeout`
- 底部闭环：用户反馈和追问会作为 `[User Feedback]` 进入下一轮上下文

讲述重点：

- 这页的关键不是“Agent 回答对了”，而是它没有被用户问题中的错误前提带偏。
- 用户说的是“服务端没有发送 keep alive response”，但可验证的问题应该是：客户端是否持续发送 PINGREQ、服务端是否收到、断连原因是什么、是否存在并发重连。
- 这个案例里，历史经验先提供了方向，LogAgent 再用日志重新验证，最终把根因收敛到客户端心跳停止和同 Client ID 多 Pod 接入造成的 `session_taken_over`。

代码证据 / speaker notes：

- `ConversationSession.sendMessage()` 原样保存用户输入为 `UserTurn`
- `onToolCallStarted()` 从 `set_context` 参数提取 sidebar context，从 handoff tool 提取 `triageContext` 和 `lastAgent`
- `ChatArea` 拦截 `metrics-chart | instance-ranking | suggestions`
- `submitFeedback()` 发送 “Please review and address my feedback above.”，真实反馈内容来自 `extractMessagesForAgent()`

案例证据（Chrome 当前会话观察）：

- instance：`mqtt-a822z7oq`
- client：`LE4LG4GB1RZ020010-CGW2`
- 关键结论：客户端停止发送 PINGREQ，不是服务端漏发 PINGRESP
- keepalive：60 秒，服务端超时阈值约 90 秒
- 断连分布：`keepalive_timeout` 3 次，`session_taken_over` 6 次，`client_disconnect` 1 次
- 历史检索：Orchestrator 找到同一实例、同一客户端、同一日期的历史诊断记录，并用 LogAgent 重新验证

### 4. 使用演示 2：知识沉淀（Good Case / Bad Case）

布局：顶部一行 Admin Summary 元信息；中间左右双栏，左栏 “Good Case / 复用经验”，右栏 “Bad Case / 优化输入”；底部一条沉淀 pipeline。

上屏内容：

- 页面标题：德赛西威上下行消息数不符合业务预期
- 上方元信息：`Conversation Summary v1` / `Instance Diagnosis` / `~1.1M tokens`
- Good Case：75 个 topic filter -> `Too many incoming subscriptions` -> 整批订阅被拒绝
  - 可复用经验：wildcard 合并订阅、拆批订阅、提高 `maxSubscriptionPerSession`、监控 SUBACK 失败码
- Bad Case：用户给错 clientId：`im_client_19`，真实日志客户端是 `sim_client_19`
  - 优化输入：不能把“查不到”当结论；topic/time 明确时要反查真实 client；订阅拒绝不能直接归因 ACL
- 沉淀路径：conversation + tool calls + feedback -> structured summary -> search/reuse/improve

讲述重点：

- Summary 的价值不是把聊天记录缩短，而是把一次诊断拆成可检索、可复用、可复盘的结构化资产。
- Good case 保存“以后遇到类似症状应该怎么查、怎么解释、怎么建议”；bad case 保存“Agent 曾经容易在哪里走偏，下一版 prompt、skill、工具边界要怎么改”。
- 这个案例同时展示了两种知识：一边是订阅超限的稳定排查经验，另一边是错 clientId 造成的失败路径。

代码证据 / speaker notes：

- `ConversationSummaryPanel` 实际展示上述字段，包括 `User Interaction Analysis`
- `generate_summary_stream()` 通过 SSE 返回 started/completed/error
- `summary_generator.extract_transcript()` 把 user、tool、agent、assistant、feedback 组装成 `[TURN N]`
- `summary_generator.ConversationSummary` schema 包含 `established_facts`、`investigated_components`、`agent_performance_issues`、`workflow_improvement_suggestions`、`related_summaries_hint`
- `conversation_summaries_db` 为 instance、category、symptom keywords、error patterns、全文搜索建立索引

### 5. 整体架构：Triage 驱动的分层诊断系统

布局：左侧大号 Triage 中枢；右侧四层架构块；底部放闭环：“问题 -> 分诊 -> 编排 -> 证据 -> 可观测 -> 沉淀”。

上屏内容：

- Triage first：所有输入先变成可路由、可校验、可执行的诊断上下文
- Presentation：React / Chat UI / Sidebar / Charts / Feedback
- Application：FastAPI `/agent` SSE / proxy APIs / conversation / summary
- Orchestration：LangGraph + Deep Agents / Triage / Orchestrator / Expert Agents
- Data & Knowledge：GraphQL / CLS / Prometheus / PostgreSQL / LLM APIs
- 职责边界：Triage 决定“谁接手”，Orchestrator 决定“查哪些证据”，专家 Agent 只负责自己的数据域

讲述重点：

- 这页只讲系统骨架，不展开实现细节。
- Triage 要放在视觉中心，因为它不是一个普通节点，而是把用户输入从自然语言世界带到工程系统世界的入口。
- 架构上的关键取舍是：LLM 不替代数据源；前端不理解 LangGraph 内部；专家 Agent 不跨越自己的证据域。

代码证据 / speaker notes：

- `workflows/diagnosis_workflow.py`：parent `StateGraph` 有 5 个节点，`START -> triage`，Triage 永远是入口；后续节点是 `orchestrator/mqtt/metrics/log`
- `agents/triage.py`：Triage 使用 `create_agent`，不是 `create_deep_agent`；说明它只做轻量分诊，不承担复杂诊断
- `agents/orchestrator.py`：Orchestrator 使用 `create_deep_agent`，挂载 MQTT/Log/Metrics 三个 subagent 和 `search_similar_issues`
- `agents/mqtt.py`、`agents/log.py`、`agents/metrics.py`：三个专家 Agent 都绑定 `ProgressReportMiddleware`，前端可以观察执行过程

### 6. Agent 拓扑：Triage 是入口，不是诊断专家

布局：Triage 在左，右侧四个出口；每个出口只放职责和典型输入，不放长说明。

上屏内容：

- Triage 只做四件事：分类、时间解析、实体校验、路由
- 四类模式：
  - Mode A：实例诊断 -> Orchestrator
  - Mode B：fleet discovery -> Orchestrator
  - Mode C：快速查询 -> 直接回答或单 Agent
  - Mode D：知识/闲聊 -> 直接回答
- 关键硬规则：
  - 未校验 client 不 handoff
  - `need_correction/not_found` 必须先澄清
  - Triage 不做根因诊断

讲述重点：

- Triage 的价值不是“多一个 Agent”，而是把基础判断提前固定下来，避免后面的 Orchestrator 在错误上下文里越查越远。
- 快速查询可以直接从校验结果回答，复杂问题才进入 Orchestrator，这也是成本控制的一部分。

代码证据 / speaker notes：

- `agents/triage.py` instruction 明确写着：`You are NOT a diagnosis agent — you do NOT investigate problems yourself`
- Triage workflow 明确 4 步：classify、resolve time、`set_context` validate、route
- Triage 的 routing table 覆盖 4 类模式：Mode A/B 到 Orchestrator，Mode C 到自己或单专家 Agent，Mode D 直接回答
- `set_context` 校验结果包括 instance、client、message；client 状态包含 `need_correction/not_found/online/offline_*`
- 硬规则：client validation 是 `need_correction` 或 `not_found` 时不能 handoff，必须先让用户确认或澄清

### 7. 多 Agent 排查：把复杂问题拆成可验证子问题

布局：三列 case-driven flow。左列“用户问题”，中列“可验证子问题”，右列“证据来源”。底部放一句多 Agent 价值判断。

上屏内容：

- Orchestrator 的核心职责不是“多叫几个 Agent”，而是把用户问题拆成可验证的调查问题
- 示例拆解：
  - 用户假设：服务端没回 keep alive
  - LogAgent：心跳日志、断连原因、Pod 时间线
  - MQTTAgent：客户端会话、连接事件、实例配置
  - MetricsAgent：只在怀疑实例级异常时查趋势和 TopK
  - Summary Search：先看历史上是否出现过相同模式
- 多 Agent 的用处：专业化、证据互补、可并行、可裁剪、可复核
- 底部结论：多 Agent 的价值不是“多查”，而是把复杂诊断变成有边界的调查过程

讲述重点：

- 这页不要讲 Handoff / Task Delegation 机制名词，讲排查问题的工程方法。
- Orchestrator 的关键能力是“选择证据”，而不是“把所有专家都叫一遍”。
- 单客户端问题是最好的例子：实例级 Metrics 可能有参考价值，但不能作为客户端行为的直接证据。

代码证据 / speaker notes：

- Orchestrator instruction 要求每次委派前回答 3 个问题：这个 Agent 回答什么、当前数据是否已足够、结果是否会改变结论；如果不会改变结论就跳过委派
- 单客户端问题有硬约束：优先 MQTTAgent + LogAgent；MetricsAgent 没有 per-client metrics，只有在怀疑实例级原因时才补查
- Orchestrator 在委派前使用 `search_similar_issues`，并用历史案例决定优先查哪个 Agent、报告里是否加入“历史参考”
- Instance Diagnosis 的 agent 选择表写在代码里：general health、instance-wide disconnect、single-client、latency、message loss、session takeover、resource pressure、message tracing、auth failures 都有不同主/辅 Agent
- Fleet Discovery 明确以 MetricsAgent 为主，且不委派 MQTTAgent 做实例元数据 enrichment；这部分由前端处理

### 8. 数据源与工具层：Agent 查证据，不替代数据源

布局：5 行证据源表格。每行包含“数据源 / 提供什么事实 / 谁使用”。右侧放一句原则。

上屏内容：

- GraphQL：实例元数据、客户端会话、连接事件 -> MQTTAgent / `set_context`
- CLS：Proxy/Audit 日志、断连原因、订阅拒绝、消息链路 -> LogAgent / `set_context`
- Prometheus：趋势、聚合指标、Fleet TopK -> MetricsAgent
- PostgreSQL：conversation、summary、全文检索、历史经验 -> Summary Search / Admin
- LLM APIs：推理、报告、结构化 summary -> Triage / Orchestrator / Summary Generator
- 原则：LLM 负责判断路径和解释证据，事实必须来自可查询系统

讲述重点：

- 工具层的职责是提供可复核事实，不是把领域结论写死。
- `set_context` 是特殊的“入口工具”：它不只是声明上下文，还完成 instance、client、message 的级联校验。
- 数据源之间是互补关系：日志回答事件，指标回答趋势，GraphQL 回答当前结构化状态，summary 回答历史经验。

代码证据 / speaker notes：

- `tools/context.py`：`set_context` 做 instance -> client -> message 的级联校验；instance 走 GraphQL，client 先 GraphQL 再 CLS fallback，message 走 CLS audit 生命周期分析
- `agents/mqtt.py`：MQTTAgent 只有一个 `graphql-query` 工具；定位是 GraphQL runtime data，高信息密度、快速查询；常用 `mqttInstances/sessionStats/clientEvents/message`
- `agents/log.py`：LogAgent 工具是 `get_error_logs/get_message_trace/execute_log_query`；要求先用聚合和 convenience tool，不要无目的扫原始日志
- `agents/metrics.py`：MetricsAgent 工具是 `query_prometheus_range/query_prometheus_instant`；输出诊断分析和 `metrics-chart` / `instance-ranking` spec
- `tools/summary_search.py`：`search_similar_issues` 从 summary 库返回 root cause、error patterns、investigated components，以及 `agent_performance_issues/workflow_improvement_suggestions`

### 9. 前端可观测性：为后续可评估性做铺垫

布局：4 张真实 UI 截图拼成 evidence wall；每张截图只加一个 5-8 字标签，不放解释段落。

上屏内容：

- 截图 1：工具调用展开态
  - 展示工具名、参数、结果、耗时、状态
  - 讲清楚“结论背后的证据链是可检查的”
- 截图 2：右侧 Agent 执行栏
  - 展示 Triage、Orchestrator、LogAgent 等执行顺序、耗时、token 使用
  - 讲清楚“多 Agent 过程不是黑盒”
- 截图 3：上下文侧栏
  - 展示 `set_context` 生成的 instance/client/time 等诊断上下文
  - 讲清楚“用户输入如何被结构化和校验”
- 截图 4：feedback / highlight / review comment
  - 展示用户可以标注关键结论、错误内容和改进意见
  - 讲清楚“可观测不是只给人看，也是为了后续评估和改进”
- 底部短句：可见过程 + 可见证据 + 可见成本 + 可反馈结果

讲述重点：

- 这页的作用是给第 14 页“评估与进化”做铺垫。
- 如果用户看不到 Agent 做了什么，就很难判断它是否走偏；如果用户不能标注具体错误，团队也很难把一次 bad case 转成系统改进。
- 截图比抽象流程更重要，应优先使用前两章演示中的真实页面。

### 10. Context is all you need：有效上下文，而不是更多上下文

布局：顶部一句定义；中间三层上下文金字塔；底部 4 个工程技巧小标签。

上屏内容：

- 有效上下文：每个 token 都应该对下一步决策有贡献
- 避免低价值上下文：无关历史、原始大结果、重复输出、全量领域文档
- 本项目三层上下文：
  - 行为上下文：代码里给 Agent 的 system prompt，约束角色、边界、停止条件、工具使用策略
  - 领域上下文：skills 中的 catalog / SOP / query template，按需给 Agent 的查询和诊断提供目录
  - 历史上下文：summary search，把 good case 和 bad case 转成可检索经验
- 工程技巧：多 Agent 切分上下文 / 工具结果压缩 / 文件承载中间产物 / 上下文保序适配 KV cache

讲述重点：

- 上下文不是越多越好。真实系统里，过多上下文会增加成本、降低注意力密度，还会把旧错误带入新判断。
- 三层上下文分别解决不同问题：行为上下文管“怎么做”，领域上下文管“查什么、怎么查”，历史上下文管“过去相似问题发生过什么”。
- 这页可以用一句话收束：Context engineering 的目标不是塞满窗口，而是提高每个 token 的决策密度。

代码证据 / speaker notes：

- `agents/triage.py`、`agents/orchestrator.py`、`agents/log.py`、`agents/metrics.py`：system prompt 定义角色、停止条件、工具限制和委派原则
- `agents/orchestrator.py`：`FilesystemBackend(root_dir=skills, virtual_mode=True)`，不同 Agent 加载 `/shared/` + 自己的 skill 目录
- `tools/summary_search.py`：历史 summary 返回 `root_cause/error_patterns/investigated_components` 以及改进建议
- `frontend/src/lib/extract-messages.ts`：明确写着 KV Cache invariant，输出 `[Triage Context] -> final assistant -> [User Feedback]` 的稳定顺序
- `summary_generator._compact_tool_result()` 和 `ProgressReportMiddleware._compact_result_for_frontend()`：分别压缩 summary transcript 和前端工具结果展示

### 11. 优质信息供给：把该进上下文的信息整理好

布局：漏斗图。上方输入两类信息：用户问题、项目知识；中间是校验/筛选；底部是进入 Agent 的紧凑上下文。

上屏内容：

- 承接上一页：有效上下文不是自然出现的，需要系统主动收集、校验、裁剪和组织
- 用户问题上下文：
  - instance：GraphQL 校验是否存在，补 customer/spec/region/TPS
  - client：GraphQL 查 session；查不到时用 CLS fallback 做 partial ID 候选补全
  - message：CLS audit 查生命周期，判断 consumed / not_found / unknown 等状态
  - time：自然语言时间统一转成 ISO 8601
- 项目知识上下文：
  - metric catalog：指标、标签、诊断相关性、close code
  - query catalog：CLS SQL 函数、proxy/auth/audit 查询模板
  - diagnosis skills：订阅失败、消息未消费等稳定排查目录
- 过滤规则：
  - 能由系统验证的，不反问用户
  - 有歧义才让用户确认，例如短 clientId 命中候选
  - 用户的因果判断只当假设，不当事实
  - 能通过工具实时查询的信息不长期塞 prompt
- 产物：进入上下文的是“已验证事实 + 必要假设 + 可用目录”，不是用户原话和全量文档

讲述重点：

- 这一页讲“优质信息怎么进上下文”。用户输入往往是不完整、带假设、甚至带错 ID 的；项目知识又太大，不可能全塞进去。
- 系统做了两件事：对用户信息做实体校验和补全，对项目知识做目录化和按需加载。
- 这也是第 10 页理论的工程落地。

代码证据 / speaker notes：

- `tools/context.py`：注释说明 `set_context` 同时声明 sidebar context 和自动校验 instance/client/message
- `agents/triage.py`：Mode C 简单查询可以直接从 `set_context` validation 结果回答，不必 handoff
- `skills/metrics/metric-catalog/SKILL.md`：Prometheus 指标目录、标签、诊断相关性、close code reference
- `skills/log/query-catalog/SKILL.md`：CLS SQL 函数、proxy/auth/audit 查询模板索引
- `skills/orchestrator/subscription-failure-diagnosis/SKILL.md`、`message-not-consumed-diagnosis/SKILL.md`：把稳定诊断流程沉淀成 Orchestrator 可读的目录

### 12. 工具设计：给原子能力，不给僵硬 SOP

布局：左右对照。左侧是“业务 SOP 工具”的反例，右侧是“原子事实工具 + skill 目录 + Agent 判断”的推荐模型。

上屏内容：

- 不推荐的工具设计：
  - `诊断订阅失败`
  - `判断 keepalive 根因`
- 问题：
  - 场景越细，工具越多，Agent 越难选择
  - SOP 固化在工具里，遇到边界场景难调整
  - 工具直接输出结论，反而削弱可复核性
- 推荐的工具设计：
  - GraphQL 查询工具：查结构化运行时数据
  - CLS 查询工具：查日志、聚合、消息链路
  - Prometheus 查询工具：查时序指标
  - Summary search：查历史经验
  - `set_context`：实体声明和校验
- SOP 放在哪里：
  - 放进 skill/catalog，让 Agent 读目录、选模板、再调用原子工具
  - 工具负责事实，skill 负责方法，Agent 负责判断
- 经验：工具越原子，越需要好的上下文和可观测性；否则 Agent 会乱试

讲述重点：

- 这页的核心观点是“不要把诊断 SOP 做成黑盒工具”。
- 如果工具直接叫“诊断订阅失败”，它会把查询路径、判断逻辑和结论封装在一起，短期看方便，长期会让 Agent 难以调整和复核。
- 本项目选择给 Agent 原子事实工具，再用 skill 提供方法目录。这样工具保持稳定，诊断策略可以随经验演进。

代码证据 / speaker notes：

- `agents/mqtt.py`：MQTTAgent 只有 `graphql-query`，不是一堆业务场景工具
- `agents/log.py`：LogAgent 只有 `get_error_logs/get_message_trace/execute_log_query` 三类工具
- `agents/metrics.py`：MetricsAgent 只有 `query_prometheus_range/query_prometheus_instant`
- `skills/log/query-catalog/SKILL.md`、`skills/metrics/metric-catalog/SKILL.md`：把查询模板和领域目录放在 skill，而不是塞进工具函数
- `agents/log.py`、`agents/metrics.py` 的 prompt 明确要求遇到不熟悉查询时先加载 catalog skill，不要猜 SQL/PromQL

### 13. 确定性输出：用 Spec 约束 AI，用前端拿真实数据

布局：三段 pipeline：LLM emits spec -> API fetches real data -> deterministic component renders。每段配一个极简代码/组件示意。

上屏内容：

- 问题：让 AI 直接输出图表数据，容易幻觉、token 高、数据过期
- 方案：Generative UI / Spec over Data
  - AI 只输出 `metrics-chart` / `instance-ranking` / `suggestions` 这样的轻量 spec
  - spec 里只包含查询、时间范围、单位、legend、排名列表等结构化字段
  - 前端解析 spec 后调用 Prometheus/GraphQL 代理拿真实数据
  - 渲染由确定性 React 组件完成
- 效果：
  - 图表数据不是 AI 编的
  - 交互和样式稳定
  - spec token 成本低
  - 分享页还能把 metrics spec 重新取数并缓存，避免历史页面失效
- 更准确的表述：不能保证“诊断结论 100% 正确”，但能保证“图表和排名展示来自真实接口，而不是模型想象”

讲述重点：

- 这页要严谨表述：Spec 方案解决的是“展示数据不让 AI 编”，不是让整份诊断报告天然正确。
- 模型仍然负责选择查询和解释结果，所以结论仍需要证据链和评估系统支撑。
- 但图表、排名、后续可视化交互由确定性系统完成，这能显著降低幻觉和 token 成本。

代码证据 / speaker notes：

- `agents/metrics.py`：明确要求输出 `metrics-chart` spec，不输出 raw data points；前端会重新从 Prometheus 拉数据
- `ChatArea.tsx` 和 `subagent-activity-group.tsx`：解析 fenced code block，分别渲染 `MetricsPanel`、`InstanceRankingPanel`、`Suggestions`
- `metrics_cache.py`：从 conversation 提取 `metrics-chart` spec，后台 fetch Prometheus 并缓存 resolved series
- `Architecture.md`：记录 spec 模式从约 5000 token 原始数据降到约 200 token 查询规格

### 14. Agent 不会一次搭好：需要明确的评估与进化路径

布局：左侧放一句判断：“Agent 不会一次搭好”；右侧放闭环：Observe -> Evaluate -> Summarize -> Improve -> Re-run。

上屏内容：

- 基本判断：
  - 业务会变，日志会变，指标会变，客户问题也会变
  - Agent 需要一条明确的评估与进化路径
- Observe：过程、证据、成本可见
- Evaluate：thumbs / highlight / comment 定位具体问题
- Summarize：good case 变经验，bad case 变反面教材
- Improve：沉淀到 skill、prompt 或工具边界
- Re-run：同类问题不再从零学习

讲述重点：

- 这一页是第三章的收束：没有评估，Agent 只能靠感觉优化；没有可观测，评估也找不到落点。
- `EvaluationTurn` 解决“用户到底指出了哪句话的问题”；summary 解决“这次会话中什么值得沉淀”；`create-agent-skill` 解决“怎么把一次错误变成下一次的行为约束”。
- 这条路径比“调一次 prompt”慢，但更适合生产系统持续演进。

代码证据 / speaker notes：

- `frontend/src/types/conversation.ts`：`EvaluationTurn` 紧跟 assistant turn，注释说明是为了稳定位置和 KV cache
- `frontend/src/lib/evaluation-utils.ts`：`buildEvaluationSummary()` 把 thumbs/highlights/comment 转成 `[User Feedback]` YAML
- `summary_generator.py`：summary schema 包含 `user_interaction_analysis.agent_performance_issues` 和 `workflow_improvement_suggestions`
- `.claude/skills/create-agent-skill/SKILL.md`：要求优先分析 EvaluationTurn，基于 thumbs-down、nonsense/review highlight 创建新 skill
- `ProgressReportMiddleware`：累加 token usage 和 cache read/create tokens，用于看成本和缓存效果

### 15. Thank You

布局：经典结束页；大号 Thank You / 感谢聆听，底部放 Q&A。

内容：

- Thank You
- 感谢聆听
- Q&A

## 建议节奏

- 使用演示：8-10 分钟，其中第 3 页完整流程、第 4 页知识沉淀是重点
- 架构：10-12 分钟，重点讲 Triage、Agent 协作、工具证据和前端可观测性
- 设计经验：12-15 分钟
- Q&A：5 分钟
