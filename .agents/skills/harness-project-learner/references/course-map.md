# 路线 A：课程 → 源码 → 练习

本文件供按章学习、继续学习和制定计划时按需读取。主线沿本地讲义的 Day01–Day11；Day 表示课程阶段，不是完成时长承诺。源码主题编号指向 [源码地图](learning-map.md)。下列链接相对于本文件，代码位置相对于项目根目录。

另一种安排见 [路线 B：推荐大纲](recommended-outline.md)。切换时按已掌握知识点衔接，不要求从头重学。

## 主线顺序

Day01 建立系统和运行认识 → Day02–04 会话与异步轮次 → Day05–06 人工和实时事件 → Day07 总结客服并进入 AI → Day08–10 执行、工具和校验 → Day11 动态技能与完整链路。

每篇文章按原文二级、三级标题的顺序讲解，保留原编号；下表只是定位提示，不替代文章完整大纲。部署文档属于 Day01，除非用户明确要求略读或跳过，否则按原文安排讲解。第一次进入 AI 阶段可补充 Harness 概念，其他补充材料按需使用。

| Day / 文档顺序 | 原文 | 优先定位的小节/概念 | 当前源码入口 | 理解练习 |
| --- | --- | --- | --- | --- |
| Day01 / 01 | [day01_电商客服_项目背景以及架构](../../../../docs/course-materials/originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/2_resource/day01_%E7%94%B5%E5%95%86%E5%AE%A2%E6%9C%8D_%E9%A1%B9%E7%9B%AE%E8%83%8C%E6%99%AF%E4%BB%A5%E5%8F%8A%E6%9E%B6%E6%9E%84.md) | 项目背景、角色、五服务交互 | 主题 1；两个前端 api.js、三个 API 入口 | 说清用户、客服、AI 和电商各负责什么；画出一条查询路径。 |
| Day01 / 02 | [电商后台项目部署](../../../../docs/course-materials/originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/2_resource/%E7%94%B5%E5%95%86%E5%90%8E%E5%8F%B0%E9%A1%B9%E7%9B%AE%E9%83%A8%E7%BD%B2.md) | Dockerfile、compose、数据库与健康检查 | 主题 9；README 本地运行、服务配置 | 区分课程容器部署与当前本地启动；先确认部署文件是否存在，不直接执行文章命令。 |
| Day01 / 03 | [day01_客服服务项目初始化](../../../../docs/course-materials/originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/2_resource/day01_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96.md) | 配置、数据库接入、事件循环、启动入口、令牌 | 主题 9；customer-service/atguigu/common/config.py、common/event_loop.py、app/app.py | 解释为什么同名 atguigu 包要在各自环境中运行；指出一个配置的使用位置。 |
| Day02 / 04 | [day02_客服会话建模与分层实现](../../../../docs/course-materials/originals/day02_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E4%BC%9A%E8%AF%9D%E7%BC%96%E7%A0%81%E4%B8%8A%EF%BC%88%E4%BC%9A%E8%AF%9D%E5%BB%BA%E6%A8%A1%E4%B8%8E%E5%88%86%E5%B1%82%E5%AE%9E%E7%8E%B0%EF%BC%89/2_resource/day02_%E5%AE%A2%E6%9C%8D%E4%BC%9A%E8%AF%9D%E5%BB%BA%E6%A8%A1%E4%B8%8E%E5%88%86%E5%B1%82%E5%AE%9E%E7%8E%B0.md) | 会话/消息/轮次、约束与索引、数据访问和事务 | 主题 2；customer-service/atguigu/models/models.py、app/services/chat/conversation.py | 解释三个对象的关系，并找出数据库约束或索引的实际定义。 |
| Day03 / 05 | [day03_用户消息接收与轮次收集](../../../../docs/course-materials/originals/day03_%E5%AE%A2%E6%9C%8D%E4%BC%9A%E8%AF%9D%E7%BC%96%E7%A0%81%E4%B8%AD%EF%BC%88%E7%94%A8%E6%88%B7%E6%B6%88%E6%81%AF%E6%8E%A5%E6%94%B6%E4%B8%8E%E8%BD%AE%E6%AC%A1%E6%94%B6%E9%9B%86%EF%BC%89/2_resource/day03_%E7%94%A8%E6%88%B7%E6%B6%88%E6%81%AF%E6%8E%A5%E6%94%B6%E4%B8%8E%E8%BD%AE%E6%AC%A1%E6%94%B6%E9%9B%86.md) | message_id 幂等、会话锁、输入版本与收集窗口 | 主题 2；app/services/chat/message.py、app/services/chat/turn.py（均在 customer-service/atguigu 下） | 推演重复发送同一条消息，以及连续发送两条不同消息时的差别。 |
| Day04 / 06 | [day04_智能处理任务的领取调用与结果保存](../../../../docs/course-materials/originals/day04_%E5%AE%A2%E6%9C%8D%E4%BC%9A%E8%AF%9D%E7%BC%96%E7%A0%81%E4%B8%8B%EF%BC%88%E6%99%BA%E8%83%BD%E5%A4%84%E7%90%86%E4%BB%BB%E5%8A%A1%E4%B8%8E%E7%BB%93%E6%9E%9C%E4%BF%9D%E5%AD%98%EF%BC%89/2_resource/day04_%E6%99%BA%E8%83%BD%E5%A4%84%E7%90%86%E4%BB%BB%E5%8A%A1%E7%9A%84%E9%A2%86%E5%8F%96%E8%B0%83%E7%94%A8%E4%B8%8E%E7%BB%93%E6%9E%9C%E4%BF%9D%E5%AD%98.md) | 领取、快照、租约、网关、准备结果与最终结算 | 主题 3/7；customer-service/atguigu/worker/ai/worker.py、ai-service/atguigu/agent/harness/run/coordinator.py | 推演 AI 执行中用户追加输入；解释当前 confirm_run 的职责。 |
| Day05 / 07 | [day05_人工客服工单与前端状态处理](../../../../docs/course-materials/originals/day05_%E4%BA%BA%E5%B7%A5%E5%AE%A2%E6%9C%8D%E5%B7%A5%E5%8D%95%E4%B8%8E%E5%89%8D%E7%AB%AF%E7%8A%B6%E6%80%81%E5%A4%84%E7%90%86/2_resource/day05_%E4%BA%BA%E5%B7%A5%E5%AE%A2%E6%9C%8D%E5%B7%A5%E5%8D%95%E4%B8%8E%E5%89%8D%E7%AB%AF%E7%8A%B6%E6%80%81%E5%A4%84%E7%90%86.md) | 工单状态、接单并发、回复、结束服务与前端事件 | 主题 8；customer-service/atguigu/app/services/admin/handoff.py | 解释工单状态和会话模式是否一回事；沿一次接单定位锁与事务。 |
| Day06 / 08 | [day06_实时推送与监控](../../../../docs/course-materials/originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/day06_%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7.md) | Outbox、发布订阅、WebSocket、补拉与监控 | 主题 7/9；customer-service/atguigu/worker/realtime.py、app/routers/realtime.py | 推演发布成功但数据库尚未标记成功的情况，不预设恰好一次送达。 |
| Day07 / 09 | [Customer Service 项目总结与面试指南](../../../../docs/course-materials/originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md) | 客服整体链路、状态、并发和面试问答 | 主题 1–3/7–9，按问题选源码 | 用自己的话讲清完整客服链路，再回答一个失败场景；不直接照搬亮点结论。 |
| Day07 / 10 | [day07_AI_Service项目框架搭建](../../../../docs/course-materials/originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md) | AI 服务边界、Run 模型、接口与认证 | 主题 4；ai-service/atguigu/app/routers/run.py、models/models.py | 区分 Turn 与 AgentRun，找到启动、确认、取消接口和认证边界。 |
| Day08 / 11 | [day08_AI_SERVICE运行核心链路实现](../../../../docs/course-materials/originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md) | 模型适配、共享 Agent、消息编译、执行与状态映射 | 主题 4；ai-service/atguigu/agent/llm/adapter.py、agent/harness/run/context.py、executor.py | 跟踪历史裁剪和结构化输出；区分本日最小链路与后来加入的工具、技能、校验。 |
| Day09 / 12 | [day09_业务查询工具与运行上下文](../../../../docs/course-materials/originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md) | 运行上下文、令牌传递、工具契约、执行记录 | 主题 5；ai-service/atguigu/agent/harness/tools/executor.py、run/runtime.py | 追踪一个订单查询工具，从用户身份到 HTTP 请求、数据契约和调用记录。 |
| Day10 / 13 | [day10_生产级Agent_Harness校验与异常治理](../../../../docs/course-materials/originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md) | 事实/页面动作校验、错误分类、纠错与终态 | 主题 6；ai-service/atguigu/agent/harness/validator/output.py、rules/fact.py、run/executor.py | 选择一个有结构但可能缺乏事实依据的回答，推演校验和纠错分支。 |
| Day11 / 14 | [day11_生产级Harness全链路打通](../../../../docs/course-materials/originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md) | Prompt/Skill/Tool/协议、动态路由、Run 闭环与观测 | 主题 4–7；ai-service/atguigu/agent/factory.py、agent/harness/skills/middleware.py、agent/harness/run/coordinator.py | 解释 load_skill 前后工具范围变化，再串起一次 Run；指出一项尚未保证的能力。 |

## 补充材料的使用时机

- [Harness_Engineering概念与原理](../../../../docs/course-materials/originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86.md)：进入 Day07 AI 阶段，或用户问 Model、Agent、Workflow、Harness 的区别时。文中的通用机制不代表本项目全部实现。
- [Docker安装指南](../../../../docs/course-materials/originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/Docker%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97.md)：用户需要搭建课程部署环境时；先核实其操作系统、现有环境和当前目标。
- [Docker镜像加速（梯子）](../../../../docs/course-materials/originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F/Docker%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F%EF%BC%88%E6%A2%AF%E5%AD%90%EF%BC%89.md)：用户明确学习或排查镜像网络配置时；示例网络、代理与端口不是当前机器事实。

课堂白板通过项目根目录 `docs/course-materials/README.md` 找到对应 Day。图片按原文章相对位置读取；没有查看图片或白板时，不声称已经核实图中细节。

## 讲义与当前实现的核查重点

以下是已发现的教学差异线索，不是永远固定的缺陷清单。备课时按需重新读取原文和当前源码；若已变更就更新解释。非关键差异留到收尾，影响正确理解的地方简短澄清，不把本表当作每节必讲内容。

| 材料线索 | 需要核对的当前实现 | 教学处理 |
| --- | --- | --- |
| Day04「7.4 第二阶段提交业务操作」示例 `commit_run(token, run_id, input_revision)`，并描述随后调用电商写接口 | `customer-service/atguigu/worker/ai/gateway.py`、`worker.py` 与 `ai-service/atguigu/agent/harness/run/coordinator.py` 中的 `confirm_run` | 当前确认主要更新 Run 状态并返回结果；页面业务写入另有入口。明确两种设计差别，不把该阶段示例说成当前行为，也不称为数据库 2PC。 |
| Day08「1.2 实现边界」列出当日暂不实现工具、能力加载等 | 当前 `ai-service/atguigu/agent/factory.py` 及 `agent/harness/skills/`、`tools/` | 解释课程逐步构建过程；“当日未实现”不代表当前没有，当前已有也不要求在 Day08 一次讲完。 |
| Day11 动态 Skill 与「生产级」标题 | `SkillScopeMiddleware`、工具目录、运行状态与实际验证材料 | 解释单 Agent 的动态能力范围；存在规则不等于已证明生产可靠性，模型技能不等于本 Markdown 导学技能。 |
| 部署文章包含 Dockerfile / compose.yml 示例 | 当前项目实际文件、README 与各服务配置 | 先确认文件和命令适用性。文档中的代码块不能充当磁盘文件存在的证据，不默认连课程示例主机。 |
| 总结与概念文章中的能力、指标或工程优点 | 对应源码、可运行测试与观测结果 | 区分设计目标、代码机制和经验证的效果；不根据文章给出性能或可靠性保证。 |

## 主线结束后的可选小节

Day11 之后可用一个短小节介绍与旧电商客服的关键设计差别，默认只参考当前项目已有材料；用户不需要即可略过，前面各 Day 专注当前项目。

## 一课如何结束

用户回答后，针对当前小节说明已经理解的关系、需要修正的点和一项源码依据，再决定继续还是补一个追问。下一课入口细化到 Day / 文章 / 小节，不只记录“学到 AI”。只讲解未作答时，状态应是“已讲解”或“待练习”。用户点名下一个主题则立即切换，不把验收变成强制通关。
