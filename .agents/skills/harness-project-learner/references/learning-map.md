# 源码学习地图

以下路径均相对项目根目录。按问题读取对应行，授课前核对当前符号和调用关系；2026-09-28 的目录结构仅作为导航基线。

这是两条路线共用的源码检索地图，不是第三条路线。路线 A 使用 [课程地图](course-map.md)，路线 B 使用 [推荐大纲](recommended-outline.md)。主题 1/9 对应 Day01，主题 2 对应 Day02–03，主题 3 对应 Day04，主题 8 对应 Day05，主题 7 对应 Day06 及 Day11，主题 4 对应 Day07–08，主题 5 对应 Day09/11，主题 6 对应 Day10。同一主题跨天时按当前讲义阶段讲解，再指出完整实现的位置。

| 顺序 / 主题 | 主要入口 | 学习结果与阅读练习 |
| --- | --- | --- |
| 1. 服务边界与页面请求 | `frontend/user-frontend/src/api.js`、`frontend/user-frontend/src/App.vue`、`customer-service/atguigu/app/app.py`、`backend/ecommerce-service/ecommerce_service/main.py` | 能区分电商 API 与聊天 API；从发送按钮追踪到消息路由，再解释为何发消息返回不等于 AI 已回复。 |
| 2. 消息与轮次 | `customer-service/atguigu/app/routers/chat/message.py`、`customer-service/atguigu/app/services/chat/message.py`、`customer-service/atguigu/app/services/chat/turn.py`、`customer-service/atguigu/models/models.py` | 追踪 `input_revision`、`snapshot_revision`、合并窗口、领取和租约；推演用户连续发送两条消息。 |
| 3. Worker 与服务调用 | `customer-service/atguigu/worker/ai/worker.py`、`customer-service/atguigu/worker/ai/gateway.py`、`customer-service/atguigu/worker/ai/parser.py`、`customer-service/atguigu/worker/ai/result.py` | 解释 `AIWorker` 与 `TurnProcessor` 的分工；从领取、AI 调用到最终结算定位旧输入被淘汰的位置。 |
| 4. Agent 创建与运行上下文 | `ai-service/atguigu/app/routers/run.py`、`ai-service/atguigu/agent/factory.py`、`ai-service/atguigu/agent/harness/run/coordinator.py`、`ai-service/atguigu/agent/harness/run/executor.py`、`ai-service/atguigu/agent/harness/run/context.py`、`ai-service/atguigu/agent/harness/run/runtime.py` | 分清共享 Agent、调用内 state/context、持久化 AgentRun；分别定位历史截断、工具调用上限、纠错上限。 |
| 5. 技能选择与工具证据 | `ai-service/atguigu/agent/harness/skills/catalog.py`、`ai-service/atguigu/agent/harness/skills/middleware.py`、`ai-service/atguigu/agent/harness/tools/skill.py`、`ai-service/atguigu/agent/harness/tools/catalog.py`、`ai-service/atguigu/agent/harness/tools/executor.py`、`ai-service/atguigu/agent/harness/tools/snapshot.py` | 追踪 `load_skill` 前后的工具范围；按一个查询工具找到参数、上游响应、契约验证和证据存储。 |
| 6. 输出校验与纠错 | `ai-service/atguigu/agent/llm/output.py`、`ai-service/atguigu/agent/harness/validator/output.py`、`ai-service/atguigu/agent/harness/validator/answer.py`、`ai-service/atguigu/agent/harness/validator/action.py`、`ai-service/atguigu/agent/harness/rules/fact.py`、`ai-service/atguigu/agent/harness/rules/correction.py` | 区分结构校验、事实匹配和页面动作约束；构造一个类型正确但事实错误的回答，依据规则推演是否会被拦截。 |
| 7. 结果发布与实时事件 | `ai-service/atguigu/agent/harness/run/output.py`、`ai-service/atguigu/agent/harness/run/coordinator.py`、`customer-service/atguigu/worker/ai/worker.py`、`customer-service/atguigu/app/services/realtime.py`、`customer-service/atguigu/worker/realtime.py`、`customer-service/atguigu/app/routers/realtime.py` | 区分 Run 确认、本地事务、Outbox 发布和客户端消费；推演发布后未标记成功的重复窗口，定位前端补拉逻辑。 |
| 8. 人工客服与业务动作 | `customer-service/atguigu/app/routers/admin/handoff.py`、`customer-service/atguigu/app/services/admin/handoff.py`、`frontend/admin-frontend/src/api.js`、`frontend/admin-frontend/src/App.vue`、`backend/ecommerce-service/ecommerce_service/main.py` | 跟踪接单、回复、结束时的会话模式；解释 AI 引导售后和用户提交售后写请求的责任边界。 |
| 9. 运行、观测与未完成能力 | 各服务 `pyproject.toml`、`common/config.py`、`ai-service/atguigu/app/routers/admin.py`、`ai-service/atguigu/agent/harness/knowledge/service.py`、`customer-service/atguigu/test/redis_pub_sub.py` | 对照 README 列出必要进程与依赖；验证管理页面是否有对应 API，将演示脚本、占位知识能力和真正自动化测试分开。 |

## 三种跨模块练习

- **订单查询**：从用户端消息进入 Turn，跟踪 AI 请求、技能加载、订单工具结果、回答校验和消息事件。每一跳写出输入、输出、负责的服务与是否持久化。
- **用户补充信息**：假设 Worker 调用 AI 期间用户再次发消息，对照实际分支推演 `input_revision` 与 `snapshot_revision`。分别考虑普通回答、页面动作和转人工，不假设它们共用完全相同的确认流程。
- **转人工及恢复**：追踪工单创建、客服接单、人工回复和结束服务。区分业务状态切换、前端展示和实时事件，核对事务边界。

## 掌握标准

学习者能够不依赖术语背诵，指出一个真实入口、描述关键数据/状态变化、解释一个失败分支和设计限制，并提供源码依据。没有运行环境时，完成源码推演即可；不能把推演结果标记为集成测试通过。
