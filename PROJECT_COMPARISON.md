# harness电商客服：源码理解与旧项目对比

记录日期：2026-09-28。

## 对象与核查范围

- harness电商客服：`E:\ZephyrLLM\Projects\harnessassistant`。
- 之前的电商客服：`E:\ZephyrLLM\Projects\commerce-service`。通过此前“解释客服项目validator作用”的讨论记录确认，并重新核查该目录源码。
- 本文依据当前本地源码，覆盖服务边界、对话执行、技能工具、校验、状态存储、实时事件、人工转接和前端调用。未启动依赖服务、未执行真实模型端到端测试，不能据此断言运行稳定性或性能。
- 未使用 GitNexus 索引；当前会话没有可调用的 GitNexus 工具，因此不存在已确认的索引新鲜度。

## 核心判断

旧项目以声明式业务流程为核心：模型理解输入，代码控制任务步骤、槽位和中断恢复。

harness 项目以受约束的工具调用 Agent 为核心：模型选择技能和查询工具，服务端管理工具范围、证据快照、输出校验、轮次有效性与结果发布。

harness 的系统外围更完整，尤其是人工客服、实时推送、异步轮次和运行观测；旧项目的显式长任务状态管理仍有独立价值。两者不是简单的全面替代关系。

## harness 的模块与职责

| 模块 | 职责 |
|---|---|
| `frontend/user-frontend` | Vue 用户端：聊天、订单/商品查看、取消订单、改地址、售后表单；HTTP 发送消息，WebSocket 接收事件 |
| `frontend/admin-frontend` | Vue 管理端：人工接单、回复、结束服务、指标和 Run 查询；知识库和评测界面还不能等同于后端已实现 |
| `customer-service` | FastAPI 会话中心：消息、轮次合并、Worker、输入版本、人工工单、Outbox、WebSocket |
| `ai-service` | FastAPI AI 服务：LangChain `create_agent`、技能路由、工具执行、结构化输出、证据校验、纠错、Run 持久化 |
| `backend/ecommerce-service` | FastAPI 电商业务接口：用户、商品、订单、物流、售后；提供实际页面写操作接口，使用演示数据与演示登录 |

主要基础设施是 PostgreSQL、SQLAlchemy 和 Redis。Redis 在此用于实时发布订阅，AI 任务领取来自数据库中的 Turn 表。

## 一次消息的实际处理过程

1. 用户端发送带 `message_id` 的 HTTP 请求。
2. Customer Service 保存消息；AI 模式下增加 `input_revision`，创建或延长收集中的 Turn。默认合并窗口为 800ms，最大收集等待为 2000ms。
3. AI Worker 领取到期 Turn，记录租约和 `snapshot_revision`，组织当前输入与历史消息。
4. AI Service 创建 AgentRun；ContextCompiler 默认最多保留 30 条历史消息、12000 字符预算。
5. Agent 初始业务工具范围仅为 `load_skill`；加载技能后开放该领域工具，可以在同一次运行中切换技能。
6. 业务工具通过用户令牌调用电商 API。ToolExecutor 保存调用参数和结果，校验返回数据结构并区分业务失败、服务调用失败和契约失败。
7. Agent 返回 AgentOutput：ANSWER、CLARIFY、DECLINE 或 REQUEST_HANDOFF。
8. 服务端校验业务事实和页面动作。可纠正错误最多进行两次执行尝试（首次加一次纠错）；单次 Agent 调用配置工具调用上限 8。
9. 带页面动作的回答进入 DECISION_PREPARED；Customer Service 检查输入版本后 confirm/cancel。普通回答和转人工不走相同的 prepared 分支，但在最终落库前仍检查输入版本。
10. Customer Service 将轮次结果、消息、必要的工单与 Outbox 事件在本地事务中提交。
11. Outbox Worker 发布到 Redis，WebSocket 转发给用户端/客服端。用户端重连后按 sequence 补取历史并按消息标识合并。

这条聊天链路发送的是完成后的业务消息事件。模型调用使用 `ainvoke`，AI Service 与 Customer Service 之间使用 JSON 响应，不能描述成模型 token 逐字流式输出。

关键源码：

- [TurnService](E:/ZephyrLLM/Projects/harnessassistant/customer-service/atguigu/app/services/chat/turn.py:22)
- [TurnProcessor](E:/ZephyrLLM/Projects/harnessassistant/customer-service/atguigu/worker/ai/worker.py:27)
- [Agent 工厂](E:/ZephyrLLM/Projects/harnessassistant/ai-service/atguigu/agent/factory.py:30)
- [AgentExecutor](E:/ZephyrLLM/Projects/harnessassistant/ai-service/atguigu/agent/harness/run/executor.py:18)
- [OutboxWorker](E:/ZephyrLLM/Projects/harnessassistant/customer-service/atguigu/worker/realtime.py:20)
- [WebSocket](E:/ZephyrLLM/Projects/harnessassistant/customer-service/atguigu/app/routers/realtime.py:16)

## 技能、校验与业务边界

代码中注册五类技能：商品、订单、物流、平台政策、实际售后记录。技能是 Python 中的固定定义，包含指导语和工具白名单，不是磁盘上的 Markdown 技能，也不是五个独立 Agent。

工具目录包括七个业务查询工具、一个知识查询工具和 `load_skill`。没有把取消订单、修改地址、提交售后直接暴露为 Agent 写工具。

| 校验层 | 实际职责 | 边界 |
|---|---|---|
| Pydantic / ToolStrategy | 约束输出结构、回复类型与附加字段的组合 | 结构合法不代表语义真实 |
| ToolExecutor | 校验工具响应外壳和业务数据模型，记录调用结果 | 不能证明上游业务数据绝对正确 |
| AnswerValidator / FactChecker | 从回复提取编号、部分数值、状态，与成功业务工具参数/结果中的标量匹配 | 是有限规则匹配；未绑定“对象—属性—值”关系，也未全面覆盖自然语言事实 |
| PageActionValidator | 要求目标订单被成功 get_order 查询；由服务端动作目录生成 URL | 校验页面引导，不执行退款、取消等业务写入 |
| snapshot_revision 检查 | 用户新输入到来时，淘汰过期轮次结果 | 保证输入时效性的机制，不是长业务任务暂停恢复 |

特别需要注意：成功工具数据中的价格 100、库存 5 被合并为标量证据后，如果回答把二者说反，当前 FactChecker 未必能够发现。政策知识也没有被纳入完整的引用一致性检查。

页面写操作的实际链路是：Agent 查询和生成允许的入口 → 用户点击并填写/确认 → 前端调用电商写接口。不能宣传为“Agent 已自动完成退款”。

## 与旧项目的对比

| 维度 | 旧电商客服 | harness电商客服 |
|---|---|---|
| 对话控制 | TurnPlanner → TurnPlanValidator → Task/Knowledge/Chitchat 三轨分派 | create_agent → load_skill → 动态工具调用 → 输出校验 |
| 业务流程 | YAML Flow，Collect/Action/SetSlots/End，确定性推进 | 技能指导与工具选择，未发现等价的持久化业务步骤引擎 |
| 跨轮任务状态 | active_task、paused_tasks、step_id、slots、focused_object | 历史消息、ConversationTurn、AgentRun；active_skill_code 是调用内状态 |
| 中断恢复 | 保留业务步骤和槽位，可显式恢复暂停任务 | 租约超时重排和失败重试，通常重跑本次推理；不等于从某个工具步骤恢复 |
| 主要校验重点 | 轨道唯一性、流程/意图存在、上下文前置条件 | 工具契约、回答事实、页面资源依据、输入版本 |
| 聊天通信 | 当前 /api/chat 等待处理后返回 JSON | HTTP 接收输入，Worker 后台处理，WebSocket 推送完成消息 |
| 用户连续输入 | 按消息处理，通过状态锁协调 | 短消息合并、输入版本更新、丢弃过期输出 |
| 状态持久化 | 同一用户 DialogueState 序列化为一份 JSON，仓储支持 MySQL/SQLite 写锁 | PostgreSQL 拆分会话、轮次、消息、工单、Outbox、Run、工具调用表 |
| 人工客服 | human_handoff Flow 仅输出转接提示 | waiting/active/resolved 工单；AI/QUEUED/HUMAN 模式；接单、回复、结束并回到 AI |
| 知识检索 | 商品/订单 Provider 调业务 API；FAQ/RAG Provider 为空实现 | search_knowledge 固定返回三条售后政策，尚无真实检索 |
| 退款/售后 | refund_request 收集信息后输出“已提交”，未调用退款接口 | Agent 负责引导，页面调用售后申请等实际业务接口 |
| 可观测性 | 可读取会话和任务状态 | 独立 Run、工具调用、耗时、Token 和 Prompt 版本记录及查询 |

旧项目源码依据：

- [DialogueEngine](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/zephyr/engine/dialogue_engine.py:39)
- [TurnPlanValidator](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/zephyr/planning/validator.py:15)
- [FlowExecutor](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/zephyr/task/flow/executor.py:40)
- [DialogueState](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/zephyr/domain/state.py:137)
- [KnowledgeProvider](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/zephyr/knowledge/provider.py:206)
- [退款占位流程](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/flow_config/user_flows.yml:139)
- [人工转接占位流程](E:/ZephyrLLM/Projects/commerce-service/customer-service-backend/flow_config/user_flows.yml:202)

## 已确认的不足与待验证事项

1. **知识能力未接通。** KnowledgeQueryService 对任意非空 query 返回相同三条规则，没有向量检索、关键词召回、重排和文档入库链路。管理前端调用 documents/upload/evaluations API，但当前 AI 后端路由未找到对应实现。
2. **事实校验覆盖有限。** 正则和标量集合匹配不能替代对象级证据绑定与完整事实验证。
3. **confirm/cancel 的业务防线不足。** Coordinator 接收 user_id 却未检查 Run 归属；confirm 未限定前置状态，也未处理不存在的 Run。路由有内部服务令牌与用户角色验证，但仍应补充资源归属及状态迁移约束。
4. **发布确认不等于分布式事务。** DECISION_PREPARED → confirm/cancel 是应用层结果发布协议，不能直接称为数据库 2PC，也不意味着业务写操作恰好执行一次。
5. **Worker 租约不等于完整故障恢复证明。** 已有超时回收，但结算代码没有检查领取者/租约代次；长执行和多 Worker 下的旧任务回写、重复执行需要专门验证。
6. **Outbox 不等于客户端可靠收件箱。** Redis Pub/Sub 无离线积压；publish 与数据库标记之间存在重复发布窗口。用户端已有历史补拉与消息合并，仍不能声称恰好一次送达。
7. **当前登录是演示机制。** demo-token 按演示用户 ID 和角色签发令牌，不能直接当成生产身份认证。
8. **验证材料不足。** 本次提供的 harness 目录未见覆盖主链路的完整自动化测试套件和部署说明；存在 Redis 发布订阅实验脚本。未进行运行验证，不给出性能优劣、生产稳定性或测试通过结论。

## 选型与组合建议

- 商品、订单、物流的灵活查询，多问题组合，以及人工客服协作：优先借鉴 harness。
- 必须逐步收集信息、确认前置条件、跨轮继续的业务，例如复杂退换货与设备排障：保留旧项目 Flow/TaskContext 思路。
- 推荐组合：harness 负责接入、消息与人工服务，Agent 负责理解和查询；关键多步骤任务由确定性流程执行，统一接入工具证据和运行审计。
- 下一阶段优先补：真实知识检索、对象级事实校验、Run 归属与状态机、Worker 租约竞争验证和关键场景端到端测试。

这些建议属于架构判断，不代表本次已经修改业务代码。
