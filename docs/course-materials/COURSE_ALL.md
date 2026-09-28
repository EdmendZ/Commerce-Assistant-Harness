# 电商小二 V2.0 · 课程文档合并版

[返回课程索引](README.md)

按 Day01–Day11 顺序汇总主讲义，文末附补充阅读。保留原文的代码、示例与结论；课程内容不等同于当前项目的运行验证。

## 阅读目录

- [01 · day01_电商客服_项目背景以及架构](#chapter-01)
- [02 · 电商后台项目部署](#chapter-02)
- [03 · day01_客服服务项目初始化](#chapter-03)
- [04 · day02_客服会话建模与分层实现](#chapter-04)
- [05 · day03_用户消息接收与轮次收集](#chapter-05)
- [06 · day04_智能处理任务的领取调用与结果保存](#chapter-06)
- [07 · day05_人工客服工单与前端状态处理](#chapter-07)
- [08 · day06_实时推送与监控](#chapter-08)
- [09 · Customer Service 项目总结与面试指南](#chapter-09)
- [10 · day07_AI_Service项目框架搭建](#chapter-10)
- [11 · day08_AI_SERVICE运行核心链路实现](#chapter-11)
- [12 · day09_业务查询工具与运行上下文](#chapter-12)
- [13 · day10_生产级Agent_Harness校验与异常治理](#chapter-13)
- [14 · day11_生产级Harness全链路打通](#chapter-14)
- [15 · Docker安装指南](#chapter-15)
- [16 · Docker镜像加速（梯子）](#chapter-16)
- [17 · Harness_Engineering概念与原理](#chapter-17)

---

<a id="chapter-01"></a>

## 01 · day01_电商客服_项目背景以及架构

> [查看本篇原文](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/2_resource/day01_%E7%94%B5%E5%95%86%E5%AE%A2%E6%9C%8D_%E9%A1%B9%E7%9B%AE%E8%83%8C%E6%99%AF%E4%BB%A5%E5%8F%8A%E6%9E%B6%E6%9E%84.md) · [返回目录](#阅读目录)

### Day 01：项目背景与架构

#### 1. 项目背景

##### 1.1 业务痛点

电商客服每天需要处理大量重复咨询，例如商品是否有货、订单当前状态、物流配送进度，以及订单能否取消或申请售后。完全依赖人工客服，会带来服务成本高、响应效率低和服务时间受限等问题。

为提高客服效率，本项目引入大模型处理重复咨询，帮助用户快速获得商品、订单、物流和售后相关信息。

但电商客服不同于普通问答。商品库存、订单状态和物流进度会不断变化，**回答必须以业务系统中的数据为准**；取消订单、修改地址和申请售后等操作还会修改业务数据，<span style="color:red">不能由大模型直接执行</span>。

因此，大模型需要与业务数据、操作约束、运行审计和人工客服配合。如果缺少这些机制，可能产生以下风险：

- 回答内容与实际库存、订单状态或物流进度不一致；
- 错误理解业务规则；如果系统没有限制 AI 的写权限，还可能绕过用户确认修改业务数据；
- 无法追踪回答依据和处理过程；
- 对于复杂或不确定的问题，如果系统不能及时识别并转接人工客服，模型可能继续给出不可靠的回答。

##### 1.2 解决方案

本项目采用 **“AI 优先、人工兜底、关键操作由用户确认”** 的解决方案：

1. Customer Service 接收并保存用户消息；
2. AI Service 理解问题，并按需查询 Ecommerce Service 或知识库；
3. AI Service 检查回复是否可靠，再返回回答、追问或操作引导；
4. 取消订单、修改地址和申请售后等操作，<span style="color:red">必须由用户在页面中确认</span>；
5. AI 无法可靠处理时请求转人工，由 Customer Service 创建工单；
6. 管理端可以处理人工工单、维护知识库并查看 AI 运行情况。

##### 1.3 核心场景

本项目覆盖四类核心场景：

1. 电商业务查询：查询商品、库存、订单、物流和售后信息；
2. 用户业务办理：用户确认后取消订单、修改地址或申请售后；
3. 智能客服：发送文本、商品或订单咨询，接收 AI 回答、追问和操作引导；
4. 人工与运营管理：人工客服接入会话，管理员维护知识库并观察 AI 运行情况。

##### 1.4 用户角色

本项目包含三类角色：

- `customer`：普通用户，浏览商品、管理自己的订单并咨询客服；
- `agent`：人工客服，接入转人工工单、查看会话并回复用户；
- `admin`：系统管理员，除人工客服能力外，还可以接管工单、维护知识库和查看系统指标。

当前使用 Demo 用户体系，由 Ecommerce Service 根据预置用户签发 JWT。

---

#### 2. 项目概览

##### 2.1 系统组成

小满电商智能客服由两个前端项目和三个后端项目组成：

- User Frontend：面向普通用户，提供商城、订单和客服功能；
- Admin Frontend：面向人工客服和管理员，提供人工工作台、知识管理和 AI 观测；
- Customer Service：管理会话、消息、AI 调度、人工工单和实时通知；
- Ecommerce Service：管理商品、订单、物流和售后业务数据；
- AI Service：负责问题理解、业务查询、知识检索、回复生成和可靠性检查。

##### 2.2 项目与端口

- `frontend/user-frontend`：用户端，默认端口 `5173`
- `frontend/admin-frontend`：客服与管理端，默认端口 `5174`
- `customer-service`：客服会话服务，默认端口 `8000`
- `ecommerce-service`：电商业务服务，默认端口 `8001`
- `ai-service`：AI 编排与知识服务，默认端口 `8002`

##### 2.3 基础设施

- PostgreSQL：三个后端服务分别保存各自负责的数据；
- Redis：发布客服消息和人工工单等实时事件；
- LLM API：为 AI Service 提供大语言模型能力；
- Embedding API：将知识文档转换为可检索的向量。

##### 2.4 业务边界

当前项目重点演示电商智能客服的完整协作流程，**不是完整电商交易平台**。以下能力尚未实现：

- 购物车、创建订单和在线支付；
- 发货管理以及库存扣减与回补；
- 售后审批和真实退款；
- 商品后台增删改；
- 正式用户注册、密码登录和刷新令牌。

Ecommerce Service 中的用户、商品和订单主要由种子数据生成，用于客服与 AI 功能联调。

---

#### 3. 项目技术栈

##### 3.1 后端基础栈

三个后端项目使用统一的 Python 技术体系：

- Python `3.12+`：后端运行环境；
- FastAPI + Uvicorn：构建和运行 HTTP API；
- Pydantic 2 + pydantic-settings：数据校验与配置管理；
- SQLAlchemy 2 + PostgreSQL + psycopg 3：数据持久化；
- PyJWT：用户身份认证。

##### 3.2 Customer 技术栈

- SQLAlchemy AsyncSession：异步访问会话、消息和任务数据；
- Redis AsyncIO：通过 Pub/Sub 发布实时事件；
- WebSocket：向用户端和管理端推送消息；
- HTTPX：AI Worker 调用 AI Service。

##### 3.3 Ecommerce 技术

- FastAPI REST API：提供商品、订单、物流和售后接口；
- SQLAlchemy Session：访问电商业务数据；
- PyJWT + 角色权限：限制不同用户的数据操作范围。

##### 3.4 AI 技术

- LangChain `1.x`：模型、Tool 调用和结构化决策；
- langchain-openai、langchain-deepseek：连接 OpenAI 兼容、DeepSeek 或 Qwen 模型接口；
- HTTPX：调用 Ecommerce Service、Embedding API 等外部接口；
- NDJSON：向 Customer Service 分阶段返回 AI 运行结果；
- PostgreSQL + pgvector：保存知识内容并执行向量检索；
- Embedding API：生成知识文本向量；
- PyPDF：解析 PDF 知识文档。

---

#### 4. 项目架构

##### 4.1 架构总览

```mermaid
flowchart TB
    subgraph actors [系统参与者]
        Customer[普通用户]
        Agent[人工客服]
        Admin[系统管理员]
    end

    subgraph frontend [前端应用]
        UserFrontend["User Frontend :5173"]
        AdminFrontend["Admin Frontend :5174"]
    end

    subgraph backend [后端服务]
        CustomerService["Customer Service :8000"]
        EcommerceService["Ecommerce Service :8001"]
        AiService["AI Service :8002"]
    end

    subgraph infrastructure [基础设施]
        CustomerDb[(Customer PostgreSQL)]
        EcommerceDb[(Ecommerce PostgreSQL)]
        AiDb[("AI PostgreSQL + pgvector")]
        Redis[(Redis Pub/Sub)]
        LlmApi[LLM API]
        EmbeddingApi[Embedding API]
    end

    Customer --> UserFrontend
    Agent --> AdminFrontend
    Admin --> AdminFrontend

    UserFrontend -->|"REST + JWT"| CustomerService
    UserFrontend -->|"WebSocket + JWT"| CustomerService
    UserFrontend -->|"REST + JWT"| EcommerceService

    AdminFrontend -->|"REST + JWT"| CustomerService
    AdminFrontend -->|"WebSocket + JWT"| CustomerService
    AdminFrontend -->|"REST + JWT"| EcommerceService
    AdminFrontend -->|"REST + JWT"| AiService

    CustomerService -->|"NDJSON + JWT + Internal Token"| AiService
    AiService -->|"只读 REST + 用户 JWT"| EcommerceService

    CustomerService --> CustomerDb
    CustomerService --> Redis
    EcommerceService --> EcommerceDb
    AiService --> AiDb
    AiService --> LlmApi
    AiService --> EmbeddingApi
```

五个项目共同组成一条完整的电商客服业务链路。

`User Frontend` 是用户入口，`Customer Service` 负责会话编排，`AI Service` 提供业务数据查询与知识问答能力，`Ecommerce Service` 保存业务事实。

`Admin Frontend` 则承接人工服务和运营管理。

整个电商系统以“**业务数据**、**知识问答**、**人工客服**”三种能力响应用户请求。



##### 4.2 五服务交互

```mermaid
flowchart LR
    UserFrontend["User Frontend"]
    CustomerService["Customer Service"]
    AiService["AI Service"]
    EcommerceService["Ecommerce Service"]
    AdminFrontend["Admin Frontend"]
    Capability{"选择处理能力"}
    Business["业务数据"]
    Knowledge["知识问答"]
    Human["人工客服"]
    KnowledgeDb["知识库"]
    AiResult["生成并检查回复"]
    Handoff["请求转人工"]

    UserFrontend -->|"发送客服消息"| CustomerService
    CustomerService -->|"AI Worker 调用"| AiService
    AiService --> Capability

    Capability -->|"查询商品、订单、物流或售后"| Business
    Business -->|"只读查询"| EcommerceService
    EcommerceService -->|"返回业务数据"| AiService

    Capability -->|"查询政策、规则或商品资料"| Knowledge
    Knowledge --> KnowledgeDb
    KnowledgeDb -->|"返回知识内容"| AiService

    Capability -->|"无法可靠处理"| Human
    Human --> Handoff
    Handoff --> CustomerService
    CustomerService -->|"创建工单并实时通知"| AdminFrontend
    AdminFrontend -->|"接入并回复"| CustomerService

    AiService --> AiResult
    AiResult -->|"返回回答、追问或操作引导"| CustomerService
    CustomerService -->|"实时推送"| UserFrontend

    UserFrontend -->|"用户确认业务操作"| EcommerceService
```

主流程体现了三种核心服务能力：

1. **业务数据**：AI Service 通过只读 Tool 查询 Ecommerce Service 中的商品、订单、物流和售后事实，再生成可验证的回答；
2. **知识问答**：AI Service 从知识库检索政策、服务规则，由模型结合知识内容生成回答；
3. **人工客服**：当问题需要人工判断或 AI 无法可靠处理时，Customer Service 创建工单并通知 Admin Frontend，由客服接入原有会话。

三条能力链路都由 **Customer Service 统一承接用户会话**。AI 回复最终通过 Customer Service 返回用户；涉及取消订单、修改地址或申请售后时，<span style="color:red">AI 只返回操作引导，不直接修改业务数据</span>，再由用户在 User Frontend 确认并直接调用 Ecommerce Service。

##### 4.3 Customer Service

整个客服系统的会话中枢：向前承接用户端和管理端请求，向后调度 AI Service，并通过 PostgreSQL、Redis 和 WebSocket 保证消息持久化与实时送达。代码按接口、服务、仓储和基础设施分层，同时运行 API、AI Worker 与 Outbox Worker 三类进程。

###### 4.3.1 运行流程

```mermaid
flowchart LR
    User["User Frontend"]
    Admin["Admin Frontend"]
    Api["Customer Service API"]
    Save["保存用户消息"]
    Mode{"会话模式"}
    Turn["创建或合并Turn"]
    Worker["AI Worker"]
    AiService["AI Service"]
    AiResult{"AI处理结果"}
    Reply["保存AI回复"]
    Handoff["创建人工工单"]
    HumanMessage["保存消息并通知客服"]
    Outbox["写入Outbox"]
    Redis["Redis Pub/Sub"]
    Realtime["WebSocket实时推送"]

    User -->|"发送客服消息"| Api
    Api --> Save
    Save --> Mode
    Save --> Outbox

    Mode -->|"AI模式"| Turn
    Turn --> Worker
    Worker -->|"调用Agent Run"| AiService
    AiService --> AiResult
    AiResult -->|"正常回答或处理失败"| Reply
    AiResult -->|"请求转人工"| Handoff

    Mode -->|"HUMAN或QUEUED"| HumanMessage
    HumanMessage --> Outbox
    Reply --> Outbox
    Handoff --> Outbox

    Outbox --> Redis
    Redis --> Realtime
    Realtime -->|"用户消息"| User
    Realtime -->|"工单事件"| Admin
    Admin -->|"接入、回复或结束"| Api
```

###### 4.3.2 核心能力

Customer Service 由三个相互配合的进程组成：

1. FastAPI API 进程负责会话、消息和人工工单的请求接入；
2. AI Worker 领取到期 Turn，调用 AI Service，并将处理结果写回消息系统；
3. Outbox Worker 读取待发布事件，通过 Redis Pub/Sub 和 WebSocket 推送给前端。

三个进程共同提供会话管理、消息持久化、AI 调度、人工接管和实时通知等核心能力。它们共享 PostgreSQL 中的业务状态，但职责彼此独立。

###### 4.3.3 小结

**Customer Service 是整个客服系统的连接中心。**它既保存完整会话和消息，也负责把用户请求交给 AI 或人工处理，并将处理结果实时推送到对应前端。

##### 4.4 Ecommerce Service

系统中的电商业务数据源，统一管理用户、商品、订单、物流与售后数据。用户端可以查询并执行受控写操作，AI Service 只能以当前用户身份通过 Tool 进行只读查询，从而将“事实查询”和“智能决策”清晰分开。

###### 4.4.1 运行流程

```mermaid
flowchart LR
    Client["前端或AI Tool"]
    Request["接收HTTP请求"]
    Auth["校验JWT与角色"]
    Validate["校验参数、用户身份及数据访问权限"]
    Rule["执行业务规则"]
    Model["查询或更新ORM模型"]
    EcommerceDb[(PostgreSQL)]
    Response["返回统一结果"]

    Client --> Request
    Request --> Auth
    Auth --> Validate
    Validate --> Rule
    Rule --> Model
    Model --> EcommerceDb
    EcommerceDb --> Response
    Response --> Client
```

###### 4.4.2 核心能力

服务持有五类核心业务数据：

- 用户：提供用户身份和角色
- 商品：保存名称、分类、价格、规格、状态和库存
- 订单与订单项：保存订单归属、金额、地址和商品明细
- 物流：保存承运公司、运单号、状态和轨迹
- 售后单：保存退款、退货或换货申请

用户端和 AI Tool 都可以发起查询，但<span style="color:red">只有用户端可以在确认后执行写操作</span>：

```mermaid
flowchart TD
    Request["收到业务请求"]
    RequestType{"只读或写入"}
    Read["查询商品订单物流或售后"]
    ReturnRead["返回业务数据"]
    CheckOwner["校验订单归属"]
    CheckState["校验订单状态"]
    Allowed{"允许操作"}
    Update["取消订单修改地址或创建售后"]
    SaveKey["保存结果"]
    ReturnWrite["返回操作结果"]
    Reject["返回业务拒绝"]

    Request --> RequestType
    RequestType -->|"只读"| Read
    Read --> ReturnRead
    RequestType -->|"写入"| CheckOwner
    CheckOwner --> CheckState
    CheckState --> Allowed
    Allowed -->|"是"| Update
    Update --> SaveKey
    SaveKey --> ReturnWrite
    Allowed -->|"否"| Reject
```



###### 4.4.3 小结

**Ecommerce Service 是系统中的业务事实来源。**它负责保存和管理业务数据，既为用户页面提供读写接口，也为 AI Tool 提供可信的只读数据；所有关键写操作仍由用户端确认后执行。

##### 4.5 AI Service

它不是一个直接面向浏览器的普通聊天接口，而是由 Customer Service 调用的 AI 编排与审计服务。该服务把模型、Tool、Skill、知识库和确定性校验组织成完整 Harness，使最终回复不仅来自 LLM，还必须经过规则验证。

###### 4.5.1 运行流程

```mermaid
flowchart TD
    Customer["Customer Service"]
    Context["用户消息与会话历史"]
    Start["接收Agent Run"]
    Understand["LLM理解用户问题"]
    Skill["识别领域，必要时加载Skill"]
    Evidence{"按需调用Tool获取信息"}
    Business["调用业务Tool"]
    Ecommerce["Ecommerce Service"]
    Knowledge["调用知识检索Tool"]
    AiDb["PostgreSQL（启用 pgvector）"]
    Generate["生成AgentDecision"]
    Validate["检查回答是否可信、操作是否合规"]
    Result{"处理结果"}
    Answer["回答、追问、无法处理或引导用户操作"]
    Handoff["REQUEST_HANDOFF"]

    Customer --> Context
    Context --> Start
    Start --> Understand
    Understand --> Skill
    Skill --> Evidence

    Evidence -->|"业务数据"| Business
    Business --> Ecommerce
    Ecommerce --> Generate

    Evidence -->|"知识问答"| Knowledge
    Knowledge --> AiDb
    AiDb --> Generate

    Evidence -->|"无需查询"| Generate

    Generate --> Validate
    Validate --> Result
    Result -->|"正常处理"| Answer
    Result -->|"转人工"| Handoff

    Answer --> Customer
    Handoff --> Customer
```

AI Service 接收 Customer Service 提交的用户消息和会话历史，使用领域 Skill 辅助处理问题，并按需通过 Tool 查询电商数据或知识库，最后检查回复是否可靠：能够处理时返回回答、追问或操作引导；无法可靠处理时通知 Customer Service 转接人工客服。

###### 4.5.2 核心能力

AI Service 主要提供以下能力：

1. 使用 Skill 为商品、订单、物流和售后问题提供处理指导
2. 使用 Tool 查询 `Ecommerce Service` 中的业务数据
3. 从知识库检索平台服务政策
4. 检查回答是否可信、操作引导是否合规
5. 无法可靠处理时请求转人工客服

AI Service 会记录每次 AI 运行和 Tool 调用，方便管理员查看处理过程和排查问题。Knowledge Worker 负责在后台处理上传的知识文档，并将文档转换为可检索的知识内容。

###### 4.5.3 小结

**AI Service 是系统的智能处理中心。**它负责理解问题、查询信息、生成回复和判断是否需要转人工。它只读取 Ecommerce Service 中的业务数据，<span style="color:red">不直接取消订单、修改地址或申请售后</span>；这些操作仍需用户在 User Frontend 中确认。

##### 4.6 User Frontend

它是普通用户访问商城和客服系统的统一入口，一侧连接 Customer Service 完成会话与实时消息，另一侧连接 Ecommerce Service 完成登录、业务查询和用户确认后的写操作。

###### 4.6.1 用户流程

```mermaid
flowchart TD
    Open["用户打开应用"]
    Login["选择Demo用户并登录"]
    EcommerceAuth["Ecommerce Service签发JWT"]
    Page{"选择功能"}
    Mall["进入商城"]
    Products["浏览商品"]
    ConsultProduct["咨询商品"]
    Me["进入我的"]
    Orders["查看订单列表"]
    OrderDetail["查看订单、物流与售后"]
    OrderChoice{"选择订单操作"}
    ConsultOrder["咨询订单"]
    OrderAction["取消订单、修改地址或申请售后"]
    Chat["发起客服咨询"]
    LoadChat["加载会话与历史消息"]
    Send["发送文本或对象消息"]
    CustomerApi["Customer Service接收消息"]
    Realtime["WebSocket接收实时事件"]
    EventType{"实时事件类型"}
    Message["展示客服消息"]
    State["更新客服状态：AI处理中、等待人工或人工服务"]
    ActionCheck{"消息是否包含操作引导"}
    Action["跳转操作页面"]
    Continue["继续咨询或结束"]
    Confirm["用户确认操作"]
    EcommerceWrite["Ecommerce Service执行写操作"]

    Open --> Login
    Login --> EcommerceAuth
    EcommerceAuth --> Page
    Page -->|"商城"| Mall
    Mall --> Products
    Products --> ConsultProduct
    ConsultProduct --> Chat
    Page -->|"我的"| Me
    Me --> Orders
    Orders --> OrderDetail
    OrderDetail --> OrderChoice
    OrderChoice -->|"咨询订单"| ConsultOrder
    ConsultOrder --> Chat
    OrderChoice -->|"办理业务"| OrderAction
    OrderAction --> Confirm
    Page -->|"直接进入客服"| Chat
    Chat --> LoadChat
    LoadChat --> Send
    Send --> CustomerApi
    CustomerApi --> Realtime
    Realtime --> EventType
    EventType -->|"客服消息"| Message
    Message --> ActionCheck
    ActionCheck -->|"否"| Continue
    ActionCheck -->|"是"| Action
    EventType -->|"客服状态"| State
    State --> Continue
    Action --> Confirm
    Confirm --> EcommerceWrite
    EcommerceWrite --> OrderDetail
```

###### 4.6.2 核心能力

User Frontend 主要提供以下能力：

1. 选择用户并完成登录
2. 浏览商城商品，并携带商品信息发起咨询
3. 查看订单、物流和售后信息，并携带订单信息发起咨询
4. 发送客服消息，查看会话历史
5. 实时接收客服回复和处理状态
6. 根据客服引导确认取消订单、修改地址或申请售后。

###### 4.6.3 小结

**User Frontend 是普通用户的统一入口。**它同时承担电商页面和客服入口，但不直接调用 AI Service。聊天消息统一经过 Customer Service，关键业务写操作则必须回到前端页面由用户确认，再提交给 Ecommerce Service。

##### 4.7 Admin Frontend

它是人工客服与管理员共用的运营入口，通过 Customer Service 处理人工工单，通过 AI Service 管理知识和观察 Agent Run，并通过 Ecommerce Service 完成登录和查看基础业务摘要。

###### 4.7.1 管理流程

```mermaid
flowchart TD
    Open["客服或管理员打开应用"]
    Login["选择员工并登录"]
    EcommerceAuth["Ecommerce Service签发JWT"]
    Section{"选择管理模块"}
    Handoffs["人工工作台"]
    CustomerApi["查询Customer Service工单"]
    Accept["接入、回复或结束工单"]
    Realtime["WebSocket接收工单事件"]
    Knowledge["知识管理"]
    Upload["上传知识文档"]
    AiKnowledge["AI Service创建索引任务"]
    Observe["AI观测"]
    Runs["查看Run、Tool与指标"]
    Evaluate["管理员触发评估"]
    BusinessSummary["业务数据摘要"]
    EcommerceSummary["查看商品、订单与售后数量"]

    Open --> Login
    Login --> EcommerceAuth
    EcommerceAuth --> Section
    Section -->|"人工工作台"| Handoffs
    Handoffs --> CustomerApi
    CustomerApi --> Accept
    Accept --> Realtime
    Realtime --> Handoffs
    Section -->|"知识管理"| Knowledge
    Knowledge --> Upload
    Upload --> AiKnowledge
    Section -->|"AI观测"| Observe
    Observe --> Runs
    Runs -->|"管理员可选"| Evaluate
    Section -->|"业务数据"| BusinessSummary
    BusinessSummary --> EcommerceSummary
```

###### 4.7.2 核心能力

Admin Frontend 主要提供以下能力：

1. 选择客服或管理员账号并完成登录
2. 查看人工工单，执行接入、回复、结束或管理员接管
3. 实时接收新工单和会话消息
4. 上传和查看知识文档
5. 查看 Agent Run、Tool 调用和运行指标
6. 触发 AI 评估，并查看商品、订单和售后数量摘要。

###### 4.7.3 小结

**Admin Frontend 是人工客服与管理员的统一工作台。**它把人工服务、知识运营和 AI 可观测性集中在一个管理界面中。`agent` 主要处理工单并查看运行信息，`admin` 还可以上传知识、接管工单和触发评估。

##### 4.8 架构原则

1. **职责分离**：Ecommerce Service 管理业务数据，Customer Service 管理会话与消息，AI Service 负责智能处理，两个前端分别服务用户和客服人员；
2. **事实驱动**：AI 通过 Tool 查询真实电商数据或知识库，并检查回复是否可靠，避免仅依赖模型自身生成答案；
3. **写操作受控**：AI 只查询业务数据，不直接取消订单、修改地址或申请售后，关键操作必须由用户确认；
4. **AI 优先、人工兜底**：常见问题由 AI 自动处理，无法可靠回答时转接人工客服，并保留原有会话内容；
5. **异步解耦与实时通知**：耗时的 AI 处理和事件发布由 Worker 完成，前端通过 WebSocket 实时接收回复与工单状态。

##### 4.9 本章小结

从整体上看，五个项目不是简单地并列部署，而是围绕一次客服请求形成协作闭环：User Frontend 提供用户入口，Customer Service 维持会话并协调实时消息，AI Service 完成推理与校验，Ecommerce Service 提供可信业务数据，Admin Frontend 则处理人工接管、知识维护和运行观测。

这种结构把界面交互、会话状态、AI 能力和电商事实分配给不同项目。每个项目只处理自身边界内的职责，又通过 JWT、REST、 WebSocket 串联起来，从而形成“AI 自动处理、人工及时兜底、业务操作受控执行”的完整客服流程。

---

#### 5. 数据与异步任务

##### 5.1 Customer Service 数据

Customer Service 使用独立数据库，核心表包括：

- `conversations`
- `messages`
- `conversation_turns`
- `handoffs`
- `realtime_outbox`

Redis 仅用于 Pub/Sub，不作为会话或业务数据缓存。

##### 5.2 AI Service 数据

AI Service 使用独立 PostgreSQL，并启用 pgvector，核心表包括：

- `agent_runs`
- `agent_tool_calls`
- `agent_conversation_cursors`
- `knowledge_jobs`
- `knowledge_documents`
- `knowledge_chunks`

AI Service 不负责保存用户会话和聊天消息，这些数据由 Customer Service 管理。AI Service 只保存每次 AI 处理的运行记录、Tool 调用记录和知识库数据，方便后续查询与审计。

##### 5.3 Ecommerce Service 数据

Ecommerce Service 使用独立数据库，核心表包括：

- `users`
- `products`
- `orders`
- `order_items`
- `logistics`
- `after_sales`

##### 5.4 异步机制

项目没有引入 Kafka、RabbitMQ 等独立消息队列，而是使用 PostgreSQL、Worker 和 Redis 完成三类后台任务：

1. **AI 消息处理**

   用户消息 → Turn 任务 → AI Worker → AI Service

   Customer Service 将需要 AI 处理的消息保存为 Turn，由 AI Worker 在后台领取并调用 AI Service，避免用户请求一直等待 AI 完成。

2. **实时消息通知**

   业务事件 → Outbox → Outbox Worker → Redis → WebSocket

   用户消息、AI 回复和人工工单等事件先写入 Outbox，再由 Worker 发布到 Redis，最后通过 WebSocket 通知用户端或管理端。

3. **知识文档处理**

   上传文档 → Knowledge Job → Knowledge Worker → 知识库

   AI Service 将上传的文档保存为后台任务，由 Knowledge Worker 完成解析、分块和向量化，避免上传请求长时间阻塞。

---

#### 6. 通信与安全

##### 6.1 通信方式

五个项目主要使用三种通信方式：

1. **REST**：用于登录、商品和订单查询、业务写操作、知识管理与人工工单操作；
2. **流式 JSON（NDJSON）**：用于 AI Service 分阶段返回处理结果，Customer Service 的 AI Worker 可以逐条接收并处理；
3. **WebSocket**：用于向用户端和管理端实时推送客服消息、处理状态和人工工单事件。

User Frontend 不直接调用 AI Service，用户聊天统一经过 Customer Service，再由 AI Worker 调用 AI Service。Admin Frontend 仅在知识管理和运行观测等管理场景中直接调用 AI Service。

##### 6.2 认证与权限

- Ecommerce Service 负责签发演示账号 JWT；
- 用户端和管理端携带 JWT 调用后端接口；
- 三个后端使用相同的 JWT 配置识别用户身份；
- Customer Service 调用 AI Service 时额外携带内部服务令牌；
- AI Service 使用当前用户身份查询 Ecommerce Service；
- 系统包含三种角色，不同角色只能访问与自身职责对应的功能：
  - `customer`：普通用户，可以浏览商品、查看自己的订单、咨询客服和确认业务操作；
  - `agent`：人工客服，可以接入人工工单、回复用户和结束人工服务；
  - `admin`：系统管理员，可以接管人工工单、上传知识文档、查看系统指标和触发 AI 评估。

##### 6.3 可靠性

系统通过以下机制减少重复处理和消息丢失：

1. 消息和业务写操作使用唯一标识，避免重复提交；
2. Worker 领取后台任务，避免同一任务被重复处理；
3. AI 运行使用版本检查，避免旧回复覆盖用户的新消息；
4. 业务事件先写入 Outbox，再发布到 Redis。

---

#### 7. 总结

小满电商智能客服由两个前端和三个后端组成：

- Ecommerce Service 保存商品、订单、物流和售后事实；
- Customer Service 管理会话、消息、AI Turn、人工接管和实时推送；
- AI Service 负责模型推理、只读工具、知识库、校验与审计；
- User Frontend 承载用户商城、订单和客服体验；
- Admin Frontend 承载人工接待、知识管理与 AI 观测。

系统最重要的主链路是：

```mermaid
flowchart LR
    UserMessage["用户发送消息"]
    CustomerService["Customer Service"]
    Turn["创建Turn"]
    AiWorker["AI Worker"]
    AiService["AI Service"]
    Source{"查询信息"}
    Business["Ecommerce Service业务数据"]
    Knowledge["知识库"]
    Generate["生成并检查回复"]
    Save["保存回复并写入Outbox"]
    Redis["Redis Pub/Sub"]
    WebSocket["WebSocket"]
    UserFrontend["User Frontend"]

    UserMessage --> CustomerService
    CustomerService --> Turn
    Turn --> AiWorker
    AiWorker -->|"调用"| AiService
    AiService --> Source
    Source -->|"业务查询"| Business
    Source -->|"知识问答"| Knowledge
    Business --> Generate
    Knowledge --> Generate
    Generate --> CustomerService
    CustomerService --> Save
    Save --> Redis
    Redis --> WebSocket
    WebSocket -->|"实时推送"| UserFrontend
```

当 AI 无法可靠处理时，系统进入：

```mermaid
flowchart LR
    AiService["AI Service请求转人工"]
    CustomerService["Customer Service"]
    Handoff["创建Handoff工单"]
    AdminFrontend["Admin Frontend"]
    Agent["人工客服接入"]
    Reply["人工客服回复"]
    UserFrontend["User Frontend接收回复"]
    Finish["结束人工服务"]
    AiMode["恢复AI模式"]

    AiService --> CustomerService
    CustomerService --> Handoff
    Handoff -->|"实时通知"| AdminFrontend
    AdminFrontend --> Agent
    Agent --> Reply
    Reply --> CustomerService
    CustomerService -->|"实时推送"| UserFrontend
    Agent --> Finish
    Finish --> CustomerService
    CustomerService --> AiMode
```

这一架构的核心思想是：**让业务系统提供事实，让 AI 负责理解与编排，让用户确认关键写操作，让人工客服处理不确定问题。**

---

<a id="chapter-02"></a>

## 02 · 电商后台项目部署

> [查看本篇原文](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/2_resource/%E7%94%B5%E5%95%86%E5%90%8E%E5%8F%B0%E9%A1%B9%E7%9B%AE%E9%83%A8%E7%BD%B2.md) · [返回目录](#阅读目录)

### ecommerce-service 部署文档

> 将 `ecommerce-service`（电商业务服务）通过 Docker 容器部署到虚拟机。部署完成后，本地的 AI Service、User Frontend 和 Admin Frontend 可通过虚拟机 IP + 端口调用电商接口。

---

#### 1、部署架构概览

只部署电商业务服务及其依赖数据库：

| 子项目 | 部署位置 | 说明 |
|--------|----------|------|
| `customer-service`（客服会话服务） | 本地开发 | 会话、消息、人工工单 |
| `ai-service`（AI 编排服务） | 本地开发 | 通过 Tool 调用电商只读接口 |
| `ecommerce-service`（电商业务服务） | **虚拟机 Docker** | 商品、订单、物流、售后与 Demo JWT |

部署后，本地服务通过 `http://<虚拟机IP>:8001` 调用电商服务接口。

当前 Compose 中还会启动 Redis，主要用于后续 Customer Service 联调；**Ecommerce Service 自身只依赖 PostgreSQL**。

---

#### 2、需要准备的文件

部署前，在 `ecommerce-service` 根目录下需要有以下文件：

```
ecommerce-service/
├── ecommerce_service/          # 业务代码（FastAPI 入口：main.py）
├── postgres/init.sql           # 初始化 ecommerce / customer_service / ai 三个服务库
├── pyproject.toml              # 已有
├── uv.lock                     # 已有
├── .env                		# 已有
├── Dockerfile                  # 已有
├── compose.yml                 # 已有
```

##### 2.1 Dockerfile

```dockerfile
# Python 3.12 轻量镜像
FROM docker.1ms.run/library/python:3.12-slim

WORKDIR /app

# 安装 uv
RUN pip install uv -i https://mirrors.aliyun.com/pypi/simple/

ENV UV_INDEX_URL=https://mirrors.aliyun.com/pypi/simple/

# 先复制依赖清单，利用分层缓存
COPY pyproject.toml uv.lock ./

# 仅安装生产依赖，并严格按 lock 文件安装
RUN uv sync --frozen --no-dev

# 再复制业务代码
COPY . .

EXPOSE 8001

# 直接使用已同步的虚拟环境，避免运行时再次安装开发依赖
CMD [".venv/bin/uvicorn", "ecommerce_service.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

##### 2.2 compose.yml

```yaml
name: ecommerce-customer-service

services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: ecommerce-cs-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-customer_service}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-customer_service}
      POSTGRES_DB: ${POSTGRES_DB:-ecommerce}
    ports:
      - "5432:5432"
    volumes:
      - ecommerce_cs_postgres:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:  
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-customer_service} -d ${POSTGRES_DB:-ecommerce}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.4-alpine
    container_name: ecommerce-cs-redis
    ports:
      - "6379:6379"
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - ecommerce_cs_redis:/data

  app:
    build: ./ecommerce-service
    container_name: ecommerce-app
    restart: always
    ports:
      - "8001:8001"
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      ECOM_DATABASE_URL: postgresql+psycopg://customer_service:customer_service@postgres:5432/ecommerce
      JWT_SECRET: ${JWT_SECRET:-ecommerce-secret}
      JWT_ALGORITHM: ${JWT_ALGORITHM:-HS256}

volumes:
  ecommerce_cs_postgres:
  ecommerce_cs_redis:
```

这两个文件里藏着不少部署细节，下面两节分别拆开讲。

---

#### 3、Dockerfile 解析

##### 3.1 分层缓存

```dockerfile
COPY pyproject.toml uv.lock ./    # 先复制依赖清单
RUN uv sync --frozen --no-dev     # 再装依赖
COPY . .                          # 最后复制业务代码
```

Docker 镜像分层构建，每一层带缓存，但缓存是**链式依赖**的：只有在它上面所有步骤都命中缓存的前提下，当前层的输入文件或命令字面没变，才会复用缓存结果——任何一层缓存失效，它下面所有层的缓存全部作废，必须重建。

再看为什么要拆成两次 COPY：

- 业务代码天天改，依赖清单很少改。
- 如果先 `COPY . .` 再 `RUN uv sync`，业务代码一变，第一层就失效，连带下面装依赖那层也失效——每改一行代码都要把全部依赖重新装一遍。
- 拆开后，把 `pyproject.toml` 和 `uv.lock` 单独前置：只要这两个文件没动，前面那层就命中缓存，装依赖那层跟着命中，直接跳过；只有最后复制业务代码那层需要重建。

所以，写 Dockerfile 的黄金法则是：**最稳定的步骤永远放最上面，最不稳定的步骤永远放最下面。**

##### 3.2 依赖锁定

- **`--frozen`**：严格按 `uv.lock` 里记录的精确版本安装，禁止 uv 自动升级或修改 lock 文件，保证本地与生产环境依赖版本完全一致。
- **`--no-dev`**：跳过 `ruff`、`httpx2` 等开发依赖，镜像更小、更安全。

##### 3.3 COPY 语义

- 第一个 `.`：**宿主机构建镜像的上下文目录**（在 Compose 中由 `build: .` 指定）
- 第二个 `.`：容器内当前工作目录（由 `WORKDIR /app` 指定，即 `/app`）

即把宿主机当前目录下的所有文件复制进容器的 `/app`。

##### 3.4 启动命令

```dockerfile
CMD [".venv/bin/uvicorn", "ecommerce_service.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

- 模块路径是 `ecommerce_service.main:app`
- `--host 0.0.0.0` 才能让容器外访问
- 端口统一为当 `8001`

---

#### 4、compose.yml 解析

`compose.yml` 把 PostgreSQL、Redis 和电商应用编排在一份文件里，几个关键设计决定了镜像怎么来、容器之间能不能互通、数据会不会丢。

##### 4.1 build 指令

```yaml
app:
  build: ./ecommerce-service
```

`build` 告诉 Compose：`app` 的镜像不从仓库拉，而是**就地构建**——构建上下文是当前目录下的ecommerce-service目录，Compose 会去该目录下找 `Dockerfile` 并按它构建镜像。

- 构建上下文决定了 `Dockerfile` 里 `COPY . .` 中第一个 `.` 指向哪：这里就是 `ecommerce-service/`
- 只有首次启动或代码变动后执行 `docker compose up -d --build` 才会触发构建；镜像已存在时 `docker compose up -d` 会直接复用

##### 4.2 服务名即主机名

`app` 服务的数据库地址里主机名直接写的是 `postgres`，而不是某个 IP：

```yaml
ECOM_DATABASE_URL: postgresql+psycopg://customer_service:customer_service@postgres:5432/ecommerce
```

Docker Compose 启动时会自动建一个默认网络，同一份 compose 文件下的服务可以通过服务名互相访问——`postgres` 这个服务名在容器网络里就是主机名。所以 `app` 容器连数据库走的是 `postgres:5432`，**无需任何 IP**，也不用在配置里硬编码虚拟机地址。

##### 4.3 启动依赖与健康检查

```yaml
app:
  depends_on:
    postgres:
      condition: service_healthy
```

这里不只是“先启动 postgres”，而是等 PostgreSQL 健康检查通过后再启动 `应用`，减少“库还没就绪、应用先连库失败”的情况。



##### 4.4 持久化与自动初始化

两件事都靠 `volumes` 解决：

- `ecommerce_cs_postgres:/var/lib/postgresql/data`：把数据库文件挂到命名卷，容器删了数据还在，重启不丢
- `./postgres/init.sql:/docker-entrypoint-initdb.d/01-init.sql`：PostgreSQL 官方镜像在**数据卷为空（首次启动）**时会自动执行初始化脚本

`postgres/init.sql` 会创建：

- `ecommerce`：电商业务库
- `customer_service`：客服会话库
- `ai_customer_service`：AI 知识库，并启用 `vector` 扩展

> 这也是为什么改了 SQL 初始化脚本却不生效：PostgreSQL 只在卷为空时跑一次。要让新脚本生效，得先 `docker compose down -v` 删掉数据卷再重新启动。

`应用` 服务设了 `restart: always`，容器异常退出或虚拟机重启后 Docker 会自动把它拉起来。

---

#### 5、部署操作流程

##### 步骤 1：本地打包

> ⚠️ **打包前务必删除（或排除）`.venv` 目录**。它体积大、文件多，且是 Windows 下生成的，在 Linux 虚拟机上完全无法使用。

1. 找到 `ecommerce-service` 文件夹。
2. 确认已包含 `Dockerfile`、`.dockerignore`、`compose.yml`、`postgres/init.sql`。
3. 右键 → “压缩为 ZIP 文件”，生成 `ecommerce-service.zip`。

##### 步骤 2：上传到虚拟机

使用 SFTP 客户端（Xftp 等）将 zip 包上传到虚拟机目录，例如：

```text
/root/web/
```

##### 步骤 3：登录虚拟机并解压

```bash
cd /root/web/
unzip ecommerce-service.zip
cd ecommerce-service
```

##### 步骤 4：构建并启动

```bash
docker compose up -d --build
```

这条命令的含义：

| 部分 | 作用 |
|------|------|
| `docker compose` | 启用 Docker Compose，自动读取当前目录的 `compose.yml` |
| `up` | 创建并启动所有服务（容器、网络、数据卷） |
| `-d` | 后台运行，关闭终端服务不停 |
| `--build` | 强制根据 `Dockerfile` 构建镜像 |

> **执行顺序补充**：Docker Compose 先读取 `compose.yml` 这张“总图纸”，解析到 `build: .` 时才会触发 `Dockerfile` 构建。即 compose 文件是大管家，Dockerfile 是它临时调用的“制造说明书”。

##### 步骤 5：验证部署状态

**① 查看容器运行状态**

```bash
docker compose ps
```

或：

```bash
docker ps
```

应当看到 `ecommerce-app`、`ecommerce-cs-postgres`、`ecommerce-cs-redis` 三个容器都处于 `Up` 状态，且 `app`、`postgres` 健康检查为 healthy。

**② 查看 API 实时日志**

```bash
docker compose logs -f app
```

看到类似 `Application startup complete` 或 `Uvicorn running on http://0.0.0.0:8001` 就说明启动成功。按 `Ctrl+C` 退出日志查看（不会停止服务）。

**③ 健康检查**

```bash
curl http://127.0.0.1:8001/health
```

期望返回：

```json
{"status":"ok","service":"ecommerce-service"}
```

##### 步骤 6：开放虚拟机防火墙端口

为了让本地能访问，需要放行 `8001` 端口（如需直连数据库调试，再放行 `5432`）：

**CentOS：**

```bash
sudo firewall-cmd --zone=public --add-port=8001/tcp --permanent
sudo firewall-cmd --reload
```

**Ubuntu：**

```bash
sudo ufw allow 8001/tcp
sudo ufw reload
```

如果是云服务器（阿里云、腾讯云等），还要在云控制台的“安全组”里放行 `8001` 端口。

##### 步骤 7：本地访问验证

打开本地浏览器：

```text
http://<虚拟机IP>:8001/docs
```

如果能看到 **E-commerce Service** 的 Swagger 接口文档界面（包含 `/health`、`/api/v1/products`、`/api/v1/orders`、`/api/v1/auth/demo-token` 等接口），说明部署完全成功。

---

#### 6、修改本地依赖服务的配置

部署成功后，回到本地相关项目：

##### 6.1 AI Service

打开 `ai-service-v1.5/.env`，把电商服务地址改为虚拟机地址：

```text
# 旧值
ECOM_BASE_URL=http://127.0.0.1:8001/api/v1

# 新值
ECOM_BASE_URL=http://<虚拟机IP>:8001/api/v1
```

##### 6.2 前端

User Frontend 和 Admin Frontend 中如果写死了电商服务地址，同样改为：

```text
http://<虚拟机IP>:8001
```

并保证三个后端共享相同的 `JWT_SECRET` / `JWT_ALGORITHM`，否则 Demo 登录签发的 JWT 无法被其他服务识别。

重启本地 AI Service 和前端后，电商接口请求会自动路由到虚拟机的 Docker 容器。

---

#### 7、Docker 常用命令

| 命令 | 用途 |
|------|------|
| `docker compose ps` | 查看本项目所有服务状态 |
| `docker compose logs -f app` | 实时查看电商应用日志 |
| `docker compose logs -f postgres` | 实时查看 PostgreSQL 日志 |
| `docker compose restart app` | 只重启电商应用容器 |
| `docker compose down` | 停止并删除所有容器（保留数据卷） |
| `docker compose down -v` | 停止并删除所有容器**及数据卷**（数据库会被清空重建） |
| `docker compose up -d --build` | 重新构建并启动（代码改动后用） |
| `docker exec -it ecommerce-cs-postgres psql -U customer_service -d ecommerce` | 进入 PostgreSQL 交互式登录 |
| `docker exec -it ecommerce-app /bin/bash` | 进入应用容器查看文件 |

---

<a id="chapter-03"></a>

## 03 · day01_客服服务项目初始化

> [查看本篇原文](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/2_resource/day01_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96.md) · [返回目录](#阅读目录)

### Day01：客服服务项目初始化

#### 1. 学习目标

##### 1.1 任务目标

今天完成客服服务的项目初始化，并打通一条**最小可用链路**：

1. 用 uv 建立 Python 项目，并梳理分层目录；
2. 完成配置管理与异步数据库接入；
3. 启动接口服务；
4. 实现访问令牌认证；
5. 实现“获取当前会话”接口：最小响应模型、服务示例、路由注册与依赖注入。

完成本节后，项目应能启动，认证逻辑可独立使用，会话接口可返回最小示例数据。

---

#### 2. 核心概念

动手写代码前，先扫一遍今天会反复出现的几个概念。后面章节只讲它们在本项目中的落点，不再重复解释原理。

##### 2.1 访问令牌

访问令牌是一段可校验的身份凭证。本项目使用 **JWT**：把用户信息编码进一段字符串，服务端用密钥校验真伪，无需再查登录态。

前端或内部服务在请求头里携带：

```text
Authorization: Bearer <token>
```

令牌解码后通常得到用户编号和角色。<span style="color:red">客服服务不负责用户登录发号，但要能解析令牌，并按角色决定能否访问接口。</span>今天的认证模块围绕这件事展开。

##### 2.2 事件循环

Python 异步程序依赖**事件循环**调度协程。<span style="color:red">Windows 默认事件循环与异步数据库驱动不完全兼容</span>，直接启动协程访问 PostgreSQL 可能出错。

因此项目统一创建选择器事件循环：

```python
asyncio.SelectorEventLoop(selectors.SelectSelector())
```

封装成统一启动方法后，服务入口和建表命令都走同一套方式，保证异步数据库可在 Windows 下稳定工作。

##### 2.3 服务启动器

接口服务需要 ASGI 服务器承接 HTTP 请求。今天不使用命令行一键启动，而是在代码里创建服务启动器：

```python
uvicorn.Server(uvicorn.Config(...))
```

关键点是 <span style="color:red">`loop="none"`</span>：让服务器复用当前已创建的事件循环，而不是自己再开一套。这样“事件循环兼容”和“服务启动”才能接到一起。

##### 2.4 配置与注入

- 配置映射：把环境文件映射成配置对象
- 依赖注入：路由按需获取认证服务、会话服务

后面进入项目结构时，可以按“配置 → 数据库 → 启动 → 认证 → 会话”的顺序，对照这些概念落在哪些文件。

---

#### 3. 目录结构

带着上面的概念看代码组织。客服服务按“**入口 → 应用层 → 公共能力 → 基础设施 → 模型**”分层。先看整体，再看各层职责。

##### 3.1 整体结构

```text
customer-service/
├── .env
├── pyproject.toml
└── atguigu/
    ├── main.py                 # 进程启动入口
    ├── common/                 # 配置与通用工具
    ├── infrastucture/          # 数据库等基础设施(后续完善)
    ├── models/                 # 数据模型
    └── app/
        ├── app.py              # 应用实例
        ├── dependencies.py     # 依赖注入组装
        ├── schemas/            # 请求与响应模型
        ├── services/           # 业务服务
        ├── routers/            # 接口路由
        └── repositories/       # 数据访问（后续完善）
```

##### 3.2 各层职责

- `common`：配置读取、事件循环等跨层工具
- `infrastucture`：数据库引擎、会话工厂、建表命令
- `models`：表结构定义
- `app/schemas`：接口输入输出结构
- `app/services`：业务规则与流程编排
- `app/routers`：接口入口
- `app/dependencies`：把服务组装给路由使用
- `main.py`：统一启动接口进程

请求处理顺序通常是：

```text
路由 → 服务 → 仓储/数据库
```

今天先完成**路由、服务、认证与基础设施**，仓储层后续再补。

---

#### 4. 配置管理

服务启动前先加载配置，避免在代码中散落硬编码。

##### 4.1 配置模型

`atguigu/common/config.py` 定义配置类，主要字段包括：

- 数据库连接
- 令牌密钥与算法
- 服务监听地址
- 前端跨域来源
- 后续能力预留项

配置类通过项目根目录的**绝对路径**读取 `.env`，未识别字段忽略。<span style="color:red">这样无论从哪个目录启动，都能找到配置文件。</span>

##### 4.2 环境变量

当前本地配置示例：

```env
DATABASE_URL=postgresql+psycopg://customer_service:customer_service@127.0.0.1:5432/customer_service
JWT_SECRET=ecommerce-secret
JWT_ALGORITHM=HS256
API_HOST=0.0.0.0
API_PORT=8000
CORS_ORIGINS=["http://localhost:5173","http://127.0.0.1:5173","http://localhost:5174","http://127.0.0.1:5174"]
```

<span style="color:red">注意：令牌密钥必须与电商服务、AI 服务保持一致，否则跨服务令牌无法互认。</span>

##### 4.3 配置读取

```python
ENV_FILE = Path(__file__).resolve().parents[2] / ".env"

model_config = SettingsConfigDict(env_file=ENV_FILE, extra="ignore")

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`ENV_FILE` 指向项目根目录下的 `.env`，**不依赖当前工作目录**。`get_settings()` 带缓存，进程内复用同一实例。认证服务、数据库引擎和启动入口都通过它读取配置。

---

#### 5. 数据库接入

配置就绪后，接入 PostgreSQL。客服服务使用异步数据访问，因此还要接上前面的事件循环方案。

##### 5.1 引擎与会话

`atguigu/infrastucture/db.py` 创建：

- 异步引擎
- 异步会话工厂
- 模型基类

```python
db_engine = create_async_engine(
    get_settings().database_url,
    pool_pre_ping=True,
)
SessionFactory = async_sessionmaker(
    bind=db_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

连接池预检用于减少连接失效后的请求失败。

##### 5.2 会话获取

所有数据模型继承统一基类。请求级数据库访问通过会话获取函数提供：

```python
class Base(DeclarativeBase):
    """ORM 数据库模型的声明性基类。

    所有业务实体的 SQLAlchemy 模型均继承此类。
    """

    pass

async def get_db():
    async with SessionFactory() as db:
        try:
            yield db
        except Exception:
            await db.rollback()
            raise
```

发生异常时回滚，避免脏事务残留。

##### 5.3 事件循环接入

第 2 章的事件循环，在项目里落在 `common/event_loop.py`

```python
def run_async(coroutine):
    return asyncio.run(
        coroutine,
        loop_factory=lambda: asyncio.SelectorEventLoop(
            selectors.SelectSelector()
        ),
    )
```

建表命令通过它执行；下一章启动接口时也会复用。

##### 5.4 建表命令

`atguigu/infrastucture/init_db.py` 负责按模型创建表

执行流程：

1. 导入模型模块，让表定义注册到元数据
2. 按元数据创建表结构

当前模型文件仍为空。命令可以执行，但还不会创建业务表；表结构随后续课程补齐。

---

#### 6. 应用入口

基础设施准备好后，把接口服务跑起来。

##### 6.1 应用实例

`atguigu/app/app.py` 创建应用实例，并注册路由与跨域：

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from atguigu.app.routers.chat import conversation
from atguigu.common.config import get_settings

app = FastAPI(description="电商智能客服")
app.include_router(conversation.router)

app.add_middleware(
    CORSMiddleware,
    allow_origins=get_settings().cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**跨域**：浏览器不允许前端页面随便请求另一个源（协议/域名/端口不同）的接口。  
**解决**：在后端加 CORS 中间件，把允许访问的前端地址写进 `cors_origins`。

另外三项含义：

- **`allow_credentials=True`**：允许携带 Cookie、Authorization 等凭证
- **`allow_methods=["*"]`**：允许前端使用的所有 HTTP 方法
- **`allow_headers=["*"]`**：允许前端携带的所有请求头

##### 6.2 进程启动

`atguigu/main.py` 把第 2 章的服务启动器落到启动流程里：

```python
server = uvicorn.Server(
    uvicorn.Config(
        "atguigu.app.app:app",
        host=settings.api_host,
        port=settings.api_port,
        loop="none",
    )
)
await server.serve()
```

外层仍用统一异步启动方法，因此接口进程和数据库初始化<span style="color:red">共用同一套事件循环策略</span>。

##### 6.3 启动验证

服务启动后访问：

```text
http://127.0.0.1:8000/docs
```

能打开接口文档页面，说明入口链路已通。接下来实现认证，并挂上第一个业务接口。

---

#### 7. 认证模块

服务能启动后，下一步让接口认得“谁在调用”。认证模块按“**当前用户 → 认证服务 → 依赖注入**”组织，落实第 2 章的访问令牌概念。

##### 7.1 当前用户

`atguigu/app/schemas/auth.py` 定义令牌解码后的用户结构：

```python
class CurrentUser(BaseModel):
    user_id: str
    role: Literal["customer", "agent", "admin"] = "customer"
```

三类角色：

- 普通用户：浏览商品、管理自己的订单并咨询客服
- 人工客服：接入转人工工单并回复用户
- 管理员：除客服能力外，还可接管工单、维护知识库和查看系统指标

##### 7.2 认证服务

`atguigu/app/services/auth.py` 封装认证能力。接口请求进入后，按以下流程完成身份校验：

```mermaid
flowchart LR
    Header["读取请求头"]
    Extract{"令牌格式正确"}
    Unauthorized["返回未授权"]
    Decode["解析并校验令牌"]
    Valid{"令牌有效"}
    CurrentUser["得到当前用户"]
    RoleCheck{"角色符合要求"}
    Forbidden["返回无权限"]
    Pass["放行业务处理"]

    Header --> Extract
    Extract -->|"否"| Unauthorized
    Extract -->|"是"| Decode
    Decode --> Valid
    Valid -->|"否"| Unauthorized
    Valid -->|"是"| CurrentUser
    CurrentUser --> RoleCheck
    RoleCheck -->|"否"| Forbidden
    RoleCheck -->|"是"| Pass
```

另外，认证服务还提供签发短期访问令牌的能力，供内部调用使用。会话接口会要求普通用户身份，<span style="color:red">确保只有客户能打开自己的当前会话</span>。

##### 7.3 依赖注入

依赖文件中注册认证服务获取方式，路由按需注入，避免在每个接口里手动创建实例。

至此，认证能力已经可被路由复用。下一步用它保护会话接口。

---

#### 8. 会话模块

今天的第一条业务接口是“**获取当前会话**”。实现顺序采用总分总：先定最小响应模型，再写服务与路由，最后注册并验证。

这里先把“会话”理解成：用户和客服系统之间的一次聊天容器。

##### 8.1 会话模型

 使用**最小响应模型**：

```python
class CurrentConversationResponse(BaseModel):
    id: str
    mode: str = "AI"
    processing: dict | None = None
```

字段含义：

- `id`：会话编号
- `mode`：当前由谁处理，先用普通字符串，默认 `"AI"`
- `processing`：是否有正在进行的处理；没有时为 `None`

后续课程再把 `mode` 收成枚举，把 `processing` 收成正式结构。今天先用简单类型把接口跑通。

##### 8.2 会话服务

`atguigu/app/services/chat/conversation.py` 负责会话编排。

```python
async def get_current_conversation(self, user_id: str) -> dict[str, Any]:
    # 1. 获取或创建当前有效会话 TODO
    # 2. 组装会话响应 TODO
    # 3. 提交会话状态变化 TODO
    return {
        "id": f"demo-conversation-{user_id}",
        "mode": "AI",
        "processing": None,
    }
```

##### 8.3 会话路由

`atguigu/app/routers/chat/conversation.py` 定义获取当前会话接口。请求进入后按以下流程处理：

```mermaid
flowchart LR
    Request["接收获取当前会话请求"]
    Auth["调用认证服务校验普通用户"]
    AuthOk{"身份校验通过"}
    Reject["返回未授权或无权限"]
    UserId["取出当前用户编号"]
    Service["调用会话服务"]
    Response["返回最小会话响应"]

    Request --> Auth
    Auth --> AuthOk
    AuthOk -->|"否"| Reject
    AuthOk -->|"是"| UserId
    UserId --> Service
    Service --> Response
```

对应代码：

```python
@router.post(
    "/conversations/current",
    response_model=CurrentConversationResponse,
)
async def get_current_conversation(
    auth_service: AuthServiceDep,
    conversation_service: ConversationServiceDep,
    authorization: Annotated[str | None, Header()] = None,
):
    current_user = auth_service.get_authorized_user(authorization, "customer")
    return await conversation_service.get_current_conversation(
        current_user.user_id
    )
```

##### 8.4 注册与验证

在应用实例中注册会话路由，并在依赖文件中提供会话服务注入。

验证方式：

1. 启动服务
2. 打开接口文档，确认出现获取当前会话接口
3. 使用电商服务签发的普通用户令牌调用接口

预期结果：

- 认证失败：返回未授权或无权限
- 认证成功：返回类似下面的数据

```json
{
  "id": "demo-conversation-user_001",
  "mode": "AI",
  "processing": null
}
```

<span style="color:red">今天目标是打通“认证 → 服务 → 最小响应”，不是完成会话持久化。</span>

---

<a id="chapter-04"></a>

## 04 · day02_客服会话建模与分层实现

> [查看本篇原文](originals/day02_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E4%BC%9A%E8%AF%9D%E7%BC%96%E7%A0%81%E4%B8%8A%EF%BC%88%E4%BC%9A%E8%AF%9D%E5%BB%BA%E6%A8%A1%E4%B8%8E%E5%88%86%E5%B1%82%E5%AE%9E%E7%8E%B0%EF%BC%89/2_resource/day02_%E5%AE%A2%E6%9C%8D%E4%BC%9A%E8%AF%9D%E5%BB%BA%E6%A8%A1%E4%B8%8E%E5%88%86%E5%B1%82%E5%AE%9E%E7%8E%B0.md) · [返回目录](#阅读目录)

### 第二天：客服会话建模与分层实现

第一天从项目背景出发，梳理了用户端、管理端、客服服务、电商服务和智能服务之间的协作关系，也明确了**客服服务是整条对话链路的会话中枢**。第二天开始进入客服服务内部，围绕“当前会话”和“会话详情”两个接口，完成会话模块的数据建模、分层查询与业务流程设计。

本章不会孤立地介绍某一个类，而是沿着一次真实请求经过的路径展开：请求先进入路由层，经过身份校验后调用业务服务层；业务服务层再通过数据访问层读取或写入数据库，最终由响应模型约束返回给前端的数据。

---

#### 1. 从系统架构进入会话模块

第一天已经明确，用户端不会直接调用智能服务，而是先把客服请求提交给客服服务。客服服务要继续完成消息接收、智能处理和人工接管，**首先必须知道用户当前处于哪一个会话**。因此，会话模块是后续消息模块和处理轮次模块的基础。

##### 1.1 本章学习目标

本章完成后，需要能够理解并说明以下问题：

2. 会话、消息和处理轮次分别负责保存什么数据；
3. 如何保证一个用户同一时间只有一个未结束会话；
4. 如何查询用户当前有效的会话；
5. 如何向客服工作台返回指定会话的完整消息记录；
6. 路由层、响应模型、业务服务层和数据访问层如何协作；

这些目标共同指向一个结果：让前端能够稳定地获得当前会话状态，同时让客服人员能够查看一段会话的完整历史。

##### 1.2 本章代码范围

本章主要涉及持久层模型，以及会话模块中的数据访问、响应模型、业务服务和路由文件。持久层定义会话、消息和处理轮次，数据访问部分只讲解会话与消息两个对象，接口部分包括当前会话和会话详情两个入口。

这些文件按照“持久化模型—数据访问—响应约束—业务实现—接口接入”的顺序组成会话模块。下面先从业务概念入手，再逐层落到代码。

---

#### 2. 客服会话领域分析

在编写数据库模型之前，需要先区分会话系统中的三个核心概念。它们虽然都与聊天有关，但承担的职责完全不同。

##### 2.1 为什么需要客服会话

**消息只能描述一次发送行为，例如谁发送了什么内容，却不适合保存整段客服对话的状态。**如果没有独立的会话模型，会话模式、开始结束时间和当前处理状态就需要重复保存在每条消息中，或者每次通过消息临时推断。

因此，系统需要**使用会话作为一段连续客服对话的业务容器**：

- 将同一次对话中的消息归为一组；
- 统一保存整段对话的处理模式和生命周期；
- 为消息、处理轮次和人工工单提供共同的关联标识；
- 区分当前有效会话与已经结束的历史会话；
- 保证同一用户同一时间只有一个有效会话。

因此，**消息记录“说了什么”，会话记录“这段对话目前处于什么状态”**。在明确会话的作用后，还需要进一步区分会话与处理轮次。

##### 2.2 会话、消息与处理轮次的关系

`Conversation` 表示一段客服对话，`Message` 表示这段对话中的一条聊天消息，`ConversationTurn` 表示一批等待或正在交给智能客服处理的用户输入。<span style="color:red">人工客服领取的是 `Handoff` 人工工单，查看和回复的是 `Message`，与 `ConversationTurn` 无关。</span>

```mermaid
erDiagram
    Conversation ||--o{ Message : "包含聊天消息"
    Conversation ||--o{ ConversationTurn : "关联智能处理轮次"

    Conversation {
        string id PK
        string user_id
        string mode
        datetime started_at
        datetime last_active_at
        datetime ended_at
    }

    Message {
        int id PK
        string message_id UK
        string conversation_id FK
        string role
        json content
        datetime created_at
    }

    ConversationTurn {
        string id PK
        string conversation_id FK
        string user_id
        string status
        datetime collect_until
        datetime locked_until
    }
```

三者的职责可以概括为：

- 一条会话可以包含多条消息；
- 一条用户消息只能属于一个会话；
- 一个会话在整个生命周期中可以产生多个处理轮次；
- 一个处理轮次可以关联一条或多条用户消息；
- 会话负责长期状态，处理轮次负责一次智能客服处理任务。

有了这个关系，系统既能还原完整聊天记录，也能独立管理智能客服的状态。接下来需要明确会话自身有哪些处理模式。

##### 2.3 四种会话处理模式

会话的 `mode` 字段共有四种业务值：

- `AI`：当前由智能客服处理。用户消息会进入处理轮次；
- `QUEUED`：已经请求转人工，正在等待客服接入；
- `HUMAN`：人工客服已经接入并正在服务；
- `CLOSED`：当前会话已经关闭，不能再作为有效会话继续使用。

<span style="color:red">这四种模式不是四种消息角色，而是整段会话当前由谁负责处理。</span>消息角色保存在 `Message.role` 中，两者不能混淆。

##### 2.4 会话生命周期

新创建的会话默认进入智能客服模式。智能客服无法可靠解决问题时，可以请求转人工；人工客服接入后进入人工服务模式；人工服务结束后可以重新回到智能客服模式。长时间没有活动的智能客服会话则会被关闭。

```mermaid
stateDiagram-v2
    [*] --> AI: 创建新会话
    AI --> QUEUED: 请求转人工
    QUEUED --> HUMAN: 人工客服接入
    HUMAN --> AI: 人工服务结束
    AI --> CLOSED: 空闲时间超过限制
    CLOSED --> [*]
```

`AI`、`QUEUED` 和 `HUMAN` 都表示尚未结束的会话，而 `CLOSED` 表示历史会话。正因为存在这个区别，<span style="color:red">数据库才能限制一个用户只能拥有一个未结束会话，同时允许该用户保留多条历史会话。</span>

---

#### 3. 数据持久层模型

领域概念明确后，就可以把会话、消息和处理轮次映射到数据库。**持久层模型不仅保存字段，还通过外键、唯一约束和索引维护并发场景下的数据正确性。**

##### 3.1 会话模型

`Conversation` 对应 `conversations` 表，保存一次客服服务的总体状态。

各字段承担的职责如下：

- `id` 是会话业务标识，使用 `conv` 前缀生成；
- `user_id` 标识会话所属用户；
- `mode` 保存当前处理模式；
- `started_at` 保存会话创建时间；
- `last_active_at` 保存最后活动时间，**是判断会话是否超时的依据**；
- `ended_at` 保存会话结束时间。

会话是本章其他模型的业务入口。接下来，消息模型通过外键归属于具体会话。

##### 3.2 消息模型

`Message` 对应 `messages` 表，保存用户、智能客服和人工客服发送的消息。

这里同时存在 `id` 和 `message_id`：

- `id` 是数据库自增主键，适合根据主键查询
- `message_id` 是业务消息标识，可以在接口、事件和其他服务之间传递。

`role` 区分消息发送方，当前响应模型允许 `user`、`ai` 和 `human`。`content` 使用 `JSONB`，因此既能保存普通文本，也能保存带导航操作提示或结构化信息的消息。

消息解决了聊天内容的持久化问题，但智能客服还需要一个可以被后台工作进程领取和执行的任务模型。

##### 3.3 会话处理轮次模型

`ConversationTurn` 对应 `conversation_turns` 表。它不代表前端看到的一条聊天消息，而是后台智能处理的一次任务。

本章只需要知道处理轮次属于某个会话，并通过 `status` 保存当前状态。消息范围、收集窗口、工作进程锁定和重试等字段将在下一天实现具体处理逻辑时再分析。

##### 3.4 数据实体之间的关系

会话模型通过 `messages` 属性建立与消息模型的一对多关系。消息模型使用 `conversation_id` 外键指向会话，并通过 `conversation` 属性建立反向关系。

`cascade="all, delete-orphan"` 表示消息的生命周期依附于会话；`lazy="selectin"` 表示查询 `Conversation` 时，SQLAlchemy 会用 `IN (...)` 批量把所有对应的 `Message` 一次性全查出来。

`cascade="all, delete-orphan"` （级联删除与孤儿删除）

**含义**：控制当“会话（父）”发生变动时，“消息（子）”该如何联动处理。

- **`all`**：最直接的效果是：**删除了主表 `Conversation`，ORM 会自动把关联的所有 `Message` 记录一并删掉**。
- **`delete-orphan`（孤儿删除）**：如果把某条消息从会话的列表里移除了，ORM会识别出 某条消息 已经成为了“孤儿（不再属于任何会话）”，执行 `session.commit()` 时**直接从数据库中将其 `DELETE` 掉**，防止产生垃圾残留数据。



##### 3.5 唯一约束与索引设计

应用代码中的“先查询再新增”不能完全解决并发问题。两个请求可能同时完成查询，并且都认为数据不存在，随后分别写入重复记录。<span style="color:red">因此，关键的唯一性规则最终必须由数据库保证。</span>

当前三个模型共包含三组业务唯一性规则。它们解决的问题不同，使用的数据库约束也不同。

**第一组：保证一个用户只有一个当前有效会话**

会话表使用 PostgreSQL 部分唯一索引，只在会话模式为 `AI`、`QUEUED` 或 `HUMAN` 时约束 `user_id` 唯一。

这三种模式虽然名称不同，但都表示会话尚未结束，共同占用“当前有效会话”这一个名额。因此：

- <span style="color:red">同一用户只能存在一个未结束会话；</span>
- 会话可以在三种模式之间切换，不需要创建新的有效会话；
- `CLOSED` 不参与这个唯一约束，同一用户可以保留多个历史会话。

**第二组：防止消息和智能处理结果被重复写入**

消息表使用 `UniqueConstraint` 定义三个不带状态条件的唯一约束：

- **`message_id` 全局唯一，防止同一条业务消息被重复保存；**
- `conversation_id` 与 `input_revision` 的组合唯一，防止同一会话中的同一个输入版本重复保存。输入版本的具体生成和使用将在下一天讲解；
- `agent_run_id` 与 `agent_outcome_seq` 的组合唯一，防止同一次智能处理中的同一个结果因重复事件而写入多次。

两个组合约束中包含允许为空的字段。PostgreSQL 默认把不同记录中的空值视为彼此不同，因此未填写输入版本或智能处理标识的消息可以正常保存；当组合字段都有实际值时，约束才会阻止相同组合重复写入。

**第三组：限制同一会话中两种活动轮次的数量**

处理轮次表使用两个独立的部分唯一索引：

- 第一个索引只约束 `COLLECTING` 状态，保证同一会话最多有一个正在收集的轮次；
- 第二个索引只约束 `RUNNING` 状态，保证同一会话最多有一个正在运行的轮次。

<span style="color:red">这里不能把两个状态合并到同一个部分唯一索引中。</span>正在运行的轮次已经确定了本次要处理的消息，此时用户新发送的消息需要由另一个收集轮次接收。如果合并约束，只要存在 `RUNNING` 轮次，数据库就会禁止创建 `COLLECTING` 轮次，系统便无法在生成上一轮回答的同时收集下一轮消息。

所以，<span style="color:red">同一会话可以同时存在一个 `RUNNING` 轮次和一个 `COLLECTING` 轮次，但不能同时存在两个 `RUNNING` 轮次，也不能同时存在两个 `COLLECTING` 轮次。</span>

三组规则可以归纳为：会话约束防止出现多个当前未结束会话，消息约束防止结果重复写入，处理轮次约束保证两个并行处理阶段各自最多只有一个任务。



##### 3.6 消息收集与任务执行状态约束

轮次最重要的两个中间状态是：

- `COLLECTING`：正在防抖窗口内收集用户消息；
- `RUNNING`：消息范围已经确定，工作进程正在调用智能服务。

“防抖”指的是：**当用户连续发送多条消息时，系统不立即触发 AI 响应，而是先“等一会儿”，把这段时间内连续发出的所有短消息合并为一次完整的上下文，再统一打包交给 AI 处理。**



```mermaid
stateDiagram-v2
    [*] --> COLLECTING: 收到需要智能处理的消息
    COLLECTING --> COLLECTING: 防抖窗口内继续合并消息
    COLLECTING --> RUNNING: 到达领取时间
    RUNNING --> COMPLETED: 回答保存成功
    RUNNING --> FAILED: 执行失败且不再重试
    RUNNING --> COLLECTING: 租约超时后回收重试
```

“租约超时后回收重试”指的是：**工作进程领取一个处理轮次后，将其改为 `RUNNING`。同时记录：`locked_by`：由哪个工作进程处理；`locked_until`：该工作进程最晚拥有到什么时间。**<span style="color:red">如果工作进程崩溃、卡死或没有及时完成，超过 `locked_until` 后，这个轮次不能永远停留在 `RUNNING`。</span>系统会解除原工作进程的占用，把轮次重新放回待处理状态，交给其他工作进程重试。

本章只完成处理轮次的数据结构设计，暂不实现处理轮次的数据访问、领取和执行逻辑。这些内容将在后面的消息发送和后台工作进程课程中继续展开。

---

#### 4. 数据访问层

持久层模型定义数据结构，数据访问层负责集中编写查询。**业务服务不直接拼接 SQLAlchemy 查询，而是通过语义明确的方法获取所需数据。**

##### 4.1 数据访问层的职责

数据访问层主要承担以下职责：

- 封装数据库查询条件；
- 把查询结果转换成实体或实体列表；
- 将新实体加入当前数据库会话；
- 避免业务服务与具体查询语句耦合。

数据访问对象持有同一个 `AsyncSession`。它们可以执行查询和 `add`，但本章由业务服务决定何时刷新或提交事务。

##### 4.2 会话数据访问对象

`ConversationRepository` 提供四个与本章相关的方法。

`list_ai_conversation()` 查询指定用户的全部智能客服会话，专门服务于空闲超时检查。<span style="color:red">它只查询 `AI` 模式，是因为人工排队或人工服务中的会话不能由智能客服空闲规则直接关闭。</span>

`find_active_conversation()` 查询用户最近的有效会话。业务服务传入 `("AI", "QUEUED", "HUMAN")`，从而排除已经关闭的历史会话。查询按最后活动时间倒序并限制一条，使业务意图更加明确。

另外，`get_by_id()` 负责按主键查询指定会话，`add()` 负责把新会话实体加入当前数据库事务。

会话数据访问对象解决了会话查询问题，消息详情则交给消息数据访问对象完成。

##### 4.3 消息数据访问对象

`MessageRepository.list_by_conversation_id()` 根据 `conversation_id` 查询指定会话的全部消息，并在数据库查询中按 `Message.id` 升序排列。

**排序直接放在数据库查询中，业务服务收到的结果已经具有稳定顺序，因此不需要在外层再次调用 `sorted()`。**

这里选择自增主键 `Message.id`，是因为本接口需要按数据库写入顺序还原聊天记录。若只按 `created_at` 排序，多条消息时间相同时可能无法得到稳定结果。

##### 4.4 数据访问与事务边界

**数据访问对象负责查询和登记实体，业务服务负责控制事务。**<span style="color:red">底层数据访问方法不能各自提交，否则一个完整业务操作会被拆成无法统一回滚的多个片段。</span>

```mermaid
flowchart LR
    R["路由层"] --> S["业务服务层"]
    S --> C["会话数据访问对象"]
    S --> M["消息数据访问对象"]
    C --> D[(数据库)]
    M --> D
    S --> F["刷新未提交变更"]
    S --> K["在业务入口统一提交"]
```

**`flush()` 与 `commit()`** **核心差异对照**：

| **维度**       | **flush() 之后**                                           | **commit() 之后**                          |
| -------------- | ---------------------------------------------------------- | ------------------------------------------ |
| **SQL 执行**   | 数据库已经接收并执行了 `INSERT/UPDATE/DELETE`              | 数据库接收并执行了 `COMMIT`                |
| **主键与约束** | **立刻拿到自增 `id`**，数据库立刻校验外键/唯一索引冲突     | 变更彻底完成，无法再触发该事务内的约束报错 |
| **数据可见性** | **仅当前数据库连接/事务可见**，其他并发请求/连接完全看不到 | **对所有事务可见**                         |
| **锁与资源**   | **继续持有行锁/排他锁**，阻塞其他试图修改该行数据的事务    | **释放所有锁**，归还数据库连接资源         |

**本质区别：**

- `flush()` 的本质： 把 ORM 内存中的变化**转成 SQL 语句**发送给数据库引擎执行，但**保持事务开启、持有行锁**。

  `commit()` 的本质： 向数据库发送 `COMMIT` 指令，**写日志落盘、释放锁、结束事务**，使修改对所有其他数据库连接可见。

- 对外接口的业务入口适合在所有步骤成功后统一 `commit()`。

#### 5. 会话响应数据模型

数据库模型面向持久化，字段较多；接口响应模型面向前端，只应返回当前页面真正需要的数据。<span style="color:red">两类模型职责不同，不能直接把数据库实体完整暴露给前端。</span>

##### 5.1 响应模型的职责

响应模型使用 Pydantic 定义，主要作用包括：

- 限制接口可以返回的字段；
- 校验字段类型和值域；
- 自动生成接口文档；
- 避免内部调度字段泄漏；
- 让前后端围绕稳定的数据契约协作。

本章包含当前会话响应、单条消息响应和会话详情响应三个模型。在定义具体响应之前，先约束会话模式。

##### 5.2 会话模式枚举

`ConversationMode` 使用字符串枚举约束模式值：

```python
class ConversationMode(StrEnum):
    AI = "AI"				 # 用户端显示“AI 客服”
    QUEUED = "QUEUED"		 # 用户端显示“等待人工”	
    HUMAN = "HUMAN"          # 用户端显示“人工服务中”
    CLOSED = "CLOSED"		 # 用户端显示“已结束” 	
```

使用枚举以后，接口不会把任意字符串作为合法会话模式。它既保留字符串在 JSON 中易于传输的特点，又让代码和接口文档明确展示允许值。

虽然“当前有效会话”正常只会返回前三种模式，但保留 `CLOSED` 能表达完整的会话状态集合，也便于其他响应模型复用。

##### 5.3 当前会话响应模型

用户端加载客服页面时需要获得当前会话的状态：

```python
class CurrentConversationResponse(BaseModel):
    id: str
    mode: ConversationMode
    is_processing: bool
```

三个字段分别用于：

- `id`：标识当前会话，用于筛选该会话的历史消息
- `mode`：显示当前由智能客服、排队或人工处理；
- `is_processing`：用户端刷新后恢复“AI 正在处理”提示

##### 5.4 单条会话消息响应模型

客服工作台需要展示会话中的每一条消息：

```python
class ConversationMessageResponse(BaseModel):
    message_id: str
    role: Literal["user", "ai", "human"]
    content: dict[str, Any]
    created_at: datetime
```

各字段面向管理端的用途是：

- `message_id`：作为消息列表中的稳定唯一标识；
- `role`：管理端决定用户、智能客服或人工客服的标签和样式；
- `content`：提供需要展示的文本或结构化内容；
- `created_at`：保留消息发送时间，便于后续展示。

##### 5.5 会话详情响应模型

会话详情响应只包含消息列表：

```python
class ConversationDetailResponse(BaseModel):
    messages: list[ConversationMessageResponse]
```

管理端调用接口时已经持有目标 `conversation_id`，遍历消息记录

##### 5.6 持久层模型与响应模型的区别

**持久层模型描述“数据库必须保存什么”，响应模型描述“当前接口允许返回什么”。**一次接口响应通常需要经过转换，而不是直接返回完整实体。

```mermaid
flowchart LR
    D[(数据库记录)] --> O["持久层实体"]
    O --> S["业务服务筛选并组装"]
    S --> P["响应模型校验"]
    P --> J["返回前端的 JSON"]
```

这种分离使数据库可以保存工作进程租约和审计标识，而前端仍然获得简单、稳定的数据结构。接下来进入本章的核心：会话业务服务如何组织这些查询与状态变化。

---

#### 6. 会话业务服务实现

会话业务服务位于接口与数据访问之间。它不负责解析请求头，也不直接编写查询，而是根据业务规则组织多个数据访问对象，并控制事务提交。

##### 6.1 业务服务层的职责

`ConversationService` 持有数据库会话，以及会话和消息数据访问对象。它通过这些对象组织跨表查询和状态变更。

它承担以下职责：

- 确保用户拥有一个当前有效会话；
- 关闭已经空闲超时的智能客服会话；
- 在没有有效会话时创建新会话；
- 查询并组装客服工作台所需的会话详情；
- 在接口业务完成后统一提交事务。

业务服务的总体调用关系如下：

```mermaid
flowchart TD
    A["会话业务服务"] --> B{"调用目标"}
    B -->|"获取当前会话"| C["确保存在有效会话"]
    C --> D["关闭超时会话"]
    C --> E["查询已有有效会话"]
    C --> F["必要时创建新会话"]
    B -->|"查询会话详情"| G["验证会话存在"]
    G --> H["查询全部消息"]
    H --> I["组装消息响应"]
```

下面分别展开两条业务链路，先从用户端进入客服页面时调用的当前会话逻辑开始。

##### 6.2 获取或创建当前有效会话

`get_current_conversation()` 是当前会话接口的业务入口。它先调用 `ensure_active_conversation()` 获得可继续使用的会话，<span style="color:red">随后统一提交本次事务</span>，最后返回当前会话信息。

完整流程如下：

```mermaid
flowchart TD
    A["接收当前用户编号"] --> B["确保存在有效会话"]
    B --> C["获得会话实体"]
    C --> E["提交本次事务"]
    E --> F["返回会话编号"]
    E --> G["返回会话模式"]
```

执行顺序不能随意交换：

1. 必须先处理超时会话，才能确定旧会话是否仍然有效；
2. 必须获取已有会话或创建新会话，才能得到当前会话；
3. 必须完成所有状态变化后再统一提交；
4. 最后才组装当前会话响应。

##### 6.3 查询会话详情

`get_conversation_detail()` 面向客服和管理员，负责查询指定会话的全部消息。它先按主键查询会话；会话不存在时抛出异常，存在时再调用消息数据访问对象加载全部消息，并逐条转换为响应结构。

这条业务链路只读取数据，不修改会话状态：

```mermaid
flowchart TD
    A["接收会话编号"] --> B["按主键查询会话"]
    B --> C{"会话是否存在"}
    C -->|"否"| D["抛出会话不存在异常"]
    C -->|"是"| E["按会话编号查询全部消息"]
    E --> F["数据库按消息主键升序排列"]
    F --> G["逐条转换为响应结构"]
    G --> H["返回消息列表"]
```



##### 6.4 组装前端消息结构

持久层消息实体包含许多内部字段，而会话详情只需要 `message_id`、`role`、`content` 和 `created_at` 四个字段。模块级辅助函数 `_build_conversation_message()` 负责从消息实体中提取这些字段。

##### 6.5 关闭空闲超时的智能客服会话

在查询当前有效会话之前，系统先加载用户已有的智能客服会话，并逐个判断是否空闲超时。超时后将会话模式修改为 `CLOSED`，把 `ended_at` 记录为最后活动时间；只有发生状态变化时才刷新数据库事务。

超时判断流程如下：

```mermaid
flowchart TD
    A["查询当前用户的智能客服会话"] --> B["逐个检查会话"]
    B --> C{"最后活动时间是否早于超时边界"}
    C -->|"否"| D["保持原状态"]
    C -->|"是"| E["模式修改为已关闭"]
    E --> F["结束时间记录为最后活动时间"]
    F --> G["累计状态变化数量"]
    D --> H{"是否还有会话"}
    G --> H
    H -->|"有"| B
    H -->|"没有"| I{"是否发生状态变化"}
    I -->|"是"| J["刷新到当前事务"]
    I -->|"否"| K["无需执行刷新"]
```

具体超时条件由 `_has_idle_timeout()` 完成：当前时间与最后活动时间之差大于或等于 30 分钟时，会话达到超时条件。

这里使用 `>=` 表示达到 30 分钟边界就视为超时。`ended_at` 使用 `last_active_at`，表达会话实际停止活动的时刻，而不是用户下一次打开页面、系统发现超时的时刻。

##### 6.6 创建新会话

当用户不存在 `AI`、`QUEUED` 或 `HUMAN` 模式的会话时，业务服务创建一个模式为 `AI` 的新会话，把它加入当前数据库事务，然后执行 `flush()` 并返回会话实体。

此处调用 `flush()` 后，数据库会生成会话主键，使后续代码可以立即使用 `conversation.id`，<span style="color:red">但事务仍由外层业务入口统一提交。</span>

创建方法不自行 `commit()`，是因为它还会被消息接收等更大的业务流程复用。如果底层方法提前提交，就无法保证“创建会话”和“保存首条消息”处于同一个原子事务。

##### 6.7 数据提交与事务处理

当前会话的完整事务关系可以表示为：

```mermaid
sequenceDiagram
    participant R as 路由层
    participant S as 会话业务服务
    participant C as 会话数据访问对象
    participant D as 数据库

    R->>S: 获取当前会话
    S->>C: 查询智能客服会话
    C->>D: 执行查询
    D-->>C: 返回会话
    S->>S: 判断并关闭超时会话
    opt 发生状态变化
        S->>D: 刷新变更
    end
    S->>C: 查询有效会话
    alt 不存在有效会话
        S->>C: 添加新会话
        S->>D: 刷新并生成主键
    end
    S->>D: 统一提交事务
    S-->>R: 返回当前会话状态
```

<span style="color:red">这里最重要的边界是：内部方法只刷新，当前接口的公开业务方法统一提交。</span>这样能够同时满足主键生成、组合业务和异常回滚三方面需求。

---

#### 7. 会话接口实现

业务服务完成后，路由层负责把 HTTP 请求转换为业务方法调用。路由层应保持轻量，只处理路径、请求头、依赖注入、权限和响应模型，不重复实现会话规则。

##### 7.1 路由层的职责

会话路由统一使用 `/api/v1` 前缀。本章提供两个接口：`POST /api/v1/conversations/current` 用于获取当前有效会话，`GET /api/v1/conversations/{conversation_id}` 用于读取指定会话详情。

第一个接口面向普通用户，第二个接口面向客服和管理员。两者复用同一个会话业务服务，但权限和返回结构不同。

##### 7.2 获取当前有效会话接口

当前会话接口从请求头中取得认证令牌，通过认证服务校验 `customer` 角色并获得当前用户编号，然后调用会话业务服务的 `get_current_conversation()` 方法。接口使用 `CurrentConversationResponse` 约束最终响应。

接口执行流程如下：

```mermaid
flowchart TD
    A["用户端发起请求"] --> B["读取认证请求头"]
    B --> C["校验普通用户身份"]
    C --> D{"认证是否通过"}
    D -->|"否"| E["返回认证或权限错误"]
    D -->|"是"| F["提取当前用户编号"]
    F --> G["调用当前会话业务方法"]
    G --> H["获得会话状态"]
    H --> I["按当前会话响应模型校验"]
    I --> J["返回用户端"]
```

**路径使用 `POST` 是因为这个接口不只是单纯读取数据。**用户没有有效会话时，它会创建新会话；智能客服会话超时时，它还会修改旧会话状态。因此该调用可能产生数据库写入。

##### 7.3 获取指定会话详情接口

会话详情接口从路径参数中取得 `conversation_id`，并通过认证服务校验 `agent` 或 `admin` 角色。权限通过后，它调用业务服务的 `get_conversation_detail()` 方法，并使用 `ConversationDetailResponse` 约束返回的消息列表。

这条接口只读取指定会话及其消息，所以使用 `GET`。`conversation_id` 来自路径参数，由管理端在进入具体会话时提供。

```mermaid
flowchart TD
    A["客服工作台发起请求"] --> B["从路径取得会话编号"]
    B --> C["读取认证请求头"]
    C --> D["校验客服或管理员身份"]
    D --> E{"认证与权限是否通过"}
    E -->|"否"| F["返回认证或权限错误"]
    E -->|"是"| G["调用会话详情业务方法"]
    G --> H{"会话是否存在"}
    H -->|"否"| I["返回会话不存在错误"]
    H -->|"是"| J["获得按写入顺序排列的消息"]
    J --> K["按会话详情响应模型校验"]
    K --> L["返回客服工作台"]
```



##### 7.4 身份认证与角色校验

两个接口都从 `Authorization` 请求头读取令牌，但<span style="color:red">允许的角色不同</span>：

- 当前会话接口只允许 `customer`；
- 会话详情接口只允许 `agent` 和 `admin`。

```mermaid
flowchart LR
    T["认证令牌"] --> A["认证服务"]
    A --> R{"用户角色"}
    R -->|"普通用户"| C["允许访问当前会话"]
    R -->|"人工客服"| D["允许访问会话详情"]
    R -->|"管理员"| D
    R -->|"角色不匹配"| X["拒绝访问"]
```

<span style="color:red">角色校验必须放在调用业务服务之前。</span>这样可以防止普通用户通过猜测会话编号读取其他用户的聊天记录。

##### 7.5 接口响应模型约束

**路由装饰器中的 `response_model` 不只是接口文档说明，还会对业务服务返回值进行校验和过滤。**当前会话接口使用 `CurrentConversationResponse`，会话详情接口使用 `ConversationDetailResponse`。

如果业务服务遗漏必填字段、返回错误类型，或者返回了不符合枚举和值域约束的数据，响应校验会暴露问题。例如，`is_processing` 必须提供布尔值，不能返回 `None`。

##### 7.6 路由层、业务服务层与数据访问层的调用关系

两个接口共用相同分层，但调用的数据访问方法不同：

```mermaid
flowchart TB
    subgraph 用户端链路
        U["当前会话接口"] --> UA["普通用户认证"]
        UA --> US["获取当前会话业务方法"]
        US --> UC["会话数据访问对象"]
    end

    subgraph 管理端链路
        A["会话详情接口"] --> AA["客服或管理员认证"]
        AA --> AS["获取会话详情业务方法"]
        AS --> AC["会话数据访问对象"]
        AS --> AM["消息数据访问对象"]
    end

    UC --> D[(客服数据库)]
    AC --> D
    AM --> D
```

路由层解决“谁可以调用”，业务服务层解决“应该执行哪些步骤”，数据访问层解决“怎样查询数据库”。职责划分清楚后，每一层都可以独立测试和推进。

---

#### 8. 两个接口的完整执行流程

前面的章节分别介绍了模型和分层实现，本节从页面行为出发，把两个接口重新串成用户能够感知的完整流程。

##### 8.1 页面初始化流程

用户进入或刷新客服页面时，前端首先请求当前会话。接口返回当前会话信息，前端再根据会话编号恢复对应的历史和实时状态。

```mermaid
sequenceDiagram
    participant U as 用户端
    participant R as 会话路由
    participant S as 会话业务服务
    participant D as 数据库

    U->>R: 请求当前有效会话
    R->>R: 校验普通用户身份
    R->>S: 传入当前用户编号
    S->>D: 关闭超时的智能客服会话
    S->>D: 查询或创建有效会话
    S->>D: 提交事务
    S-->>R: 返回会话状态
    R-->>U: 返回编号、模式和处理状态
```

这个接口是整个用户端聊天流程的入口。前端获得当前会话后，后续发送消息和接收实时事件才有明确的会话归属。

##### 8.2 获取当前会话流程

获取当前会话包含三个连续决策：

```mermaid
flowchart TD
    A["开始"] --> B["检查智能客服会话是否超时"]
    B --> C["查询未结束会话"]
    C --> D{"是否找到"}
    D -->|"是"| E["复用已有会话"]
    D -->|"否"| F["创建新的智能客服会话"]
    E --> H["提交事务"]
    F --> H
    H --> I["返回当前会话响应"]
```

这里的“复用”很重要。如果用户刷新页面就创建新会话，历史消息会被切断，正在执行的处理轮次也无法正确恢复。

##### 8.3 加载历史消息流程

客服或管理员进入人工工作台中的某个会话时，管理端使用会话编号调用详情接口。

```mermaid
sequenceDiagram
    participant A as 管理端
    participant R as 会话路由
    participant S as 会话业务服务
    participant C as 会话数据访问对象
    participant M as 消息数据访问对象

    A->>R: 请求指定会话详情
    R->>R: 校验客服或管理员身份
    R->>S: 传入会话编号
    S->>C: 按主键查询会话
    alt 会话不存在
        C-->>S: 返回空
        S-->>R: 抛出会话不存在异常
        R-->>A: 返回错误
    else 会话存在
        C-->>S: 返回会话
        S->>M: 查询该会话全部消息
        M-->>S: 返回按写入顺序排列的消息
        S-->>R: 返回消息列表
        R-->>A: 返回会话详情
    end
```

消息查询在数据库层完成排序，业务服务只负责转换字段，响应模型再保证每条消息都满足前端契约。

##### 8.4 会话不存在或超时后的处理

两个接口面对“会话不可用”时的行为不同：

- 当前会话接口发现旧会话超时，会关闭旧会话并创建新会话；
- 会话详情接口按编号查不到会话时，应返回会话不存在错误；
- 当前会话接口不会把 `CLOSED` 会话作为有效会话返回；
- 会话详情接口可以按编号读取历史会话，只要该记录仍然存在。

```mermaid
flowchart TD
    A["请求会话"] --> B{"接口类型"}
    B -->|"获取当前会话"| C{"旧会话是否有效"}
    C -->|"是"| D["返回旧会话"]
    C -->|"否或不存在"| E["创建并返回新会话"]
    B -->|"获取指定会话详情"| F{"指定编号是否存在"}
    F -->|"是"| G["返回消息详情"]
    F -->|"否"| H["返回会话不存在错误"]
```

这个差异来自两个接口不同的业务目标：<span style="color:red">用户端必须始终获得一个可继续聊天的会话，而管理端详情接口必须查询指定会话，不能创建其他会话。</span>

---

#### 9. 会话模块分层设计总结

经过前面的实现，会话模块已经形成从数据库到接口的完整分层。最后需要重新审视各层边界和数据流向，为后续消息发送功能做好衔接。

##### 9.1 各层职责对照

持久层模型负责定义会话、消息和轮次的数据结构以及数据库约束。

数据访问层负责按用户、会话编号和状态查询实体。

响应模型负责定义前端真正能够收到的字段和值域。

业务服务层负责关闭超时会话、查询或创建有效会话、判断处理状态、加载消息以及控制事务。

路由层负责接收请求、读取认证信息、限制访问角色并声明响应模型。

这些职责沿调用方向逐层收敛：

```mermaid
flowchart LR
    A["请求与身份"] --> B["业务规则"]
    B --> C["数据访问语义"]
    C --> D["持久化实体与约束"]
    D --> E[(数据库)]
    E --> D
    D --> C
    C --> B
    B --> F["响应数据"]
    F --> G["前端页面"]
```

任何一层都不应该越过相邻层承担全部工作，否则会导致接口函数过大、查询散落或内部字段泄漏。

##### 9.2 数据流转过程

当前会话的数据从认证令牌开始。路由层取得当前用户编号后，业务服务查询或创建会话实体，组装当前会话数据并经过响应模型校验，最后返回用户端。

会话详情的数据从路径中的会话编号开始。业务服务先查询会话实体，再查询按写入顺序排列的消息实体，将每条消息转换为响应字段，经过会话详情响应模型校验后返回管理端。

**两条链路都没有直接把数据库实体原样返回。**这保证了数据访问结构与接口契约之间存在清晰边界。

---

#### 10. 本章总结

本章从第一天的整体架构进入客服服务内部，围绕会话模块完成了以下内容：

1. 使用会话、消息和处理轮次三个模型表达客服业务；
2. 使用会话模式描述智能客服、人工排队、人工服务和关闭状态；
3. 使用部分唯一索引保证一个用户只有一个未结束会话；
4. 使用两个轮次索引支持“一个正在运行、一个继续收集”的处理方式；
5. 使用数据访问层封装当前会话和会话详情查询；
6. 使用业务服务组织超时关闭、查询、创建和事务提交；
7. 使用两个路由分别服务普通用户和客服工作台。

**当前会话接口解决“用户现在应该继续使用哪个会话”，会话详情接口解决“客服人员应该看到哪些历史消息”。**这两个接口共同建立了客服系统最基础的会话读取能力。

在此基础上，下一阶段可以继续实现用户消息接收、处理轮次的防抖涉及与合并、后台工作进程领取任务，以及智能回复写回与实时通知，从而把静态会话管理扩展成完整的异步客服对话链路。

---

<a id="chapter-05"></a>

## 05 · day03_用户消息接收与轮次收集

> [查看本篇原文](originals/day03_%E5%AE%A2%E6%9C%8D%E4%BC%9A%E8%AF%9D%E7%BC%96%E7%A0%81%E4%B8%AD%EF%BC%88%E7%94%A8%E6%88%B7%E6%B6%88%E6%81%AF%E6%8E%A5%E6%94%B6%E4%B8%8E%E8%BD%AE%E6%AC%A1%E6%94%B6%E9%9B%86%EF%BC%89/2_resource/day03_%E7%94%A8%E6%88%B7%E6%B6%88%E6%81%AF%E6%8E%A5%E6%94%B6%E4%B8%8E%E8%BD%AE%E6%AC%A1%E6%94%B6%E9%9B%86.md) · [返回目录](#阅读目录)

### 第三天：用户消息接收与轮次收集

第二天完成了客服会话的数据建模、当前会话查询和会话详情查询。此时，用户进入客服页面后已经能够获得当前会话，客服人员也能够查看指定会话的历史消息，但系统还不能真正接收一条新的用户消息。

第三天继续沿着同一条业务链路向后实现：用户提交消息后，客服服务先识别重复请求，再获取并锁定当前有效会话，随后保存消息，并根据会话模式决定是收集到智能处理轮次，还是创建一条等待推送给人工客服工作台的消息事件。

本章的重点不是单独完成一次数据库新增，而是保证**消息、会话状态、智能处理轮次和待发布事件在同一个业务事务中保持一致**。

---

#### 1. 从会话查询进入消息接收

第二天实现的当前会话接口解决了“用户现在属于哪个会话”的问题。第三天的消息接收接口需要在这个基础上继续回答三个问题：

1. 这条消息是否已经提交过；
2. 这条消息应该进入哪个当前有效会话；
3. 这条消息后续应该交给智能客服还是人工客服。

只有把这三个问题按照稳定顺序处理，才能避免消息重复、会话状态错乱以及智能处理轮次遗漏。

##### 1.1 本章学习目标

完成本章后，需要能够理解并说明

3. 为什么保存用户消息前需要锁定当前会话；
4. 如何通过输入版本标记等待智能客服处理的消息；
5. 如何把短时间内连续发送的消息合并到同一个处理轮次；
6. 为什么收集轮次需要同时设置短等待时间和最大等待时间；
7. 智能模式、排队模式和人工模式下的消息处理有什么区别；
8. 为什么消息和待发布事件必须在同一个事务中提交。

这些目标最终组成一条完整的用户消息接收链路。

##### 1.2 本章代码范围

本章主要涉及以下内容：

- 用户消息的请求模型与响应模型；
- 消息、会话和处理轮次的数据访问；
- 当前有效会话的锁定；
- 用户消息保存；
- 智能处理轮次的防抖收集；
- 待发布事件模型与写入服务；
- 消息接收接口和依赖注入。

处理轮次的领取、智能服务调用、租约、失败重试和智能回复保存不属于本章，它们将在下一天展开。

##### 1.3 用户消息接收链路概览

一条用户消息进入客服服务后，会按照下面的顺序流转：

```mermaid
flowchart TD
    A["用户端提交消息"] --> B["根据令牌识别当前用户"]
    B --> C["根据消息编号查询重复消息"]
    C --> D{"消息是否已经存在"}
    D -- "是" --> E["返回原会话编号和模式"]
    D -- "否" --> F["获取或创建当前有效会话"]
    F --> G["锁定会话"]
    G --> H["创建用户消息"]
    H --> I{"当前会话模式"}
    I -- "智能模式" --> J["增加输入版本"]
    J --> K["创建或延长收集轮次"]
    I -- "排队或人工模式" --> L["创建管理端消息事件"]
    K --> M["提交事务"]
    L --> M
    M --> N["返回会话编号和模式"]
```

---

#### 2. 用户消息请求与响应

消息接收链路从接口数据开始。请求模型负责限制前端可以提交什么，响应模型只返回前端在消息发送完成后继续维护页面状态所需的数据。

##### 2.1 用户消息请求结构

用户发送消息时提交三个字段：

```python
class ChatMessageRequest(BaseModel):
    message_id: str = Field(min_length=4, max_length=80)
    type: MessageType
    content: dict[str, Any]
```

三个字段分别承担不同职责：

- `message_id`：由用户端提前生成，用于标识同一次消息提交；
- `type`：说明消息是普通文本还是业务对象；
- `content`：保存消息的实际内容。

文本消息可以使用下面的结构：

```json
{
  "message_id": "msg_1001",
  "type": "text",
  "content": {
    "text": "我的订单什么时候发货？"
  }
}
```

消息内容使用对象结构，而不是直接使用字符串，是为了兼容订单、商品等业务对象消息，也为后续智能服务返回操作按钮和选择项保留扩展空间。

##### 2.2 消息类型与消息角色

消息类型描述内容结构：

```python
class MessageType(StrEnum):
    TEXT = "text"
    OBJECT = "object"
```

消息角色描述发送者：

```python
class MessageRole(StrEnum):
    USER = "user"
    AI = "ai"
    HUMAN = "human"
```

二者不能混为一谈。例如，一条用户发送的订单卡片消息，其角色是 `user`，类型是 `object`；一条智能客服返回的文本消息，其角色是 `ai`，类型是 `text`。

第三天接收的是用户消息，因此保存时角色固定为 `user`，类型和内容来自前端请求。

##### 2.3 消息编号的作用

用户端在发起请求前生成消息编号，并在网络重试时继续使用同一个编号。这样即使同一个请求因为超时、重复点击或重试到达服务端多次，服务端也能够识别它们属于同一次业务提交。

<span style="color:red">消息编号标识的是一次业务消息，不是数据库自增主键。</span>

数据库主键用于表内存储和排序，消息编号用于前后端共同识别消息。二者职责不同，不能相互替代。

##### 2.4 消息接收响应结构

消息接收成功后只返回两个字段：

```python
class AcceptUserMessageResponse(BaseModel):
    conversation_id: str
    mode: ConversationMode
```

- `conversation_id`：让用户端确认这条消息所属的会话，比如加载历史消息就是使用这个最新会话 ID。
- `mode`：让用户端确认消息提交后仍由智能客服、排队流程还是人工客服处理，比如页面右上角区域的样式变化。

---

#### 3. 消息幂等与数据访问

请求模型解决了消息编号从哪里来，数据访问层接下来需要根据这个编号判断消息是否已经保存。

##### 3.1 为什么需要消息幂等

假设用户端发送一条消息，服务端已经成功提交，但响应在网络中丢失。用户端无法判断消息是否保存，只能使用同一个消息编号重新请求。

如果服务端每次都直接新增，就会出现两条内容相同的消息，完整的接口幂等要求：

> 同一个消息编号无论提交多少次，都只能对应同一条数据库消息，并返回相同的会话结果。

##### 3.2 联表查询消息及所属会话

消息数据访问对象通过一次联表查询同时返回消息和会话：

```python
async def find_with_conversation_by_message_id(
    self,
    message_id: str,
) -> tuple[Message, Conversation] | None:
    result = await self.session.execute(
        select(Message, Conversation)
        .join(
            Conversation,
            Conversation.id == Message.conversation_id,
        )
        .where(Message.message_id == message_id)
    )
    return result.tuples().one_or_none()
```

如果只查询消息，业务服务为了返回会话模式还要再查询一次会话。联表查询把两次数据库访问合并为一次，直接获取结果。

##### 3.3 重复消息的返回处理

业务服务收到查询结果后，先判断消息所属会话是否属于当前用户，再直接返回原结果：

```python
duplicate_result = (
    await self.message_repository.find_with_conversation_by_message_id(
        chat_message.message_id
    )
)
if duplicate_result:
    duplicate, conversation = duplicate_result
    return {
        "conversation_id": duplicate.conversation_id,
        "mode": conversation.mode,
    }
```

这个分支不会再次保存消息、增加输入版本或创建处理轮次。重复请求因此不会产生新的业务副作用。

##### 3.4 消息唯一约束的最终保证

应用代码中的查询能够处理先后到达的普通重试，数据库唯一约束则负责保证相同消息编号不能重复落库：

```python
UniqueConstraint("message_id", name="uq_message_id")
```

这个约束保证任意两条数据库消息都不能使用相同的消息编号，但它本身只能保证数据不重复，不能保证并发请求都得到正常的幂等响应。

因此，两个完全并发的相同消息请求仍可能同时通过应用层查询，其中一个最终会被数据库唯一约束拒绝。这里保留这一取舍，是为了把本章重点放在消息接收主链路；生产版本可以增加用户级事务锁，或者捕获唯一约束异常后重新查询。

完成重复消息识别后，下一步才进入会话获取和消息保存。

---

#### 4. 当前会话的获取与锁定

消息不能脱离会话独立存在。业务服务需要先确保当前用户拥有一个有效会话，再锁定这条会话记录，才能安全地修改输入版本和最后活跃时间。

##### 4.1 获取或创建当前有效会话

第二天实现的 `ensure_active_conversation()` 已经能够：

1. 检查并关闭空闲超时的智能会话；
2. 查询 `AI`、`QUEUED`、`HUMAN` 三种模式下的当前有效会话；
3. 没有有效会话时创建新的智能会话。

第三天在这个能力上增加锁定步骤：

```python
async def ensure_locked_active_conversation(
    self,
    user_id: str,
) -> Conversation:
    conversation = await self.ensure_active_conversation(user_id)
    await self.session.refresh(
        conversation,
        with_for_update=True,
    )
    return conversation
```

这里使用 `refresh()` 重新读取已经加载的会话对象，并通过 `with_for_update=True` 在读取时取得数据库行锁。它同时解决两个问题：

- 覆盖当前 Session 身份映射中可能存在的旧字段值；
- 阻止其他事务在当前消息事务结束前修改同一条会话。

如果只在第二次查询中使用 `SELECT ... FOR UPDATE`，SQLAlchemy 虽然会取得数据库锁，但可能继续返回当前 Session 中已经加载的同一个 Python 对象，不自动覆盖旧的输入版本。`refresh()` 明确要求重新加载字段，因此等待锁结束后可以获得数据库中的最新会话状态。

##### 4.2 为什么需要锁定会话

智能模式下，每条用户消息都会执行：

```python
conversation.input_revision += 1
```

如果两个请求同时读取到 `input_revision = 5`，又都把它修改为 `6`，两条不同消息就会得到相同的输入版本。

加锁查询能够刷新最新状态后，请求会按顺序修改：

```text
请求一：5 → 6
请求二：等待请求一提交，再执行 6 → 7
```

因此，会话锁保护的不是消息内容本身，而是同一会话中需要连续变化的公共状态。

##### 4.3 会话锁保护的业务数据

消息接收期间主要修改以下会话字段：

- `input_revision`：智能客服尚待处理的最新输入版本；
- `last_active_at`：会话最近一次活动时间；
- `mode`：决定消息进入智能处理还是人工处理分支。

锁定后读取会话模式，还能避免消息处理过程中会话模式被同时切换，导致同一条消息既进入智能轮次又被推送给人工客服。

##### 4.4 会话锁定流程图

```mermaid
sequenceDiagram
    participant A as 消息请求一
    participant DB as 数据库会话行
    participant B as 消息请求二

    A->>DB: 查询并锁定当前会话
    DB-->>A: 返回 input_revision=5
    B->>DB: 请求锁定同一会话
    Note over B,DB: 等待请求一提交
    A->>DB: input_revision 更新为6
    A->>DB: 提交并释放锁
    DB-->>B: 强制刷新并返回最新 input_revision=6
    B->>DB: input_revision 更新为7
    B->>DB: 提交并释放锁
```

会话锁会一直持有到消息接收事务提交或回滚。对于数据库中已经存在的会话，正确刷新加锁查询结果后，业务服务才能获得稳定的最新状态。

首次创建会话是另一种并发场景：会话记录尚不存在时没有可锁定的行，两个首次消息请求仍可能同时尝试创建有效会话。最终由“同一用户只能有一个有效会话”的部分唯一索引阻止重复创建。当前没有捕获并恢复该唯一约束异常，这与完全并发消息的幂等处理一样，属于生产版本需要补充的并发异常恢复。

---

#### 5. 用户消息保存

用户消息保存方法负责创建消息实体并把它加入当前事务，但不负责决定消息交给谁处理，也不负责提交事务。这样同一个方法未来可以继续保存智能回复和人工回复。

##### 5.1 创建并保存用户消息

通用消息保存方法接收会话、角色、类型和内容：

```python
def add_message(
    self,
    conversation: Conversation,
    role: MessageRole,
    content: dict[str, Any],
    *,
    message_id: str | None = None,
    message_type: str = "text",
) -> Message:
    message = Message(
        message_id=message_id or get_uid("msg"),
        conversation_id=conversation.id,
        role=role,
        message_type=message_type,
        content=content,
    )
    self.message_repository.add(message)
    conversation.last_active_at = get_utcnow()
    return message
```

消息接收服务调用时，把角色固定为用户：

```python
message = self.add_message(
    conversation,
    MessageRole.USER,
    chat_message.content,
    message_id=chat_message.message_id,
    message_type=chat_message.type,
)
```

##### 5.2 更新会话最后活跃时间

用户发送新消息代表会话仍在活动，因此保存消息时同步更新：

```python
conversation.last_active_at = get_utcnow()
```

后续判断智能会话是否空闲超时，会以该字段为依据。如果只保存消息而不更新会话时间，仍在持续聊天的会话可能被错误关闭。

##### 5.3 消息保存与事务边界

`add_message()` 只执行：

```python
self.message_repository.add(message)
```

它不调用 `flush()` 或 `commit()`。原因是此时业务流程还没有结束：

- 智能模式还需要更新输入版本和处理轮次；
- 排队或人工模式还需要创建待发布事件；
- 任意一步失败时，前面创建的消息也必须一起回滚。

<span style="color:red">如果在消息保存方法中提前提交，后续轮次或事件创建失败时，就会留下“消息已经存在，但没有后续处理任务”的不完整状态。</span>

##### 5.4 为什么输入版本不在通用消息方法中

通用方法只负责消息实体本身，因此输入版本的增加没有放在这里。

输入版本只属于“智能模式下的用户输入”。人工回复、智能回复以及人工模式下的用户消息都不应该增加智能输入版本。把版本逻辑放到处理轮次服务中，可以保证只有真正进入智能处理链路的用户消息才会获得输入版本。

保存消息后，业务服务开始根据会话模式选择不同的后续处理路径。

---

#### 6. 智能处理轮次收集

智能客服不需要用户每输入一个字或连续发送一句补充，就立即发起一次模型调用。系统会先在一个很短的时间窗口内收集消息，再把这一批消息作为一个处理轮次交给后续 Worker。

##### 6.1 会话输入版本

`Conversation.input_revision` 表示当前会话已经接收到的最新智能输入版本，`Conversation.answered_revision` 表示智能客服已经处理完成的输入版本。

例如：

```text
input_revision = 5
answered_revision = 3
```

说明版本 `4～5` 的用户输入还没有处理完成。

每收到一条智能模式下的用户消息，就先增加输入版本：

```python
conversation.input_revision += 1
message.input_revision = conversation.input_revision
```

这样每条等待智能处理的用户消息都有唯一且连续的版本。

##### 6.2 消息与输入版本绑定

消息表使用下面的组合唯一约束：

```python
UniqueConstraint(
    "conversation_id",
    "input_revision",
    name="uq_conversation_input_revision",
)
```

它保证同一会话中的一个输入版本最多对应一条用户消息。

排队模式和人工模式下的消息不进入智能处理轮次，因此 `input_revision` 保持为空。PostgreSQL 允许唯一约束中存在多条空值记录，所以人工消息不会受到输入版本唯一约束影响。

##### 6.3 查询并锁定收集轮次

增加输入版本后，系统查询当前会话是否已经存在 `COLLECTING` 轮次：

```python
async def find_and_lock_collecting_by_conversation_id(
    self,
    conversation_id: str,
) -> ConversationTurn | None:
    return await self.session.scalar(
        select(ConversationTurn)
        .where(
            ConversationTurn.conversation_id == conversation_id,
            ConversationTurn.status == "COLLECTING",
        )
        .with_for_update()
    )
```

这里的行锁用于协调消息请求与未来的 Worker。二者都会根据 Turn 当前状态做出不同修改：

- 消息请求看到 `COLLECTING` 后，会延长 `collect_until`；
- Worker 看到已经到期的 `COLLECTING` 后，会将其改为 `RUNNING` 并固定输入快照。

如果不加锁，消息请求可能先查到 `COLLECTING`，随后 Worker 已经把它改成 `RUNNING`，消息请求仍然按照之前的判断延长 collect_until，这样消息请求会误以为新消息已经加入当前收集轮次，不再创建新 Turn，但 Worker 固定的输入快照可能不包含这条新消息。

这就是典型的“**查询时状态有效，修改时状态已经变化**”。

**`with_for_update()` 的核心作用，是把“确认状态仍为 `COLLECTING`”和“决定延长当前 Turn”保护成一个不可被其他事务插入的整体。**最终只允许两种正确结果：

- 消息请求先获得锁：延长当前 `COLLECTING` Turn，Worker 等待；
- Worker 先获得锁：原 Turn 变成 `RUNNING`，消息请求随后查询不到它，并创建下一条 `COLLECTING` Turn。

行锁会一直持有到整个消息事务 `commit()` 或 `rollback()`，并不是查询方法返回后立即释放。虽然 Worker 要到下一天才实现，但第三天必须先把这个并发边界设计正确。

##### 6.4 延长消息收集时间

如果已经存在收集轮次，新消息不再创建重复轮次，而是延长等待时间：

```python
turn.collect_until = min(
    now + delay,
    turn.max_collect_until,
)
```

`now + delay` 表示从最新消息到达时间开始，再等待一个短暂防抖时间。只要用户继续输入，收集时间就会向后延长。

##### 6.5 限制最大等待时间

如果每条新消息都无限延长等待时间，持续输入的用户可能永远得不到回复。因此每个轮次创建时还会固定：

```python
max_collect_until
```

`min()` 会选择短等待截止时间和最大等待截止时间中更早的一个。

例如：

```text
当前时间：10:00:09
防抖延迟：3秒
短等待截止时间：10:00:12
最大等待截止时间：10:00:10
最终 collect_until：10:00:10
```

这样既能合并连续消息，又能保证用户不会等待过久。

##### 6.6 创建新的收集轮次

如果不存在可复用的收集轮次，就创建新轮次：

```python
turn = ConversationTurn(
    conversation_id=conversation.id,
    user_id=conversation.user_id,
    start_revision=conversation.answered_revision + 1,
    collect_until=now + delay,
    max_collect_until=now
    + timedelta(
        milliseconds=self.settings.message_merge_max_wait_ms
    ),
)
self.turn_repository.add(turn)
```

`start_revision` 从上一次已经回答的版本之后开始，表示这个轮次未来需要覆盖的最早未回答输入。

新轮次默认状态为 `COLLECTING`。此时只完成消息收集，不会调用智能服务。

##### 6.7 消息防抖流程图

```mermaid
flowchart TD
    A["收到智能模式用户消息"] --> B["会话输入版本加一"]
    B --> C["消息绑定输入版本"]
    C --> D["查询并锁定收集轮次"]
    D --> E{"是否存在收集轮次"}
    E -- "存在" --> F["计算当前时间加防抖延迟"]
    F --> G["与最大等待时间取较早值"]
    G --> H["更新收集截止时间"]
    E -- "不存在" --> I["创建新的收集轮次"]
    I --> J["设置起始输入版本"]
    J --> K["设置短等待与最大等待时间"]
    H --> L["等待事务统一提交"]
    K --> L
```

这一流程完成后，一条或多条连续消息就被组织成了下一天可以领取的处理任务。

---

#### 7. 智能处理期间继续接收消息

防抖收集不仅解决短时间连续输入，还为“智能客服正在处理时，用户继续发送消息”提供了基础。第三天只负责接收新消息并创建下一条收集轮次，但必须理解它为什么从上一次已经回答的版本开始收集。

##### 7.1 运行轮次与收集轮次并存

轮次表分别使用两个部分唯一索引：

```text
同一会话最多一个 COLLECTING
同一会话最多一个 RUNNING
```

两个索引相互独立，因此下面的状态是允许的：

```text
Turn 1：RUNNING
Turn 2：COLLECTING
```

但下面的状态不允许：

```text
Turn 1：RUNNING
Turn 2：RUNNING
```

也不允许同时存在两个 `COLLECTING` 轮次。

##### 7.2 智能处理期间继续接收消息

假设第一批消息已经被 Worker 领取，轮次状态从 `COLLECTING` 变成 `RUNNING`。此时用户又发送一条补充消息。

消息请求只查询 `COLLECTING` 轮次，因此不会把新消息继续追加到已经固定输入快照的 `RUNNING` 轮次，而是创建新的 `COLLECTING` 轮次。

这保证正在执行的输入范围不再变化，同时新消息也不会被拒绝。

##### 7.3 下一轮覆盖全部尚未回答输入

两个轮次分别承担不同职责：

- `RUNNING`：尝试处理领取时已经固定的输入快照；
- `COLLECTING`：记录运行期间又有新输入到达，并准备下一次处理。

新轮次使用：

```python
start_revision = conversation.answered_revision + 1
```

因此，它不是只包含运行期间新增加的消息，而是从“最后一次已经成功回答的版本”开始，覆盖当前仍未回答的全部输入。

下一天 Worker 在保存旧轮次结果前会校验输入快照。如果运行期间输入版本已经增加，旧快照会被标记为 `SUPERSEDED`，旧结果不会作为最终回复保存；随后新的收集轮次重新处理全部尚未回答输入。这里提前说明这一点，是为了准确解释第三天 `start_revision` 的计算方式，具体校验和状态修改将在下一天实现。

##### 7.4 两类轮次的协调流程图

```mermaid
sequenceDiagram
    participant U as 用户
    participant C as 客服服务
    participant T1 as 轮次一
    participant T2 as 轮次二
    participant W as 智能处理任务

    U->>C: 发送第一条消息
    C->>T1: 创建收集轮次
    W->>T1: 领取并改为运行状态
    W->>W: 处理第一批输入
    U->>C: 智能处理期间发送补充消息
    C->>T1: 查询收集轮次
    Note over C,T1: 轮次一已经运行，不能继续收集
    C->>T2: 创建新的收集轮次
    Note over T2: 起点仍是最后已回答版本加一
    W->>T1: 保存前发现输入快照已变化
    W->>T1: 标记为已被新输入替代
    W->>T2: 后续领取下一轮
    Note over W,T2: 重新处理全部尚未回答输入
```

第三天的职责仍然止于保存新消息和创建收集轮次。图中的快照淘汰只用于说明为什么下一轮不能仅处理补充消息，其具体代码属于下一天。

---

#### 8. 人工模式与待发布事件

并不是所有用户消息都需要进入智能处理轮次。会话进入排队或人工模式后，用户消息应该通知客服工作台，而不是继续调用智能客服。

##### 8.1 排队模式与人工模式

两种模式分别表示：

- `QUEUED`：用户已经请求人工服务，正在等待客服接入；
- `HUMAN`：人工客服已经接入，当前会话由人工处理。

这两种模式仍然属于同一个有效会话，只是会话的处理方式发生了变化。

##### 8.2 人工模式下不创建智能处理轮次

消息接收服务按照会话模式分支：

```python
if conversation.mode == "AI":
    await self.turn_service.add_message_to_turn(
        conversation,
        message,
    )
elif conversation.mode in {"QUEUED", "HUMAN"}:
    # 创建管理端消息事件
```

排队和人工模式下：

- 消息仍然保存到消息表；
- 消息角色仍然是 `user`；
- 不增加智能输入版本；
- 不创建处理轮次；
- 创建发往客服工作台的消息事件。

##### 8.3 消息创建事件

前端实时业务事件为两种：

```python
class RealtimeEventType(StrEnum):
    MESSAGE_CREATED = "message_created"
    HANDOFF_CHANGED = "handoff_changed"
```

第三天只实际创建 `MESSAGE_CREATED`。`HANDOFF_CHANGED` 为后续人工工单状态变化预留。

```mermaid
flowchart TD
    M["MESSAGE_CREATED<br/>消息创建"]
    M --> MU["用户频道"]
    M --> MA["管理端频道"]
    MU --> MU1["智能回复"]
    MU --> MU2["人工回复"]
    MU1 --> U1["用户端新增或替换消息"]
    MU2 --> U1
    U1 --> U2["根据角色显示消息样式"]
    MU1 --> U3["关闭智能处理提示<br/>失败时显示错误"]
    MA --> MA1["会话模式为 QUEUED 或 HUMAN 时<br/>用户发送的新消息"]
    MA --> MA2["人工回复<br/>同步其他客服工作台"]
    MA1 --> A1["管理端防抖刷新人工工单列表"]
    MA2 --> A1
    A1 --> A2["当前已打开工单时<br/>刷新会话详情"]

    H["HANDOFF_CHANGED<br/>人工工单状态变化"]
    H --> HW["创建人工工单<br/>status = waiting"]
    HW --> HWU["用户端<br/>切换为排队状态"]
    HW --> HWA["管理端<br/>刷新列表并显示新工单"]

    H --> HC["客服接单<br/>status = active"]
    HC --> HCU["用户端<br/>切换为人工服务状态"]
    HC --> HCA["管理端<br/>显示处理状态和当前负责人"]

    H --> HT["工单被接管<br/>status = active<br/>负责人发生变化"]
    HT --> HTA["主要发送到管理端<br/>刷新当前负责人"]

    H --> HR["人工服务结束<br/>status = resolved"]
    HR --> HRU["用户端<br/>恢复智能会话模式"]
    HR --> HRA["管理端<br/>移除工单"]
    HRA --> HRD["正在查看该工单时<br/>关闭会话详情"]
```

<span style="color:red">第三天只实现：`QUEUED` 或 `HUMAN` 会话中的用户消息通过 `MESSAGE_CREATED` 发送到管理端频道。</span>

##### 8.4 事件数据与接收频道

消息创建事件使用统一数据结构：

```python
def build_message_created_data(
    message: Message,
) -> dict[str, Any]:
    return {
        "message": {
            "message_id": message.message_id,
            "role": message.role,
            "content": message.content,
        }
    }
```

当前前端只需要：

- `message_id`：识别和去重消息；
- `role`：决定用户、智能客服或人工客服样式；
- `content`：展示消息内容。

排队或人工模式下写入管理端频道：

```python
self.realtime_service.add_outbox_event(
    STAFF_CHANNEL,
    RealtimeEventType.MESSAGE_CREATED,
    build_message_created_data(message),
    conversation_id=conversation.id,
    request_message_id=chat_message.message_id,
)
```

##### 8.5 消息和事件的原子提交

待发布事件不会在当前请求中直接发送到网络，而是先写入 `realtime_outbox` 表。事件发布 Worker、Redis 和 WebSocket 属于后续内容。

第三天需要保证：

```text
消息保存成功，事件也必须保存成功；
事件保存失败，消息也必须一起回滚。
```

```mermaid
flowchart TD
    A["开始消息接收事务"] --> B["保存用户消息"]
    B --> C["创建消息事件"]
    C --> D{"事务是否成功"}
    D -- "成功" --> E["同时提交消息和待发布事件"]
    D -- "失败" --> F["同时回滚消息和待发布事件"]
    E --> G["后续发布任务读取事件"]
```

`add_outbox_event()` 只把事件加入当前 Session，不执行 `flush()`、`commit()` 或实际网络发布。最终提交权仍然属于消息接收服务。

---

#### 9. 消息接收接口完整实现

前面的章节分别完成了请求约束、幂等查询、会话锁定、消息保存、轮次收集和待发布事件。本章把它们重新组合成完整接口，观察每一层如何协作。

##### 9.1 消息服务依赖组装

消息服务依赖四个对象：

- 当前数据库 Session；
- 会话服务；
- 处理轮次服务；
- 实时事件服务。

依赖注入必须保证它们共享同一个 Session，才能让消息、会话、轮次和事件参与同一个数据库事务。

```python
async def get_chat_message_service(
    session: Annotated[AsyncSession, Depends(get_db)],
    conversation_service: ConversationServiceDep,
    turn_service: TurnServiceDep,
    realtime_service: RealtimeServiceDep,
) -> ChatMessageService:
    return ChatMessageService(
        session,
        conversation_service,
        turn_service,
        realtime_service,
    )
```

如果每个服务各自创建 Session，就无法通过一次提交保证所有数据同时成功。

##### 9.2 根据会话模式选择处理分支

消息服务的主流程可以归纳为六步：

1. 根据消息编号识别重复请求；
2. 获取并锁定当前有效会话；
3. 创建用户消息；
4. 根据会话模式创建处理轮次或待发布事件；
5. 统一提交事务；
6. 返回会话编号和模式。

核心实现如下：

```python
async def accept_user_message(
    self,
    user_id: str,
    chat_message: ChatMessageRequest,
) -> dict[str, Any]:
    duplicate_result = (
        await self.message_repository
        .find_with_conversation_by_message_id(
            chat_message.message_id
        )
    )
    if duplicate_result:
        duplicate, conversation = duplicate_result
        if conversation.user_id != user_id:
            raise ValueError("会话不存在")
        return {
            "conversation_id": duplicate.conversation_id,
            "mode": conversation.mode,
        }

    conversation = (
        await self.conversation_service
        .ensure_locked_active_conversation(user_id)
    )

    message = self.add_message(
        conversation,
        MessageRole.USER,
        chat_message.content,
        message_id=chat_message.message_id,
        message_type=chat_message.type,
    )

    if conversation.mode == "AI":
        await self.turn_service.add_message_to_turn(
            conversation,
            message,
        )
    elif conversation.mode in {"QUEUED", "HUMAN"}:
        self.realtime_service.add_outbox_event(
            STAFF_CHANNEL,
            RealtimeEventType.MESSAGE_CREATED,
            build_message_created_data(message),
            conversation_id=conversation.id,
            request_message_id=chat_message.message_id,
        )
    else:
        raise RuntimeError(
            f"不支持的有效会话模式：{conversation.mode}"
        )

    await self.session.commit()

    return {
        "conversation_id": conversation.id,
        "mode": conversation.mode,
    }
```

##### 9.3 消息接收接口

路由层只负责身份校验、接收请求和调用业务服务：

```python
@router.post(
    "/messages",
    response_model=AcceptUserMessageResponse,
)
async def accept_user_message(
    chat_message: ChatMessageRequest,
    message_service: ChatMessageServiceDep,
    authorization: Annotated[str | None, Header()] = None,
):
    current_user = get_auth_service().get_authorized_user(
        authorization,
        "customer",
    )
    return await message_service.accept_user_message(
        current_user.user_id,
        chat_message,
    )
```

路由层不直接查询数据库，也不判断消息应该进入哪个处理分支。业务规则全部集中在消息服务中。

##### 9.4 身份认证与用户校验

消息请求中的用户编号不能由前端自由提交。路由层从 Bearer Token 中解析当前用户，再把可信的 `user_id` 传给消息服务。

重复消息分支还会校验消息所属会话的用户编号，避免用户通过猜测其他人的消息编号获取会话状态。

因此，消息编号负责幂等，Token 中的用户身份负责数据归属，二者不能相互替代。

##### 9.5 消息接收完整流程图

```mermaid
sequenceDiagram
    participant U as 用户端
    participant R as 消息接口
    participant M as 消息服务
    participant C as 会话服务
    participant T as 轮次服务
    participant E as 事件服务
    participant DB as 数据库

    U->>R: 提交消息编号、类型和内容
    R->>R: 校验令牌并获得用户编号
    R->>M: 接收用户消息
    M->>DB: 查询重复消息及所属会话

    alt 消息已经存在
        DB-->>M: 返回消息和会话
        M-->>R: 返回原会话编号和模式
        R-->>U: 幂等响应
    else 消息不存在
        M->>C: 获取并锁定有效会话
        C->>DB: 查询或创建会话，并锁定已存在的会话行
        M->>DB: 加入用户消息

        alt 智能模式
            M->>T: 收集消息到处理轮次
            T->>DB: 增加版本并创建或延长轮次
        else 排队或人工模式
            M->>E: 创建管理端消息事件
            E->>DB: 加入待发布事件
        end

        M->>DB: 统一提交事务
        M-->>R: 返回会话编号和模式
        R-->>U: 消息发送成功
    end
```

#### 10. 本章总结

第三天从第二天已经存在的会话模块继续向后，实现了用户消息从接口进入数据库，再进入智能处理轮次或人工消息事件的完整接收链路。

##### 10.1 用户消息接收链路总结

消息接收的稳定顺序是：

```text
身份校验
→ 消息幂等查询
→ 获取并锁定有效会话
→ 保存用户消息
→ 按会话模式选择后续处理
→ 统一提交事务
```

任何一步都不能脱离整个业务事务单独提交。

##### 10.2 会话锁与轮次锁总结

两种锁保护的对象不同：

- 会话锁：保护输入版本、最后活跃时间和会话模式，加锁查询还必须刷新 Session 中已经存在的实体；
- 收集轮次锁：协调消息防抖延时与未来 Worker 领取。

##### 10.3 事务一致性总结

智能模式下需要保证：

```text
用户消息 + 会话输入版本 + 收集轮次
```

同时提交。

排队或人工模式下需要保证：

```text
用户消息 + 管理端消息事件
```

同时提交。

待发布事件表把业务事务与网络发布解耦，避免数据库已经提交但实时通知丢失。

##### 10.4 下一天内容预告

第三天结束时，数据库中已经存在等待处理的 `COLLECTING` 轮次。下一天将继续实现：

1. 查询并领取已经结束收集的轮次；
2. 将轮次从收集状态改为运行状态；
3. 固定本轮输入快照；
4. 设置 Worker 租约；
5. 构造智能服务请求；
6. 调用智能服务；
7. 处理失败重试和超时恢复；
8. 保存智能回复和消息创建事件。

至此，第三天负责的“接收并组织用户输入”与第四天负责的“领取并处理输入”形成清晰边界。

---

<a id="chapter-06"></a>

## 06 · day04_智能处理任务的领取调用与结果保存

> [查看本篇原文](originals/day04_%E5%AE%A2%E6%9C%8D%E4%BC%9A%E8%AF%9D%E7%BC%96%E7%A0%81%E4%B8%8B%EF%BC%88%E6%99%BA%E8%83%BD%E5%A4%84%E7%90%86%E4%BB%BB%E5%8A%A1%E4%B8%8E%E7%BB%93%E6%9E%9C%E4%BF%9D%E5%AD%98%EF%BC%89/2_resource/day04_%E6%99%BA%E8%83%BD%E5%A4%84%E7%90%86%E4%BB%BB%E5%8A%A1%E7%9A%84%E9%A2%86%E5%8F%96%E8%B0%83%E7%94%A8%E4%B8%8E%E7%BB%93%E6%9E%9C%E4%BF%9D%E5%AD%98.md) · [返回目录](#阅读目录)

### 第四天：智能处理任务的领取、调用与结果保存

第三天完成了用户消息接收链路：客服服务保存用户消息、增加输入版本，并通过短暂的防抖窗口把连续消息组织成 `COLLECTING` 轮次。第四天继续处理这些已经收集完成的轮次，实现后台任务领取、智能服务调用、结果保存、失败重试和租约恢复。

本章重点不是模型如何生成答案，而是客服服务如何可靠地组织一次智能处理。模型能力属于后续智能服务课程；当前客服服务只负责准备输入、调用接口、校验结果是否仍然有效，并把最终结果保存到会话中。

---

#### 1. 从消息收集进入后台处理

第三天结束时，数据库中已经存在等待处理的轮次。它们不会自动执行，需要独立后台任务持续查询并领取。本章从这条边界开始，把“等待处理的输入”转换为“用户最终能够看到的智能回复”。

##### 1.1 本章学习目标

完成本章后，需要能够说明：

1. 为什么消息接口不直接等待智能服务；
2. 什么样的轮次可以被后台任务领取；
3. 如何固定本次处理的输入快照；
4. 租约如何记录当前处理权；
5. 如何构造当前输入和历史上下文；
6. 普通回复与业务写操作为什么使用不同流程；
7. 为什么最终保存结果前还要再次校验版本；
8. 失败重试和租约恢复分别解决什么问题；
9. 消息和待发布事件如何在同一事务中提交。

这些问题共同构成智能处理任务的完整生命周期。

##### 1.2 本章代码范围

本章涉及以下模块：

- 处理轮次的数据访问与业务服务；
- 后台轮询任务与单轮处理器；
- 智能服务调用网关；
- 智能事件解析器；
- 智能结果保存服务；
- 用户身份令牌与内部服务令牌；
- 消息、轮次和待发布事件模型。

人工工单的创建、接单、回复和结束暂不实现。`RUN_HANDOFF_REQUESTED` 只作为后续协议预留。

##### 1.3 第三天与第四天的职责边界

第三天负责接收并组织输入，第四天负责领取并处理输入：

```mermaid
flowchart LR
    A["用户发送消息"] --> B["保存用户消息"]
    B --> C["增加输入版本"]
    C --> D["创建或延长收集轮次"]
    D --> E["COLLECTING"]
    E --> F["后台任务领取"]
    F --> G["RUNNING"]
    G --> H["调用智能服务"]
    H --> I["保存回复或处理失败"]
    I --> J["COMPLETED / FAILED / SUPERSEDED"]
```

消息接口提交成功后立即返回，不等待右半部分完成。用户端先显示“智能客服正在处理”，最终回复通过消息创建事件到达。

---

#### 2. 后台处理模块的职责划分

智能处理包含轮询、远程调用、版本校验和结果保存。如果全部放在一个方法中，调度规则与单轮业务会互相混杂。因此将其拆成以下职责明确的组件：

- `AIWorker`：负责持续轮询、回收租约过期的 Turn，并领取下一个可以处理的 Turn；
- `TurnProcessor`：负责处理单个 Turn，组织智能服务调用、版本校验、两阶段提交和最终结算；
- `TurnService`：负责 Turn 的领取、请求数据构造、状态转换、重试和租约释放等业务规则；
- `ConversationTurnRepository`：负责查询 Turn、执行行锁和读取租约过期的 Turn；
- `AIServiceGateway`：负责携带用户令牌和内部服务令牌，通过 HTTP 调用 AI Service；
- `AIEventParser`：负责识别 AI Service 返回的事件类型，并提取 Run ID 和最终处理结果；
- `AIResultService`：负责在当前事务中保存 AI 消息，并创建发送给用户端的 Outbox 事件；
- `AuthService`：负责为 Worker 生成代表当前用户身份的访问令牌。

其中，`AIWorker` 负责“什么时候处理”，`TurnProcessor` 负责“如何完成一次处理”，其余组件分别提供轮次业务、数据访问、远程通信、协议解析、结果保存和身份认证能力。

##### 2.1 消息接口不直接调用智能服务

智能服务调用通常需要数秒甚至更久。如果消息接口直接调用，会带来以下问题：

- HTTP 请求长时间占用；
- 用户连续消息难以防抖合并；
- 网络失败后难以独立重试；
- 服务进程中断后无法恢复未完成任务；
- 数据库事务可能跨越远程调用。

因此，消息接口只负责把输入可靠地写入数据库，后台任务异步完成后续处理。

##### 2.2 轮询调度与单轮处理分离

`AIWorker` 负责调度：

```text
回收过期租约
→ 领取一个到期轮次
→ 交给 TurnProcessor
→ 没有任务时短暂等待
```

`TurnProcessor` 负责单个轮次：

```text
创建身份令牌
→ 调用智能服务
→ 处理两阶段业务操作
→ 结算轮次并保存结果
```

这样，`AIWorker` 不需要了解智能事件细节，`TurnProcessor` 也不需要管理无限轮询。

##### 2.3 网关、事件解析与结果保存

单轮处理又拆成三个协作组件：

- `AIServiceGateway`：负责 HTTP 请求和鉴权请求头；
- `AIEventParser`：负责识别智能服务返回的事件；
- `AIResultService`：负责保存智能消息和用户端 Outbox 事件。

这三个组件分别对应外部通信、协议解析和本地持久化，避免后台任务直接承担所有细节。

##### 2.4 后台处理模块整体结构图

```mermaid
flowchart TD
    W["AIWorker<br/>轮询与领取"] --> P["TurnProcessor<br/>处理单个轮次"]
    P --> G["AIServiceGateway<br/>调用智能服务"]
    G --> A["AI Service"]
    P --> E["AIEventParser<br/>解析单个事件"]
    P --> T["TurnService<br/>轮次状态与版本"]
    P --> R["AIResultService<br/>保存回复和事件"]
    T --> DB[("PostgreSQL")]
    R --> DB
```

组件拆分没有改变事务边界。每次数据库操作仍然使用当前短事务中的同一个 `AsyncSession`。

---

#### 3. 领取等待处理的轮次

后台任务不能领取所有 `COLLECTING` 轮次。只有收集时间已经结束、会话仍由智能客服处理，并且同一会话没有其他运行任务时，轮次才真正可执行。

##### 3.1 哪些轮次可以被领取

候选轮次需要同时满足：

```text
status = COLLECTING
collect_until <= 当前时间
Conversation.mode = AI
同一会话不存在 RUNNING Turn
```

查询按 `collect_until` 和 `created_at` 排序，每次只领取最早到期的一条。这让等待时间更长的任务优先执行。

##### 3.2 排除已有运行轮次的会话

同一会话允许同时存在：

```text
Turn 1：RUNNING，处理上一批输入
Turn 2：COLLECTING，收集运行期间的新输入
```

但不能让 Turn 2 在 Turn 1 结束前也变成 `RUNNING`。相关条件是：

```python
~exists(
    select(1).where(
        running_turn.conversation_id
        == ConversationTurn.conversation_id,
        running_turn.status == "RUNNING",
    )
)
```

它表示“候选轮次所属会话不存在运行中的另一条轮次”。数据库唯一索引是最后保障，这个查询条件则主动跳过当前不可领取的任务，避免依赖唯一约束异常控制流程。

##### 3.3 锁定轮次与会话

领取查询使用：

```python
.with_for_update()
```

它需要协调两类并发操作：

- 用户消息请求可能延长 `COLLECTING` 轮次的 `collect_until`；
- 后台任务准备把同一轮次改成 `RUNNING`。

用户消息先获得锁时，轮次收集时间被延长，后台任务不应继续领取。后台任务先获得锁时，轮次变成 `RUNNING`，后续用户消息会创建下一条 `COLLECTING` 轮次。

##### 3.4 固定输入快照

领取成功后执行：

```python
turn.status = "RUNNING"
turn.snapshot_revision = conversation.input_revision
```

`snapshot_revision` 表示本次处理最多覆盖到哪个输入版本。后续即使用户继续发送消息，这个值也不再改变。

例如：

```text
start_revision = 4
snapshot_revision = 6
```

表示本轮处理版本 4、5、6。版本 7 以后到达的消息属于下一次处理。

##### 3.5 设置处理租约

领取时还会记录：

```python
turn.locked_by = worker_id
turn.locked_until = now + timedelta(
    seconds=settings.ai_worker_lease_seconds
)
turn.attempts += 1
turn.started_at = turn.started_at or now
```

从业务含义上说，租约表示某个后台进程在指定时间内拥有本轮处理权：

- `locked_by`：当前处理者，由主机名和进程号组成；
- `locked_until`：处理权截止时间；
- `attempts`：已经领取并尝试处理的次数；
- `started_at`：第一次开始处理的时间。

##### 3.6 领取轮次流程图

```mermaid
flowchart TD
    A["查询 COLLECTING Turn"] --> B{"收集时间是否到期"}
    B -- "否" --> X["本次不领取"]
    B -- "是" --> C{"会话是否为 AI 模式"}
    C -- "否" --> X
    C -- "是" --> D{"同一会话是否已有 RUNNING"}
    D -- "是" --> X
    D -- "否" --> E["锁定轮次和会话"]
    E --> F["状态改为 RUNNING"]
    F --> G["固定 snapshot_revision"]
    G --> H["设置处理租约"]
    H --> I["增加 attempts"]
    I --> J["提交领取事务"]
```

领取事务提交后立即释放数据库行锁，但租约字段继续保存在数据库中，覆盖后续远程调用阶段。

---

#### 4. 构造智能服务请求

领取轮次后，后台任务需要根据固定的版本范围查询当前输入，同时补充之前的会话历史。这两部分数据承担不同职责。

##### 4.1 查询本轮输入版本范围

当前输入按照下面的范围查询：

```text
conversation_id 相同
role = user
input_revision >= start_revision
input_revision <= snapshot_revision
```

并按 `input_revision` 正序排列。这样，即使一轮合并了多条连续消息，智能服务也能按照用户实际发送顺序理解输入。

##### 4.2 查询本轮之前的历史消息

系统先找到本轮第一条用户消息的数据库序号，再查询它之前最近的 30 条消息：

```text
Message.id < 本轮第一条消息的 id
按 id 倒序查询
限制 30 条
最后重新翻转为正序
```

历史消息可以包含 `user`、`ai` 和 `human` 三种角色，帮助智能服务理解之前已经讨论过什么。

##### 4.3 生成请求编号

请求编号使用：

```python
request_id = f"{turn.id}:attempt:{turn.attempts}"
```

同一轮重试时 `attempts` 会增加，因此每次处理尝试都有不同请求编号：

```text
turn_001:attempt:1
turn_001:attempt:2
turn_001:attempt:3
```

这个编号用于区分重试尝试；`turn.id` 标识逻辑任务，`run_id` 则由智能服务在真正启动运行后生成。

##### 4.4 当前消息与历史消息的区别

发送给智能服务的数据分成：

```text
messages：本轮尚未回答的用户输入
history：本轮之前已经存在的上下文
```

`messages` 只包含用户角色，因为它表示本轮需要回答的内容。`history` 保留角色和创建时间，因为它用于还原完整对话背景。

`build_agent_request_data()` 最终返回的完整字典结构如下：

```python
{
    "conversation_id": claimed_turn.conversation_id,
    "user_id": claimed_turn.user_id,
    "turn_id": claimed_turn.id,
    "request_id": f"{claimed_turn.id}:attempt:{claimed_turn.attempts}",
    "input_revision": snapshot_revision,
    "messages": [
        {
            "message_id": message.message_id,
            "type": message.message_type,
            "content": message.content,
        }
        for message in current_messages
    ],
    "history": [
        {
            "message_id": message.message_id,
            "role": message.role,
            "type": message.message_type,
            "content": message.content,
            "created_at": message.created_at.isoformat()
        }
        for message in history_messages
    ],
}, current_messages[-1].message_id
```

##### 4.5 请求数据构造流程图

```mermaid
flowchart TD
    A["读取 Turn"] --> B["获得 start_revision"]
    A --> C["获得 snapshot_revision"]
    B --> D["查询版本范围内的用户消息"]
    C --> D
    D --> E["取得本轮第一条消息序号"]
    E --> F["查询之前最近 30 条历史消息"]
    F --> G["历史消息恢复正序"]
    D --> H["构造 messages"]
    G --> I["构造 history"]
    H --> J["生成 Agent Payload"]
    I --> J
    J --> K["返回请求数据和最后一条用户消息编号"]
```

请求构造仍在领取事务中完成。消息读取完成并提交后，后台任务才离开数据库事务去调用智能服务。

---

#### 5. 调用智能服务

客服服务不会直接执行模型，也不会直接修改订单数据库。它通过网关调用智能服务，由智能服务完成意图理解和工具编排。

##### 5.1 生成用户身份令牌

后台任务根据请求中的用户编号创建客户身份令牌：

```python
token = auth_service.create_access_token(
    CurrentUser(user_id=request["user_id"])
)
```

智能服务调用电商工具时需要知道操作属于哪个用户。该令牌用于传递用户身份，防止只依靠请求体中的自由字段执行跨用户操作。

##### 5.2 内部服务令牌的作用

网关同时发送：

```text
Authorization: Bearer <客户身份令牌>
X-Internal-Service-Token: <内部服务令牌>
```

两种令牌回答不同问题：

- 客户身份令牌：这次请求代表哪个用户；
- 内部服务令牌：调用方是否为可信的客服服务。

智能服务需要同时校验服务身份和用户身份。

##### 5.3 单个事件响应协议

单个 JSON 事件，每次启动或提交请求只返回一个业务事件：

```json
{
  "event_type": "run_completed",
  "event_data": {
    "run_id": "run_001",
    "message_id": "msg_001",
    "content": {
      "text": "这是智能客服回复"
    }
  }
}
```

因此，网关返回 `dict[str, Any]`，事件解析器也只处理一个字典。

##### 5.4 启动、提交与取消接口

网关提供三个入口：

```text
start_run   → 启动一次智能处理并返回事件
commit_run  → 提交已经准备好的业务写操作
cancel_run  → 取消尚未提交的业务决策
```

对应接口为：

```text
POST /internal/v1/agent/runs
POST /internal/v1/agent/runs/{run_id}/commit
POST /internal/v1/agent/runs/{run_id}/cancel
```

启动和提交返回单个 JSON 事件，取消成功不需要响应内容，使用 `204 No Content` 即可。

##### 5.5 网关调用流程图

```mermaid
sequenceDiagram
    participant P as TurnProcessor
    participant G as AIServiceGateway
    participant A as AI Service

    P->>G: start_run(token, request)
    G->>A: POST /agent/runs
    A-->>G: 单个 JSON 事件
    G-->>P: dict

    opt 返回待提交业务决策
        P->>G: commit_run(token, run_id, revision)
        G->>A: POST /runs/{id}/commit
        A-->>G: 最终 JSON 事件
        G-->>P: dict
    end

    opt 决策失效
        P->>G: cancel_run(token, run_id)
        G->>A: POST /runs/{id}/cancel
        A-->>G: 204 No Content
    end
```

网关只负责通信协议，不判断轮次版本，也不修改本地数据库。

---

#### 6. 解析智能处理事件

智能服务返回的事件不能直接交给持久化逻辑。事件解析器先识别类型，再提取运行编号和最终数据。

##### 6.1 四种内部事件类型

定义了四种内部事件：

```text
RUN_DECISION_PREPARED
RUN_COMPLETED
RUN_FAILED
RUN_HANDOFF_REQUESTED
```

前三种参与当前智能回复和业务写操作流程，第四种仅为后续人工工单模块预留。

##### 6.2 普通完成事件

`RUN_COMPLETED` 表示已经获得可保存的最终结果：

```json
{
  "event_type": "run_completed",
  "event_data": {
    "run_id": "run_001",
    "message_id": "msg_ai_001",
    "content": {
      "text": "订单预计明天发货"
    }
  }
}
```

普通知识回答和只读查询可以在第一阶段直接返回该事件；业务写操作则在提交成功后返回。

##### 6.3 待提交业务决策事件

`RUN_DECISION_PREPARED` 表示：

```text
智能服务已经理解用户意图
已经准备好业务操作参数
但还没有真正执行写操作
```

例如取消订单、申请退款和修改地址。客服服务必须先校验输入快照，再决定提交还是取消。

##### 6.4 失败事件

`RUN_FAILED` 表示智能处理失败。解析器把事件转换为异常：

```python
if event_type == AgentEventType.RUN_FAILED:
    raise RuntimeError(message)
```

异常由 `TurnProcessor` 捕获，再进入重试或最终失败流程。这样事件解析器只负责协议语义，不负责修改数据库状态。

##### 6.5 转人工事件的预留

`RUN_HANDOFF_REQUESTED` 计划用于表示智能服务判断当前问题需要人工处理。未来它可以作为第一阶段直接返回的终态决策，因为真正的人工工单应由客服服务本地事务创建。

当前课程尚未实现人工工单模型和业务服务，`AIEventParser.parse_outcome()` 也不支持该事件。如果现在收到它，会被当成协议错误进入失败重试。后续需要同时补充解析和工单处理：

```text
解析转人工原因
→ 创建 waiting 工单
→ Conversation.mode 改为 QUEUED
→ 创建 HANDOFF_CHANGED 事件
```

##### 6.6 事件解析流程图

```mermaid
flowchart TD
    A["接收单个 JSON 事件"] --> B["读取 event_type"]
    B --> C{"事件类型"}
    C -- "RUN_COMPLETED" --> D["返回 event_data"]
    C -- "RUN_FAILED" --> E["抛出处理异常"]
    C -- "RUN_DECISION_PREPARED" --> F["进入提交前版本校验"]
    C -- "RUN_HANDOFF_REQUESTED" --> G["当前不支持<br/>报告协议错误"]
    C -- "其他类型" --> H["报告协议错误"]
```

事件类型只描述智能运行结果，发送给用户端和管理端的 `MESSAGE_CREATED`、`HANDOFF_CHANGED` 属于另一套实时业务事件。

---

#### 7. 业务写操作的两阶段处理

普通回答只产生文本，过期后丢弃即可。取消订单、退款等操作会改变真实业务数据，不能在客服服务确认输入仍然有效前直接执行。

##### 7.1 为什么写操作不能立即执行

假设用户先发送：

```text
取消我的订单
```

智能服务分析期间，用户又发送：

```text
先不要取消
```

如果智能服务第一阶段已经真正取消订单，客服服务即使发现旧输入过期，也无法通过丢弃回复撤销业务影响。因此，第一阶段只能准备决策，不能执行写操作。

##### 7.2 第一阶段准备业务决策

第一阶段调用：

```python
event = await ai_gateway.start_run(token, request)
```

涉及写操作时返回：

```text
RUN_DECISION_PREPARED
```

智能服务保存待提交决策及其参数，等待客服服务调用 `commit_run()` 或 `cancel_run()`。

##### 7.3 提交前校验输入快照

只有待提交业务决策才执行第一次版本校验：

```text
conversation.input_revision
==
turn.snapshot_revision
```

校验时锁定 `Conversation`，避免比较版本与作出提交决定之间被用户消息请求插入。

如果版本已经变化，Turn 被标记为 `SUPERSEDED`，客服服务会尽力取消待提交决策。普通 `RUN_COMPLETED` 没有外部写操作，因此不需要这次提前校验，只在最终保存前校验。

##### 7.4 第二阶段提交业务操作

版本有效时调用：

```python
event = await ai_gateway.commit_run(
    token,
    run_id,
    input_revision,
)
```

智能服务再调用对应电商业务接口。真正修改订单数据的是订单或电商服务，智能服务只是工具调用编排者，客服服务则是对话流程协调者。

##### 7.5 输入过期时取消待提交决策

只有同时满足下面两个条件时才调用取消：

```python
if prepared and not committed:
    await self._safe_cancel_run(token, run_id)
```

- `prepared`：智能服务已经准备了业务写操作；
- `not committed`：客服服务尚未确认提交成功。

`cancel_run()` 只是尽力取消，果远端操作实际已经提交，取消也不能撤销业务副作用。普通完成和失败都是终态，不存在等待提交的操作。

##### 7.6 普通回复不需要两阶段

普通回答、知识检索和只读订单查询不会修改外部业务数据。它们可以在第一阶段直接返回 `RUN_COMPLETED`：

```text
start_run
→ 得到最终回复
→ 最终保存前校验版本
→ 有效则保存，过期则丢弃
```

即使结果过期，也只是不保存旧回复，不会留下外部业务副作用。

##### 7.7 两阶段处理流程图

```mermaid
flowchart TD
    A["start_run"] --> B{"返回事件"}
    B -- "RUN_COMPLETED" --> C["进入最终结算"]
    B -- "RUN_FAILED" --> D["进入失败处理"]
    B -- "RUN_HANDOFF_REQUESTED" --> E["当前版本报告协议错误<br/>后续再创建人工工单"]
    B -- "RUN_DECISION_PREPARED" --> F["锁定会话并校验输入版本"]
    F --> G{"版本是否有效"}
    G -- "否" --> H["标记 Turn 为 SUPERSEDED"]
    H --> I["cancel_run"]
    G -- "是" --> J["commit_run"]
    J --> K{"最终事件"}
    K -- "RUN_COMPLETED" --> C
    K -- "RUN_FAILED" --> D
```

这张图体现了两次校验的不同职责：**提交前校验只保护待执行写操作，最终保存前校验保护所有本地结果落库。**

---

#### 8. 结算轮次并保存智能回复

智能调用结束后，后台任务重新打开短事务，锁定会话并结算轮次。无论普通回复、业务操作结果还是最终失败消息，都在这里完成本地持久化。

##### 8.1 为什么保存前还要校验版本

第一次校验事务提交后，会话锁已经释放。调用 `commit_run()`、解析响应期间，用户仍可能发送新消息。因此，最终保存前必须再次比较：

```text
conversation.input_revision
与
turn.snapshot_revision
```

用户消息先提交时，后台任务会读到新版本并淘汰旧 Turn；后台任务先提交时，当前回复先保存，新消息随后进入下一轮。最终结果的有效性因此具有确定顺序。

##### 8.2 记录智能运行编号

智能服务在事件中返回 `run_id`。最终结算时写入：

```python
turn.run_id = run_id
```

普通回复不会经过提交前校验，因此必须在最终结算中记录 Run ID。业务写操作虽然已经在提交前记录过，再次赋值仍保持相同结果。

Run ID 还会写入最终 AI 消息，用于关联智能服务运行记录并支持结果幂等。

##### 8.3 保存智能回复消息

`AIResultService` 创建角色为 `ai` 的消息：

```text
message_id：智能服务提供，缺省时由客服服务生成
conversation_id：所属会话
role：ai
message_type：text
content：最终展示内容
agent_run_id：智能运行编号
agent_outcome_seq：当前固定为1
```

当前协议规定一个 Run 只产生一条最终消息，所以结果序号固定为 1。未来若一个 Run 支持多个结果，应由智能服务返回真实序号。

##### 8.4 推进会话已处理版本

正常回复或最终失败消息保存后，执行：

```python
conversation.answered_revision = turn.snapshot_revision
```

这里的“已回答”包含正常回复和最终失败提示，准确含义是这批输入已经得到终态处理结果。推进版本后，后续新轮次不会再次包含已经达到最大重试次数的旧输入。

##### 8.5 创建用户端消息事件

保存 AI 消息后，同时创建：

```text
频道：customer-service:user:{user_id}
事件：MESSAGE_CREATED
数据：message_id、role、content
关联：conversation_id
```

用户端收到后新增或替换消息，根据 `role="ai"` 显示智能客服样式，并关闭“正在处理”提示；失败内容还会切换为失败状态。

普通 AI 回复不发送管理端。管理端主要处理 `QUEUED` 和 `HUMAN` 会话，未来发生转人工时由 `HANDOFF_CHANGED` 通知。

##### 8.6 消息与事件的原子提交

`AIResultService` 中的 `MessageRepository` 和 `RealtimeService` 使用同一个 Session：

```text
保存 AI Message
创建 RealtimeOutbox
更新 Conversation
更新 ConversationTurn
→ 最外层统一 commit
```

任意 SQL 失败时，消息、轮次状态、已处理版本和 Outbox 事件都会一起回滚，避免用户消息已经保存但实时通知缺失。

##### 8.7 结果结算流程图

```mermaid
flowchart TD
    A["打开最终结算事务"] --> B["查询 Turn 并锁定 Conversation"]
    B --> C["记录 run_id"]
    C --> D{"输入快照是否仍有效"}
    D -- "否" --> E["Turn 标记 SUPERSEDED"]
    E --> F["提交并丢弃旧结果"]
    D -- "是" --> G{"处理是否成功"}
    G -- "成功" --> H["Turn 标记 COMPLETED"]
    G -- "失败但可重试" --> I["重新放回 COLLECTING"]
    G -- "达到最大次数" --> J["Turn 标记 FAILED"]
    J --> K["生成用户可见失败消息"]
    H --> L["保存 AI 消息"]
    K --> L
    L --> M["推进 answered_revision"]
    M --> N["创建用户端 MESSAGE_CREATED"]
    N --> O["统一提交事务"]
```



#### 9. 失败重试与租约恢复

远程调用失败不一定表示用户问题永远无法处理。短暂网络异常可以重试，进程意外中断则需要租约恢复。两者触发时机和处理方式不同。

##### 9.1 普通调用失败如何重试

智能调用抛出异常后，`TurnProcessor` 进入最终结算，调用：

```python
turn_service.retry_or_fail(turn, error)
```

只要尝试次数没有达到上限，Turn 就会：

```text
RUNNING → COLLECTING
collect_until → 当前时间 + 重试延迟
run_id → None
```

它不会立即再次执行，而是在短暂等待后重新进入领取队列。

##### 9.2 最大重试次数

每次领取都会增加 `attempts`。对于能够被当前进程捕获的普通处理异常，当前默认最多尝试 3 次：

```text
attempt 1 失败 → 重新排队
attempt 2 失败 → 重新排队
attempt 3 失败 → 标记 FAILED
```

请求编号包含尝试次数，智能服务可以区分同一 Turn 的不同调用。



##### 9.3 最终失败消息

达到最大次数后，系统不会静默结束，而是保存：

```json
{
  "kind": "error",
  "text": "AI 处理失败，请稍后重试。"
}
```

这条消息的角色仍然是 `ai`，通过用户频道发送。用户能够明确知道处理失败，而不是一直停留在“正在处理”状态。

最终失败也会推进 `answered_revision`，表示当前输入已经得到终态结果，避免后续新消息再次自动带上这批失败输入。

##### 9.4 为什么需要释放租约

租约只属于当前一次处理尝试。无论重新排队还是最终失败，当前 Worker 都已经结束本次处理，因此需要清空：

```python
turn.locked_by = None
turn.locked_until = None
```

正确状态约束是：

```text
只有 RUNNING Turn 持有租约
其他状态的租约字段为空
```

重试时下一次领取会重新设置新的租约。

##### 9.5 查询租约过期轮次

```text
status = RUNNING
locked_until <= 当前时间
```

##### 9.7 重新排队与标记失效

恢复时先比较输入版本：

```text
当前输入版本 != 旧快照版本
→ 旧 Turn 标记 SUPERSEDED

当前输入版本 == 旧快照版本
→ 旧 Turn 重新放回 COLLECTING
```

版本变化说明运行期间已经出现新消息，新的收集轮次会重新覆盖全部未回答输入；版本没变则说明原任务仍然有效，可以立即重新领取。

完整的租约恢复代码如下：

```python
async def _recover_expired_turns(self) -> None:
    """回收 Worker 中断遗留的过期租约。"""
    async with self.session_factory() as session:
        turn_service = ConversationTurnService(session)
        contexts = await turn_service.list_expired_running_turns_with_conversations()

        for turn, conversation in contexts:
            if conversation.input_revision != turn.snapshot_revision:
                turn_service.mark_superseded(turn)
            else:
                turn_service.requeue(
                    turn,
                    RuntimeError("Worker 租约超时"),
                )

        if contexts:
            await session.commit()
```

##### 9.8 过期 Turn 中的消息去了哪里

Turn 本身不保存消息，只通过 `start_revision` 和 `snapshot_revision` 引用 `messages` 表中的一段用户消息。因此，Turn 过期或失效时，用户消息不会被删除。

这里需要区分两种“过期”：

1. **租约过期，但输入版本没有变化**：原 Turn 被重新放回 `COLLECTING`，下次仍然处理原来的消息；
2. **输入快照过期**：原 Turn 被标记为 `SUPERSEDED`，但不会推进 `answered_revision`，原来的未回答消息会和新消息一起交给下一条 Turn。

例如：

```text
answered_revision = 3

Turn 1 正在处理版本 4～5
用户又发送版本 6
→ Turn 1 的快照失效，标记为 SUPERSEDED
→ 新 Turn 的 start_revision 仍然是 4
→ 新 Turn 领取时 snapshot_revision = 6
→ 下一次处理版本 4、5、6
```

```mermaid
flowchart LR
    A["Message 4、5<br/>属于旧快照"] --> D["仍保存在 messages 表"]
    B["旧 Turn<br/>SUPERSEDED"] --> D
    C["新 Message 6"] --> D
    D --> E["新 Turn<br/>start_revision = 4<br/>snapshot_revision = 6"]
    E --> F["重新处理消息 4、5、6"]
```

所以，`SUPERSEDED` 淘汰的是旧的处理结果和旧的执行任务，不是用户消息。只有成功回复或最终失败消息保存后，系统才推进 `answered_revision`，表示这批输入已经得到终态处理。

##### 9.9 失败恢复流程图

```mermaid
flowchart TD
    A["处理失败"] --> B{"Worker 是否仍能执行异常处理"}
    B -- "是" --> C{"是否达到最大尝试次数"}
    C -- "否" --> D["释放租约并延迟重试"]
    C -- "是" --> E["标记 FAILED"]
    E --> F["保存用户可见失败消息"]
    B -- "否，进程中断" --> G["数据库遗留 RUNNING Turn"]
    G --> H["新 Worker 等待租约到期"]
    H --> I["锁定 Conversation 并比较输入版本"]
    I --> J{"快照是否过期"}
    J -- "是" --> K["标记 SUPERSEDED"]
    J -- "否" --> L["重新放回 COLLECTING"]
```

---

#### 10. 后台任务的轮询运行

前面的章节描述了单个 Turn 如何处理。本章把领取、处理和恢复重新放回持续运行的后台循环中。

##### 10.1 没有任务时短暂等待

`poll_and_process()` 没有领取到轮次时返回 `False`，主循环按照配置短暂休眠：

```python
await asyncio.sleep(
    settings.ai_worker_poll_interval_ms / 1000
)
```

这样既能及时发现新任务，又避免空队列时持续查询数据库造成忙循环。

##### 10.2 每轮执行顺序

每轮固定执行：

```text
先回收租约过期的 RUNNING Turn
→ 再领取一个已经到期的 COLLECTING Turn
→ 构造请求数据
→ 交给 TurnProcessor 处理
```

一次只领取一轮，处理完成后再进入下一轮。

##### 10.3 为什么远程调用不占用数据库事务

领取阶段使用短事务：

```text
锁定任务
→ 更新 RUNNING、快照和租约
→ 构造输入
→ commit
```

随后关闭 Session，再调用智能服务。最终结果返回后重新打开另一个短事务保存。

如果在数据库事务内等待远程调用，会长时间占用连接和行锁，阻塞用户继续发送消息。

##### 10.4 后台任务完整流程图

```mermaid
flowchart TD
    A["启动 AIWorker"] --> B["回收过期租约"]
    B --> C["查询可领取 Turn"]
    C --> D{"是否存在任务"}
    D -- "否" --> E["短暂休眠"]
    E --> B
    D -- "是" --> F["短事务领取并固定快照"]
    F --> G["关闭数据库事务"]
    G --> H["TurnProcessor 调用智能服务"]
    H --> I{"是否为待提交写操作"}
    I -- "是" --> J["提交前校验并 commit_run"]
    I -- "否" --> K["直接解析最终事件"]
    J --> L["打开最终结算事务"]
    K --> L
    L --> M["校验、保存或重试"]
    M --> N["提交并返回下一轮"]
    N --> B
```

---

#### 11. 本章总结

第四天把第三天创建的收集轮次接入后台处理链路，完成了从领取、调用到最终落库的闭环。

##### 11.1 轮次生命周期总结

主要状态变化为：

```text
COLLECTING
→ 领取后 RUNNING
→ 成功后 COMPLETED
→ 可重试失败时重新 COLLECTING
→ 达到上限后 FAILED
→ 输入快照过期后 SUPERSEDED
```

`COMPLETED`、`FAILED` 和 `SUPERSEDED` 都是终态，不会再次被领取。

##### 11.2 两阶段处理总结

普通回复与业务写操作的核心区别是是否产生外部副作用：

```text
普通回复、只读查询
→ 第一阶段直接得到结果
→ 最终保存前校验

取消、退款、修改等写操作
→ 第一阶段准备决策
→ 提交前校验
→ 第二阶段执行
→ 最终保存前再次校验
```

只有“已经准备但尚未确认提交”的业务决策需要调用 `cancel_run()`。

##### 11.3 事务边界总结

第四天使用多个短事务：

```text
领取事务
→ 远程调用
→ 提交前校验事务（仅业务写操作）
→ 远程提交
→ 最终结算事务
```

远程调用期间不持有数据库连接和行锁。最终结算事务把 Turn 状态、Conversation 版本、AI Message 和 RealtimeOutbox 一起提交。

##### 11.4 后续人工工单内容预告

当前协议已经预留 `RUN_HANDOFF_REQUESTED`，但尚未实现人工工单业务。后续需要继续完成：

```text
创建等待中的人工工单
→ 会话切换为 QUEUED
→ 通知用户端和管理端
→ 客服接单后切换 HUMAN
→ 人工回复与结束工单
```

至此，客服服务已经完成智能消息处理主链路，下一阶段可以围绕人工接管和智能服务自身的工具调用能力继续扩展。

---

<a id="chapter-07"></a>

## 07 · day05_人工客服工单与前端状态处理

> [查看本篇原文](originals/day05_%E4%BA%BA%E5%B7%A5%E5%AE%A2%E6%9C%8D%E5%B7%A5%E5%8D%95%E4%B8%8E%E5%89%8D%E7%AB%AF%E7%8A%B6%E6%80%81%E5%A4%84%E7%90%86/2_resource/day05_%E4%BA%BA%E5%B7%A5%E5%AE%A2%E6%9C%8D%E5%B7%A5%E5%8D%95%E4%B8%8E%E5%89%8D%E7%AB%AF%E7%8A%B6%E6%80%81%E5%A4%84%E7%90%86.md) · [返回目录](#阅读目录)

第五天：人工客服工单与前端状态处理

第四天已经完成 AI Turn 的领取、调用与结果保存。第五天继续处理 AI 无法独立完成的问题，实现**从 AI 请求转人工，到客服接单、回复、结束服务，再切回 AI**的完整业务闭环。

本章先说明**完整流程和事件职责**，再进入数据模型、并发控制与接口实现。<span style="color:red">Day05 只确定事件应当发给谁、前端收到后展示什么；Outbox Worker、Redis、WebSocket 和管理员监控指标留到 Day06。</span>

---

#### 1. 从智能客服进入人工服务

##### 1.1 本章学习目标

完成本章后，需要能够说明：

1. 人工工单和会话模式如何配合；
2. `MESSAGE_CREATED` 与 `HANDOFF_CHANGED` 分别表示什么；
3. AI 请求转人工后如何创建工单；
4. 客服如何接单、回复并结束服务；
5. 为什么工单和会话都需要加锁；
6. 两个前端收到事件后分别展示什么；
7. 消息、状态和 Outbox 如何保持事务一致。

##### 1.2 本章范围与权限边界

```text
Day04：完成一次 AI Turn
Day05：接住转人工结果，完成人工服务流程
Day06：完成 Redis/WebSocket 实时传输和管理员监控指标
```

本章主要涉及：

- `Handoff` 数据模型；
- `HandoffRepository`；
- `HandoffService`；
- `AIEventParser` 与 `AIResultService`；
- `RealtimeService`；
- 工单 Schema、依赖和 Router；
- 用户端与客服端的事件处理。

- `customer`：发送消息，查看 AI 或人工回复；
- `agent`：查看开放工单、接单、回复和结束自己的工单；
- `admin`：只访问独立监控指标，不参与工单处理。

本章实现的是客服 `agent` 的工作台，不提供管理员接管工单功能。

---

#### 2. 完整人工服务流程

##### 2.1 流程概览

人工服务围绕两组核心状态变化展开：

```text
工单状态：waiting → active → resolved
会话模式：QUEUED  → HUMAN  → AI
```

<span style="color:red">Handoff 记录人工服务进度；Conversation.mode 决定新消息路由：`AI` 模式创建 AI Turn，`QUEUED` 或 `HUMAN` 模式不创建 AI Turn，只保存用户消息并通知客服端。</span>

##### 2.2 顺序步骤

1. 用户消息进入 AI Turn；
2. AI 返回 `RUN_HANDOFF_REQUESTED`；
3. Customer Service 创建 `waiting` 工单；
4. 会话模式切换为 `QUEUED`；
5. 用户端显示“等待人工”，客服端显示新工单；
6. 客服接单，工单变为 `active`；
7. 会话模式切换为 `HUMAN`；
8. 用户和客服通过原会话继续发送消息；
9. 客服结束服务，工单变为 `resolved`；
10. 会话模式切回 `AI`，后续消息重新进入 AI 流程。

##### 2.3 完整业务流程图

```mermaid
flowchart LR
    A["用户发送消息"] --> B["AI 处理"]
    B --> C["请求转人工"]
    C --> D["创建 waiting 工单"]
    D --> E["会话进入 QUEUED"]
    E --> F["客服接单"]
    F --> G["工单进入 active"]
    G --> H["会话进入 HUMAN"]
    H --> I["人工回复"]
    I --> J["客服结束服务"]
    J --> K["工单进入 resolved"]
    K --> L["会话切回 AI"]
```

---

#### 3. 两种实时事件与前端展示

本章只保留两种前端实时事件：

- **`MESSAGE_CREATED`**：有一条聊天消息需要展示；
- **`HANDOFF_CHANGED`**：工单状态或会话模式发生变化。

<span style="color:red">产生聊天内容时使用 `MESSAGE_CREATED`，工单或会话状态变化时使用 `HANDOFF_CHANGED`；两者同时发生就创建两种事件。</span>

```text
用户频道：customer-service:user:{user_id}
客服频道：customer-service:staff
```

**用户频道属于单个用户；客服频道由所有客服工作台共同订阅。**

##### 3.1 人工流程中的事件

###### 1. AI 请求转人工

```text
MESSAGE_CREATED → 用户端
展示“正在为你转接人工客服，请稍候”

HANDOFF_CHANGED → 用户端
显示“等待人工”

HANDOFF_CHANGED → 客服端
刷新列表并显示新的 waiting 工单
```

###### 2. 客服接单

```text
HANDOFF_CHANGED → 用户端
显示“人工服务中”

HANDOFF_CHANGED → 客服端
显示 active 状态和负责人
```

###### 3. 用户发送消息

```text
MESSAGE_CREATED → 客服端
同步用户在 QUEUED/HUMAN 模式下发送的新消息
```

**用户发送的消息已经由用户端直接显示，不需要再推送回用户频道。**

###### 4. 客服回复

```text
MESSAGE_CREATED → 用户端
追加人工客服回复

MESSAGE_CREATED → 客服端
同步当前会话消息
```

###### 5. 结束人工服务

```text
MESSAGE_CREATED → 用户端
展示人工服务结束语

HANDOFF_CHANGED → 用户端
切回“AI 客服”

HANDOFF_CHANGED → 客服端
从开放工单列表移除工单
```

##### 3.2 事件与前端展示流程图

```mermaid
flowchart TD
    A["业务操作"] --> B{"产生什么变化"}
    B -->|"产生聊天内容"| C["MESSAGE_CREATED"]
    B -->|"工单或模式变化"| D["HANDOFF_CHANGED"]
    C --> E["用户端追加消息"]
    C --> F["按需同步客服端消息"]
    D --> G["用户端切换服务状态"]
    D --> H["客服端刷新工单状态"]
```

---

#### 4. 人工工单的数据模型与状态

**Conversation 保存聊天上下文，Handoff 保存一次人工服务过程。**独立建模后，可以分别表达：

- 会话当前由谁处理；
- 工单是否等待、处理中或已结束；
- 当前负责人是谁；
- 何时接单、何时结束。

##### 4.1 工单状态与会话模式

- **`waiting`**：已经请求转人工，尚未接单；
- **`active`**：客服已经接单，正在人工处理；
- **`resolved`**：人工服务已经结束。

- **`AI`**：新消息进入 AI Turn；
- **`QUEUED`**：正在等待人工客服；
- **`HUMAN`**：新消息交给人工客服。

```text
Handoff.waiting  ↔ Conversation.QUEUED
Handoff.active   ↔ Conversation.HUMAN
Handoff.resolved ↔ Conversation.AI
```

##### 4.2 核心字段

```python
class Handoff(Base):
    id: Mapped[str]
    conversation_id: Mapped[str]
    user_id: Mapped[str]
    summary: Mapped[str]
    status: Mapped[str]
    assigned_agent_id: Mapped[str | None]
    created_at: Mapped[datetime]
    accepted_at: Mapped[datetime | None]
    resolved_at: Mapped[datetime | None]
```

##### 4.3 状态流转图

```mermaid
stateDiagram-v2
    [*] --> waiting: AI 请求转人工
    waiting --> active: 客服接单
    active --> resolved: 客服结束服务
    resolved --> [*]
```

---

#### 5. 工单的数据访问与并发控制

`HandoffRepository` 只负责数据库访问：

```text
add()                         新增工单
find_open_by_user_id()        查询用户的 waiting/active 工单
list_open()                   查询客服工作台开放工单
find_and_lock_by_id()         查询并锁定指定工单
```

##### 5.1 查询开放工单

客服工作台只关心尚未结束的工单：

```python
select(Handoff).where(
    Handoff.status.in_(["waiting", "active"])
)
```

##### 5.2 锁定工单和会话

接单、回复和结束服务都会先执行：

```text
锁定 Handoff
→ 检查工单状态
→ 锁定 Conversation
→ 执行业务修改
→ 提交事务并释放锁
```

**Handoff 锁防止两个客服同时修改一个工单；Conversation 锁保证消息路由和会话模式不会被并发请求错误覆盖。**

<span style="color:red">两个锁都会一直保持到当前事务提交或回滚，因此状态校验、业务修改和 Outbox 写入必须在同一事务中完成；任一步失败都统一回滚。</span>

##### 5.3 两个客服同时接单

```mermaid
sequenceDiagram
    participant A as 客服 A
    participant DB as PostgreSQL
    participant B as 客服 B

    A->>DB: 查询工单 FOR UPDATE
    DB-->>A: waiting，并锁定
    B->>DB: 查询同一工单 FOR UPDATE
    Note over B,DB: 等待客服 A 提交
    A->>DB: 设置 active 和负责人 A
    A->>DB: COMMIT
    DB-->>B: 返回最新 active 工单
    B->>B: 负责人校验失败
```

---

#### 6. AI 请求转人工的具体实现

<span style="color:red">AI 请求转人工属于一次正常的 AI 最终结果，不属于处理失败，因此当前 Turn 需要标记为 `COMPLETED`。</span> 后续等待、接单和人工处理由 Handoff 负责；Customer Service 还需要保存转接提示、创建工单、修改会话模式并创建状态事件。

##### 6.1 转人工事件的解析

`AIEventParser` 按以下顺序解析事件：

1. 读取 `event_type` 和 `event_data`；
2. 判断事件是否为 `RUN_HANDOFF_REQUESTED`；
3. 设置 `outcome_type="handoff"`；
4. 提取工单摘要，缺失时使用默认摘要；
5. 提取面向用户的转接提示，缺失时使用默认提示；
6. 返回供 Worker 保存的统一结果。

```python
if event_type == AgentEventType.RUN_HANDOFF_REQUESTED:
    return {
        **event_data,
        "outcome_type": "handoff",
        "summary": str(
            event_data.get("summary") or "用户请求人工客服"
        ),
        "content": {
            "text": str(
                event_data.get("message")
                or "正在为你转接人工客服，请稍候。"
            )
        },
    }
```

解析过程可以概括为：

```mermaid
flowchart LR
    A["读取 AI 事件"] --> B{"是否请求转人工"}
    B -->|"否"| C["继续解析其他结果"]
    B -->|"是"| D["设置 handoff 类型"]
    D --> E["提取 summary"]
    E --> F["构造用户提示 content"]
    F --> G["返回统一 outcome"]
```

##### 6.2 转人工结果的保存

Worker 得到 `handoff` 结果后，按以下顺序处理：

1. 在已经锁定当前 Turn 和 Conversation 的结算事务中，把 Turn 标记为 `COMPLETED`；
2. 查询用户现有的开放工单；
3. 没有开放工单时创建 `waiting` 工单，已有开放工单时复用原工单；
4. 根据工单状态设置 `QUEUED` 或 `HUMAN`；
5. 保存用户可见的 AI 转接提示；
6. 创建用户端 `MESSAGE_CREATED`；
7. 创建两个端的 `HANDOFF_CHANGED`；
8. 统一提交事务。

关键实现：

```python
handoff = await self.handoff_service.get_or_create_open_handoff(
    conversation,
    summary=str(outcome["summary"]),
)
conversation.mode = (
    "HUMAN" if handoff.status == "active" else "QUEUED"
)

self.save_ai_result(
    conversation,
    turn,
    outcome["content"],
    message_id=outcome.get("message_id"),
)

self.realtime_service.add_handoff_changed_events(
    conversation,
    handoff,
)
```

`get_or_create_open_handoff()` 可能返回：

```text
新建 waiting  → 会话设置为 QUEUED
已有 waiting  → 会话保持 QUEUED
已有 active   → 会话保持 HUMAN
```

<span style="color:red">这个判断可以防止重复事件把正在人工服务的会话错误退回排队状态。</span>

**Turn 表示一次 AI 处理任务。**AI 已经成功给出“转人工”结果，因此 AI Turn 已完成；后续等待和处理属于 Handoff 生命周期。

```text
ConversationTurn.status = COMPLETED
Handoff.status = waiting
```

结果保存流程如下：

```mermaid
flowchart TD
    A["得到 handoff outcome"] --> B["Turn 标记 COMPLETED"]
    B --> C["获取或创建开放工单"]
    C --> D{"工单是否 active"}
    D -->|"是"| E["Conversation = HUMAN"]
    D -->|"否"| F["Conversation = QUEUED"]
    E --> G["保存 AI 转接提示"]
    F --> G
    G --> H["创建 MESSAGE_CREATED"]
    H --> I["创建 HANDOFF_CHANGED"]
    I --> J["统一提交事务"]
```

---

#### 7. HandoffService 与三个工单操作

##### 7.1 公共处理逻辑

三个写操作都遵循相同的处理顺序：

```text
锁定工单
→ 排除 resolved 工单
→ 锁定所属会话
→ 校验业务条件
→ 修改数据并创建事件
→ 提交事务
```

<span style="color:red">人工回复和结束服务只能由当前工单负责人执行。</span>

```python
if handoff.assigned_agent_id != agent_id:
    raise HTTPException(
        status_code=403,
        detail="工单不属于当前客服",
    )
```

##### 7.2 客服接单

顺序步骤：

1. 锁定工单和会话；
2. 如果已经是 `active`，原负责人重复接单时直接结束，其他客服接单时返回 `403`；
3. 把工单改为 `active`；
4. 保存当前客服 ID 和接单时间；
5. 把会话改为 `HUMAN`；
6. 给两个端创建 `HANDOFF_CHANGED`；
7. 提交事务。

```mermaid
flowchart LR
    A["waiting 工单"] --> B["客服接单"]
    B --> C["status = active"]
    C --> D["记录负责人"]
    D --> E["mode = HUMAN"]
    E --> F["HANDOFF_CHANGED"]
```

##### 7.3 客服回复

顺序步骤：

1. 锁定工单和会话；
2. 校验当前客服是负责人；
3. 根据 `message_id` 查询已有消息；
4. 已存在时直接结束，避免重复消息；
5. 不存在时保存 `role=human` 的消息；
6. 给用户端和客服端创建 `MESSAGE_CREATED`；
7. 提交事务。

```mermaid
flowchart TD
    A["客服提交回复"] --> B["锁定工单和会话"]
    B --> C["校验负责人"]
    C --> D{"message_id 已存在"}
    D -->|"是"| E["直接结束"]
    D -->|"否"| F["保存 human 消息"]
    F --> G["两个端 MESSAGE_CREATED"]
    G --> H["提交事务"]
```

##### 7.4 结束人工服务

顺序步骤：

1. 锁定工单和会话；
2. 校验当前客服是负责人；
3. 把工单改为 `resolved`；
4. 记录结束时间；
5. 把会话模式切回 `AI`；
6. 保存服务端生成的结束语；
7. 只给用户端创建结束语 `MESSAGE_CREATED`；
8. 给两个端创建 `HANDOFF_CHANGED`；
9. 提交事务。

```mermaid
flowchart LR
    A["active 工单"] --> B["客服结束服务"]
    B --> C["status = resolved"]
    C --> D["mode = AI"]
    D --> E["用户端结束语"]
    E --> F["两个端状态事件"]
```

---

#### 8. 工单接口与两个前端

##### 8.1 工单接口

```text
GET  /api/v1/handoffs
查询 waiting 和 active 工单

POST /api/v1/handoffs/{handoff_id}/accept
当前客服接单

POST /api/v1/handoffs/{handoff_id}/messages
当前负责人发送人工回复

POST /api/v1/handoffs/{handoff_id}/resolve
当前负责人结束人工服务
```

**写接口成功后不返回额外业务数据。前端通过重新查询和状态事件获得最新结果。**

##### 8.2 接口顺序

1. Router 校验当前身份为 `agent`；
2. 从令牌中取得当前客服 ID；
3. 调用 `HandoffService`；
4. Service 完成锁定、校验、保存和事件创建；
5. Router 返回成功状态。

##### 8.3 用户端处理

用户端维护当前会话模式：

```javascript
function conversationModeLabel(mode) {
  return {
    AI: 'AI 客服',
    QUEUED: '等待人工',
    HUMAN: '人工服务中',
  }[mode] || mode
}
```

收到事件后：

```text
MESSAGE_CREATED → 追加聊天消息
HANDOFF_CHANGED → 更新 conversationMode，并刷新会话
```

**用户自己发送的消息由页面直接显示，不依赖 WebSocket 再推送一次。**

页面刷新时还会查询当前会话，从数据库恢复 `AI`、`QUEUED` 或 `HUMAN`。

##### 8.4 客服端处理

客服端收到事件后：

```text
新建 waiting 工单 → 刷新开放工单列表
工单变为 active  → 显示负责人和处理中状态
用户发送新消息   → 刷新当前会话
收到人工消息     → 刷新当前会话
工单变为 resolved → 移除工单并关闭详情
```

<span style="color:red">只有 `assigned_agent_id` 等于当前客服 ID 时，前端才启用回复和结束操作；其他客服不能操作，后端仍会返回 `403`。</span>

##### 8.5 前后端协作图

```mermaid
flowchart TD
    A["用户端"] -->|"用户消息"| B["Customer Service"]
    B -->|"waiting 状态事件"| A
    B -->|"新工单状态事件"| C["客服端"]
    C -->|"接单 / 回复 / 结束"| B
    B -->|"人工消息"| A
    B -->|"工单状态变化"| C
```

---

#### 9. 事务一致性

##### 9.1 工单与会话状态的一致性

接单和结束服务会同时修改 Handoff 与 Conversation：

```text
接单：waiting + QUEUED → active + HUMAN
结束：active + HUMAN → resolved + AI
```

<span style="color:red">Handoff 与 Conversation 的状态修改必须使用同一个 Session 和同一次事务提交。</span>

##### 9.2 消息与 Outbox 的原子提交

人工回复必须保证：

```text
消息保存成功，事件也保存成功；
消息保存失败，事件也不能留下。
```

<span style="color:red">Message、Handoff、Conversation 和 RealtimeOutbox 必须在当前事务中写入，最后统一 `commit()`。</span>

##### 9.3 Day06 预告

Day05 确定事件内容、接收频道和前端行为。Day06 继续完成实时传输和管理员监控：

```text
读取 RealtimeOutbox
→ 发布 Redis 消息
→ WebSocket 订阅频道
→ 把事件转发给用户端和客服端
→ 实现管理员监控指标查询与前端指标卡片
```


---

<a id="chapter-08"></a>

## 08 · day06_实时推送与监控

> [查看本篇原文](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/day06_%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7.md) · [返回目录](#阅读目录)

第六天：实时推送与监控

第五天已经完成人工工单流程，并在业务事务中创建了 `MESSAGE_CREATED` 和 `HANDOFF_CHANGED` 事件。第六天继续完成事件传输，让事件经过 **Outbox Worker、Redis Pub/Sub 和 WebSocket** 到达用户端与客服端，并补充历史消息和管理员监控指标。

---

#### 1. 整体架构

##### 1.1 本日目标

本日主要完成：

1. 将数据库中的 Outbox 事件发布到 Redis；
2. 通过 WebSocket 把 Redis 消息转发给前端；
3. 支持用户端和客服端订阅不同频道；
4. 支持历史消息的完整查询和增量查询；
5. 提供管理员监控指标接口。

##### 1.2 核心组件

实时链路由四个组件组成：

- **RealtimeOutbox**：保存等待发布的事件；
- **Outbox Worker**：轮询并发布事件；
- **Redis Pub/Sub**：完成实时消息广播；
- **WebSocket Router**：把 Redis 消息转发给浏览器。

<span style="color:red">业务 Service 只负责写入 Outbox，不直接连接 Redis；Outbox Worker 只负责发布事件，不修改业务数据。</span>

##### 1.3 完整链路

```text
业务 API
→ 保存 Message / Handoff / Conversation
→ 同一事务写入 RealtimeOutbox
→ Outbox Worker 查询待发布事件
→ Redis Pub/Sub
→ WebSocket
→ 用户端或客服端
```

```mermaid
flowchart LR
    A["业务 API"] --> B["业务数据"]
    A --> C["RealtimeOutbox"]
    C --> D["Outbox Worker"]
    D --> E["Redis Pub/Sub"]
    E --> F["WebSocket"]
    F --> G["用户端"]
    F --> H["客服端"]
```

---

#### 2. 实时事件

##### 2.1 事件结构

Outbox Worker 发布前，会把数据库记录转换成统一事件：

```python
def build_realtime_event(
    event_id: str,
    event_type: str,
    event_data: dict[str, Any],
    conversation_id: str,
    event_created_at: datetime
) -> dict[str, Any]:
    return {
        "event_id": event_id,
        "event_type": event_type,
        "event_data": event_data,
        "event_created_at": event_created_at.isoformat(),
        "conversation_id": conversation_id
    }
```

字段职责：

- `event_id`：事件唯一标识；
- `event_type`：事件类型；
- `event_data`：事件业务数据；
- `event_created_at`：事件创建时间；
- `conversation_id`：关联会话。

##### 2.2 消息事件

`MESSAGE_CREATED` 表示有一条新消息需要处理：

```python
{
    "sequence": message.id,
    "message_id": message.message_id,
    "conversation_id": message.conversation_id,
    "role": message.role,
    "type": message.message_type,
    "content": message.content,
    "created_at": message.created_at.isoformat()
}
```

用户端收到后可以直接追加 **AI回复 或人工回复**；客服端收到后可以**刷新当前会话**。

<strong style='color:red'>用户端:</strong>

**人工回复**

AI转人工阶段（开场白）：

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/1.png)

人工转AI阶段（结束语）：

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/2.png)

**AI回复**

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/3.png)

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/4.png)

<strong style='color:red'>客服端:</strong>

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/5.png)

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/6.png)

##### 2.3 工单事件

`HANDOFF_CHANGED` 表示工单状态或会话模式发生变化：

```python
{
    "handoff_id": handoff.id,
    "status": handoff.status,
    "summary": handoff.summary,
    "assigned_agent_id": handoff.assigned_agent_id,
    "conversation_mode": conversation.mode
}
```

前端主要使用：

- `status`：展示 `waiting`、`active` 或 `resolved`；
- `assigned_agent_id`：展示负责人；
- `conversation_mode`：展示 `AI`、`QUEUED` 或 `HUMAN`。

用户端收到后可以**更新会话模式**；客服端收到后可以**刷新工单列表和工单状态**。

<strong style="color:red">用户端</strong>：

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/7.png)

<strong style="color:red">客服端</strong>：

![](originals/day06_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%AE%9E%E6%97%B6%E6%8E%A8%E9%80%81%E4%B8%8E%E7%9B%91%E6%8E%A7%EF%BC%89/2_resource/images/8.png)

##### 2.4 事件频道

```python
STAFF_CHANNEL = "customer-service:staff"
USER_CHANNEL_PREFIX = "customer-service:user:"
```

频道规则：

```text
customer → customer-service:user:{user_id}
agent    → customer-service:staff
```

用户频道相互隔离；全部客服共享一个工作台频道。

---

#### 3. Outbox 写入

##### 3.1 Outbox 模型

`RealtimeOutbox` 保存事件发布需要的数据：

```text
channel       发布频道
event_type    事件类型
conversation_id 关联会话
data          业务数据
attempts      发布次数
created_at    创建时间
published_at  发布时间
```

新事件的 `published_at` 为 `NULL`，表示尚未发布。

##### 3.2 事件创建

`RealTimeOutBoxService` 根据接收端确定频道：

```python
def add_message_created_events(
    self,
    conversation: Conversation,
    message: Message,
    *,
    notify_user: bool,
    notify_staff: bool
):
    channels = []
    if notify_user:
        channels.append(user_channel(conversation.user_id))
    if notify_staff:
        channels.append(STAFF_CHANNEL)

    for channel in channels:
        self._add_outbox_event(
            channel,
            RealTimeOutBoxType.MESSAGE_CREATED,
            build_message_created_data(message),
            conversation_id=conversation.id
        )
```

##### 3.3 事务提交

业务数据和 Outbox 必须使用同一个 Session：

```text
保存消息
→ 修改会话或工单
→ 创建 Outbox 事件
→ 统一 commit
```

如果事务失败，消息和事件一起回滚；不会出现业务数据失败但事件仍然发布的情况。

<span style="color:red">Outbox 的核心作用是保证业务事务与事件记录的一致性。</span>

##### 3.4 消息刷新

`Message.id` 是数据库自增序号，`created_at` 也会在写入时生成。因此构建消息事件前需要：

```python
await self.session.flush()
```

`flush()` 会执行 SQL 并回填数据库生成字段，但不会提交事务。

```text
创建 Message
→ flush
→ 获取 message.id 和 message.created_at
→ 构建 Outbox 事件
→ commit
```

---

#### 4. 事件发布

##### 4.1 Worker 轮询

Outbox Worker 独立运行：

```python
async def start(self):
    while True:
        published = await self.publish_next_event()
        if not published:
            await asyncio.sleep(
                self.settings.outbox_worker_poll_interval_seconds
            )
```

没有待发布事件时暂停一段时间，避免持续空查询数据库。

##### 4.2 查询事件

```python
async def find_next_unpublished(self) -> RealtimeOutbox | None:
    return await self.session.scalar(
        select(RealtimeOutbox)
        .where(RealtimeOutbox.published_at.is_(None))
        .order_by(RealtimeOutbox.sequence)
        .limit(1)
    )
```

`published_at IS NULL` 表示查询待发送队列。

##### 4.3 构建数据

Worker 将数据库记录转换成 JSON：

```python
event_json = json.dumps(
    build_realtime_event(
        event_id=event.id,
        event_type=event.event_type,
        event_data=event.data,
        conversation_id=event.conversation_id,
        event_created_at=event.created_at
    ),
    ensure_ascii=False
)
```

`ensure_ascii=False` 用于保留中文内容。

##### 4.4 发布结果

```python
try:
    await self.redis.publish(event.channel, event_json)
except RedisError:
    outbox_repo.mark_publish_failed(event)
else:
    outbox_repo.mark_publish_succeeded(event)

await session.commit()
```

发布成功后设置 `published_at`；失败时只增加 `attempts`，事件仍保持待发布状态，下一轮继续处理。

```mermaid
flowchart TD
    A["查询 published_at 为 NULL"] --> B{"存在事件？"}
    B -->|"否"| C["等待下一轮"]
    B -->|"是"| D["构建 JSON"]
    D --> E["发布 Redis"]
    E --> F{"发布成功？"}
    F -->|"是"| G["设置 published_at"]
    F -->|"否"| H["增加 attempts"]
    G --> I["提交事务"]
    H --> I
```

---

#### 5. Redis 通信

##### 5.1 发布与订阅

Outbox Worker 是发布者：

```python
await redis_client.publish(channel, event_json)
```

WebSocket Router 是订阅者：

```python
pubsub = redis_client.pubsub()
await pubsub.subscribe(channel)
```

##### 5.2 频道划分

同一条人工工单状态事件会写入两个频道：

```text
用户频道 → 更新用户端会话模式
客服频道 → 刷新客服工单列表
```

消息事件则根据业务场景选择接收端：

```text
AI 回复       → 用户频道
人工回复      → 用户频道 + 客服频道
人工阶段用户消息 → 客服频道
```

##### 5.3 职责边界

```text
Message 表        保存聊天历史
RealtimeOutbox 表 保证服务端最终发布
Redis Pub/Sub     完成实时通知
History 接口      补充前端遗漏消息
```

<span style="color:red">Outbox 解决数据库到 Redis 的发布问题。</span>

---

#### 6. WebSocket 推送

##### 6.1 连接认证

浏览器建立连接后，先发送 Token：

```json
{
  "type": "authenticate",
  "token": "..."
}
```

服务端解析用户身份：

```python
auth_message = await websocket.receive_json()
current_user = auth_service.decode_access_token(
    str(auth_message["token"])
)
```

##### 6.2 频道订阅

```python
channel = (
    user_channel(current_user.user_id)
    if current_user.role == "customer"
    else STAFF_CHANNEL
)
```

订阅完成后通知前端：

```python
await websocket.send_json({"event_type": "connected"})
```

##### 6.3 消息转发

```python
async def forward_events():
    async for item in pubsub.listen():
        if item["type"] == "message":
            await websocket.send_text(item["data"])
```

Redis 中已经保存的是 JSON 字符串，因此可以直接通过 WebSocket 转发。

##### 6.4 断开清理

服务端不接收业务消息，只等待浏览器断开：

```python
try:
    while True:
        await websocket.receive_text()
except WebSocketDisconnect:
    pass
finally:
    forward_task.cancel()
    await asyncio.gather(
        forward_task,
        return_exceptions=True
    )
    await pubsub.aclose()
```

`receive_text()` 的作用不是处理业务数据，而是及时感知前端断开。

---

#### 7. 历史消息

##### 7.1 历史页面

用户进入历史页面时调用：

```text
GET /api/v1/chat/history
```

接口返回当前用户的历史消息，前端再按照会话进行分组展示。

##### 7.2 增量查询

历史页面再次查询时，可以携带上次最后一条消息的序号：

```text
GET /api/v1/chat/history?after_sequence=最后序号
```

Repository 增加条件：

```python
if after_sequence is not None:
    statement = statement.where(Message.id > after_sequence)
```

这样只返回新增的历史消息，减少数据库查询结果和网络传输量。

##### 7.3 会话分组

历史响应保留：

```text
conversation_id
conversation_started_at
```

前端按 `conversation_id` 分组，使用 `conversation_started_at` 展示会话开始时间：

```text
今天 09:30 开始的会话
昨天 16:20 开始的会话
```

---

#### 8. 监控指标

##### 8.1 角色边界

```text
agent → 查看和处理人工工单
admin → 只查看监控指标
```

管理员不订阅客服频道，也不能接单、回复或结束工单。

##### 8.2 指标统计

Customer Service 提供五个指标：

```text
conversations 会话总数
messages      消息总数
handoffs      工单总数
queued        当前等待人工的会话数
human         当前人工服务中的会话数
```

`handoffs` 包含历史工单；已经结束并切回 AI 的工单不会计入 `queued` 或 `human`。

##### 8.3 指标接口

```text
GET /api/v1/admin/metrics
```

接口只允许 `admin` 访问。

```python
@router.get("/metrics", response_model=AdminMetricsResponse)
async def get_admin_metrics(...):
    auth_service.get_authorized_user(
        authorization,
        "admin"
    )
    return await metrics_service.get_metrics()
```

##### 8.4 前端展示

管理员前端使用指标卡片展示：

```text
会话总量
消息总量
人工工单总量
等待人工数量
人工处理中数量
```

管理员页面只发起 HTTP 指标查询，不建立客服 WebSocket 连接。

---

#### 9. 启动与预告

##### 9.1 服务启动

Customer Service 需要启动三个进程：

```powershell
# Customer Service API
uvicorn atguigu.app.app:app --port 8000

# AI Worker
python -m atguigu.worker.ai.worker

# Outbox Worker
python -m atguigu.worker.realtime
```

同时需要保证 PostgreSQL 和 Redis 已经启动。

##### 9.2 链路验证

```text
用户发送消息
→ Message 与 Turn 写入数据库
→ AI Worker 保存 AI 结果和 Outbox
→ Outbox Worker 发布 Redis
→ WebSocket 转发
→ 用户端显示 AI 回复
```

人工流程：

```text
AI 请求转人工
→ 创建 waiting 工单
→ 客服端收到状态事件
→ 客服接单并回复
→ 用户端收到人工消息
→ 客服结束服务
→ 会话切回 AI
```

##### 10.  Day07 预告

Day07 上午先在 Customer Service 项目中启动 Mock AI Service：

```text
普通问题     → 模拟一阶段只读结果
我要取消订单 → 模拟两阶段写操作
我要转人工   → 模拟人工工单
```

完成 Customer Service 全链路验证后，下午开始搭建正式 AI Service。

---

<a id="chapter-09"></a>

## 09 · Customer Service 项目总结与面试指南

> [查看本篇原文](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md) · [返回目录](#阅读目录)

### Customer Service 项目总结与面试指南



#### 目录

- [1. 项目概述](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#1-项目概述)
- [2. 整体架构设计](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#2-整体架构设计)
- [3. 核心数据模型与状态设计](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#3-核心数据模型与状态设计)
- [4. 用户消息接收与会话管理](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#4-用户消息接收与会话管理)
- [5. ConversationTurn 与 AI Worker](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#5-conversationturn-与-ai-worker)
- [6. 人工客服转接](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#6-人工客服转接)
- [7. 实时消息推送](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#7-实时消息推送)
- [8. 管理后台与监控](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#8-管理后台与监控)
- [9. 认证与内部服务安全](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#9-认证与内部服务安全)
- [10. 事务、并发与一致性](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#10-事务并发与一致性)
- [11. 异常处理与系统恢复](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#11-异常处理与系统恢复)
- [12. 设计取舍与优化思路](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#12-设计取舍与优化思路)
- [13. 面试高频问题](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#13-面试高频问题)
- [附录](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/Customer%20Service%20%E9%A1%B9%E7%9B%AE%E6%80%BB%E7%BB%93%E4%B8%8E%E9%9D%A2%E8%AF%95%E6%8C%87%E5%8D%97.md#附录)

---

#### 1. 项目概述

##### 1.1 项目背景

Customer Service 是电商智能客服系统中的会话中枢，负责连接用户端、人工客服端和 AI Service。

它不负责模型推理，也不直接修改订单、地址或售后单，而是负责：

1. 接收并持久化用户消息；
2. 管理用户当前会话；
3. 将连续用户消息合并为一个处理轮次；
4. 异步调用 AI Service；
5. 校验 AI 结果是否仍然有效；
6. 保存 AI 回复或创建人工工单；
7. 通过 Outbox、Redis 和 WebSocket 推送实时事件；
8. 为管理后台提供客服业务指标。

##### 1.2 核心业务目标

- 用户发送消息后，接口快速完成处理，不同步等待 AI。
- 用户短时间连续发送的消息可以合并为一次 AI 请求。
- 同一会话中的 AI 回复保持顺序，避免并行生成导致答非所问。
- AI 处理期间出现新输入时，旧结果不能覆盖新上下文。
- AI 无法处理时，可以平滑转入人工客服。
- 消息、工单和实时事件保持事务一致。
- Worker 崩溃后，未完成的 Turn 可以重新处理。

##### 1.3 技术选型

- Web 框架：FastAPI
- 数据校验：Pydantic
- ORM：SQLAlchemy 2.0 Async
- 数据库：PostgreSQL
- PostgreSQL 异步驱动：Psycopg 3
- 实时消息：Redis Pub/Sub + WebSocket
- 服务调用：httpx
- 身份认证：JWT
- 运行模型：API 服务、AI Worker、Outbox Worker 独立运行

##### 1.4 项目主要功能

- 用户会话创建、超时关闭与当前会话查询；
- 文本消息和对象消息接收；
- 基于 `message_id` 的消息去重；
- 用户连续消息合并；
- Turn 领取、租约、重试和超时恢复；
- AI Service 的 `start`、`commit`、`cancel` 调用；
- AI 回复保存与转人工；
- 人工工单列表、接单、回复和结束；
- 消息事件和工单事件实时推送；
- 历史消息完整查询与增量查询；
- 管理员客服指标聚合。

##### 1.5 项目技术亮点

1. 使用 `ConversationTurn` 将“消息写入”和“AI 执行”解耦。
2. 使用收集窗口合并用户短时间连续输入。
3. 使用输入版本和快照版本阻止过期 AI 结果落库。
4. 使用数据库行锁和部分唯一索引控制并发。
5. 使用租约机制恢复 Worker 中断遗留任务。
6. 使用 Transactional Outbox 保证业务数据与实时事件同时提交。
7. 使用双重身份信息保护 Customer Service 到 AI Service 的内部调用。

---

#### 2. 整体架构设计

##### 2.1 系统组成

```mermaid
flowchart LR
    U[用户前端] -->|HTTP / JWT| API[Customer Service API]
    A[客服工作台] -->|HTTP / JWT| API
    API --> DB[(PostgreSQL)]

    AW[AI Worker] --> DB
    AW -->|内部令牌 + 用户 JWT| AI[AI Service]
    AI --> AW

    OW[Outbox Worker] --> DB
    OW --> R[(Redis Pub/Sub)]
    R --> WS[WebSocket 路由]
    WS --> U
    WS --> A

    ADM[管理后台] -->|管理员 JWT| API
```

API 服务、AI Worker 和 Outbox Worker 使用同一数据库，但承担不同职责：

- API 服务处理 HTTP、WebSocket、认证和业务事务；
- AI Worker 消费待处理 Turn；
- Outbox Worker 发布待推送事件；
- AI Service 独立负责模型执行和 AgentRun 生命周期。

##### 2.2 Customer Service 分层架构

项目主要分为以下层次：

- `app/routers`：接收请求、读取 Header、执行角色校验；
- `app/schemas`：定义请求和响应结构；
- `app/services`：组织会话、消息、Turn、工单等业务流程；
- `app/respositories`：封装 SQLAlchemy 查询和持久化；
- `models/models.py`：定义数据库模型；
- `worker/ai`：异步 AI 处理；
- `worker/realtime.py`：Outbox 事件发布；
- `infrastucture`：数据库和 Redis 连接；
- `common`：配置、ID、时间和事件循环工具。

Router 不直接写 SQL，Repository 不负责决定业务状态，事务通常由 Service 或 Worker 在完整业务步骤结束后显式提交。

##### 2.3 用户消息完整处理链路

```mermaid
sequenceDiagram
    participant U as 用户
    participant API as Message Router
    participant MS as MessageService
    participant CS as ConversationService
    participant TS as TurnService
    participant DB as PostgreSQL

    U->>API: POST /api/v1/chat/messages
    API->>API: 校验 customer JWT
    API->>MS: accept_user_message()
    MS->>DB: 按 message_id 查询重复消息
    alt 重复消息
        DB-->>MS: 已存在消息和会话
        MS-->>U: 返回原 conversation_id、mode
    else 新消息
        MS->>CS: 获取并锁定当前有效会话
        CS->>DB: 查询/创建 Conversation
        MS->>DB: 新增 Message
        alt mode = AI
            MS->>TS: 加入 COLLECTING Turn
        else mode = QUEUED/HUMAN
            MS->>DB: 新增客服端消息 Outbox
        end
        MS->>DB: 原子提交
        MS-->>U: conversation_id、mode
    end
```

##### 2.4 AI 回复处理链路

1. API 保存用户消息并创建或更新 `COLLECTING` Turn。
2. AI Worker 查询收集时间已结束的 Turn。
3. Worker 将 Turn 更新为 `RUNNING` 并固定 `snapshot_revision`。
4. Worker 查询本轮消息和历史消息，构建 AI 请求。
5. Worker 调用 AI Service 的 `start_run()`。
6. 如果 AI 返回 `run_decision_prepared`，Customer Service 再校验输入快照。
7. 快照有效时调用 `commit_run()`；失效时调用 `cancel_run()`。
8. Worker 再次锁定会话，完成最终版本校验。
9. 保存 AI 消息或人工工单，同时写入 Outbox。

##### 2.5 人工客服处理链路

```mermaid
flowchart TD
    A[AI 请求转人工] --> B[创建或复用 waiting 工单]
    B --> C[Conversation.mode = QUEUED]
    C --> D[客服工作台收到 handoff_changed]
    D --> E[客服接单]
    E --> F[Handoff.status = active]
    F --> G[Conversation.mode = HUMAN]
    G --> H[客服与用户互发消息]
    H --> I[负责人结束服务]
    I --> J[Handoff.status = resolved]
    J --> K[Conversation.mode = AI]
```

##### 2.6 实时事件推送链路

业务 Service 不直接连接 Redis，而是在当前事务中写入 `RealtimeOutbox`。Outbox Worker 再按顺序读取事件、发布到 Redis，WebSocket 路由订阅授权频道并转发给前端。

这样可以避免“数据库提交成功但实时通知丢失”或“通知已经发送但数据库事务回滚”的明显不一致。

##### 2.7 Customer Service 与 AI Service 的职责边界

Customer Service 是会话状态和输入版本的权威来源：

- 管理 Conversation、Message、Turn、Handoff；
- 判断 AI 结果是否基于当前输入；
- 决定结果能否写回会话；
- 保存面向用户的最终消息。

AI Service 负责：

- 创建并执行 AgentRun；
- 调用模型和只读业务工具；
- 生成回复、转人工建议或页面操作引导；
- 保存 AgentRun 的执行状态和监控信息。

对于取消订单、修改地址等真正的业务写操作，AI 只返回受控页面链接，最终操作由用户进入业务页面后确认。

---

#### 3. 核心数据模型与状态设计

##### 3.1 Conversation 会话模型

`Conversation` 表示一个用户当前的客服会话，核心字段包括：

- `user_id`：会话所属用户；
- `mode`：当前处理模式；
- `last_active_at`：最后活跃时间；
- `input_revision`：已经接收的用户输入版本；
- `answered_revision`：已经完成回答的输入版本；
- `ended_at`：会话结束时间。

数据库通过部分唯一索引限制每个用户最多只有一个 `AI`、`QUEUED` 或 `HUMAN` 状态的开放会话。

##### 3.2 Message 消息模型

`Message` 同时保存用户、AI 和人工客服消息：

- `role`：`user`、`ai` 或 `human`；
- `message_type`：`text` 或 `object`；
- `content`：JSONB 消息内容；
- `input_revision`：用户消息所属输入版本；
- `agent_run_id`：生成 AI 消息的 AgentRun；
- `agent_outcome_seq`：AgentRun 输出序号。

`message_id` 具有唯一约束，用于阻止同一业务消息被重复插入。

##### 3.3 ConversationTurn 处理轮次

一个 Turn 表示“一批需要一起交给 AI 处理的用户输入”。

关键字段：

- `start_revision`：本轮第一条输入版本；
- `snapshot_revision`：Worker 领取时固定的最后输入版本；
- `collect_until`：最早可领取时间；
- `max_collect_until`：连续消息合并的最迟等待时间；
- `locked_by`、`locked_until`：Worker 租约；
- `attempts`、`last_error`：重试信息；
- `run_id`：对应的 AI Service AgentRun。

##### 3.4 Handoff 人工工单

`Handoff` 保存人工转接过程：

- `waiting`：等待客服；
- `active`：已有负责人；
- `resolved`：已经结束。

部分唯一索引保证同一用户或同一会话最多只有一个开放工单。

##### 3.5 RealtimeOutbox 实时事件

`RealtimeOutbox` 保存尚未发布或已经发布的实时事件：

- `sequence`：全局递增发布顺序；
- `channel`：Redis 频道；
- `event_type`：消息或工单事件；
- `data`：事件数据；
- `attempts`：发布尝试次数；
- `published_at`：成功发布时间。

##### 3.6 会话模式状态流转

```mermaid
stateDiagram-v2
    [*] --> AI
    AI --> QUEUED: AI 请求转人工
    QUEUED --> HUMAN: 客服接单
    HUMAN --> AI: 客服结束人工服务
    AI --> CLOSED: AI 会话空闲超时
    CLOSED --> [*]
```

当前 `_close_timeout_conversation()` 只查询并关闭 `AI` 模式的超时会话，因此 `QUEUED` 和 `HUMAN` 不会被这段逻辑自动关闭。

##### 3.7 Turn 状态流转

```mermaid
stateDiagram-v2
    [*] --> COLLECTING
    COLLECTING --> RUNNING: Worker 领取
    RUNNING --> COMPLETED: AI 结果有效并保存
    RUNNING --> SUPERSEDED: 输入快照过期
    RUNNING --> COLLECTING: 调用失败且允许重试
    RUNNING --> COLLECTING: 租约超时且快照有效
    RUNNING --> FAILED: 达到最大重试次数
```

##### 3.8 Handoff 状态流转

```mermaid
stateDiagram-v2
    [*] --> waiting
    waiting --> active: 客服接单
    active --> resolved: 负责人结束服务
    resolved --> [*]
```

##### 3.9 input_revision 与 answered_revision

每保存一条新的用户消息：

```text
conversation.input_revision += 1
message.input_revision = conversation.input_revision
```

Worker 领取 Turn 时：

```text
turn.snapshot_revision = conversation.input_revision
```

结果落库前比较：

```text
conversation.input_revision == turn.snapshot_revision
```

相等表示 AI 执行期间没有新输入，结果仍然有效；不相等表示旧结果已经过期，应将 Turn 标记为 `SUPERSEDED`。

最终结算完成后：

```text
conversation.answered_revision = turn.snapshot_revision
```

这里不仅包括正常 AI 回复和转人工，也包括重试耗尽后已经向用户保存失败提示的情况。因为该轮输入已经得到一个终结性响应，不应继续被后续 Turn 重复处理。

---

#### 4. 用户消息接收与会话管理

##### 4.1 用户身份校验

消息接口只允许 `customer` 角色访问。Router 从 Authorization Header 提取 Bearer Token，通过 `AuthService` 解码为 `CurrentUser`，业务层只接收已经认证的 `user_id`。

##### 4.2 获取或创建当前会话

`ConversationService.ensure_activate_conversation()`：

1. 检查当前 AI 会话是否空闲超时；
2. 查询 `AI`、`QUEUED`、`HUMAN` 中的当前有效会话；
3. 没有有效会话时创建新的 `AI` 会话。

项目通过数据库部分唯一索引提供最终约束，防止一个用户出现多个开放会话。

##### 4.3 消息幂等处理

`MessageService.accept_user_message()` 首先按照 `message_id` 查询消息。已经存在时不重复插入，而是直接返回原会话 ID 和模式。

这解决的是“防止重复写入”的基础幂等，不等于完整的请求结果重放。

当前 `message_id` 是全局唯一，查询重复消息时没有同时校验消息所属用户。生产环境应在返回原会话信息前核对会话所有者，避免不同用户复用同一 `message_id` 时泄露会话信息。

##### 4.4 用户消息并发控制

当前代码在获得有效会话后，通过：

```text
session.refresh(conversation, with_for_update=True)
```

锁定会话行，使同一会话的后续消息依次修改 `input_revision`。

这个方案可以保护已经存在的会话行，但两个请求同时为新用户创建首个会话时仍可能竞争，最终依赖部分唯一索引阻止重复开放会话。生产版本可以增加用户维度的 PostgreSQL advisory lock，把“查询或创建会话”整体串行化。

##### 4.5 消息序号与输入版本

`Message.id` 是数据库递增序号，负责稳定排序和增量历史查询；`input_revision` 是会话输入版本，负责判断 AI 结果是否过期。二者用途不同：

- sequence 回答“消息写入顺序是什么”；
- revision 回答“AI 处理的是哪一版用户输入”。

##### 4.6 连续消息合并

第一条用户消息创建 `COLLECTING` Turn，默认在 800ms 后允许 Worker 领取。期间有新消息时延长 `collect_until`，但不能超过 `max_collect_until`，默认最大等待 2000ms。

这既能减少模型调用次数，也不会让用户因为持续输入而无限等待。

##### 4.7 消息、Turn 与 Outbox 的原子提交

`MessageService` 在一个数据库事务中完成：

- 保存用户消息；
- 更新 Conversation；
- 创建或更新 Turn，或者创建客服端 Outbox；
- 最后统一 `commit()`。

其中任一步骤抛出异常，FastAPI 的 Session 依赖执行 `rollback()`。

##### 4.8 历史消息查询

`GET /api/v1/chat/history` 支持：

- 不传 `after_sequence`：返回当前用户全部历史消息；
- 传入 `after_sequence`：只返回 `Message.id` 更大的消息。

历史接口读取 PostgreSQL 中已经保存的事实；WebSocket 负责推送连接期间的新事件。两者互相补充，但不是同一条连接。

---

#### 5. ConversationTurn 与 AI Worker

##### 5.1 为什么需要 ConversationTurn

如果消息接口直接调用 AI，会产生三个问题：

1. HTTP 请求需要长时间等待；
2. 用户连续消息会触发多次模型调用；
3. 多个模型请求可能乱序返回。

Turn 将用户消息先持久化为待处理任务，让 API 快速返回，由 Worker 异步执行。

##### 5.2 COLLECTING 消息收集阶段

`COLLECTING` 表示 Turn 还可以继续接收用户消息。新消息到达时优先锁定并复用当前 `COLLECTING` Turn；如果当前 Turn 已经被 Worker 领取为 `RUNNING`，则创建下一轮 Turn。

部分唯一索引保证同一会话最多存在一个 `COLLECTING` Turn。

##### 5.3 Worker 领取 Turn

Turn 可被领取必须同时满足：

- `status == COLLECTING`；
- `collect_until <= 当前时间`；
- Conversation 仍处于 `AI` 模式；
- 同一会话不存在 `RUNNING` Turn。

领取后写入：

- `status = RUNNING`；
- `snapshot_revision`；
- `locked_by`；
- `locked_until`；
- `attempts += 1`；
- `started_at`。

##### 5.4 行锁领取

当前查询使用 `FOR UPDATE` 锁定候选记录，

因此多个 Worker 同时领取时，后来的 Worker 可能等待第一个 Worker 持有的候选行。业务正确性仍由行锁和唯一索引保护，但横向扩容能力有限。

生产环境可以使用：

```sql
FOR UPDATE SKIP LOCKED
```

让其他 Worker 跳过已经锁定的任务，继续领取下一条可执行 Turn。

##### 5.5 Worker 租约机制

数据库行锁只存在于短事务中，不能覆盖整个模型调用。因此 Worker 领取后通过 `locked_by` 和 `locked_until` 保存逻辑租约。

租约让系统知道：

- 哪个 Worker 正在处理 Turn；
- 最晚应在什么时候完成；
- 超时后是否需要重新排队。

##### 5.6 固定输入快照

Worker 领取 Turn 时固定 `snapshot_revision`，然后查询：

```text
start_revision <= Message.input_revision <= snapshot_revision
```

得到本轮用户消息。早于本轮第一条消息的记录作为 history，按正序交给 AI Service。

##### 5.7 构建 AI Service 请求

- `conversation_id`
- `turn_id`
- `messages`
- `history`

`user_id` 不放在请求体中，而是由 Worker 从 Turn 单独取得并写入 JWT。`input_revision` 和重试次数仍由 Customer Service 自己管理，不再传给 AI Service。

##### 5.8 AI Service Gateway

`AIServiceGateway` 使用 httpx 调用三个内部接口：

- `start_run()`：启动 AgentRun；
- `commit_run()`：确认快照仍然有效后发布结果；
- `cancel_run()`：取消已经过期或未完成确认的 Run。

请求同时携带用户 JWT 和 `X-Internal-Service-Token`。

##### 5.9 AI 结果解析

`AIEventParser` 识别四类事件：

- `run_decision_prepared`
- `run_completed`
- `run_failed`
- `run_handoff_requested`

其中 `run_id` 必须返回，因为后续 `commit_run()`、`cancel_run()` 和 Turn 关联都需要它。

##### 5.10 AI 结果有效性校验

Customer Service 在两个节点校验：

1. 调用 `commit_run()` 前，锁定 Conversation 并比较快照版本；
2. 最终写入结果前，再次比较 Conversation 与 Turn 的版本。

第二次校验用于覆盖 commit 返回到最终持久化之间又有新消息到达的情况。

##### 5.11 两阶段结果确认

这里的两阶段不是数据库分布式事务：

1. AI Service 先生成并保存 `DECISION_PREPARED`；
2. Customer Service 校验输入快照；
3. 有效则调用 `commit_run()` 发布结果；
4. 失效则调用 `cancel_run()`。

`commit_run()` 不执行取消订单、修改地址等外部业务写操作。AI 返回的写操作只是页面引导，用户进入电商页面后再主动确认。

##### 5.12 Turn 重试与失败处理

AI 调用失败时：

- `attempts < 最大次数`：释放租约，Turn 回到 `COLLECTING`，延迟后重试；
- 达到最大次数：Turn 进入 `FAILED`，保存一条面向用户的失败消息，并把 `answered_revision` 推进到本轮快照。

默认最大尝试次数为 3，重试延迟为 2 秒。

##### 5.13 过期 Turn 恢复

AI Worker 每轮处理前查询 `RUNNING` 且 `locked_until` 已过期的 Turn：

- 会话版本已经变化：标记为 `SUPERSEDED`；
- 快照仍有效：重新放回 `COLLECTING`。

因此 Worker 进程意外退出后，任务不会永久停留在 `RUNNING`。

##### 5.14 为什么同一会话不能并行执行多个 Turn

如果同一会话同时执行两个 Turn，后发送的消息可能先得到回复，导致对话顺序混乱。

项目同时使用：

- 查询条件中的“不存在 RUNNING Turn”；
- `RUNNING` 状态的部分唯一索引；

共同限制同一会话只能有一个正在执行的 Turn。

---

#### 6. 人工客服转接

##### 6.1 为什么需要人工兜底

AI 不适合处理所有问题。信息不足、规则复杂、需要人工判断或用户明确要求人工时，系统创建 Handoff，把 Conversation 从 AI 轨道切换到人工轨道。

##### 6.2 AI 发起转人工

Worker 收到 `run_handoff_requested` 后，在同一事务中：

1. 创建或复用开放工单；
2. 将 Conversation 改为 `QUEUED`；
3. 保存 AI 的转人工提示；
4. 创建消息事件；
5. 创建工单状态事件。

##### 6.3 工单排队

`waiting` 工单会显示在客服工作台。`list_open_handoffs()` 返回 `waiting` 和 `active` 工单，并按创建时间排序。

##### 6.4 客服接入工单

客服调用 `/api/v1/handoffs/{id}/accept` 后：

- 工单改为 `active`；
- 写入 `assigned_agent_id` 和 `accepted_at`；
- Conversation 改为 `HUMAN`；
- 用户端和客服端都收到 `handoff_changed`。

##### 6.5 多客服并发抢单

接单前通过 `find_and_lock_with_conversation_by_id()` 对工单和会话执行 `FOR UPDATE`。第一个事务提交后，第二个事务读取到最新的 `active` 状态，只允许原负责人幂等重复接单，其他客服得到 403。

##### 6.6 客服回复消息

人工回复保存为 `role = human` 的 Message，并同时向：

- 用户频道；
- 客服公共频道；

写入 `message_created` Outbox。公共频道事件可以让其他打开同一会话的客服工作台同步刷新。

##### 6.7 结束人工服务

只有当前负责人可以结束工单。结束时：

- Handoff 改为 `resolved`；
- Conversation 切回 `AI`；
- 保存一条人工服务结束提示；
- 用户收到消息事件；
- 用户端和客服端收到工单状态事件。

##### 6.8 会话模式与工单状态的一致性

工单状态、会话模式、提示消息和 Outbox 事件在同一事务中提交。这样不会出现工单已经结束，但会话仍显示 `HUMAN` 的部分更新。

##### 6.9 工单操作为什么需要行锁

工单操作都是“先判断旧状态，再修改新状态”。如果没有行锁，两个客服可能同时看到 `waiting` 并都认为自己接单成功。行锁将这段读改写过程串行化。

---

#### 7. 实时消息推送

##### 7.1 HTTP 与 WebSocket 的职责

- HTTP：提交命令、查询数据库事实；
- WebSocket：推送连接期间的新事件。

消息必须先写入 PostgreSQL。WebSocket 不是消息事实来源。

##### 7.2 用户频道与客服频道

- 用户频道：`customer-service:user:{user_id}`
- 客服频道：`customer-service:staff`

customer 只能订阅自己的用户频道；agent 和 admin 订阅客服公共频道。

##### 7.3 MESSAGE_CREATED 消息事件

消息事件包含序号、消息 ID、会话 ID、角色、类型、内容和创建时间。

用户端收到后可以追加 AI 回复或人工回复；客服端收到后可以刷新当前会话。

##### 7.4 HANDOFF_CHANGED 工单事件

工单事件包含工单 ID、状态、摘要、负责人和会话模式。

用户端收到后可以更新“排队中、人工服务中”等状态；客服端收到后可以刷新工单列表和当前会话。

##### 7.5 Transactional Outbox

业务数据和待发送事件写入同一个 PostgreSQL 事务：

```mermaid
flowchart LR
    S[业务 Service] --> T[数据库事务]
    T --> M[Message / Handoff / Conversation]
    T --> O[RealtimeOutbox]
    T --> C{commit}
    C -->|成功| W[Outbox Worker]
    C -->|失败| R[全部回滚]
```

##### 7.6 Outbox Worker

Worker 按 `sequence` 查询最早的未发布事件：

1. 构建统一事件信封；
2. 发布到 Redis；
3. 成功时设置 `published_at`；
4. 失败时只增加 `attempts`，保留为待发布状态。

##### 7.7 Redis Pub/Sub

Redis 负责低延迟广播，不负责长期保存消息。数据库中的 Message 和 Handoff 才是可恢复的业务事实。

##### 7.8 WebSocket 连接与认证

连接建立后，客户端首先发送包含 token 的 JSON。服务端解码 JWT，根据角色选择唯一频道，随后启动后台任务转发 Redis 消息。

客户端与服务端通过接收循环维持连接；断开时取消转发任务并关闭 Pub/Sub。

##### 7.9 为什么不能在业务事务中直接发送 WebSocket

直接发送会产生无法原子化的两个系统：

- 先发送、后回滚：前端看到数据库中不存在的数据；
- 先提交、发送失败：数据库有数据但前端没有通知。

Outbox 把“必须发送”先保存为数据库事实，再异步重试发布。

---

#### 8. 管理后台与监控

##### 8.1 管理员权限控制

`GET /api/v1/admin/metrics` 只允许 `admin` 角色访问。人工工单接口当前只允许 `agent` 角色，管理员没有自动继承客服操作权限。

##### 8.2 会话统计

`conversations` 表示 Conversation 总数，包含已关闭会话。

##### 8.3 消息统计

`messages` 表示 Message 总数，包括用户、AI 和人工消息。

##### 8.4 人工工单统计

`handoffs` 表示历史工单总数；`queued` 和 `human` 分别统计当前排队与人工服务中的 Conversation。

##### 8.5 AI Run 监控

Customer Service 当前只保存 Turn 的 `run_id`、尝试次数和错误，没有提供 AI Run 指标接口。AI Run 的模型名称、Prompt 版本、Token、耗时和错误应由 AI Service 保存并提供监控接口。

##### 8.6 服务指标聚合

`AdminMetricsRepository.get_metrics()` 使用一个 SQL statement，通过多个标量子查询聚合核心指标，减少管理后台多次请求数据库。

当前指标是全量计数快照，还没有时间范围、waiting/active 工单分项、Turn 队列深度、Outbox 积压量和 Worker 健康状态。

##### 8.7 部分监控接口失败时的降级处理

管理前端同时请求 Customer Service、AI Service 和电商服务指标时，应允许各请求独立成功。使用 `Promise.allSettled()` 或为每个请求单独捕获异常，可以避免单个服务不可用导致整个监控区域空白。

---

#### 9. 认证与内部服务安全

##### 9.1 JWT 用户认证

外部接口从 Authorization Header 读取：

```text
Authorization: Bearer <token>
```

JWT 载荷至少包含 `user_id` 和 `role`。

##### 9.2 Customer、Agent 与 Admin 角色

- customer：发送消息、查询当前会话和历史消息；
- agent：查看、接入、回复和结束人工工单；
- admin：查看客服指标；
- WebSocket：customer 进入个人频道，agent/admin 进入客服频道。

##### 9.3 Customer Service 生成内部 JWT

AI Worker 根据 Turn 的 `user_id` 创建内部用户 JWT，使 AI Service 能够知道本次 Run 代表哪个用户。

##### 9.4 内部服务令牌

Gateway 同时发送：

```text
X-Internal-Service-Token: <internal token>
```

该令牌证明调用方是受信任的 Customer Service。

##### 9.5 AI Service 用户身份校验

AI Service 应同时校验内部服务令牌和用户 JWT，并将 AgentRun 绑定到 JWT 中的用户，而不是信任请求体自行声明的用户。

##### 9.6 AgentRun 所有者校验

`commit_run()` 和 `cancel_run()` 接收外部传入的 `run_id`，必须校验 Run 所属用户与 JWT 用户一致，防止用户操作其他人的 Run。

Customer Service 自身也存在资源级授权边界：当前会话详情接口只校验调用者是 `agent`，没有校验该客服是否为目标会话对应工单的负责人。若会话内容要求按负责人隔离，后续还需要补充归属校验。

##### 9.7 JWT 与内部服务令牌为什么要同时存在

- 内部令牌回答“哪个服务发起调用”；
- 用户 JWT 回答“本次调用代表哪个用户”。

两者保护的身份维度不同，不能互相替代。

当前 `AuthService.decode_access_token()` 没有把 PyJWT 解码异常统一转换为 401，WebSocket 认证也只捕获了部分输入异常。这是后续需要补强的错误边界。

---

#### 10. 事务、并发与一致性

##### 10.1 AsyncSession 事务边界

FastAPI 每个请求获得独立 AsyncSession。Session 依赖负责异常回滚，Service 负责在业务操作完整完成时显式提交。

Worker 也为领取、校验、最终结算分别创建短 Session，避免模型调用期间长期持有数据库事务。

##### 10.2 flush 与 commit 的区别

- `flush()`：把当前变更发送到数据库，使当前事务获得主键等数据，其他事务不可见；
- `commit()`：提交事务，使其他事务可见并释放事务锁。

创建消息后需要先 `flush()`，才能使用数据库生成的 Message 序号构建实时事件。

##### 10.3 为什么采用显式提交

显式提交可以让业务代码清楚表达原子边界，例如：

- 用户消息 + Turn；
- AI 消息 + Turn 状态 + Outbox；
- 工单状态 + 会话模式 + Outbox。

如果 Session 依赖在请求结束时自动提交，耗时流程中间的持久化时机不够直观，也不适合 Worker 的多阶段事务。

##### 10.4 行锁

当前版本主要使用 `FOR UPDATE` 或 `refresh(..., with_for_update=True)`：

- 锁定 Conversation，串行修改输入版本；
- 锁定 COLLECTING Turn，防止并发延长；
- 锁定 Handoff 和 Conversation，保护接单与结束；
- 最终结算时锁定 Conversation。

##### 10.5 部分唯一索引

项目使用 PostgreSQL 部分唯一索引约束：

- 每个用户最多一个开放 Conversation；
- 每个会话最多一个 COLLECTING Turn；
- 每个会话最多一个 RUNNING Turn；
- 每个用户或会话最多一个开放 Handoff。

业务判断提升可读性，数据库唯一约束提供并发下的最终防线。

##### 10.7 消息幂等与业务幂等

`message_id` 唯一约束防止消息重复插入；`agent_run_id + agent_outcome_seq` 唯一约束防止同一 AI 输出重复生成消息。

当前业务层在插入用户消息前主动查重，但并发请求仍必须依赖数据库唯一约束。若发生唯一冲突，还需要补充捕获异常并重查原记录的完整并发幂等流程。

##### 10.8 数据库事务与 Redis 的一致性

项目不尝试让 PostgreSQL 与 Redis 参与同一个分布式事务，而是通过 Outbox 实现最终一致：

1. PostgreSQL 原子保存业务和事件；
2. Worker 至少尝试发布；
3. 发布失败继续保留事件。

##### 10.9 常见并发场景分析

- 同一用户并发发消息：当前使用 Conversation 行锁、输入版本和唯一索引；首次创建会话时可增加 advisory lock。
- 多 Worker 领取 Turn：当前使用 `FOR UPDATE` 和 RUNNING 唯一索引；可增加 `SKIP LOCKED`。
- 用户在 AI 执行时继续输入：当前使用 `snapshot_revision` 二次校验；应保持最终校验与结果落库处于同一事务。
- 多客服抢同一工单：当前同时锁定 Handoff 和 Conversation；还可将不存在异常统一转换为 404。
- 多 Outbox Worker 发布：当前仅按顺序查询；应增加行锁和 `SKIP LOCKED`，防止重复发布。

---

#### 11. 异常处理与系统恢复

##### 11.1 AI Service 调用失败

Gateway 使用连接超时和总超时，并通过 `raise_for_status()` 把非成功响应转为异常。TurnProcessor 捕获异常后进入重试或失败结算。

##### 11.2 Worker 执行超时

当前恢复依据是数据库租约到期，不是主动中断正在运行的模型协程。AI Gateway 自身的 httpx timeout 负责限制 HTTP 调用时间，租约负责处理 Worker 中断后的遗留状态。

##### 11.3 Worker 意外退出

Turn 保持 `RUNNING`，但 `locked_until` 到期后会被恢复流程发现。快照有效则重排队，快照失效则淘汰。

##### 11.4 Outbox 发布失败

Redis 发布失败时：

- `attempts += 1`；
- `published_at` 保持为空；
- 当前事务提交尝试次数；
- Worker 下轮再次选择该事件。

当前没有最大尝试次数、退避、死信队列和告警。

##### 11.5 WebSocket 断开

断开后服务端取消转发任务并关闭 Pub/Sub。由于 Redis Pub/Sub 不重放离线消息，前端重新建立连接后应以历史消息接口中的数据库结果为准。

##### 11.6 重复请求

顺序重复请求可以由 `message_id` 查询直接返回。完全并发的重复请求仍可能同时通过预查询，最终由唯一约束阻止重复写入，但当前 Service 尚未将唯一冲突转换为幂等成功响应。

##### 11.7 输入快照过期

发现版本不一致时，Turn 进入 `SUPERSEDED`，释放租约，不保存旧 AI 回复。若 AI Service 已经生成 prepared 结果，Customer Service 调用 `cancel_run()`。

##### 11.8 数据库事务回滚

FastAPI Session 依赖捕获业务异常并执行 rollback。Service 不应吞掉数据库异常后继续提交，否则可能把不完整状态写入数据库。

---

#### 12. 设计取舍与优化思路

##### 12.1 为什么采用分层架构

分层让接口、业务和 SQL 可以独立变化，也让复杂流程集中在 Service 和 Worker 中，而不是散落在 Router。

##### 12.2 为什么业务逻辑不放在 Router

Router 只负责协议边界：

- 解析参数；
- 身份认证；
- 调用 Service；
- 返回响应。

这样 HTTP 接口、Worker 或未来消息队列消费者都能复用业务服务。

##### 12.3 为什么引入 Repository

Repository 隔离 SQLAlchemy 查询细节，使 Service 可以用业务语言表达“查找开放工单”“领取可执行 Turn”“查询历史消息”。

##### 12.4 为什么不在请求线程中直接调用 AI

模型调用耗时长且可能失败。异步 Worker 可以让消息接口快速返回，并提供合并、租约、重试和恢复能力。

##### 12.5 为什么需要独立 AI Worker

独立 Worker 可以单独扩容，也不会让 API 服务的并发连接被模型调用占满。数据库 Turn 充当了可恢复任务队列。

##### 12.6 为什么使用 Transactional Outbox

Outbox 用本地事务解决数据库与消息发布之间的双写问题，不要求 PostgreSQL 和 Redis 支持分布式事务。

---

#### 13. 面试高频问题

##### 13.1 你这个项目整体是怎么工作的？

用户消息先由 Customer Service 保存到 PostgreSQL，并加入一个可合并的 ConversationTurn。AI Worker 领取 Turn 后固定输入快照，再调用 AI Service。结果返回后，Customer Service 再次校验快照，保存 AI 回复或创建人工工单。业务变化同时写入 Outbox，由独立 Worker 经 Redis 和 WebSocket 推送给前端。

##### 13.2 为什么需要 ConversationTurn？

Turn 把消息接收和 AI 执行解耦，同时提供连续消息合并、顺序控制、租约、重试、快照和故障恢复能力。没有 Turn，每条消息都直接调用 AI，容易重复调用和乱序回复。

##### 13.3 为什么要合并用户连续消息？

用户经常把一个问题拆成多条消息。800ms 收集窗口可以把这些消息作为一次完整输入交给 AI，减少调用成本并提升语义完整性；最大等待时间防止无限延迟。

##### 13.4 如何避免 AI 回复顺序错乱？

同一会话只允许一个 `RUNNING` Turn。查询条件排除已有 RUNNING 的会话，数据库部分唯一索引再提供最终约束。

##### 13.5 input_revision 有什么作用？

它是会话输入版本。Worker 领取时保存快照，结果落库前比较当前版本。如果版本变化，说明 AI 基于旧输入生成结果，不能再写入当前会话。

##### 13.6 Worker 如何避免重复消费？

领取 Turn 时使用数据库行锁，并立即将状态改为 `RUNNING`。同一会话还存在 RUNNING 部分唯一索引。不过当前没有 `SKIP LOCKED`，多 Worker 扩展仍可优化。

##### 13.7 Worker 崩溃后如何恢复？

领取时写入 `locked_until`。恢复任务扫描租约过期的 RUNNING Turn，快照有效则重新进入 COLLECTING，快照失效则标记 SUPERSEDED。

##### 13.8 为什么使用两阶段结果确认？

模型执行期间用户可能继续输入。AI Service 先准备结果，Customer Service 校验输入版本后再 commit；旧结果则 cancel。这样由掌握会话事实的 Customer Service 决定结果是否仍可发布。

##### 13.9 为什么需要人工客服状态机？

waiting、active、resolved 可以明确表示等待、服务中和结束，配合 Conversation 的 QUEUED、HUMAN、AI 模式，避免用户消息被 AI 和人工同时处理。

##### 13.10 多客服同时抢单如何处理？

接单前使用 `FOR UPDATE` 锁定 Handoff 和 Conversation。第一个事务提交后，其他事务读取最新状态，只有负责人可以继续操作。

##### 13.11 为什么需要 Transactional Outbox？

因为数据库和 Redis 不能在普通本地事务中原子提交。Outbox 先把实时事件和业务数据一起提交，再由 Worker 异步发布，实现可重试的最终一致性。

##### 13.12 如何保证消息幂等？

业务层先按 `message_id` 查询，数据库再用唯一约束兜底。AI 消息还通过 `agent_run_id + agent_outcome_seq` 防止同一输出重复落库。完整并发幂等还应捕获唯一冲突并重查原记录。

##### 13.13 如何保证数据库与实时事件一致？

Message、Handoff、Conversation 和 RealtimeOutbox 在同一 PostgreSQL 事务中提交。事务失败时全部回滚，成功后 Outbox Worker 才发布事件。

##### 13.14 项目中最大的技术难点是什么？

最大的难点不是调用模型，而是异步执行期间的状态一致性：用户可能继续输入、Worker 可能崩溃、多个客服可能抢单、项目通过输入快照、行锁、部分唯一索引、租约和 Outbox 分别解决这些问题。

---

#### 附录

##### A. 核心业务流程图

```mermaid
flowchart TD
    U[用户发送消息] --> M[保存 Message]
    M --> MODE{Conversation.mode}
    MODE -->|AI| T[创建或更新 COLLECTING Turn]
    MODE -->|QUEUED/HUMAN| SO[创建客服端 Outbox]
    T --> W[AI Worker 领取]
    W --> AI[调用 AI Service]
    AI --> V{快照仍有效?}
    V -->|否| X[SUPERSEDED + cancel_run]
    V -->|是| R{结果类型}
    R -->|回复| AM[保存 AI Message]
    R -->|转人工| H[创建 Handoff + QUEUED]
    AM --> O[创建 Outbox]
    H --> O
    O --> REDIS[Outbox Worker 发布 Redis]
    REDIS --> WS[WebSocket 推送]
```

##### B. 数据模型关系图

```mermaid
erDiagram
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ CONVERSATION_TURN : schedules
    CONVERSATION ||--o{ HANDOFF : creates
    CONVERSATION ||--o{ REALTIME_OUTBOX : emits

    CONVERSATION {
        string id PK
        string user_id
        string mode
        int input_revision
        int answered_revision
    }
    MESSAGE {
        int id PK
        string message_id UK
        string conversation_id FK
        string role
        json content
        int input_revision
    }
    CONVERSATION_TURN {
        string id PK
        string conversation_id FK
        string status
        int start_revision
        int snapshot_revision
        string run_id
    }
    HANDOFF {
        string id PK
        string conversation_id FK
        string status
        string assigned_agent_id
    }
    REALTIME_OUTBOX {
        string id PK
        int sequence UK
        string channel
        string event_type
        datetime published_at
    }
```

##### C. 状态流转速记

```text
Conversation：
AI -> QUEUED -> HUMAN -> AI
AI -> CLOSED

Turn：
COLLECTING -> RUNNING -> COMPLETED
COLLECTING -> RUNNING -> COLLECTING（重试/租约恢复）
COLLECTING -> RUNNING -> FAILED
COLLECTING -> RUNNING -> SUPERSEDED

Handoff：
waiting -> active -> resolved
```

##### D. 30 秒项目介绍

这是一个基于 FastAPI、PostgreSQL 和 Redis 构建的电商智能客服中台。它负责管理用户会话和消息，通过 ConversationTurn 异步合并并调度 AI 请求，使用输入版本快照防止过期回复，通过人工工单实现 AI 到人工客服的切换，并使用 Transactional Outbox 保证业务数据和 WebSocket 实时事件的一致性。

##### E. 2 分钟项目介绍

项目的核心不是简单调用大模型，而是解决 AI 异步执行过程中的一致性问题。

用户消息先写入 PostgreSQL。如果会话处于 AI 模式，消息会进入 ConversationTurn。Turn 有短暂收集窗口，可以合并用户连续输入。AI Worker 领取 Turn 时固定输入版本并设置租约，然后调用独立 AI Service。

AI 返回结果后，Customer Service 会比较当前会话版本和 Turn 快照。如果执行期间用户又发送了新消息，旧结果会被淘汰；如果结果仍有效，则保存 AI 回复或创建人工工单。

转人工后，会话依次进入 QUEUED 和 HUMAN，工单使用行锁处理多客服并发抢单。所有消息、工单状态和实时事件都先在一个数据库事务中提交到业务表与 Outbox，再由独立 Worker 发布到 Redis，最后通过 WebSocket 推送给用户端和客服端。

##### F. 5 分钟项目介绍

1. 我做的是一个电商智能客服系统中的 Customer Service。它位于用户前端、人工客服工作台和 AI Service 之间，主要负责会话管理、消息持久化、AI 任务调度、人工转接以及实时事件推送。这个服务本身不负责大模型推理，也不会直接执行取消订单、修改地址等业务写操作。AI Service 负责生成回复、转人工建议或者受控的业务操作链接，Customer Service 则负责判断这些结果能不能安全地写回当前会话。

   用户发送消息时，请求首先经过 JWT 认证，只有 customer 角色才能访问聊天接口。系统会根据 `message_id` 查询消息是否已经存在，重复请求直接返回原来的会话信息，避免重复插入。之后系统会获取用户当前的开放会话；如果没有，就创建一个 AI 模式的新会话。为了避免同一个用户并发发送消息时覆盖会话版本，保存消息前会对 Conversation 执行行锁。消息、会话状态以及后续创建的 Turn 或 Outbox 事件，最后都在一个数据库事务中提交。

   消息接口没有直接同步调用 AI，因为模型调用耗时较长，而且用户经常会把一个完整问题拆成多条消息连续发送。为了解决这个问题，我设计了 ConversationTurn。用户在 AI 模式下发送第一条消息时，系统创建一个 `COLLECTING` 状态的 Turn，并设置一个短暂的消息收集窗口。窗口内继续收到消息时，只延长 `collect_until`，但不能超过最大等待时间。这样既能把连续输入合并成一次完整的 AI 请求，也不会让用户一直等待。

   独立的 AI Worker 会轮询已经结束收集的 Turn。一个 Turn 只有在状态为 `COLLECTING`、收集时间已经到期、会话仍处于 AI 模式，并且同一会话不存在其他 `RUNNING` Turn 时才能被领取。领取后，Worker 会把状态改为 `RUNNING`，记录 Worker 标识和租约截止时间，同时把 Conversation 当前的 `input_revision` 保存为 `snapshot_revision`。数据库中还通过部分唯一索引限制同一会话最多只有一个 `COLLECTING` Turn 和一个 `RUNNING` Turn，避免并发处理导致回复顺序混乱。

   版本机制是这个项目中比较核心的设计。每收到一条新的用户消息，Conversation 的 `input_revision` 就会增加；Worker 领取 Turn 时固定 `snapshot_revision`；结果成功结算后，再把 `answered_revision` 推进到这次快照。AI 调用期间不会一直持有数据库锁，因为那样会占用连接并阻塞用户的新消息。等 AI 返回结果后，Customer Service 再比较当前 `input_revision` 和 Turn 的 `snapshot_revision`。如果两者不一致，就说明 AI 执行期间用户又发送了新消息，旧结果已经过期，Turn 会被标记为 `SUPERSEDED`，不会写入会话。

   Customer Service 与 AI Service 之间采用两阶段结果确认。Worker 先调用 `start_run()`，如果 AI Service 返回 `run_decision_prepared`，Customer Service 会先锁定会话并校验输入快照。快照仍然有效时调用 `commit_run()` 发布结果，已经过期时调用 `cancel_run()`。这里的 commit 只是确认并发布 AI 已准备的结果，不是直接执行外部业务写操作。像取消订单这样的操作，AI 只返回可信页面链接，最终仍由用户进入业务页面核对并确认。

   AI 结果最终分为普通回复和转人工两条路径。普通回复会保存为 `role=ai` 的 Message。如果 AI 请求转人工，系统会创建或复用一个 `waiting` 状态的 Handoff，把 Conversation 从 `AI` 切换为 `QUEUED`，同时保存一条面向用户的转人工提示。客服接单时，工单从 `waiting` 变为 `active`，Conversation 变为 `HUMAN`；客服结束服务后，工单变为 `resolved`，Conversation 再切回 `AI`。接单、回复和结束操作都会锁定 Handoff 与 Conversation，并校验 `assigned_agent_id`，防止多个客服同时抢到同一工单或者非负责人操作工单。

   实时推送部分使用了 Transactional Outbox。业务 Service 不会在数据库事务中直接发送 Redis 或 WebSocket，而是把 Message、Handoff、Conversation 的变化和 RealtimeOutbox 事件一起提交到 PostgreSQL。独立的 Outbox Worker 按事件序号读取尚未发布的记录，发送到 Redis Pub/Sub，成功后更新 `published_at`，失败则保留记录等待下次重试。WebSocket 根据用户角色订阅不同频道：普通用户订阅自己的用户频道，客服和管理员订阅公共客服频道。这样可以降低数据库事务与实时推送之间双写不一致的风险。

   系统还提供了历史消息和管理指标接口。历史消息以数据库中的 Message 为准，可以通过 `after_sequence` 增量查询。管理端目前可以统计会话总数、消息总数、工单总数，以及当前处于 `QUEUED` 和 `HUMAN` 模式的会话数量。Customer Service 调用 AI Service 时还会同时携带内部服务令牌和内部用户 JWT，前者证明调用方是 Customer Service，后者说明本次调用代表哪个用户。

   在异常恢复方面，AI 调用失败后 Turn 可以按配置重新进入 `COLLECTING`，达到最大次数后进入 `FAILED` 并向用户保存失败提示。Worker 领取 Turn 时还会设置 `locked_until`，如果进程意外退出，租约过期后系统会重新检查该 Turn：快照仍有效就重新排队，已经过期就标记为 `SUPERSEDED`。因此任务不会因为某个 Worker 崩溃而永久停留在 `RUNNING`。

   当前版本仍有一些可以继续优化的地方。例如 Turn 领取使用了 `FOR UPDATE`，但还没有使用 `SKIP LOCKED`，多个 Worker 横向扩展时可能互相等待；Outbox 领取还缺少行锁，多 Worker 部署时可能重复发布；并发重复消息主要依赖数据库唯一约束兜底，还没有把唯一冲突完整转换为幂等成功；后续我会从 Worker 并发领取与更完整的监控指标几个方向继续完善。

##### G. 技术亮点速记

- Turn 异步解耦；
- 连续消息合并；
- 输入版本快照；
- 同会话单 RUNNING；
- Worker 租约恢复；
- AI 结果两阶段确认；
- 人工工单状态机；
- 行锁解决并发抢单；
- 部分唯一索引兜底；
- Transactional Outbox；
- 用户频道与客服频道隔离；
- 显式事务边界。

##### H. STAR 面试回答模板

**Situation：** 用户可能连续发消息，AI 调用耗时较长，执行期间还可能产生新输入；同时 Worker、Redis 和人工客服都存在并发或故障场景。

**Task：** 保证用户消息不丢失、AI 回复不乱序、旧结果不覆盖新输入，并支持人工转接和实时推送。

**Action：** 引入 ConversationTurn、输入版本快照、租约、数据库行锁、部分唯一索引和 Transactional Outbox；把 API、AI Worker 和 Outbox Worker 拆分运行。

**Result：** 消息接口可以快速返回，连续输入能够合并，过期 AI 结果会被淘汰，Worker 崩溃后可以恢复，工单并发操作保持一致，实时事件失败后可以重试发布。

##### I. 项目复习清单

- 能画出用户消息到 AI 回复的完整链路；
- 能解释 Conversation、Message、Turn、Handoff、Outbox 的职责；
- 能说明三个 revision 字段的关系；
- 能解释为什么模型调用期间不能持有数据库锁；
- 能说明租约和数据库行锁的区别；
- 能解释两阶段确认不是分布式事务；
- 能说明 AI 为什么只返回业务操作链接；
- 能解释工单抢单的并发控制；
- 能说明 Outbox 解决了什么、没有解决什么；
- 能指出当前代码至少三个真实不足及改进方案。

---

<a id="chapter-10"></a>

## 10 · day07_AI_Service项目框架搭建

> [查看本篇原文](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md) · [返回目录](#阅读目录)

### Day07 AI Service 项目框架搭建

#### 目录

- [1. 今日目标](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#1-今日目标)
- [2. 服务边界](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#2-服务边界)
- [3. 项目架构](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#3-项目架构)
- [4. 运行设计](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#4-运行设计)
- [5. 接口与认证](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#5-接口与认证)
- [6. 服务协议](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#6-服务协议)
- [7. 代码框架](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#7-代码框架)
- [8. 本日总结](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/2_resource/day07_AI_Service%E9%A1%B9%E7%9B%AE%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA.md#8-本日总结)

---

#### 1. 今日目标

##### 1.1 独立AI Service

Customer Service 已经负责用户会话、消息、Turn、人工工单和实时事件。如果继续把模型调用、Agent 执行和运行监控都放在 Customer Service 中，会让会话业务与 AI 能力紧密耦合。

因此项目把 AI 能力拆分为独立的 AI Service：

- Customer Service 管理会话事实，决定某个 AI 结果是否仍然有效；
- AI Service 管理 AgentRun，负责后续的模型执行和结果生成；
- 两个服务通过内部 HTTP 接口和统一事件协议协作。

拆分以后，AI 模型、Prompt 和 Agent 工具可以独立迭代，不需要修改 Customer Service 的会话主流程。

##### 1.2 实现边界

 先搭建可持续扩展的项目骨架，不急于实现真正的 AgentExecutor。

本日完成：

1. 创建 FastAPI 项目和分层目录；
2. 配置 PostgreSQL 异步连接；
3. 建立 AgentRun 模型和 Repository；
4. 建立 AgentRunCoordinator 生命周期框架；
5. 提供 `start`、`commit`、`cancel` 三个内部接口；
6. 校验内部服务令牌和用户 JWT；
7. 定义 Customer Service 可以解析的响应事件；
8. 为后续模型监控预留字段。

---

#### 2. 服务边界

##### 2.1 客服服务职责

Customer Service 是会话数据的权威来源，主要负责：

- 接收并保存用户消息；
- 管理 Conversation 和 ConversationTurn；
- 在 Worker 领取 Turn 时固定输入快照；
- 组织当前消息和历史消息；
- 调用 AI Service；
- 校验 AI 执行期间是否产生了新输入；
- 保存最终 AI 回复或人工工单。

##### 2.2 智能服务职责

AI Service 主要负责：

- 为一次 Turn 创建 AgentRun；
- 保存本次运行的输入上下文；
- 后续调用 AgentExecutor；
- 保存执行结果、状态和监控数据；
- 根据 AgentRun 状态生成统一响应事件；
- 管理已经准备结果的发布或取消。

AI Service 不保存 Customer Service 的完整 Conversation，也不直接决定旧输入是否仍然有效。

##### 2.3 调用流程

```mermaid
flowchart LR
    U[用户消息] --> CS[Customer Service]
    CS --> T[ConversationTurn]
    T --> W[AI Worker]
    W -->|start_run| AIS[AI Service]
    AIS --> R[AgentRun]
    R --> E{响应事件}
    E -->|run_completed| W
    E -->|run_handoff_requested| W
    E -->|run_decision_prepared| V[Customer Service 校验快照]
    V -->|有效 commit_run| AIS
    V -->|过期 cancel_run| AIS
    W --> DB[(Customer Service 数据库)]
```

这条链路中，Customer Service 负责“结果能不能写入会话”，AI Service 负责“本次 Run 执行到了什么状态”。

---

#### 3. 项目架构

##### 3.1 技术选型

- Python 3.12+
- FastAPI：内部 HTTP 接口
- Pydantic：请求和身份数据校验
- SQLAlchemy 2.0 Async：异步 ORM
- PostgreSQL：AgentRun 持久化
- Psycopg 3：PostgreSQL 异步驱动
- PyJWT：用户 JWT 校验
- Uvicorn：ASGI 服务启动

##### 3.2 分层设计

```mermaid
flowchart TD
    R[Router] --> D[Dependencies]
    D --> C[AgentRunCoordinator]
    C --> RP[AgentRunRepository]
    RP --> DB[(PostgreSQL)]
    C --> EB[Event Builder]
    R --> AS[AuthService]
    AS --> JWT[JWT 配置]
```

各层职责如下：

- Router：处理 HTTP 协议、Header 和参数；
- Dependencies：组装 Session、AuthService 和 Coordinator；
- Coordinator：协调 AgentRun 生命周期和事务；
- Repository：封装 AgentRun 数据访问；
- Event Builder：把持久化状态转换为响应事件；
- Models：定义 AgentRun 相关表；
- Infrastructure：管理数据库引擎和 Session；
- Common：管理配置、ID、时间和事件循环。

##### 3.3 目录结构

```text
ai-srevice/
├── atguigu/
│   ├── agent/
│   │   └── harness/
│   │       └── run/
│   │           ├── coordinator.py
│   │           └── events.py
│   ├── app/
│   │   ├── repositories/
│   │   │   └── run.py
│   │   ├── routers/
│   │   │   └── run.py
│   │   ├── schemas/
│   │   │   ├── auth.py
│   │   │   └── run.py
│   │   ├── services/
│   │   │   └── auth.py
│   │   ├── app.py
│   │   └── dependencies.py
│   ├── common/
│   │   ├── config.py
│   │   ├── event_loop.py
│   │   └── utils.py
│   ├── infrastructure/
│   │   └── db.py
│   ├── models/
│   │   └── models.py
│   └── main.py
├── tests/
├── .env
└── pyproject.toml
```

---

#### 4. 运行设计

##### 4.1 数据模型

AgentRun 表示 AI Service 对一个 ConversationTurn 的一次处理过程。

核心关联字段：

- `id`：AI Service 生成的 Run ID；
- `conversation_id`：所属会话；
- `turn_id`：对应 Customer Service 的 Turn；
- `user_id`：从用户 JWT 中得到的 Run 所有者。

执行字段：

- `state`：当前运行状态；
- `input_context`：本次固定输入；
- `result`：最终回复、转人工或失败结果；
- `started_at`、`finished_at`：运行时间。

监控字段：

- `model_name`：本次使用的模型；
- `prompt_version`：本次使用的 Prompt 版本；
- `input_tokens`：模型输入 Token 数；
- `output_tokens`：模型输出 Token 数；
- `latency_ms`：执行耗时；
- `error`：内部错误详情。

##### 4.2 状态流转

```mermaid
stateDiagram-v2
    [*] --> RUNNING: start_run
    RUNNING --> COMPLETED: 直接生成普通回复
    RUNNING --> HANDED_OFF: 直接生成转人工结果
    RUNNING --> DECISION_PREPARED: 结果等待确认
    RUNNING --> FAILED: 执行失败
    RUNNING --> SUPERSEDED: 执行被取消
    DECISION_PREPARED --> COMPLETED: commit 普通结果
    DECISION_PREPARED --> HANDED_OFF: commit 转人工结果
    DECISION_PREPARED --> SUPERSEDED: cancel_run
```

各状态含义：

- `RUNNING`：Agent 正在执行；
- `DECISION_PREPARED`：结果已经生成，等待 Customer Service 确认；
- `COMPLETED`：普通回复已经发布；
- `HANDED_OFF`：转人工结果已经发布；
- `FAILED`：执行失败；
- `SUPERSEDED`：结果被取消，不再允许发布。

##### 4.3 输入与输出

`input_context` 保存 AI Service 收到的请求快照，包括：

- `conversation_id`
- `turn_id`
- `messages`
- `history`

`result` 保存的是 AI Service 生成的结构化结果。它可能包含：

- 普通回复的 `message_id` 和 `content`；
- 转人工所需的 `summary` 和 `message`；
- 失败信息。

##### 4.4 监控字段

监控字段不会改变 AgentRun 的核心状态机，但可以支持后续管理后台：

- 按模型分析运行次数；
- 对比不同 Prompt 版本的效果；
- 统计 Token 使用量；
- 计算平均执行耗时；
- 查询失败 Run 的内部错误。

这些字段会在后续实现 `start_run()` 时写入。当前框架只完成字段定义。

---

#### 5. 接口与认证

##### 5.1 启动接口

接口：

```text
POST /internal/v1/agent/runs
```

后续完整流程：

1. 从 JWT 获取 `user_id`；
2. 创建 `RUNNING` AgentRun；
3. 保存输入快照、模型和 Prompt 信息；
4. 提交事务，使长时间执行期间可以查询该 Run；
5. 调用 AgentExecutor；
6. 重新查询并锁定 AgentRun；
7. 保存结果、终态和监控信息；
8. 构建 Customer Service 响应事件。

当前该接口返回 501，因为 AgentExecutor 尚未实现。

##### 5.2 提交接口

接口：

```text
POST /internal/v1/agent/runs/{run_id}/commit
```

后续完整流程：

1. 查询并锁定 AgentRun；
2. 校验 Run 属于 JWT 用户；
3. 校验状态为 `DECISION_PREPARED`；
4. 将准备结果推进为 `COMPLETED` 或 `HANDED_OFF`；
5. 提交 AgentRun 状态；
6. 构建最终响应事件。

`commit_run()` 只确认并发布 AI 结果，不直接执行取消订单、修改地址等外部业务写操作。真正的写操作需要用户进入对应业务页面确认。

##### 5.3 取消接口

接口：

```text
POST /internal/v1/agent/runs/{run_id}/cancel
```

后续完整流程：

1. 查询 AgentRun；
2. 不存在时返回 404；
3. 校验 Run 所有者；
4. 如果状态为 `DECISION_PREPARED`，改为 `SUPERSEDED`；
5. 保存结束时间并提交；

取消接口设计为幂等操作。Run 不处于 `DECISION_PREPARED` 时不会重复修改状态。

##### 5.4 双重认证

三个接口都要求两种凭证：

```text
X-Internal-Service-Token: teaching-internal-token
Authorization: Bearer <user-jwt>
```

- 内部服务令牌证明请求来自 Customer Service；
- 用户 JWT 表示本次调用对应的用户。

```mermaid
sequenceDiagram
    participant W as Customer Service AI Worker
    participant R as AI Service Router
    participant I as Internal Token Dependency
    participant A as AuthService
    participant C as AgentRunCoordinator

    W->>R: 内部接口请求
    R->>I: 校验 X-Internal-Service-Token
    I-->>R: 服务身份通过
    R->>A: 解析 Authorization JWT
    A-->>R: CurrentUser
    R->>C: request + current_user.user_id
    C-->>W: 响应事件或 204
```

---

#### 6. 服务协议

##### 6.1 请求结构

Customer Service 当前发送：

```json
{
  "conversation_id": "conv_001",
  "turn_id": "turn_001",
  "messages": [
    {
      "message_id": "msg_001",
      "type": "text",
      "content": {
        "text": "我的订单什么时候发货"
      }
    }
  ],
  "history": [
    {
      "message_id": "msg_history_001",
      "role": "ai",
      "type": "text",
      "content": {
        "text": "您好，请问需要什么帮助？"
      }
    }
  ]
}
```

请求体不再携带 `user_id`，因为用户身份已经包含在 JWT 中。它也不再携带 `input_revision` 和 `request_id`，会话版本与重试由 Customer Service 管理。

##### 6.2 响应事件

统一事件结构：

```json
{
  "event_type": "run_completed",
  "event_data": {
    "run_id": "run_001",
    "message_id": "msg_ai_001",
    "content": {
      "text": "您的订单预计今天发货。"
    }
  }
}
```

支持的事件类型：

- `run_completed`
- `run_handoff_requested`
- `run_decision_prepared`
- `run_failed`

##### 6.3 运行标识

`run_id` 必须包含在事件中，因为 Customer Service 会用它：

- 调用 `commit_run(run_id)`；
- 调用 `cancel_run(run_id)`；
- 写入 `ConversationTurn.run_id`；
- 关联最终 AI 消息和 AgentRun。

##### 6.4 两阶段确认

```mermaid
sequenceDiagram
    participant CS as Customer Service
    participant AI as AI Service

    CS->>AI: start_run(request)
    AI-->>CS: run_decision_prepared + run_id
    CS->>CS: 比较 Turn 快照与 Conversation 当前版本
    alt 输入仍然有效
        CS->>AI: commit_run(run_id)
        AI-->>CS: run_completed / run_handoff_requested
    else 输入已经过期
        CS->>AI: cancel_run(run_id)
        AI-->>CS: 204 No Content
    end
```

Customer Service 掌握最新的 Conversation，因此由它判断即可。



#### 7. 代码框架

##### 7.1 配置与数据库

`Settings` 集中管理：

- AI Service 数据库地址；
- API Host 和 Port；
- 内部服务令牌；
- JWT Secret 和算法；
- 模型名称；
- Prompt 版本。

`infrastructure/db.py` 创建异步 Engine 和 SessionFactory。Session 依赖只负责生命周期和异常回滚，业务提交由 Coordinator 显式执行。

##### 7.2 数据访问层

`AgentRunRepository` 当前提供：

- `add()`：把 AgentRun 加入当前事务；
- `find_by_id()`：按主键查询，找不到时返回 `None`。

Repository 只负责数据访问，不直接返回 HTTPException。404、403 和状态校验由 Coordinator 处理。

后续 commit 需要并发保护时，应增加带 `FOR UPDATE` 的查询方法。

##### 7.3 生命周期协调

Coordinator 位于 `atguigu/agent/run/coordinator.py`，而不是普通 `app/services`，因为它负责的是 AgentRun 的完整执行生命周期。

它的核心职责是：

- 创建 Run；
- 控制事务提交时机；
- 调用后续 AgentExecutor；
- 校验状态；
- 保存结果；
- 处理发布和取消；
- 调用 Event Builder 构建响应。

##### 7.4 事件构建

`events.py` 根据 AgentRun 当前状态选择事件类型，并把 `run.id` 放入 `event_data`。

```text
COMPLETED         -> run_completed
HANDED_OFF        -> run_handoff_requested
DECISION_PREPARED -> run_decision_prepared
FAILED            -> run_failed
```

`RUNNING` 还没有结果，不能构建最终响应事件。`SUPERSEDED` 由取消接口使用 204 表示，因此当前 Event Builder 也不为它生成事件。

##### 7.5 路由与依赖注入

`dependencies.py` 负责组装：

- `SessionDep`
- `AuthServiceDep`
- `AgentRunCoordinatorDep`
- `require_internal_service`

Router 只完成以下工作：

1. 接收参数；
2. 校验内部服务令牌；
3. 从 JWT 获取当前用户；
4. 调用 Coordinator；
5. 返回结果。

数据库操作、状态变化和事务提交都不放在 Router 中。

---

#### 8. 本日总结

##### 8.1 已完成能力

Day07 已经建立了 AI Service 的核心边界：

- Customer Service 与 AI Service 的调用协议已经对齐；
- AgentRun 模型和状态已经确定；
- 数据库、Repository、Coordinator、Event Builder 和 Router 已经分层；
- 内部服务身份和用户身份都得到校验；
- `cancel_run()` 已经可以取消待发布结果；

##### 8.2 后续扩展

下一阶段只需要沿着现有边界继续实现：

1. 在 `start_run()` 中创建并提交 RUNNING AgentRun；
2. 构建 Agent 运行上下文；
3. 调用 AgentExecutor；
4. 记录模型、Token、耗时和错误；
5. 保存普通回复、转人工或待确认结果；
6. 在 `commit_run()` 中发布 prepared 结果；
7. 增加 AI Run 管理指标接口。

后续重点将从“服务框架”转向“Agent 如何执行和生成可信结果”。

---

<a id="chapter-11"></a>

## 11 · day08_AI_SERVICE运行核心链路实现

> [查看本篇原文](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md) · [返回目录](#阅读目录)

### 第八天 智能运行核心链路实现

#### 目录

- [1. 今日目标](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#1-今日目标)
- [2. 整体流程](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#2-整体流程)
- [3. 模型准备](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#3-模型准备)
- [4. 消息编译](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#4-消息编译)
- [5. 执行与校验](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#5-执行与校验)
- [6. 结果与状态](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#6-结果与状态)
- [7. 启动运行](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#7-启动运行)
- [8. 测试与总结](originals/day08_AI%E6%9C%8D%E5%8A%A1%28%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%26%26%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86%29%E7%BC%96%E7%A0%81%E5%AE%9E%E7%8E%B0/2_resouce/day08_AI_SERVICE%E8%BF%90%E8%A1%8C%E6%A0%B8%E5%BF%83%E9%93%BE%E8%B7%AF%E5%AE%9E%E7%8E%B0.md#8-测试与总结)

---

#### 1. 今日目标

##### 1.1 内容

上一阶段已经完成智能服务的项目骨架、运行记录、内部接口、身份认证和响应事件。本阶段继续实现启动运行接口，使服务能够真正调用大语言模型并返回结构化结果。

本日完成以下内容：

1. 配置并创建对话模型；
2. 创建只读客服智能体；
3. 在应用启动时创建并复用智能体；
4. 定义模型的结构化输出；
5. 编译历史消息和本轮消息；
6. 实现智能体的三步执行流程；
7. 建立校验前后的输出边界；
8. 将校验结果映射为运行状态和持久化结果；
9. 完成启动运行接口的核心闭环。

##### 1.2 实现边界

本日重点是打通最小运行链路，不一次性实现全部生产能力。

本日暂不实现：

- 业务查询工具；
- 能力加载机制；
- 动态提示词；
- 运行时上下文；
- 页面动作校验；
- 澄清候选项构建；
- 待确认结果的提交与取消；
- 自定义纠错重试流程。

这些能力已经保留清晰的扩展位置，后续可以在不破坏当前主流程的前提下继续增加。

---

#### 2. 整体流程

##### 2.1 核心处理链路

用户消息首先由客服服务保存并形成一个固定输入快照。工作进程调用智能服务的启动接口后，智能服务创建运行记录、编译模型消息、调用智能体、校验结构化输出，最后保存结果并返回统一事件。

```mermaid
flowchart LR
    A[客服服务固定输入快照] --> B[调用启动运行接口]
    B --> C[创建运行记录]
    C --> D[编译模型消息]
    D --> E[调用只读客服智能体]
    E --> F[解析结构化输出]
    F --> G[执行服务端校验]
    G --> H[映射运行状态和结果]
    H --> I[保存最终运行记录]
    I --> J[返回统一响应事件]
```

这条链路明确区分了三个阶段：

- 模型调用前：编译可信、有限的输入消息；
- 模型调用后：解析并校验结构化输出；
- 数据保存前：把输出转换成运行状态和稳定的结果结构。

##### 2.2 模块职责划分

```mermaid
flowchart TD
    A[模型适配层] --> B[智能体工厂]
    B --> C[共享智能体]
    D[消息编译器] --> E[智能体执行器]
    C --> E
    E --> F[输出校验器]
    F --> G[运行结果映射器]
    G --> H[运行协调器]
    H --> I[(运行记录表)]
    H --> J[响应事件构建器]
```

各模块职责如下：

- 模型适配层：隔离模型创建参数和结构化结果格式；
- 智能体工厂：组装模型、工具、提示词和输出协议；
- 消息编译器：将历史消息和本轮消息转换为模型消息；
- 智能体执行器：协调编译、调用和校验三个步骤；
- 输出校验器：建立模型输出与可信输出之间的边界；
- 运行结果映射器：生成运行状态和持久化结果；
- 运行协调器：管理事务、执行过程、监控字段和响应事件。

---

#### 3. 模型准备

##### 3.1 模型配置与适配

模型配置集中放在统一配置类中：

```python
llm_api_key: str = ""
llm_base_url: str | None = None
llm_model: str = ""
prompt_version: str = "customer-support-v1"
```

模型适配层负责检查配置并创建模型：

```python
class ModelAdapter:
    @classmethod
    def create_model(cls):
        settings = get_settings()
        model_kwargs: dict[str, Any] = {
            "model": settings.llm_model,
            "base_url": settings.llm_base_url,
            "api_key": settings.llm_api_key,
            "timeout": settings.llm_timeout_seconds,
            "temperature": 0,
            "retry": 1
        }
        if settings.llm_provider == "deepseek":
            return ChatDeepSeek(
                **model_kwargs,
                extra_body={"thinking": {"type": settings.llm_thinking_mode}}
            )
        if settings.llm_provider == "qwen":
            return ChatOpenAI(
                **model_kwargs,
                use_responses_api=False,
                extra_body={
                    "enable_thinking": settings.llm_thinking_mode == "enabled"
                }

            )

        return ChatOpenAI(**model_kwargs, use_responses_api=False)

```

它还负责从智能体执行结果中提取 `structured_response`，并转换成 `AgentOutput`：

```python
@staticmethod
def normalize_output(value: Any) -> AgentOutput:
    structured_output = (
        value.get("structured_response")
        if isinstance(value, dict)
        else value
    )
    if isinstance(structured_output, AgentOutput):
        return structured_output
    return AgentOutput.model_validate(structured_output)
```

执行器不需要理解不同模型的创建参数，也不需要了解框架返回值的具体结构。

##### 3.2 智能体创建与复用

智能体工厂使用 `create_agent()` 组装只读客服智能体：

```python
def create_support_agent(
    settings: Settings | None = None
) -> Any:
    settings = settings or get_settings()
    model = ModelAdapter.create(settings)
    tools: list[BaseTool] = []
    return create_agent(
        model=model,
        tools=tools,
        system_prompt=SYSTEM_PROMPT,
        response_format=ToolStrategy(AgentOutput),
        name="ecommerce-customer-support"
    )
```

当前工具列表为空，后续接入业务查询工具时再进行扩展。

如果每次请求都重新创建模型和智能体，会重复组装执行结构。项目使用应用生命周期，在服务启动时完成一次创建：

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await create_tables()
    app.state.agent = create_support_agent()
    yield
    await db_engine.dispose()
```

后续请求通过依赖获取共享智能体：

```python
def get_agent_executor(request: Request) -> AgentExecutor:
    return AgentExecutor(agent=request.app.state.agent)
```

```mermaid
sequenceDiagram
    participant S as 应用启动
    participant F as 智能体工厂
    participant A as 应用状态
    participant D as 请求依赖
    participant E as 智能体执行器

    S->>F: 创建模型和智能体
    F-->>S: 返回共享智能体
    S->>A: 保存共享智能体
    D->>A: 读取共享智能体
    A-->>D: 返回同一实例
    D->>E: 创建本次请求的执行器
```

共享的是无会话状态的智能体实例。每次调用使用独立消息，因此不同用户的对话不会写入同一个内存会话。

##### 3.3 提示词与结构化输出

系统提示词首先明确只读边界：

```text
你是电商平台的只读智能客服。你的目标是使用可信数据回答问题，
并把所有业务变更引导到用户端的确定性页面。
```

当前提示词只包含两个章节：

1. 职责边界；
2. 终态输出。

后续实现能力加载、事实校验和页面动作后，再增加对应章节。

模型每次必须返回四种回复类型中的一种：

```python
class ReplyType(StrEnum):
    ANSWER = "ANSWER"
    CLARIFY = "CLARIFY"
    DECLINE = "DECLINE"
    REQUEST_HANDOFF = "REQUEST_HANDOFF"
```

四种类型分别表示：

- `ANSWER`：正常回答电商问题；
- `CLARIFY`：请求用户补充信息或选择业务对象；
- `DECLINE`：拒绝非电商问题或没有可靠依据的问题；
- `REQUEST_HANDOFF`：请求客服服务创建人工工单。

转人工信息与用户回复属于不同用途，因此使用独立结构：

```python
class HandoffReason(StrEnum):
    """转人工的业务原因。"""

    USER_REQUESTED = "USER_REQUESTED"
    COMPLAINT = "COMPLAINT"
    RISK_REVIEW = "RISK_REVIEW"

class HandoffRequest(BaseModel):
    """定义 Customer Service 创建人工工单需要的信息。"""
    reason_code: HandoffReason
    summary: str


class AgentOutput(BaseModel):
    """定义大语言模型必须返回的结构化结果。"""
    reply_type: ReplyType
    reply_content: str
    handoff_request: HandoffRequest | None = None
```

其中：

- `reply_content` 面向用户；
- `handoff_request.summary` 面向人工客服；
- `handoff_request.reason_code` 用于标记转人工原因。

只有 `REQUEST_HANDOFF` 可以携带 `handoff_request`。不能仅仅因为模型不知道答案就转人工；非电商问题应当拒答，信息不足应当澄清。

---

#### 4. 消息编译

##### 4.1 历史消息裁剪

客服服务发送的请求中，历史消息与本轮消息已经分开：

```python
class AgentRunRequest(BaseModel):
    conversation_id: str
    turn_id: str
    messages: list[CurrentMessage]
    history: list[HistoryMessage]
```

消息编译器先按照条数限制取得最近历史，再从最新消息向前计算字符预算：

```python
history_message_limit: int = 30
history_character_budget: int = 12000
```

历史裁剪采用“保留连续尾部”的方式。一旦继续加入更早消息会超过字符预算，就停止向前选择，避免历史上下文无限增长。

裁剪完成后还要保证第一条历史由用户发送。历史可能因为条数限制从一条智能回复开始，这种没有前置用户问题的孤立回复不应直接交给模型。

```mermaid
flowchart TD
    A[完整历史消息] --> B[截取最近指定条数]
    B --> C[从最新消息开始倒序累加字符数]
    C --> D{加入后是否超出预算}
    D -->|否| E[保留当前消息]
    E --> C
    D -->|是| F[停止继续向前]
    F --> G[恢复原始顺序]
    G --> H[移除开头的非用户消息]
    H --> I[得到有效历史]
```

##### 4.2 本轮消息聚合

一个对话轮次可能收集用户连续发送的多条消息。

单条消息直接返回原内容：

```text
查询我的订单
```

多条消息按照原顺序聚合：

```text
用户连续发送了以下内容，请作为一个整体理解，并以最新表达为准：
1. 查询我的订单
2. 是刚才购买的手机
```

这样既保留了连续输入，又明确要求模型以最新表达为准。

##### 4.3 消息格式转换

历史中的用户消息转换为 `HumanMessage`，智能回复和人工回复转换为 `AIMessage`。本轮消息全部来自当前用户，因此聚合后转换为一个 `HumanMessage`。

文本消息直接读取文本内容。结构化对象转换成稳定的文本：

```text
[用户选择的业务对象]
{"object_id":"order_001","object_type":"order"}
```

```mermaid
flowchart LR
    A[历史用户消息] --> B[用户消息对象]
    C[历史智能或人工回复] --> D[助手消息对象]
    E[本轮一条或多条消息] --> F[聚合用户消息对象]
    B --> G[最终模型消息列表]
    D --> G
    F --> G
```

当前阶段保留历史对象消息，让模型能够理解简单指代。等业务工具接入后，再建立确定性的对象焦点和运行时上下文，避免让模型自行猜测业务对象。

---

#### 5. 执行与校验

##### 5.1 三步执行流程

智能体执行器只负责三个步骤：

```python
async def execute(
    self,
    request: AgentRunRequest
) -> ValidatedAgentOutput:
    messages = self.context_compiler.compile_messages(request)

    raw_output = await self.agent.ainvoke(
        {"messages": messages}
    )
    output = ModelAdapter.normalize_output(raw_output)

    return self.output_validator.validate(output)
```

```mermaid
flowchart LR
    A[固定输入快照] --> B[编译模型消息]
    B --> C[调用共享智能体]
    C --> D[解析结构化输出]
    D --> E[执行服务端校验]
    E --> F[返回可信输出]
```

执行器不负责：

- 创建智能体；
- 数据库事务；
- 修改运行状态；
- 构建响应事件；
- 捕获并保存运行失败。

这些职责分别交给应用生命周期、运行协调器和事件构建器。

##### 5.2 校验前后模型

`AgentOutput` 是大语言模型生成的结构化对象。字段格式正确并不代表业务内容已经可信，因此不能直接把它等同于最终结果。

```text
模型结构化输出
        ↓
AgentOutput
        ↓ 服务端校验
ValidatedAgentOutput
```

校验后的模型继承基础输出，并为后续可信数据预留位置：

```python
class PageAction(BaseModel):
    """定义服务端校验后生成的安全页面动作。"""

    action_id: str
    label: str
    description: str
    href: str


class ClarificationOption(BaseModel):
    """定义服务端根据真实业务数据生成的澄清选项。"""

    object_type: str
    object_id: str
    label: str
    description: str
        
class ValidatedAgentOutput(AgentOutput):
     """保存经过服务端校验并补充可信数据的 Agent 输出。"""
    page_action: PageAction | None = None
    clarification_options: list[ClarificationOption] = Field(
        default_factory=list
    )
```

- `page_action` 由服务端动作目录生成，不接受模型直接生成页面地址；
- `clarification_options` 由真实业务查询结果生成，不允许模型编造订单或商品编号。

##### 5.3 后续校验边界

本日校验器只完成 `AgentOutput` 到 `ValidatedAgentOutput` 的类型转换：

```python
def validate(
    self,
    output: AgentOutput
) -> ValidatedAgentOutput:
    return ValidatedAgentOutput.model_validate(
        output.model_dump()
    )
```

后续将在这个边界逐步加入：

- 回答依据校验；
- 工具调用证据校验；
- 页面动作白名单校验；
- 页面资源编号校验；
- 澄清候选项构建；
- 转人工条件校验。

把这些规则集中到校验器，可以避免运行协调器和结果映射器承担模型安全职责。

---

#### 6. 结果与状态

##### 6.1 四种回复决策

四种回复决策并不等于四种运行状态。

当前对应关系如下：

```mermaid
flowchart TD
    A[可信输出] --> B{回复类型}
    B -->|正常回答| C[完成]
    B -->|请求澄清| C
    B -->|拒绝回答| C
    B -->|请求转人工| D[已转人工]
```

正常回答、澄清和拒答最终都会向用户发送一条智能消息，所以当前都映射为 `COMPLETED`。请求转人工需要客服服务创建人工工单，因此映射为 `HANDED_OFF`。

##### 6.2 结果映射

`AgentRunResultMapper` 将可信输出映射为两个值：

```python
tuple[AgentRunState, dict[str, Any]]
```

普通回复结果：

```json
{
  "message_id": "ai_msg_xxx",
  "content": {
    "text": "您的问题已经收到。",
    "reply_type": "ANSWER"
  }
}
```

转人工结果：

```json
{
  "reason_code": "USER_REQUESTED",
  "summary": "用户明确要求人工客服处理。",
  "message": "正在为您转接人工客服，请稍候。"
}
```

智能服务只返回转人工结果，不直接创建客服工单。客服服务收到 `run_handoff_requested` 后负责：

1. 创建或复用开放工单；
2. 将会话切换为等待人工；
3. 保存面向用户的转接提示；
4. 向用户端和客服工作台发送实时事件。

##### 6.3 运行状态变化

当前启动接口会产生以下状态：

```mermaid
stateDiagram-v2
    [*] --> RUNNING: 创建运行记录
    RUNNING --> COMPLETED: 回答、澄清或拒答
    RUNNING --> HANDED_OFF: 请求转人工
    RUNNING --> FAILED: 执行发生异常
```

状态职责并不全部属于结果映射器：

- `RUNNING`：运行协调器创建记录时设置；
- `COMPLETED`：结果映射器处理普通回复时设置；
- `HANDED_OFF`：结果映射器处理转人工时设置；
- `FAILED`：运行协调器捕获执行异常时设置；
- `DECISION_PREPARED`：后续实现需要确认的页面动作时由结果映射器产生；
- `SUPERSEDED`：取消待发布结果时由运行协调器设置。

---

#### 7. 启动运行

##### 7.1 创建运行记录

启动接口收到请求后，首先创建 `RUNNING` 状态的运行记录：

```python
run = AgentRun(
    conversation_id=request.conversation_id,
    user_id=user_id,
    turn_id=request.turn_id,
    state=AgentRunState.RUNNING,
    model_name=self.settings.llm_model,
    prompt_version=self.settings.prompt_version,
    input_context=request.model_dump(mode="json")
)
```

输入快照会保存当前消息和历史消息，使后续能够知道本次模型执行基于什么输入。

##### 7.2 调用智能体

运行记录创建后先提交一次事务，再调用智能体：

```python
self.run_repository.add(run)
await self.session.commit()

output = await self.executor.execute(request)
```

第一次提交用于：

- 让数据库能够查询正在运行的记录；
- 让管理后台观察运行中任务；
- 发生进程异常时保留运行痕迹；
- 避免模型调用期间长期占用数据库事务。

##### 7.3 保存结果与监控数据

智能体执行成功时，结果映射器生成状态和结果：

```python
run.state, run.result = AgentRunResultMapper.map(output)
```

执行发生异常时，由运行协调器统一记录失败：

```python
except Exception as exc:
    run.state = AgentRunState.FAILED
    run.error = str(exc) or type(exc).__name__
    run.result = {
        "message": "AI Service 处理失败"
    }
```

最后记录耗时和结束时间，并进行第二次提交：

```python
run.latency_ms = int((perf_counter() - started) * 1000)
run.finished_at = get_utcnow()
await self.session.commit()
```

两次提交保存的是不同生命周期状态，并不是业务上的两阶段确认：

```text
第一次提交：RUNNING
第二次提交：COMPLETED、HANDED_OFF 或 FAILED
```

当前已经记录模型名称、提示词版本、执行耗时和错误信息。输入、输出令牌字段暂时保留默认值，后续接入模型用量信息后再写入。

##### 7.4 构建响应事件

最终状态保存成功后，根据数据库中的运行状态构建客服服务能够解析的事件：

```text
COMPLETED  → run_completed
HANDED_OFF → run_handoff_requested
FAILED     → run_failed
```

```mermaid
sequenceDiagram
    participant W as 客服工作进程
    participant C as 运行协调器
    participant D as 数据库
    participant E as 智能体执行器
    participant M as 结果映射器

    W->>C: 请求启动运行
    C->>D: 保存运行中状态
    D-->>C: 第一次提交完成
    C->>E: 执行智能体
    E-->>C: 返回可信输出
    C->>M: 映射状态和结果
    M-->>C: 完成或转人工
    C->>D: 保存结果和监控数据
    D-->>C: 第二次提交完成
    C-->>W: 返回统一响应事件
```

页面动作尚未接入，因此本日不会产生 `DECISION_PREPARED`。提交和取消接口继续保留框架，等待后续页面动作课程完成。

##### 8.2 本日总结

本日完成了智能服务从框架到可执行链路的关键跨越：

1. 模型创建与框架返回格式被适配层隔离；
2. 智能体在应用启动时创建，并由所有请求复用；
3. 只读职责和四种终态回复通过提示词与结构化模型固定；
4. 历史消息受到条数与字符预算控制；
5. 执行器只保留编译、调用、校验三个核心步骤；
6. 模型输出和服务端可信输出已经分层；
7. 结果映射、运行状态、失败处理和事件构建职责清晰；
8. 启动运行接口已经形成完整闭环。

后续将在当前边界上继续增加业务查询工具、运行时上下文、动态提示词、事实校验、澄清候选项和安全页面动作。

---

<a id="chapter-12"></a>

## 12 · day09_业务查询工具与运行上下文

> [查看本篇原文](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md) · [返回目录](#阅读目录)

### day09 业务查询工具与运行上下文

#### 目录

- [1. 本日概览](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#1-本日概览)
- [2. 运行信息](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#2-运行信息)
- [3. 服务访问](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#3-服务访问)
- [4. 数据契约](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#4-数据契约)
- [5. 执行与记录](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#5-执行与记录)
- [6. 查询工具](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#6-查询工具)
- [7. 工具管理](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#7-工具管理)
- [8. 本日总结](originals/day09_AI%E6%9C%8D%E5%8A%A1%28%E5%B7%A5%E5%85%B7%E5%AE%9A%E4%B9%89%26%26%E6%89%A7%E8%A1%8C%29/2_resource/day09_%E4%B8%9A%E5%8A%A1%E6%9F%A5%E8%AF%A2%E5%B7%A5%E5%85%B7%E4%B8%8E%E8%BF%90%E8%A1%8C%E4%B8%8A%E4%B8%8B%E6%96%87.md#8-本日总结)

---

#### 1. 本日概览

<span style="color:red">作用</span>：**说明本日建设目标、整体链路和实现边界。**

##### 1.1 学习目标

<span style="color:red">作用</span>：**明确本日需要完成的功能以及这些功能解决的业务问题。**

上一阶段已经打通智能服务的核心运行链路。智能服务能够创建运行记录、编译消息、调用模型、解析结构化输出并返回统一事件。

但是模型只能根据消息内容回答，无法取得用户当前的订单、商品库存、物流和售后数据。模型训练数据也不能代替实时业务数据，否则容易产生过期或虚构的回答。

本日围绕业务查询完成以下内容：

2. 定义一次运行共享的可信运行信息；
3. 将用户访问令牌安全传递给业务工具；
3. 接入订单、商品、库存、物流和售后查询工具；
4. 使用统一执行器处理工具调用；
5. 建立工具目录并集中注册到智能体；

##### 1.2 总体链路

<span style="color:red">作用</span>：**用户提出问题到模型获得真实业务数据的完整处理顺序。**

模型根据用户问题决定是否调用工具。工具从运行信息中取得用户令牌，通过电商服务访问层查询真实数据。统一工具执行器在调用前后保存工具记录，并将经过校验的结果返回模型。

```mermaid
flowchart LR
    A[用户提出业务问题] --> B[智能体判断是否需要工具]
    B --> C[生成工具名称和参数]
    C --> D[读取运行信息]
    D --> E[携带用户令牌访问电商服务]
    E --> F[校验工具返回数据]
    F --> G[保存工具调用结果]
    G --> H[返回安全结果给模型]
    H --> I[模型生成结构化回复]
```

这条链路形成了三个边界：

- **身份边界**：工具只能使用当前用户的访问令牌查询数据；
- **数据边界**：电商服务返回的数据必须经过业务数据模型校验；
- **证据边界**：每一次工具调用都要保存，后续才能判断模型回答是否有真实依据。



#### 2. 运行信息

<span style="color:red">作用</span>：**说明业务工具执行时需要共享哪些可信信息，以及这些信息如何安全传递。**

##### 2.1 操作请求

<span style="color:red">作用</span>：**衔接页面操作中的资源编号与后续业务工具查询结果。**

上一阶段已经定义页面操作请求。本日先回顾其中的资源编号，明确它与业务工具查询结果之间的关系，为下一阶段的页面动作校验做准备。

模型不能直接生成前端地址。它只能表达“希望用户执行什么操作”以及“操作哪个业务对象”。

```python
class PageActionRequest(BaseModel):
    action_code: str
    resource_id: str | None = None
```

例如，用户询问如何取消订单时，模型可以返回：

```json
{
  "action_code": "CANCEL_ORDER",
  "resource_id": "ORDER_001"
}
```

这里没有页面地址。后续由服务端完成以下工作：

1. 判断动作编码是否在白名单中；
2. 判断订单是否经过业务工具确认；
3. 判断动作与资源类型是否匹配；
4. 使用服务端路径模板生成页面地址。

**当前 `resource_id` 实际表示订单编号。**查看订单详情、取消订单、修改地址、申请售后和查看售后进度都属于订单页面动作，后续会使用成功的订单查询记录验证这个编号。

商品编号当前只作为商品查询工具的参数使用，还没有对应的商品页面动作，因此不会写入 `PageActionRequest`。这里保留通用名称 `resource_id`，是为了以后扩展商品详情等其他资源页面动作。

##### 2.2 运行上下文

<span style="color:red">作用</span>：**定义一次智能运行中由协调器、执行器和业务工具共享的可信数据。**

业务工具不只需要模型传入的查询参数，还需要本次运行的可信信息。

```python
@dataclass(frozen=True)
class AgentRuntimeContext:
    run_id: str
    conversation_id: str
    user_id: str
    access_token: str
```

四个字段的职责如下：

- `run_id`：关联本次工具调用与运行记录；
- `conversation_id`：标记工具调用属于哪个会话；
- `user_id`：标记当前可信用户；
- `access_token`：以当前用户身份访问电商服务。

**运行信息由服务端创建，不由模型生成。**模型只能通过工具运行环境间接使用这些信息。

##### 2.3 令牌传递

<span style="color:red">作用</span>：**保证业务工具能够以当前用户身份访问私有业务数据，同时避免令牌进入模型消息。**

接口收到请求后，先从请求头中提取用户令牌，再经过运行协调器和执行器传入智能体。

```text
接口请求
→ 提取用户访问令牌
→ 运行协调器
→ 创建运行信息
→ 智能体执行器
→ 工具运行环境
→ 电商服务访问层
```

**令牌不能拼接到提示词或用户消息中。**它只保存在运行信息中，由工具代码读取。

##### 2.4 传递流程

<span style="color:red">作用</span>：**用调用顺序展示令牌和运行信息从接口进入业务工具的过程。**

```mermaid
sequenceDiagram
    participant C as 客服服务
    participant R as 智能服务接口
    participant O as 运行协调器
    participant E as 智能体执行器
    participant T as 业务工具
    participant S as 电商服务

    C->>R: 用户令牌和运行请求
    R->>R: 校验身份并提取令牌
    R->>O: 请求和用户令牌
    O->>O: 创建运行信息
    O->>E: 执行智能体
    E->>T: 注入运行信息
    T->>S: 携带用户令牌查询
    S-->>T: 返回当前用户数据
```

运行信息还通过 `context_schema` 注册到智能体：

```python
return create_agent(
    model=model,
    tools=TOOL_CATALOG.get_agent_tools(),
    system_prompt=SYSTEM_PROMPT,
    context_schema=AgentRuntimeContext,
    response_format=ToolStrategy(AgentOutput),
    name="ecommerce-customer-support"
)
```

这样每个工具都可以从 `ToolRuntime` 中取得同一份可信运行信息。

---

#### 3. 服务访问

<span style="color:red">作用</span>：**建立智能服务访问电商业务接口的统一边界。**

##### 3.1 访问职责

<span style="color:red">作用</span>：**明确电商服务客户端负责的请求工作以及不应承担的业务决策。**

`EcommerceClient` 负责访问电商服务，只处理以下职责：

- 拼接电商服务地址；
- 添加用户认证请求头；
- 添加查询参数；
- 设置请求超时时间；
- 检查响应状态；
- 返回响应中的数据。

```python
class EcommerceClient:
    def __init__(
        self,
        access_token: str,
        settings: Settings | None = None,
        http_transport: httpx.AsyncBaseTransport | None = None
    ):
        self.access_token = access_token
        self.settings = settings or get_settings()
        self.http_transport = http_transport
```

##### 3.2 请求链路

<span style="color:red">作用</span>：**业务工具通过访问层请求电商服务并处理响应过程。**

```mermaid
flowchart TD
    A[业务工具] --> B[创建电商服务客户端]
    B --> C[读取运行信息中的用户令牌]
    C --> D[构建认证请求头]
    D --> E[发送只读请求]
    E --> F{响应是否正常}
    F -->|是| G[返回响应数据]
    F -->|否| H[抛出请求异常]
    H --> I[统一执行器转换失败结果]
```

#### 4. 数据契约

<span style="color:red">作用</span>：**统一工具结果格式，并限定模型能够读取的业务数据字段。**

##### 4.1 模型分层

<span style="color:red">作用</span>：**解释统一工具返回模型与最小业务数据模型之间的职责分工。**

业务工具需要两类数据模型，它们解决的问题不同。

**第一类是统一工具返回模型** `ToolResult`。它负责描述一次工具调用是否成功、返回什么结果编码、向模型提供什么说明，以及是否携带业务数据。

如果每个工具分别设计成功和失败格式，模型、统一执行器以及后续校验器都需要编写不同的解析逻辑。使用 `ToolResult` 后，所有工具都遵守同一个外层结构：

```text
工具调用
→ success 表示是否成功
→ code 表示稳定结果编码
→ message 表示安全说明
→ data 保存具体业务数据
```

**第二类是最小业务数据模型。**不同工具的 `data` 内容不同，因此仍然需要订单、商品、库存、物流和售后模型分别描述真实业务字段。

例如：

```text
查询订单详情
→ ToolResult[OrderData]

查询商品库存
→ ToolResult[ProductStockData]

查询物流信息
→ ToolResult[LogisticsData]
```

两类模型组合后形成稳定结构：

```mermaid
flowchart LR
    A[统一工具返回模型] --> C[稳定的成功与失败格式]
    B[最小业务数据模型] --> D[不同工具的具体业务字段]
    C --> E[完整工具结果]
    D --> E
    E --> F[统一执行器和模型]
```

##### 4.2 统一结果

<span style="color:red">作用</span>：**定义所有业务工具共同使用的成功、失败和业务数据外层结构。**

所有业务工具使用同一个结果外壳：

```python
class ToolResult[ResultData](BaseModel):
    success: bool
    code: str
    message: str
    data: ResultData | None = None
```

字段职责如下：

- `success`：本次查询是否成功；
- `code`：稳定的业务结果编码；
- `message`：安全、可理解的结果说明；
- `data`：成功时返回的业务数据。

成功结果示例：

```json
{
  "success": true,
  "code": "OK",
  "message": "订单查询成功",
  "data": {
    "id": "ORDER_001",
    "status": "pending_shipment",
    "total_amount": "899.00"
  }
}
```

失败结果示例：

```json
{
  "success": false,
  "code": "TOOL_CALL_FAILED",
  "message": "暂时无法取得可靠的业务数据",
  "data": null
}
```

##### 4.3 业务模型

<span style="color:red">作用</span>：**定义订单、商品、库存、物流和售后工具需要的最小数据字段。**

商品基础信息与实时库存使用不同模型：

```python
class ProductData(BaseModel):
    id: str
    name: str
    category: str
    price: Decimal = Field(ge=0)
    status: str
    description: str
    specs: dict[str, Any]


class ProductStockData(BaseModel):
    product_id: str
    stock: int = Field(ge=0)
    status: str
```

订单模型由订单基础信息和商品项组成：

```python
class OrderItemData(BaseModel):
    product_id: str
    product_name: str
    quantity: int = Field(gt=0)
    unit_price: Decimal = Field(ge=0)


class OrderData(BaseModel):
    id: str
    status: str
    total_amount: Decimal = Field(ge=0)
    created_at: datetime
    items: list[OrderItemData]
```

物流模型由物流基础信息和轨迹事件组成：

```python
class LogisticsEventData(BaseModel):
    time: str
    description: str


class LogisticsData(BaseModel):
    order_id: str
    company: str
    tracking_number: str
    status: str
    latest_event: str
    events: list[LogisticsEventData]
```

售后模型保存客服回答处理进度需要的字段：

```python
class AfterSaleData(BaseModel):
    id: str
    order_id: str
    kind: str
    reason: str
    status: str
    created_at: datetime
```

工具为不同业务请求指定对应的数据模型：

```text
商品搜索        → list[ProductData]
商品详情        → ProductData
商品库存        → ProductStockData
订单列表        → list[OrderData]
订单详情        → OrderData
物流信息        → LogisticsData
售后记录        → list[AfterSaleData]
```

---

#### 5. 执行与记录

<span style="color:red">作用</span>：**统一业务工具的执行过程，并保存后续审计和可信校验需要的调用记录。**

##### 5.1 调用记录

<span style="color:red">作用</span>：**记录工具名称、调用参数、执行结果、成功状态和耗时。**

每次工具调用都保存到 `AgentToolCall`：

```python
class AgentToolCall(Base):
    id: Mapped[str]
    run_id: Mapped[str]
    tool_call_id: Mapped[str]
    tool_name: Mapped[str]
    arguments: Mapped[dict[str, Any]]
    result: Mapped[dict[str, Any] | None]
    success: Mapped[bool | None]
    latency_ms: Mapped[int | None]
    created_at: Mapped[datetime]
```

其中：

- `run_id`：关联本次智能运行；
- `tool_call_id`：关联模型产生的具体工具调用；
- `tool_name`：记录调用了哪个工具；
- `arguments`：记录模型传入并规范化后的参数；
- `result`：记录成功或失败的统一结果；
- `success`：便于统计成功率；
- `latency_ms`：记录工具调用耗时。

**工具记录有两个用途：**

1. 审计与监控：知道模型调用了什么、是否成功以及耗时多久；
2. 后续校验：判断模型回答中的订单号、金额、状态和页面动作是否有真实工具数据支持。

##### 5.2 统一执行

<span style="color:red">作用</span>：**把各业务工具重复的记录、调用、校验和保存流程集中到一个执行入口。**

所有业务工具都通过 `ToolExecutor.execute()` 执行：

```python
async def execute(
    self,
    runtime_context: AgentRuntimeContext,
    tool_call_id: str,
    tool_name: str,
    arguments: dict[str, Any],
    operation: ToolOperation,
    output_schema: Any
) -> str:
    ...
```

调用方只需要提供：

- 本次运行信息；
- 工具调用编号；
- 工具名称；
- 规范化后的参数；
- 真正访问业务服务的异步函数；
- 成功结果对应的数据模型。

工具执行器负责：

1. 在请求业务服务前创建工具调用记录；
2. 执行传入的业务请求函数；
3. 将请求异常转换成安全失败结果；
4. 校验统一结果结构；
5. 校验成功结果中的业务数据；
6. 保存结果、成功状态和耗时；
7. 序列化成模型可以读取的JSON字符串。

##### 5.3 结果处理

<span style="color:red">作用</span>：**校验工具结果结构和业务数据，并将可信结果保存到工具调用记录。**

统一执行器先验证外层结果，再验证成功结果中的 `data`：

```python
tool_result = ToolResult[Any].model_validate(response_data)
if not tool_result.success:
    return tool_result

data_adapter = TypeAdapter(output_schema)
validated_data = data_adapter.validate_python(tool_result.data)
return tool_result.model_copy(
    update={
        "data": data_adapter.dump_python(
            validated_data,
            mode="json"
        )
    }
)
```

`model_copy()` 不修改原对象，而是根据原结果创建一个新对象，并用经过业务模型校验和裁剪的数据替换原始 `data`。

统一执行器还会把两类异常转换成失败结果：

- 业务请求抛出异常时，返回 `TOOL_CALL_FAILED`；
- 返回数据不符合业务模型时，返回 `INVALID_TOOL_RESULT`。

成功结果、业务失败结果和异常转换后的失败结果都会保存到工具调用记录，原因包括：

1. 保留本次运行调用过哪些工具、传入什么参数以及最终是否成功；
2. 为管理指标提供工具调用次数、失败次数和执行耗时；
3. 为后续可信校验提供依据，失败结果不能作为业务事实和页面动作的证据。

工具结果还需要返回给模型，原因包括：

1. 让模型明确知道本次查询是否取得可靠数据，避免把失败调用误认为查询成功；
2. 让模型根据成功或失败结果组织最终回复；
3. 只向模型提供安全的结构化失败信息，不暴露原始异常和内部服务细节。

工具记录分两次保存：

```text
执行前：保存工具名称和调用参数
执行后：保存结果、成功状态和耗时
```

先保存调用记录可以保留调用痕迹。即使业务请求长时间阻塞或进程异常退出，也能知道本次运行曾经开始执行哪个工具。

##### 5.4 执行流程

<span style="color:red">作用</span>：**展示工具从创建调用记录到返回模型结果的完整生命周期。**

```mermaid
flowchart TD
    A[收到工具调用] --> B[保存工具名称和参数]
    B --> C[提交调用记录]
    C --> D[执行业务请求]
    D --> E{请求是否抛出异常}
    E -->|是| F[构建安全失败结果]
    E -->|否| G[校验统一结果结构]
    G --> H{业务查询是否成功}
    H -->|否| I[保留业务失败结果]
    H -->|是| J[按业务模型保留约定字段]
    J --> K[保存结果和耗时]
    I --> K
    F --> K
    K --> L[序列化后返回模型]
```

“按业务模型保留约定字段”是指使用当前工具指定的数据模型验证返回值，并只留下模型中已经定义的字段。

工具调用记录使用独立数据库会话，避免模型调用工具时与启动运行接口长期共享同一个数据库事务。

---

#### 6. 查询工具

<span style="color:red">作用</span>：**按照订单、商品、库存、物流和售后场景提供只读业务查询能力。**

##### 6.1 订单查询

<span style="color:red">作用</span>：**根据用户是否提供订单编号选择订单列表或订单详情查询。**

订单工具包括：

- `list_orders`：用户没有明确订单编号时查询订单列表；
- `get_order`：用户已经明确订单编号时查询订单详情。

工具参数通过注解描述给模型：

```python
order_id: Annotated[
    str,
    "用户明确提供或订单列表返回的订单编号，例如 ORDER_001"
]
```

订单编号进入业务请求前进行规范化：

```python
def _normalize_resource_id(resource_id: str) -> str:
    return resource_id.strip().upper().replace("-", "_")
```

规范化只处理空格、大小写和连接符，不负责猜测或创造订单编号。

订单详情工具的核心结构如下：

```python
@tool
async def get_order(
    order_id: Annotated[str, "用户明确提供或订单列表返回的订单编号"],
    runtime: ToolRuntime[AgentRuntimeContext]
) -> str:
    normalized_order_id = _normalize_resource_id(order_id)
    ecommerce_client = EcommerceClient(
        runtime.context.access_token
    )

    async def request_order() -> object:
        return await ecommerce_client.get_json(
            f"/orders/{quote(normalized_order_id, safe='')}"
        )

    return await tool_executor.execute(
        runtime_context=runtime.context,
        tool_call_id=runtime.tool_call_id,
        tool_name="get_order",
        arguments={"order_id": normalized_order_id},
        operation=request_order,
        output_schema=OrderData
    )
```

##### 6.2 商品查询

<span style="color:red">作用</span>：**根据用户需求查询商品信息，并在需要时单独取得实时库存。**

商品工具包括：

- `search_products`：根据名称、分类、规格或购买需求搜索商品；
- `get_product`：根据商品编号查询基础详情；
- `get_product_stock`：查询具体商品的实时库存和销售状态。

商品详情与库存拆成两个工具，是因为基础商品信息和实时库存具有不同更新频率。用户只询问商品介绍时，不需要额外查询库存；用户询问是否有货时，必须调用库存工具。

工具用途写在函数文档和参数注解中，不重复堆积到系统提示词：

```python
@tool
async def get_product_stock(...):
    """查询一个商品的实时库存和销售状态。

    用户询问具体商品是否有货或剩余库存时调用，不用于查询其他
    商品详情。
    """
```

##### 6.3 物流售后

<span style="color:red">作用</span>：**查询指定订单的配送进度以及当前用户的售后处理记录。**

物流与售后工具包括：

- `get_logistics`：查询物流公司、运单号、配送状态和轨迹；
- `list_after_sales`：查询当前用户的退款、退货或换货记录。

物流查询必须提供明确订单编号。售后查询的订单编号是可选参数：

```text
提供订单编号 → 只查询该订单的售后记录
没有订单编号 → 查询当前用户全部售后记录
```

**所有工具都是只读工具。**即使用户表达“取消订单”或“申请退款”，模型也不能把查询工具当作写操作工具，只能查询真实状态并返回页面操作意图。

##### 6.4 查询流程

<span style="color:red">作用</span>：**展示模型根据资源编号和用户意图选择业务工具的基本路径。**

```mermaid
flowchart TD
    A[用户业务问题] --> B{是否明确资源编号}
    B -->|没有订单编号| C[查询订单列表]
    B -->|已有订单编号| D{询问内容}
    D -->|订单状态或金额| E[查询订单详情]
    D -->|配送进度| F[查询物流]
    D -->|退款退货进度| G[查询售后]
    A --> H{是否为商品问题}
    H -->|描述购买需求| I[搜索商品]
    H -->|已有商品编号| J{询问内容}
    J -->|商品详情| K[查询商品详情]
    J -->|是否有货| L[查询实时库存]
```

---

#### 7. 工具管理

<span style="color:red">作用</span>：**集中管理工具、数据类别和智能体注册，避免多处维护工具列表。**

##### 7.1 工具分类

<span style="color:red">作用</span>：**标记工具返回业务数据还是知识数据，为后续可信校验提供依据。**

工具目录使用 `ToolCategory` 标记工具返回的数据类别：

```python
class ToolCategory(StrEnum):
    BUSINESS = "BUSINESS"
    KNOWLEDGE = "KNOWLEDGE"
```

当前所有工具都属于业务工具。知识类别为后续知识库检索预留。

工具分类不是为了决定模型是否能调用，而是为了让后续校验器知道某条工具记录属于业务事实还是平台知识。

##### 7.2 工具目录

<span style="color:red">作用</span>：**集中保存工具对象和所属类别，并提供统一查询入口。**

一个工具定义组合工具本身和数据类别：

```python
@dataclass(frozen=True)
class ToolDefinition:
    tool: BaseTool
    category: ToolCategory
```

`ToolCatalog` 集中提供：

- 获取全部智能体工具；
- 根据工具名称取得定义；
- 根据工具名称取得数据类别；
- 拒绝重复工具名称。

```python
TOOL_CATALOG = ToolCatalog(
    (
        ToolDefinition(search_products, ToolCategory.BUSINESS),
        ToolDefinition(get_product, ToolCategory.BUSINESS),
        ToolDefinition(get_product_stock, ToolCategory.BUSINESS),
        ToolDefinition(list_orders, ToolCategory.BUSINESS),
        ToolDefinition(get_order, ToolCategory.BUSINESS),
        ToolDefinition(get_logistics, ToolCategory.BUSINESS),
        ToolDefinition(list_after_sales, ToolCategory.BUSINESS),
    )
)
```

**工具只需要在目录中注册一次。**智能体创建和后续工具分类都使用同一份定义。

##### 7.3 工具注册

<span style="color:red">作用</span>：**将工具目录中的全部工具一次性注册到共享智能体。**

智能体工厂不再维护单独工具列表，而是直接读取工具目录：

```python
return create_agent(
    model=model,
    tools=TOOL_CATALOG.get_agent_tools(),
    system_prompt=SYSTEM_PROMPT,
    context_schema=AgentRuntimeContext,
    response_format=ToolStrategy(AgentOutput),
    name="ecommerce-customer-support"
)
```

```mermaid
flowchart LR
    A[订单工具] --> H[工具目录]
    B[商品工具] --> H
    C[库存工具] --> H
    D[物流工具] --> H
    E[售后工具] --> H
    H --> I[智能体工厂]
    I --> K[共享智能体]
```

智能体工厂负责注册工具。下一阶段的回答校验器也会使用同一目录判断哪些调用属于业务工具，避免两套分类规则产生差异。

---

#### 8. 本日总结

本日完成了智能服务从“只根据消息回答”到“能够查询真实业务数据”的关键扩展：

1. 页面操作请求中的资源编号与业务工具结果建立了清晰衔接；
2. 运行信息在不污染模型消息的前提下传递用户身份；
3. 电商服务访问层统一处理用户认证和只读请求；
4. 工具结果通过统一外壳和最小数据模型建立安全契约；
5. 每一次工具调用都保存参数、结果、状态和耗时；
6. 统一执行器消除了各业务工具中的重复流程；
7. 订单、商品、库存、物流和售后查询已经接入；
8. 工具调用异常会转换成模型能够理解的安全失败结果；
9. 工具目录成为智能体注册和后续校验的统一来源。

下一阶段将在这些工具调用记录之上建立可信输出校验：**提取回答中的业务事实，与成功工具结果比较，拒绝没有证据支持的回答**，并根据经过确认的业务对象生成安全页面动作。

---

<a id="chapter-13"></a>

## 13 · day10_生产级Agent_Harness校验与异常治理

> [查看本篇原文](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md) · [返回目录](#阅读目录)

### Day10 生产级 Agent Harness：校验与异常治理

本日目标：在 Agent 已经能够调用模型和业务工具的基础上，建立“结果有依据、动作可验证、错误可纠正、失败可追踪”的服务端治理链路。

Day08 打通了 Agent 的基础执行链路，Day09 让 Agent 能够查询真实业务数据。即使没有 Day10，Agent 仍然可以运行：

```text
用户提问 → 模型判断 → 调用工具 → 组织回答 → 返回结果
```

但是，“能够返回结果”不等于“结果可以直接用于生产”。模型可能引用工具中不存在的订单状态，可能为没有查询成功的订单生成取消入口，也可能在工具失败后继续编造业务结论。

Day10 在模型和最终业务结果之间增加 Agent Harness。Harness 不代替模型回答问题，而是负责验证 Agent 的待校验输出、限制危险行为、组织有限纠正，并把无法恢复的异常保存为稳定的运行结果。

```mermaid
flowchart LR
    A[模型生成待校验输出] --> B[Agent Harness]
    B --> C{服务端校验}
    C -->|通过| D[可信输出]
    C -->|可纠正| E[反馈模型重新生成]
    C -->|不可纠正| F[保存失败结果]
    E --> A
```

**目录**

- [一、校验](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#一校验)
  - [1. 事实校验](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#1-事实校验)
  - [2. 页面动作校验](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#2-页面动作校验)
- [二、异常](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#二异常)
  - [1. 异常边界](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#1-异常边界)
  - [2. Tool 执行层](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#2-tool-执行层)
  - [3. Output 校验层](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#3-output-校验层)
  - [4. Agent 执行层](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#4-agent-执行层)
  - [5. Run 协调层](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#5-run-协调层)
  - [6. 完整异常处理流程](originals/day10_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E7%94%9F%E6%80%81%EF%BC%89/2_resource/day10_%E7%94%9F%E4%BA%A7%E7%BA%A7Agent_Harness%E6%A0%A1%E9%AA%8C%E4%B8%8E%E5%BC%82%E5%B8%B8%E6%B2%BB%E7%90%86.md#6-完整异常处理流程)

---

#### 一、校验

<span style="color:red">目标</span>：服务端为什么不能直接信任 Agent 的待校验输出，并掌握回答事实与页面动作的两条可信校验链路。</span>

模型生成的 `AgentOutput` 属于**待校验输出**。即使它满足 JSON 结构，也只能说明字段格式正确，不能证明回答中的订单号、金额和状态来自真实业务数据，更不能证明某个页面动作可以安全返回给前端。

因此，Day10 把输出分成两个阶段：

1. 模型生成 `AgentOutput`，表达回复内容和页面动作意图；
2. 服务端读取当前 Run 的工具证据，生成 `ValidatedAgentOutput`。

```mermaid
flowchart TD
    A[模型结构化输出] --> B[读取当前 Run 的工具记录]
    B --> C[校验回答事实]
    C --> D[校验页面动作]
    D --> E[生成服务端可信输出]
    E --> F[映射运行状态和最终结果]
```

##### 1. 事实校验

<span style="color:red">目标</span>：保证回答中的**业务编号、数字和状态**都能在当前 Run 的**成功工具结果**中找到依据。

结构化输出只能解决“模型是否按照约定返回字段”的问题，**不能解决“字段内容是否真实”的问题**。

例如：

```text
模型回答：
订单 ORDER_001 当前已发货，总金额为 399 元。

真实工具结果：
订单 ORDER_001 当前待发货，总金额为 899.00 元。
```

虽然订单号正确，但“已发货”和“399”都没有工具证据，不能直接返回给用户。

```text
结构正确：reply_type、reply_content 等字段完整
语言通顺：回答符合自然语言表达
业务真实：订单号、金额、状态来自本次成功查询
```

生产环境真正需要的是第三层。大模型具有生成能力，可能根据上下文补全一个看似合理但并不存在的金额或状态。服务端必须把真实业务系统返回的数据作为最终依据。

采用**“允许模型组织语言，限制模型创造业务事实”**的原则：

- 普通解释性文本可以由模型生成；
- 订单号、商品号、售后编号需要工具支持；
- 金额、价格、库存和数量需要工具支持；
- 订单、物流和售后状态需要工具支持。

###### 1.1 校验依据

<span style="color:red">目标</span>：理解工具调用记录为什么既是**审计数据**，也是**回答校验的证据**。

Day09 已经在每次工具调用前后保存以下信息：

- 工具名称；
- 模型传入的参数；
- 工具返回的结果；
- 是否成功；
- 失败类型；
- 调用耗时。

Day10 按 `run_id` 读取这些记录，并转换成 `ToolCallSnapshot`，形成当前 Run 的工具证据集合。使用快照有两个目的：

1. **固定当前 Run 的证据范围** 
   只使用相同 `run_id` 下已经保存的工具调用，避免其他 Run 的数据被用于当前回答。

2. **形成统一的校验证据** 
   将工具名称、参数、结果、成功状态和失败类型组合成完整证据，同时供事实校验和页面动作校验使用。

```mermaid
flowchart LR
    A[数据库工具记录] --> B[按 Run 查询]
    B --> C[读取参数和结果]
    C --> D[只读工具快照]
    D --> E[回答校验]
    D --> F[页面动作校验]
```



###### 1.2 事实范围

<span style="color:red">目标</span>：**三类关键业务事实**及其**边界**。

当前实现提取**三类事实**：

| 类别 | 典型内容 | 示例 |
|---|---|---|
| 业务编号 | 订单、商品、售后编号 | `ORDER_001`、`PRODUCT_001`、`AS_001` |
| 业务数字 | 金额、价格、库存、数量 | `899 元`、`剩余 5 件` |
| 业务状态 | 订单、物流、售后状态 | `待发货`、`运输中`、`已退款` |

`FactCategory` 使用 `IDENTIFIER`、`NUMBER` 和 `STATUS` 表示这三类事实。

当前实现不会提取所有自然语言中的数字，数字只有带有金额、总价、价格、库存、数量等标签，或者带有元、件、个、台、套等单位时才参与校验。这种设计是有意控制范围：**先覆盖客服回答中风险最高、规则最清晰的事实，再逐步扩展**

###### 1.3 提取事实

<span style="color:red">目标</span>：理解如何把一段**自然语言回答**转换成可逐项比较的**业务事实**。

`FactExtractor` **按以下顺序提取事实**：

1. 使用模式匹配查找订单、商品和售后编号；
2. 统一编号大小写和连接符；
3. 提取带业务标签的数字；
4. 提取带货币或数量单位的数字；
5. 查找预先支持的中文业务状态；
6. 在保持顺序的同时去除重复事实。

例如：

```text
订单 ORDER-001 当前待发货，总金额为 899 元，
其中商品 PRODUCT_001 共 1 件。
```

会得到：

```text
业务编号：ORDER_001
业务编号：PRODUCT_001
业务数字：899
业务数字：1
业务状态：待发货
```

```mermaid
flowchart TD
    A[模型回复文本] --> B[提取业务编号]
    B --> C[提取金额和数量]
    C --> D[提取业务状态]
    D --> E[规范化表示]
    E --> F[按值去重]
    F --> G[待校验事实集合]
```

提取阶段只回答“回复中出现了哪些关键事实”，还不会判断事实是否正确。

###### 1.4 整理证据

<span style="color:red">目标</span>：把嵌套的**工具参数和结果**整理成可以统一比较的**证据集合**。

工具结果可能包含字典、列表和多层嵌套结构：

```json
{
  "id": "ORDER_001",
  "status": "pending_shipment",
  "total_amount": "899.00",
  "items": [
    {
      "product_id": "PRODUCT_001",
      "quantity": 1
    }
  ]
}
```

校验器递归读取成功工具调用中的两部分数据：

```text
工具参数 arguments
工具结果 result.data
```

**工具参数也属于证据。**例如 `get_order(order_id="ORDER_001")` 成功后，参数可以证明本次调用查询的是哪个订单；结果数据则证明订单的金额、状态和商品项。

读取到的标量值会进入两个集合：

- 文本证据集合：用于比较编号和状态；
- 数字证据集合：用于比较金额、库存和数量。

文本会统一 Unicode 宽窄字符、大小写以及 `-`、`_` 分隔符。数字使用 `Decimal` 比较，所以回答中的 `899` 可以与工具结果中的 `"899.00"` 匹配。

###### 1.5 匹配证据

<span style="color:red">目标</span>：理解不同事实为什么需要采用不同的**等价比较规则**。

**三类事实的匹配方式不同：**

```text
业务编号 → 规范化后进行文本匹配
业务数字 → 转成 Decimal 后进行数值匹配
业务状态 → 同时匹配中文标签和服务端状态编码
```

例如：

```text
ORDER-001       等价于 ORDER_001
899             等价于 899.00
待发货          等价于 pending_shipment
```

**状态需要建立中文标签和服务端编码之间的映射**，是因为模型通常使用“待发货”回答用户，而电商服务返回的可能是 `pending_shipment`。

`FactChecker.get_unsupported_facts()` 最终只返回无法匹配的事实。调用方不需要知道内部如何提取和归一化，只需要判断返回集合是否为空。

###### 1.6 识别无依据内容

<span style="color:red">目标</span>：在发现模型引用了工具中不存在的**业务事实**时，生成**可识别、可纠正的校验错误**。

如果所有事实都能匹配，原始回答继续进入页面动作校验。如果存在不受支持的事实，则抛出稳定错误：

```text
错误码：UNSUPPORTED_FACT
错误消息：回复中的业务事实缺少工具支持：399、已发货
```

**稳定错误码用于程序判断，具体错误消息用于日志和下一轮模型纠正。**不能只返回模糊的“回答错误”，否则模型不知道应该删除或修改哪些内容。

这里不会立即把整个 Run 标记为失败，因为“模型第一次回答引用了错误事实”属于可能通过重新生成修复的输出问题。

###### 1.7 生成安全回答

<span style="color:red">目标</span>：根据**业务失败、服务失败和契约失败**选择不同的**安全降级方式**。

事实校验前，回答校验器会先查看本次 Run 的业务工具调用结果。

**当业务工具全部失败时，分为两种情况：**

第一种是业务系统明确返回的业务失败，例如：

```text
订单尚未产生物流信息
没有找到该订单的售后记录
```

这类结果本身是可信业务结论。服务端使用最后一条业务失败消息生成 `ANSWER`，同时清除转人工请求和页面动作请求。

第二种是技术或数据契约失败，例如：

```text
电商服务连接失败
请求超时
返回字段不符合约定
```

这时无法确认真实业务状态。服务端生成固定的 `DECLINE`：

```text
暂时无法取得可靠的订单、商品、物流或售后信息，
因此现在不能确认结果，请稍后重试。
```

如果当前 Run 中同时存在成功和失败的业务调用，则**只允许成功调用支持回答事实**，不会让失败结果污染证据集合。

###### 1.8 完整流程

<span style="color:red">目标</span>：串联**工具快照、失败降级、事实提取和证据匹配**的完整顺序。

```mermaid
flowchart TD
    A[读取当前 Run 工具快照] --> B[筛选业务工具调用]
    B --> C[筛选成功业务调用]
    C --> D{存在业务调用且全部失败}
    D -->|是| E{是否全部为业务失败}
    E -->|是| F[使用可信业务失败消息]
    E -->|否| G[生成固定拒绝回复]
    D -->|否| H[提取回复中的关键事实]
    H --> I[收集成功工具参数和结果]
    I --> J[规范化文本和数字]
    J --> K{所有事实都有证据}
    K -->|是| L[回答校验通过]
    K -->|否| M[产生事实缺失错误]
    M --> N[进入有限纠正流程]
    F --> O[清除页面动作请求]
    G --> O
    O --> L
```

**这条链路保证：**

- 失败工具不能支持业务事实；
- 技术失败不会被解释成业务结论；
- 业务系统明确返回的“无数据”可以安全回答；
- 模型引用错误事实时有机会根据证据重新生成。

##### 2. 页面动作校验

<span style="color:red">目标</span>：保证返回给前端的**页面动作**来自**服务端白名单**，并且具体业务对象已经由成功工具调用确认。

页面动作会影响用户下一步操作，比普通文本具有更高风险。**模型可以表达“建议用户去取消订单”，但不能直接决定前端地址，也不能为未经查询的订单生成操作入口。**

如果允许模型直接返回 URL，可能出现：

- 跳转到不存在的页面；
- 拼接错误订单编号；
- 生成服务端未允许的操作；
- 将一个用户提供但未确认的编号直接写入操作地址；
- 在工具查询失败后仍返回取消或售后入口。

因此，模型只返回页面动作请求：

```json
{
  "action_code": "CANCEL_ORDER",
  "resource_id": "ORDER_001"
}
```

服务端校验通过后才生成前端页面动作：

```json
{
  "label": "前往取消订单",
  "description": "进入订单页面核对状态并确认取消",
  "href": "/me/orders/ORDER_001/cancel"
}
```

**模型负责表达意图，服务端负责授权动作、验证资源并构建地址。**

###### 2.1 区分请求与结果

<span style="color:red">目标</span>：区分模型可以提出的**最小意图**与前端可以执行的**可信结果**。

`PageActionRequest` 属于模型输出协议，只包含：

- `action_code`：模型希望引导用户执行的动作；
- `resource_id`：具体订单动作引用的订单编号。

`PageAction` 属于服务端可信输出，只包含：

- `label`：前端按钮文案；
- `description`：动作说明；
- `href`：服务端构建的页面地址。

```mermaid
flowchart LR
    A[模型] --> B[动作编码和资源编号]
    B --> C[服务端校验]
    C --> D[固定文案和路径模板]
    D --> E[前端可信页面动作]
```

**两层模型避免模型同时控制“做什么”和“跳到哪里”。**

###### 2.2 建立动作目录

<span style="color:red">目标</span>：把**允许的动作、展示文案和路径模板**集中在**服务端维护**。

服务端动作目录当前允许：

| 动作 | 是否需要订单编号 | 页面用途 |
|---|---:|---|
| 查看订单列表 | **否** | 进入个人订单列表 |
| 查看订单详情 | 是 | 查看订单状态和可用操作 |
| 取消订单 | 是 | 进入取消确认页 |
| 修改地址 | 是 | 进入地址修改页 |
| 申请售后 | 是 | 进入售后申请页 |
| 查看售后进度 | 是 | 查看订单售后记录 |

每个动作定义保存动作编码、固定文案、路径模板和资源类型。**模型不能在动作目录之外创造新的动作。**

集中目录还有两个好处：

1. 前端路径变化时只修改服务端定义；
2. 文案和路径不会被模型提示注入影响。

###### 2.3 定义动作要求

<span style="color:red">目标</span>：在待校验输出进入**业务校验**前，先拒绝明显不合法的**字段组合**。

页面动作请求首先经过结构规则：

```text
查看订单列表 → 不允许携带 resource_id
具体订单动作 → 必须携带 resource_id
未知动作编码 → 直接拒绝
额外字段       → 直接拒绝
```

这层规则回答的是“字段组合是否合法”，**还不能证明订单真实存在。**

例如：

```json
{
  "action_code": "CANCEL_ORDER",
  "resource_id": "ORDER_999"
}
```

结构上是合法的，但只有当前 Run 成功调用订单详情工具确认 `ORDER_999` 后，才能生成页面动作。

###### 2.4 查找动作证据

<span style="color:red">目标</span>：明确具体订单动作必须满足的**成功工具证据条件**。

订单列表页不关联具体业务对象，可以直接由服务端目录生成。

**具体订单动作必须同时满足三个条件：**

1. 当前 Run 调用过订单详情工具；
2. 该工具调用明确成功；
3. 工具参数中的订单编号与动作请求一致。

```text
页面动作：取消 ORDER_001

有效证据：
get_order(order_id="ORDER_001") 且 success=true

无效证据：
get_order(order_id="ORDER_002") 且 success=true
get_order(order_id="ORDER_001") 且 success=false
list_orders() 中曾经出现 ORDER_001
```

当前规则要求订单详情工具，是因为具体订单页面动作需要对目标订单做明确确认。**订单列表只能说明用户可能拥有该订单，不能证明详情查询已经成功完成。**

###### 2.5 校验资源编号

<span style="color:red">目标</span>：避免**大小写和连接符差异**造成误判，同时禁止服务端**猜测资源编号**。

比较前会对订单编号执行有限规范化：

```text
去除首尾空格
统一为大写
将连接符 - 转换为 _
```

所以 `order-001` 可以与 `ORDER_001` 匹配。

**规范化不是模糊搜索，也不会创造编号。**`ORDER_001` 与 `ORDER_002` 仍然是两个不同订单。

如果没有找到匹配的成功订单详情调用，则产生：

```text
错误码：UNVERIFIED_ACTION_RESOURCE
错误消息：页面动作引用的订单未经工具确认：ORDER_001
```

该错误允许进入模型纠正，让模型删除页面动作，或者基于已经确认的订单重新生成。

###### 2.6 生成可信动作

<span style="color:red">目标</span>：使用**服务端固定定义**生成**展示文案和安全页面地址**。

校验通过后，**服务端从动作目录取得定义**，并执行两步操作：

1. 对资源编号进行 URL 路径编码；
2. 将编码后的编号填入固定路径模板。

例如：

```text
动作编码：CANCEL_ORDER
资源编号：ORDER_001
路径模板：/me/orders/{resource_id}/cancel

最终地址：/me/orders/ORDER_001/cancel
```

**返回给前端的结果不再包含动作编码，而是可以直接展示的 `label`、`description` 和 `href`。**

###### 2.7 统一校验

<span style="color:red">目标</span>：先**校验回答**，再校验回答中保留下来的**页面动作请求**。

统一输出校验器**按照固定顺序执行**：

1. 根据 `run_id` 读取工具快照；
2. 校验工具失败降级和回答事实；
3. 从降级后的回答中读取页面动作请求；
4. 校验并生成可信页面动作；
5. 合并为 `ValidatedAgentOutput`。

###### 2.8 完整流程

<span style="color:red">本节目标</span>：串联**结构约束、服务端目录、工具证据**和最终页面动作生成过程。

```mermaid
flowchart TD
    A[模型生成页面动作请求] --> B{字段组合是否合法}
    B -->|否| C[结构化输出错误]
    B -->|是| D[读取服务端动作定义]
    D --> E{是否关联具体订单}
    E -->|否| J[使用固定定义生成动作]
    E -->|是| F[规范化订单编号]
    F --> G{存在匹配的成功订单详情调用}
    G -->|否| H[产生资源未确认错误]
    G -->|是| I[安全编码订单编号]
    I --> J
    J --> K[生成可信页面文案和地址]
    C --> L[进入有限纠正流程]
    H --> L
```

最终输出不再是“模型建议的页面动作”，而是**“经过服务端白名单和工具证据验证的页面动作”**。

---

#### 二、异常

<span style="color:red">本章目标</span>：从工具调用、输出校验、有限纠正到运行失败持久化的分层异常处理机制。

异常治理的目标不是把所有异常都捕获后返回同一句话，而是先判断异常发生在哪一层、是否可以恢复、应该告诉模型什么、应该告诉用户什么，以及最终需要保存什么。

##### 1. 异常边界

<span style="color:red">目标</span>：建立**四层异常边界**和**三组错误码**的整体视图，理解异常如何从底层向上层传播。

下面三种情况都表现为“没有得到正常答案”，但**处理方式完全不同**：

```text
订单没有物流信息       → 可信业务结果，可以直接回答
电商服务连接失败       → 内部服务故障，不能回答业务状态
模型把 899 回答成 399  → 输出错误，可以让模型重新生成
```

如果不分类：

- 业务上的“没有数据”可能被错误地当成系统故障；
- 服务故障可能被模型误认为业务结论；
- 可纠正的模型错误会直接导致整次 Run 最终失败；
- 真正的程序异常可能被反复重试并掩盖。

问题在最接近来源的位置被识别，再转换成上层能够理解的稳定语义。底层不越权决定 Run 终态，上层也不重新猜测底层技术细节。

###### 1.1 四层职责

<span style="color:red">目标</span>：掌握 **Tool 执行层、Output 校验层、Agent 执行层和 Run 协调层**各自负责的问题。

**Tool 执行层**把工具失败转换成安全 `ToolResult`；**Output 校验层**检查结构、事实和页面动作；**Agent 执行层**负责模型调用、有限纠正和调用上限；**Run 协调层**保存最终失败状态并返回统一事件。

```mermaid
flowchart LR
    A[Tool 执行层] -->|安全 ToolResult| C[Agent 执行层]
    C -->|待校验输出| B[Output 校验层]
    B -->|可信输出或校验错误| C
    C -->|可信结果或执行异常| D[Run 协调层]
    D --> E[保存终态并返回事件]
```

###### 1.2 三组错误码

<span style="color:red">目标</span>：区分 **CorrectableErrorCode、TerminalErrorCode 和 ToolFailureCode**，避免**三套协议混用**。

三组错误码不属于同一层，它们分别回答三个问题：

- **`CorrectableErrorCode`：待校验输出哪里不可信？** 
  包含 `MODEL_OUTPUT_INVALID`、`UNSUPPORTED_FACT`、`UNVERIFIED_ACTION_RESOURCE`。这类错误由 `AgentOutputValidationError` 携带，用**于判断是否让模型重新生成**。

- **`TerminalErrorCode`：当前 Run 为什么无法继续？** 
  包含 `MODEL_CALL_FAILED`、`OUTPUT_VALIDATION_FAILED`、`AGENT_EXECUTION_FAILED`。这类错误由 `AgentExecutionError` 携带，并交**给协调器保存失败状态**。

- **`ToolFailureCode`：工具执行层生成了哪种统一失败结果？** 
  包含 `TOOL_CALL_FAILED`、`INVALID_TOOL_RESULT`，保存在 `ToolResult.code` 中并返回模型。业务服务正常返回失败时不使用这两个代码，而是保留 Ecommerce Service 的业务码，例如 `LOGISTICS_NOT_AVAILABLE`。

`ToolFailureCode` 表示**具体失败代码**，`ToolFailureType` 则表示**失败性质**：

| 失败类型 | 含义 | 示例 |
|---|---|---|
| `BUSINESS` | 业务服务明确返回的正常失败 | 尚未产生物流信息 |
| `SERVICE_CALL` | 调用业务服务时发生技术异常 | 连接失败、超时 |
| `CONTRACT` | 返回数据不符合约定 | 字段缺失、类型错误 |

因此，三组错误码分别服务于**输出纠正、Run 终止和工具失败结果**，不能互相替代。

| 错误码类型 | 所属层 | 主要载体 | 核心作用 | 是否触发输出纠正 | 最终处理 |
|---|---|---|---|---|---|
| `CorrectableErrorCode` | Output 校验层 | `AgentOutputValidationError` | 描述待校验输出中的可纠正问题 | 是 | 在次数上限内让模型重新生成，次数耗尽后终止 Run |
| `TerminalErrorCode` | Agent 执行层 | `AgentExecutionError` | 描述当前 Run 无法继续的原因 | 否 | 交给协调器保存 `FAILED` 状态 |
| `ToolFailureCode` | Tool 执行层 | `ToolResult.code` | 描述工具执行层生成的统一失败结果 | 否 | 保存调用记录并返回模型，由模型生成安全回复 |

###### 1.3 异常传播方向

<span style="color:red">目标</span>：理解具体失败如何逐层转换成**稳定语义**，并始终沿**工具、校验、执行、协调**方向向上传播。

三类失败沿不同路径传播：

```mermaid
flowchart TD
    A{失败发生位置}
    A -->|工具执行失败| B[转换为 ToolResult]
    B --> C[保存调用记录并返回模型]
    C --> D[模型继续生成待校验输出]

    A -->|输出不可信| E[抛出 AgentOutputValidationError]
    E --> F{仍可纠正且未达上限}
    F -->|是| G[生成纠正反馈并再次调用模型]
    F -->|否| H[转换为 AgentExecutionError<br/>保留原始错误码]

    A -->|执行无法继续| I[抛出 AgentExecutionError]
    H --> J[协调器保存 FAILED]
    I --> J
```

工具失败是**返回给模型的结构化结果**，不会直接终止 Run；输出校验失败先尝试有限纠正；只有无法继续执行的错误，才由 `AgentExecutionError` 交给协调器保存为 `FAILED`。

整个传播过程遵守同一原则：**下层负责把原始失败转换成稳定语义，上层负责纠正、终止或持久化，任何一层都不越权处理。**

##### 2. Tool 执行层

<span style="color:red">目标</span>：把 AI Service 调用 Ecommerce Service 时的**三类失败**统一保存并安全返回模型，同时排除**不可信证据**。

###### 2.1 三种工具失败

<span style="color:red">目标</span>：使用相同模板区分 **SERVICE_CALL、BUSINESS 和 CONTRACT** 三种**工具失败**。

AI Service 的工具调用 Ecommerce Service 后，只有三种失败情况。

**第一种：调用过程直接抛出异常。**

触发条件：连接失败、请求超时或 HTTP 请求异常。

统一结果：

```json
{
  "success": false,
  "code": "TOOL_CALL_FAILED",
  "message": "暂时无法取得可靠的业务数据",
  "data": null,
  "failure_type": "SERVICE_CALL"
}
```

含义：业务服务调用失败，没有取得可用业务响应。原始异常只记录到服务端日志。

**第二种：调用正常完成，但业务结果为 `success=false`。**

触发条件：Ecommerce Service 明确返回业务失败，例如订单尚未产生物流信息。

统一结果：

```json
{
  "success": false,
  "code": "LOGISTICS_NOT_AVAILABLE",
  "message": "订单尚未产生物流信息",
  "data": null,
  "failure_type": "BUSINESS"
}
```

含义：调用过程正常，这是业务服务明确返回的可信业务结论。

**第三种：调用正常返回，但响应数据不符合契约。**

触发条件：外层结果不符合统一 `ToolResult`，或者 `success=true` 时的 `data` 不符合当前工具的业务模型。

统一结果：

```json
{
  "success": false,
  "code": "INVALID_TOOL_RESULT",
  "message": "业务工具返回的数据不符合约定",
  "data": null,
  "failure_type": "CONTRACT"
}
```

含义：业务服务已经返回响应，但字段缺失、类型错误或结构变化导致响应不可使用，原始无效数据不会保留在 `data` 中。

###### 2.2 统一失败结果

<span style="color:red">目标</span>：理解三类工具失败为什么都要转换成**结构一致、安全可见的 ToolResult**。

统一结果固定包含 `success=false`、稳定 `code`、安全 `message`、`data=null` 和 `failure_type`。其中服务调用失败使用 `TOOL_CALL_FAILED`，契约失败使用 `INVALID_TOOL_RESULT`，业务失败保留 Ecommerce Service 给出的业务码与安全消息。

字段职责如下：

- `success`：是否得到成功业务数据；
- `code`：稳定结果编码；
- `message`：可以安全提供给模型的说明；
- `data`：成功时的业务数据，失败时固定为 `null`；
- `failure_type`：失败原因分类。

**统一的是失败外壳，而不是抹平失败性质。**服务端仍根据 `BUSINESS`、`SERVICE_CALL`、`CONTRACT` 选择不同的证据和降级策略。

###### 2.3 保存并返回模型

<span style="color:red">目标</span>：清晰区分**模型能够看到的工具结果**与**服务端能够采信的业务证据**。

三种工具失败虽然原因不同，但后续都经过三个阶段。

**模型可见**表示安全 `ToolResult` 会进入完整消息轨迹，模型能够读取错误码、消息和失败类型。**服务端可信**表示结果能够支持回答事实或页面动作；所有 `success=false` 结果都不具备这种资格。

**第一阶段：保存记录。**

统一执行器保存本次工具的参数、失败结果、失败类型和耗时。保存记录是为了后续审计、监控和服务端校验。

**第二阶段：通知模型。**

统一执行器把安全 `ToolResult` 返回模型，让模型知道本次查询没有取得成功数据。返回内容不包含原始异常和无效业务数据。

模型收到失败结果后，可以组织回复，也可以决定再次调用工具，但不会由框架自动重试，并且仍然受工具调用次数限制。

**第三阶段：服务端决定最终结果。**

模型能看到工具结果，不代表服务端会信任该结果。回答校验器按照以下规则处理：

```text
没有调用业务工具
→ 不进入工具失败降级
→ 按普通输出继续校验

调用过业务工具，并且至少有一个 success=true
→ 只使用成功调用校验回答事实

调用过业务工具，并且所有调用都是 success=false
→ 继续判断失败类型

    全部属于 BUSINESS
    → 使用最后一条可信业务失败消息回答

    存在 SERVICE_CALL 或 CONTRACT
    → 生成固定拒绝回复，说明暂时无法确认业务结果
```

所有 `success=false` 的结果都不能支持回答事实和页面动作。

因此，这一流程可以归纳为：

```text
工具失败
→ 保存失败记录
→ 返回安全结果给模型
→ 服务端根据成功证据和失败类型决定最终回复
```

**模型负责理解失败信息并生成待校验回复，服务端负责决定哪些数据可信以及最终能否返回。**

###### 2.4 排除无效证据

<span style="color:red">目标</span>：根据是否调用业务工具、是否存在**成功调用**以及**失败类型**选择证据与降级策略。

没有业务工具调用时，不进入工具失败降级；至少存在一次成功调用时，只有成功结果能够进入证据集合；调用过但全部失败时，全为 `BUSINESS` 就使用最后一条业务消息，否则返回固定 `DECLINE`。

回答事实可以使用成功业务调用中的数据，页面动作还要进一步满足动作规则，例如具体订单动作必须存在订单编号一致的成功 `get_order` 调用。

失败记录可以保存、可以被模型看到，但不能污染证据集合。

###### 2.5 工具失败流程

<span style="color:red">目标</span>：用一条流程展示**三种工具失败**及其共同的**后续处理**。

```mermaid
flowchart TD
    A[调用 Ecommerce Service] --> B{调用是否抛出异常}
    B -->|是| C[生成服务调用失败结果]
    B -->|否| D{外层结果是否符合契约}
    D -->|否| E[生成契约失败结果]
    D -->|是| F{success 是否为 true}
    F -->|否| G[标记业务失败]
    F -->|是| H{data 是否符合业务模型}
    H -->|否| E
    H -->|是| Q[保留成功业务数据]
    C --> I[保存工具调用结果和耗时]
    E --> I
    G --> I
    Q --> I
    I --> J[返回结构化结果给模型]
    J --> K[模型生成待校验回复]
    K --> L{是否存在成功业务调用}
    L -->|存在| M[只使用成功调用校验回答]
    L -->|不存在| N{是否全部为业务失败}
    N -->|是| O[使用可信业务失败消息]
    N -->|否| P[生成固定拒绝回复]
    M --> R[进入页面动作校验]
    O --> R
    P --> R
```

##### 3. Output 校验层

<span style="color:red">目标</span>：把**结构化输出、事实和页面动作问题**转换成准确错误码，并隔离**校验器自身的未知失败**。

###### 3.1 处理输出格式错误

<span style="color:red">目标</span>：把结构化输出解析时的 **Pydantic ValidationError** 转换成 **MODEL_OUTPUT_INVALID**。

待校验输出缺少字段、类型错误、枚举非法或字段组合不满足 `AgentOutput` 协议时，Pydantic 会抛出 `ValidationError`。该异常准确转换为 `CorrectableErrorCode.MODEL_OUTPUT_INVALID`。

###### 3.2 处理事实校验错误

<span style="color:red">目标</span>：在待校验回答包含成功工具结果无法支持的**业务事实**时产生 **UNSUPPORTED_FACT**。

事实校验只使用 `success=true` 的工具调用形成证据。发现价格、订单状态或物流状态等内容没有成功证据支持时，产生 `CorrectableErrorCode.UNSUPPORTED_FACT`。

###### 3.3 处理页面动作错误

<span style="color:red">目标</span>：在页面动作引用未经成功工具结果确认的**资源**时产生 **UNVERIFIED_ACTION_RESOURCE**。

页面动作只能引用成功工具调用已经确认的业务资源。资源仅出现在用户文本、待校验输出或失败工具结果中时，产生 `CorrectableErrorCode.UNVERIFIED_ACTION_RESOURCE`。

###### 3.4 处理校验内部失败

<span style="color:red">目标</span>：把**已知规则错误**与**校验器自身故障**分开，避免未知异常被误判为可纠正问题。

三种 `CorrectableErrorCode` 保留原错误码向上传播；事实或动作校验过程中出现的其他未知异常统一转换为 `TerminalErrorCode.OUTPUT_VALIDATION_FAILED`，因为重新生成文本无法修复校验器故障。

已知 `AgentOutputValidationError` 原样抛给 Agent 执行层，由执行策略判断是否纠正。

###### 3.5 输出校验流程

<span style="color:red">目标</span>：用流程展示**结构、事实、动作校验**以及**校验器内部失败**的准确分流。

```mermaid
flowchart TD
    A[待校验输出] --> B{结构解析通过}
    B -->|ValidationError| C[MODEL_OUTPUT_INVALID]
    B -->|是| D[事实校验]
    D -->|无证据事实| E[UNSUPPORTED_FACT]
    D -->|通过| F[页面动作校验]
    F -->|资源未验证| G[UNVERIFIED_ACTION_RESOURCE]
    F -->|通过| H[可信输出]
    D -->|内部异常| I[OUTPUT_VALIDATION_FAILED]
    F -->|内部异常| I
```

##### 4. Agent 执行层

<span style="color:red">目标</span>：管理**模型调用、可纠正错误的有限重生成、纠正耗尽**和**工具调用总量边界**。

###### 4.1 处理模型调用失败

<span style="color:red">目标</span>：将模型服务或 Agent 调用中的**未知异常**统一转换成**稳定执行错误**。

Agent 调用边界会保留工具超限异常，让主循环生成固定安全结果。**其他模型调用异常统一转换为：**

```text
TerminalErrorCode.MODEL_CALL_FAILED
```

**它不进入输出纠正**，因为模型服务调用本身已经失败，再要求同一个执行流程重新组织答案没有可靠基础。

###### 4.2 判断是否允许纠正

<span style="color:red">目标</span>：只让 **CorrectableErrorCode** 且仍有**剩余次数**的输出问题进入纠正流程。

执行层只会把携带 `CorrectableErrorCode` 的 `AgentOutputValidationError` 交给纠正流程，并继续判断是否还有剩余纠正次数。

`TerminalErrorCode` 会直接终止当前 Run。`ToolFailureCode` 不属于输出校验异常，它会随安全 `ToolResult` 返回模型，由 Agent 继续正常推理，但不会自动触发服务端输出纠正。

###### 4.3 执行有限纠正

<span style="color:red">目标</span>：在保留**完整消息轨迹**的前提下追加**校验反馈**，并限制重新生成次数。

执行器根据错误原因构建明确反馈，说明上一轮未通过的具体问题，并要求模型只使用成功工具结果、删除无依据事实和未经确认的页面动作。反馈以系统消息追加，不伪造新的用户意图。

纠正时保留完整 `messages`，包括原始用户问题、模型工具调用、工具结果、上一轮生成过程和服务端反馈。这样模型可以直接复用已有成功证据，不必重新开始 Run。默认 `correction_limit=1`，即**首次生成加最多一次纠正**

###### 4.4 处理纠正次数耗尽

<span style="color:red">目标</span>：在最后一次待校验输出仍未通过时**终止纠正**，并保留最能说明**根因的错误码**。

纠正次数耗尽后，输出校验错误转换成 `AgentExecutionError`。其 `code` 可以继续保留原始 `CorrectableErrorCode`.

###### 4.5 处理工具调用超限

<span style="color:red">目标</span>：在**工具调用失控**时停止 Agent，并返回不依赖业务事实的**固定安全回复**。

模型可能因业务结果暂不可用、服务持续失败或在多个工具之间循环而反复调用。 Agent 因此限制一次 Run 中**所有工具的总调用次数**，默认上限为 8。

达到上限时抛出 `ToolCallLimitExceededError`。执行器直接生成固定、可信的 `DECLINE`，不再调用模型，也不进入输出校验。

工具调用超限描述的是整个 Run 的资源边界，**不属于 `ToolFailureCode`，也不会保存为 `FAILED`**。

###### 4.6 Agent 执行流程

<span style="color:red">目标</span>：用流程串联**模型调用、输出校验、有限纠正、纠正耗尽和工具调用超限**。

```mermaid
flowchart TD
    A[调用模型] --> B{执行结果}
    B -->|模型失败| C[MODEL_CALL_FAILED]
    B -->|工具超限| D[可信 DECLINE]
    B -->|待校验输出| E[输出校验]
    E -->|通过| F[可信输出]
    E -->|终止错误| G[AgentExecutionError]
    E -->|可纠正错误| H{允许且有次数}
    H -->|是| I[保留轨迹并追加反馈]
    I --> A
    H -->|否| G
    C --> G
```

##### 5. Run 协调层

<span style="color:red">目标</span>：在最外层区分**已知与未知异常**、保留**稳定错误码**、保存失败终态并返回统一事件。

###### 5.1 区分已知与未知异常

<span style="color:red">目标</span>：保证执行边界之外的**意外异常**不会让 Run 永久停留在**运行中状态**。

**运行协调器捕获最终异常后先判断：**

```text
AgentExecutionError → 使用异常自带的稳定错误码
其他 Exception      → 使用 TerminalErrorCode.AGENT_EXECUTION_FAILED
```

两类异常使用不同日志，便于区分已知执行失败和未知程序失败，但最终共用同一套失败状态保存逻辑。

###### 5.2 保留稳定错误码

<span style="color:red">目标</span>：确保错误经过**多层传播**后仍能表达最初的**可诊断原因**。

已知 `AgentExecutionError` 保留异常自带的 `code`，包括纠正耗尽后的 `CorrectableErrorCode` 和已有 `TerminalErrorCode`。只有未知 `Exception` 才统一使用 `AGENT_EXECUTION_FAILED`。

###### 5.3 保存运行失败状态

<span style="color:red">目标</span>：无论失败发生在哪一层，都为当前 Run 保存**统一终态、错误信息、结果和耗时**。

**运行失败时统一保存：**

```text
state       = FAILED
error       = 原始错误消息
result.code = 稳定错误码
result.message = AI Service 处理失败
latency_ms  = 本次执行耗时
finished_at = 完成时间
```

`run.error` 面向内部排查，`result.message` 面向服务调用方。对外消息不直接暴露原始异常。

**最终状态和监控字段在同一次提交中保存**，调用方收到的是统一响应事件，而不是未处理的内部异常。

###### 5.4 返回统一失败事件

<span style="color:red">目标</span>：让调用方收到**结构一致、安全**且与**落库错误码一致**的失败事件。

协调器保存失败状态后返回统一事件。调用方无需识别内部异常类，也不会看到调用栈、数据库异常或无效工具响应。

##### 6. 完整异常处理流程

<span style="color:red">目标</span>：把**四层边界**组合成一条从工具调用到最终事件的**完整异常治理链路**。

###### 6.1 四层协作流程

<span style="color:red">目标</span>：从**工具失败、待校验输出错误**到最终 Run 状态，建立完整的**异常传播视图**。

```mermaid
flowchart TD
    A[开始执行 Run] --> B[调用 Agent]
    B --> C{执行结果}
    C -->|工具未成功| D[分类并保存安全工具结果]
    D --> E[返回工具结果给模型]
    E --> B
    C -->|产生待校验输出| F[解析结构化输出]
    F --> G{结构是否有效}
    G -->|否| H[可纠正的格式错误]
    G -->|是| I[事实和动作校验]
    I --> J{校验是否通过}
    J -->|是| K[映射成功或转人工结果]
    J -->|否| L[可纠正的输出错误]
    H --> M{还有纠正次数}
    L --> M
    M -->|是| N[追加反馈重新生成]
    N --> B
    M -->|否| O[保留原错误码并终止]
    C -->|模型调用失败| P[终止执行错误码]
    C -->|工具调用超限| Q[生成固定安全拒绝]
    O --> R[协调器保存 FAILED]
    P --> R
    C -->|未知异常| S[统一未知错误码]
    S --> R
    K --> T[保存最终状态和耗时]
    Q --> T
    R --> T
    T --> U[返回统一响应事件]
```

###### 6.2 最终处理结果

<span style="color:red">目标</span>：归纳**四层异常治理**最终形成的**稳定处理结果**。

最终结果只有三类：校验通过后返回可信业务结果；工具调用超限后返回可信 `DECLINE` 且不保存 `FAILED`；执行无法恢复时保存统一 `FAILED` 终态并返回失败事件。

完成 Day10 后，**AI Service 的核心链路从“模型能够回答”升级为：**

```text
模型能够回答
→ 工具数据能够追踪
→ 关键事实能够验证
→ 页面动作能够约束
→ 输出错误能够有限纠正
→ 工具失败能够安全降级
→ 未知异常能够保存终态
```

这就是 Agent Harness 在当前项目中的核心价值：**模型负责生成，业务系统提供事实，服务端规则建立信任，运行协调器保证结果最终可控。**

---

<a id="chapter-14"></a>

## 14 · day11_生产级Harness全链路打通

> [查看本篇原文](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md) · [返回目录](#阅读目录)

### Day11 生产级 Harness 全链路打通

本日目标：把全局提示词、领域 Skill、工具说明、输出协议、中间件、Run 生命周期和运行观测组合成一条完整的 Agent Harness 链路。

Day10 建立了事实校验、页面动作校验和异常治理，但 Agent 仍然一次看到全部业务工具。随着商品、订单、物流、售后和平台规则不断增加，工具选择会越来越困难，领域提示也会相互干扰。

Day11 采用**单 Agent + 动态 Skill + 中间件**完成能力治理：

```text
用户问题
→ 识别领域
→ 加载 Skill
→ 注入领域提示
→ 限制可见工具
→ 查询业务数据
→ 生成结构化输出
→ 服务端校验
→ 保存 Run
→ 管理端观测
```

<span style="color:black"><strong>Prompt 负责说明规则，Skill 负责组织领域能力，中间件负责限制执行范围，Validator 负责建立最终信任。</strong></span>

**目录**

- [一、认识 Harness 全链路](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#一认识-harness-全链路)
  - [1. 本章学习目标](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#1-本章学习目标)
  - [2. 为什么需要完整 Harness](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#2-为什么需要完整-harness)
  - [3. Harness 整体架构](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#3-harness-整体架构)
  - [4. 一次请求的完整链路](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#4-一次请求的完整链路)
  - [5. 核心组件与代码结构](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#5-核心组件与代码结构)
  - [6. 本章实现路线](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#6-本章实现路线)
- [二、建立提示与协议体系](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#二建立提示与协议体系)
  - [1. 提示与协议整体设计](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#1-提示与协议整体设计)
  - [2. 四层约束的协作流程](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#2-四层约束的协作流程)
  - [3. Prompt 定义全局行为](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#3-prompt-定义全局行为)
  - [4. SkillCatalog 定义领域流程](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#4-skillcatalog-定义领域流程)
  - [5. 工具描述定义调用条件](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#5-工具描述定义调用条件)
  - [6. 数据模型定义输出协议](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#6-数据模型定义输出协议)
  - [7. 提示与协议完整链路](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#7-提示与协议完整链路)
- [三、实现动态 Skill 路由](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#三实现动态-skill-路由)
  - [1. 动态 Skill 整体设计](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#1-动态-skill-整体设计)
  - [2. Skill 路由执行流程](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#2-skill-路由执行流程)
  - [3. 建立 Skill 目录](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#3-建立-skill-目录)
  - [4. 使用 load_skill 加载领域](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#4-使用-load_skill-加载领域)
  - [5. 保存当前 Skill 状态](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#5-保存当前-skill-状态)
  - [6. 使用 Agent 一级中间件](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#6-使用-agent-一级中间件)
  - [7. 根据 Skill 过滤工具](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#7-根据-skill-过滤工具)
  - [8. 动态生成系统提示词](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#8-动态生成系统提示词)
  - [9. 在一次 Run 中切换 Skill](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#9-在一次-run-中切换-skill)
  - [10. 组装动态 Skill Agent](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#10-组装动态-skill-agent)
- [四、完成 Run 闭环与观测](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#四完成-run-闭环与观测)
  - [1. Run 闭环整体设计](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#1-run-闭环整体设计)
  - [2. Run 完整执行流程](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#2-run-完整执行流程)
  - [3. 执行并校验 Agent 输出](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#3-执行并校验-agent-输出)
  - [4. 限制工具调用次数](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#4-限制工具调用次数)
  - [5. 生成页面动作决策](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#5-生成页面动作决策)
  - [6. 确认与取消页面动作](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#6-确认与取消页面动作)
  - [7. 统计并保存 Token 用量](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#7-统计并保存-token-用量)
  - [8. 提供管理员观测接口](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#8-提供管理员观测接口)
  - [9. 前端展示运行详情](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#9-前端展示运行详情)
  - [10. 测试单 Skill 与跨 Skill 查询](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#10-测试单-skill-与跨-skill-查询)
  - [11. 回顾 Harness 全链路](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/day11_%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A.md#11-回顾-harness-全链路)

---

#### 一、认识 Harness 全链路

##### 1. 本章学习目标

<span style="color:red">目标</span>：建立完整认知，再逐层理解 Harness 中的提示、协议、路由、执行和观测。

本章完成以下能力：

1. 使用四层信息指导模型；
2. 使用 Skill 拆分领域能力；
3. 使用中间件限制工具范围；
4. 使用动态提示词注入当前领域流程；
5. 在一次 Run 中切换多个 Skill；
6. 完成页面动作的两阶段状态流转；
7. 统计 Token 并提供运行观测接口。

最终形成：

```text
表达规则
→ 限制能力
→ 执行业务查询
→ 校验最终结果
→ 保存运行状态
→ 提供审计信息
```

##### 2. 为什么需要完整 Harness

<span style="color:red">目标</span>：理解仅有 Prompt、工具或结构化输出为什么都不足以独立保证 Agent 可靠运行。

如果把全部信息写入一个 Prompt，会出现三个问题：

- 全局规则、领域流程和工具细节混在一起；
- 工具数量增加后，模型更容易选错工具。

如果只注册工具，只完成了能力接入：

- 全部工具同时暴露，无法按当前领域收敛能力范围；
- 工具描述只定义单次调用，不定义跨工具的领域处理流程；

```text
注册工具
→ 让模型能够调用

完整 Harness
→ 决定工具何时可见、如何调用
```

如果只使用数据模型，只能保证返回结构合法：

```text
字段存在 ≠ 业务事实真实
动作编码合法 ≠ 页面动作可信
```

因此，生产级 Harness 需要同时回答四个问题：

```text
模型应该遵守什么规则？
当前问题属于哪个领域？
当前允许调用哪些工具？
最终结果是否具有服务端证据？
```

<span style="color:black"><strong>完整 Harness 不是增加一个组件，而是让多个组件各自承担一种稳定职责。</strong></span>

##### 3. Harness 整体架构

<span style="color:red">目标</span>：从整体上认识一次 Agent 请求经过的核心层次。

这条链路可以归纳为五层：

![](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/1.jpg)

1. **输入层**：编译对话和当前请求；
2. **路由层**：选择 Skill 并限制工具；
3. **执行层**：调用模型和业务工具；
4. **信任层**：校验事实与页面动作；
5. **运行层**：保存状态、Token、耗时和结果。

##### 4. 一次请求的完整链路

<span style="color:red">目标</span>：按照执行顺序串联动态 Skill、业务工具、输出校验和 Run 状态。

一次请求按以下顺序运行：

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/2.png" style="zoom: 67%;" />

其中，第 4 至第 11 步可以在同一个 `agent.ainvoke()` 中循环多次。模型可以加载 Skill、调用工具、切换 Skill，最后再生成结构化输出。

##### 5. 核心组件与代码结构

核心代码分布如下：

![](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/3.jpg)

各目录只解决一类问题：

```text
llm       → 模型协议
skills    → 领域能力
tools     → 外部数据
validator → 结果可信
run       → 生命周期
app       → HTTP 接口
```

##### 6. 本章实现路线

<span style="color:red">目标</span>：按依赖顺序完成实现，避免先写中间件却没有状态和 Skill 定义。

实现顺序如下：

![](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/4.png)

先建立协议，再实现运行机制，最后验证完整链路。

---

#### 二、建立提示与协议体系

##### 1. 提示与协议

<span style="color:red">目标</span>：先理解四层信息的整体分工，再分别实现每一层。

Agent 需要四层信息：

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/5.jpg" style="zoom:50%;" />

<span style="color:black"><strong>全局规则保持稳定，领域规则按需加载，工具规则靠近实现，输出规则由模型强制校验。</strong></span>

##### 2. 四层约束

<span style="color:red">目标</span>：理解四层约束不是重复描述，而是在不同阶段回答不同问题。

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/6.jpg" style="zoom:50%;" />

##### 3. Prompt 全局行为

<span style="color:red">目标</span>：把跨领域、长期稳定的行为规则集中在 `BASE_PROMPT`。

`prompt.py` 只保存全局规则：

```python
BASE_PROMPT = """你是电商平台的只读智能客服。

一、职责边界
...

二、信息来源与工具结果
...

三、回复决策
...

四、页面引导
...

五、最终回复
...
"""
```

全局 Prompt 负责五类规则：

1. **职责边界**：只处理电商问题，不声称已经执行写操作；
2. **事实来源**：实时业务事实必须来自成功工具结果；
3. **回复决策**：选择回答、澄清、拒绝或转人工；
4. **页面引导**：只在用户明确需要时请求页面动作；
5. **最终回复**：不承诺稍后查询或执行。

这里不写具体 Skill 指导，也不拼接页面地址。原因是：

```text
Skill 会变化
页面路径由服务端维护
全局 Prompt 应保持稳定
```

Prompt 属于行为引导。模型仍可能不遵守，因此后续必须使用中间件和 Validator 建立硬约束。

##### 4. Skill 领域流程

<span style="color:red">目标</span>：把商品、订单、物流、规则和售后拆成边界清晰的领域能力。

一个 Skill 包含四项信息：

```python
@dataclass(frozen=True)
class SkillDefinition:
    code: SkillCode
    description: str
    guidance: tuple[str, ...]
    tools: tuple[str, ...]
```

字段职责如下：

- `code`：稳定领域编码；
- `description`：模型选择 Skill 时看到的简短说明；
- `guidance`：Skill 激活后的处理流程；
- `tools`：当前领域允许使用的工具名称。

例如物流 Skill：

```python
SkillDefinition(
    code=SkillCode.LOGISTICS_SERVICE,
    description="具体订单的发货状态、承运公司、运单号和物流轨迹",
    guidance=(
        "查询入口：用户没有提供订单编号时，先使用 list_orders 查询订单列表。",
        "工具选择：订单基础信息使用 get_order，配送进度和物流轨迹使用 get_logistics。",
        "可信依据：物流公司、运单号、状态和轨迹只能来自 get_logistics 的成功结果。",
        "边界处理：物流工具失败或没有结果时不得猜测配送进度。"
    ),
    tools=("list_orders", "get_order", "get_logistics")
)
```

所有 Skill 使用相同 guidance 结构：

```text
查询入口
→ 工具选择
→ 可信依据
→ 边界处理
```

`SkillCatalog` 负责保存和读取定义：

```python
class SkillCatalog:
    def get_skill(self, skill_code: SkillCode) -> SkillDefinition:
        return self.definitions[skill_code]

    def render_index(self) -> str:
        return "\n".join(
            f"- {skill.code}: {skill.description}"
            for skill in self.definitions.values()
        )
```

`render_index()` 只输出编码和描述，用于首次路由；完整 guidance 只在 Skill 激活后注入。

##### 5. Tool Call 调用

<span style="color:red">目标</span>：让每个工具准确说明“什么时候调用、需要什么参数、返回什么数据”。

工具描述由两部分组成：

```python
@tool
async def get_order(
    order_id: Annotated[
        str,
        "用户明确提供或订单列表返回的订单编号，例如 ORDER_001"
    ],
    runtime: ToolRuntime[AgentRuntimeContext]
) -> str:
    """
    查询当前用户的一条订单详情。
    用户已经明确订单编号，并询问该订单的状态、金额、
    创建时间或商品信息时调用。
    只读取数据，不取消订单、退款或修改地址。
    """
```

其中：

- docstring 说明调用场景和能力边界；
- `Annotated` 说明参数来源和格式；
- 返回模型说明业务结果；
- `ToolExecutor` 负责执行、校验和保存记录。

**工具描述与 Skill guidance 的区别是：**

```text
Skill guidance → 当前领域如何完成一个用户目标
工具描述       → 当前工具如何完成一个具体查询
```

##### 6. BaseModel数据模型

<span style="color:red">目标</span>：使用 Pydantic 定义模型必须返回的字段，并约束字段之间的组合关系。

模型最终返回 `AgentOutput`：

```python
class AgentOutput(BaseModel):
    reply_type: ReplyType
    reply_content: str
    handoff_request: HandoffRequest | None = None
    page_action_request: PageActionRequest | None = None
```

<span style="color:black"><strong>数据模型保证“结构正确”，Validator 继续保证“业务可信”。</strong></span>

##### 7. 提示与协议完整链路

<span style="color:red">目标</span>：把 Prompt、Skill、工具描述和数据模型重新组合成一条连续链路。

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/7.jpg" style="zoom:67%;" />

四层设计可以归纳为：

```text
Prompt 管长期规则
Skill 管当前领域
工具描述管单次调用
数据模型管最终结构
```

完成协议分层后，下一步不是继续增加 Prompt，而是使用状态和中间件让这些协议在运行时生效。

---

#### 三、实现动态 Skill 路由

##### 1. 动态 Skill 整体设计

<span style="color:red">目标</span>：先理解动态 Skill 如何在一个共享 Agent 中按需切换领域能力。

当前方案不是创建多个 Agent，而是创建一个 Agent：

```text
共享模型
+ 全量工具注册
+ 当前 Skill 状态
+ 动态 Prompt
+ 工具过滤中间件
```

运行时只激活一个 Skill：

```text
未加载 Skill
→ 只允许 load_skill

已加载 Skill
→ 允许 load_skill + 当前 Skill 工具
```

需要处理其他领域时，再次调用 `load_skill` 覆盖当前 Skill。

<span style="color:black"><strong>Skill 是能力切片，不是新的 Agent 实例。</strong></span>

##### 2. Skill 路由执行流程

<span style="color:red">目标</span>：从整体看Skill 选择、状态更新、提示注入和工具过滤。

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/8.jpg" style="zoom: 67%;" />

每次模型调用前都会重新执行动态 Prompt 和中间件，因此状态更新可以立即影响下一步模型调用。

##### 3. 建立 Skill 目录

<span style="color:red">目标</span>：通过固定目录集中管理可用 Skill，避免模型创造不存在的领域编码。

系统当前定义五个 Skill：

```text
product-service
→ 商品搜索、详情、规格、价格和库存

order-service
→ 订单列表、详情、状态和金额

logistics-service
→ 发货状态、承运公司、运单号和物流轨迹

policy-service
→ 平台规则、服务政策和帮助说明

after-sales-service
→ 用户实际售后记录、状态和进度
```

每个 Skill 同时提供：

```text
路由描述
领域指导
工具白名单
```

Skill 之间需要保持边界清晰。例如：

```text
policy-service
→ 解释平台通用规则

after-sales-service
→ 查询用户实际售后记录
```

这样可以避免把知识内容当作实时业务状态，也避免使用售后记录解释平台通用政策。

##### 4. load_skill 加载领域

<span style="color:red">目标</span>：通过工具调用把模型选择的领域写入 Agent 执行状态。

`load_skill` 是未激活 Skill 时唯一可见的工具：

```python
@tool
async def load_skill(
    skill_code: Annotated[
        SkillCode,
        "与当前用户问题最匹配的客服领域技能编码"
    ],
    runtime: ToolRuntime[
        AgentRuntimeContext,
        AgentExecutionState
    ]
) -> Command:
```

它完成三件事：

1. 从 `SKILL_CATALOG` 读取固定定义；
2. 把 Skill 内容作为 `ToolMessage` 返回模型；
3. 使用 `Command` 更新 `active_skill_code`。

```python
return Command(
    update={
        "active_skill_code": skill_code,
        "messages": [
            ToolMessage(
                content=result_json,
                tool_call_id=runtime.tool_call_id
            )
        ]
    }
)
```

`load_skill` 不查询业务数据，也不参与业务事实校验。它只负责切换运行能力。

##### 5. 保存当前 Skill 状态

<span style="color:red">目标</span>：单次 Run 内可变化的执行状态与始终不变的运行上下文。

运行上下文保存固定信息：

```python
@dataclass(frozen=True)
class AgentRuntimeContext:
    run_id: str
    conversation_id: str
    user_id: str
    access_token: str
```

执行状态保存运行过程中会变化的信息：

```python
class AgentExecutionState(AgentState[AgentOutput]):
    active_skill_code: NotRequired[SkillCode]
```

两者职责不同：

```text
AgentRuntimeContext
→ 谁在执行、属于哪个 Run、使用哪个令牌

AgentExecutionState
→ 当前执行到哪个 Skill
```

`active_skill_code` 只在本次 `agent.ainvoke()` 中生效。新的 Run 会重新从未加载 Skill 开始。

##### 6.  Agent 中间件

<span style="color:red">目标</span>：中间件在模型调用前统一修改请求，不侵入业务工具。

`SkillScopeMiddleware` 继承 `AgentMiddleware`：

```python
class SkillScopeMiddleware(
    AgentMiddleware[
        AgentExecutionState,
        AgentRuntimeContext,
        AgentOutput
    ]
):
```

三个泛型分别描述：

```text
AgentExecutionState
→ 中间件可以读取的 Agent 状态

AgentRuntimeContext
→ 当前 Run 的上下文

AgentOutput
→ Agent 的结构化输出类型
```

中间件实现 `awrap_model_call()`，表示它在每次异步模型调用前包装请求：

```python
async def awrap_model_call(
    self,
    request: ModelRequest[AgentRuntimeContext],
    handler: Callable[..., Awaitable[ModelResponse[AgentOutput]]]
) -> ModelResponse[AgentOutput]:
```

它不直接调用业务服务，只负责调整模型可见的工具和调用设置。

##### 7. Skill 过滤工具

<span style="color:red">目标</span>：使用工具可见性建立比 Prompt 更强的领域边界。

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/9.jpg" style="zoom:67%;" />

这里形成两个关键约束：

1. 模型看不到当前 Skill 之外的工具；
2. 模型一次只能串行调用一个工具。

```text
Prompt：告诉模型不要越界
中间件：让模型无法越界
```

##### 8. 动态提示词

<span style="color:red">目标</span>：根据当前 Skill 在每次模型调用前生成不同的系统提示词。

系统提示词由三部分组成：

```text
BASE_PROMPT
+ 当前 Skill 提示
+ 白名单页面动作
```

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/10.jpg" style="zoom:67%;" />

`@dynamic_prompt` 会在每次模型调用前执行，而不是只在 Run 开始时执行一次。这是 Skill 切换后提示词能够同步变化的关键。

##### 9.Agent切换 Skill

<span style="color:red">目标</span>：跨领域问题如何在同一个 `ainvoke` 中串行完成。

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/11.jpg" style="zoom:67%;" />

系统始终只开放一个领域的工具，避免多个 Skill 同时扩大工具范围。

##### 10. 组装动态 Skill Agent

<span style="color:red">目标</span>：在 Agent 工厂中统一注册工具、状态、输出协议和中间件。

```python
def create_support_agent():
    return create_agent(
        model=ModelAdapter.create_model(),
        tools=TOOL_CATALOG.get_agent_tools(),
        response_format=ToolStrategy(AgentOutput),
        state_schema=AgentExecutionState,
        context_schema=AgentRuntimeContext,
        middleware=[
            support_prompt,
            SkillScopeMiddleware(),
            ToolCallLimitMiddleware(
                run_limit=8,
                exit_behavior="error"
            )
        ],
        name="智能客服专家"
    )
```

组装关系如下：

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/12.jpg" style="zoom:67%;" />

<span style="color:black"><strong>工具全量注册，能力按需暴露，是动态 Skill Agent 的核心设计。</strong></span>

---

#### 四、完成 Run 闭环与观测

##### 1. Run 闭环整体设计

<span style="color:red">目标</span>： Coordinator、Executor、Validator 和 Mapper 共同完成一次 Run。

四个组件分别负责：

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/13.jpg" style="zoom: 80%;" />

它们共同形成：

```text
运行开始
→ Agent 推理
→ 工具调用
→ 输出校验
→ 状态映射
→ 结果持久化
→ 事件返回
```

##### 2. Run 完整执行流程

<span style="color:red">目标</span>：按照状态变化理解普通回答、页面决策和执行失败的不同出口。

![](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/14.jpg)

##### 3. 执行并校验 Agent 输出

<span style="color:red">目标</span>：明确 AgentExecutor 负责执行，AgentOutputValidator 负责建立信任。

`AgentExecutor.execute()` 先编译消息，再创建 Token 回调：

```python
messages = self.context_compiler.compile_messages(request)
usage_callback = UsageMetadataCallbackHandler()
```

随后调用 Agent：

```python
raw_agent_output = await self.agent.ainvoke(
    {"messages": messages},
    context=runtime_context,
    config={"callbacks": [usage_callback]}
)
```

模型结果经过三步处理：

```text
原始 Agent 结果
→ ModelAdapter 解析 AgentOutput
→ AgentOutputValidator 校验
→ ValidatedAgentOutput
```

`ValidatedAgentOutput` 在模型协议上增加：

```python
class ValidatedAgentOutput(AgentOutput):
    page_action: PageAction | None = None
    token_usage: dict[str, int] = ...
```

其中：

- `page_action_request` 是模型意图；
- `page_action` 是服务端可信动作；
- `token_usage` 是执行器统计结果。

如果输出错误允许纠正，Executor 会保留当前轨迹并追加反馈，再调用一次模型。默认最多执行两次生成。

##### 4. 限制工具调用次数

<span style="color:red">目标</span>：在模型发生工具循环时及时终止，避免一次 Run 无限消耗资源。

Agent 工厂注册：

```python
ToolCallLimitMiddleware(
    run_limit=8,
    exit_behavior="error"
)
```

达到上限时，LangChain 抛出 `ToolCallLimitExceededError`。Executor 不继续调用模型，而是返回固定结果：

```python
ValidatedAgentOutput(
    reply_type=ReplyType.DECLINE,
    reply_content=(
        "本次查询调用业务工具次数过多，"
        "暂时无法完成查询。"
    )
)
```

该结果是可安全展示的降级回复，因此 Run 最终进入 `COMPLETED`，而不是 `FAILED`。

```text
工具超限
→ 停止循环
→ 返回固定 DECLINE
→ 正常结束 Run
```

##### 5. 生成页面动作决策

<span style="color:red">目标</span>：把模型提出的动作意图转换成服务端可信页面动作。

模型只能输出：

```json
{
  "action_code": "CANCEL_ORDER",
  "resource_id": "ORDER_001"
}
```

`PageActionValidator` 继续校验：

1. 动作编码必须存在于 `ACTION_CATALOG`；
2. 具体订单动作必须携带订单编号；
3. 当前 Run 必须成功调用过相同订单的 `get_order`；
4. 页面地址由服务端固定模板生成。

校验后得到：

```json
{
  "label": "前往取消订单",
  "description": "进入订单页面核对状态并确认取消",
  "href": "/me/orders/ORDER_001/cancel"
}
```

Mapper 将动作写入：

```text
result.content.action
```

只要可信输出中存在 `page_action`，Run 状态就映射为：

```text
DECISION_PREPARED
```

##### 6. 确认与取消页面动作

<span style="color:red">目标</span>：使用两阶段状态流转表达“AI 提出页面引导，客户端决定是否采用”。

生成动作后，Run 尚未真正结束：

```text
RUNNING
→ DECISION_PREPARED
```

此时不设置 `finished_at`，表示仍在等待客户端决定。

客户端确认：

```text
POST /internal/v1/agent/runs/{run_id}/confirm
→ state = COMPLETED
→ 设置 finished_at
→ 返回 run_completed
```

客户端取消：

```text
POST /internal/v1/agent/runs/{run_id}/cancel
→ state = SUPERSEDED
→ 设置 finished_at
→ 返回 204
```

<span style="color:black"><strong>AI 只准备决策，客户端负责确认决策。</strong></span>

##### 7. 统计并保存 Token 用量

<span style="color:red">目标</span>：统计一次 Run 中全部模型调用消耗，而不是只记录最后一次生成。

`UsageMetadataCallbackHandler` 挂载在整个 Agent 调用上，因此可以覆盖：

- Skill 路由；
- 工具调用后的继续推理；
- 结构化输出；
- 输出纠正。

执行完成后汇总：

```python
return {
    "input_tokens": sum(
        item.get("input_tokens", 0)
        for item in usage_items
    ),
    "output_tokens": sum(
        item.get("output_tokens", 0)
        for item in usage_items
    )
}
```

Token 先写入 `ValidatedAgentOutput`，再由 Coordinator 保存到 `AgentRun`：

```text
input_tokens
output_tokens
```

这种设计让 Token 与当前 Run 保持同一统计边界。

##### 8. 提供管理员观测接口

<span style="color:red">目标</span>：通过汇总、列表和详情三个层次观测 Agent 运行情况。

管理员接口统一要求：

```python
get_auth_service().get_authorized_user(
    authorization,
    "admin"
)
```

系统提供三个接口：

```text
GET /api/v1/admin/metrics
→ Run 数量、平均耗时、输入 Token、输出 Token

GET /api/v1/runs
→ 最近 Run 的状态、模型、耗时和 Token

GET /api/v1/runs/{run_id}
→ Run 结果和工具调用详情
```

##### 9. 前端展示运行详情

<span style="color:red">目标</span>：把后端观测数据组织成可用于联调和排查的管理界面。

<img src="originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/15.jpg" style="zoom:50%;" />

Run 详情用于回答三个问题：

```text
模型最终返回了什么？
本次调用了哪些工具？
本次消耗了多少时间和 Token？
```

##### 10. 总结 Harness 全链路

<span style="color:red">目标</span>：从整体重新归纳 Day11 完成的生产级 Harness。

![](originals/day11_AI%E6%9C%8D%E5%8A%A1%EF%BC%88%E7%94%9F%E4%BA%A7%E7%BA%A7Harness%E5%85%A8%E9%93%BE%E8%B7%AF%E6%89%93%E9%80%9A%EF%BC%89/2_resource/images/16.jpg)

完整信任边界可以归纳为三层：

```text
第一层：提示边界
BASE_PROMPT + Skill guidance + 工具描述

第二层：执行边界
SkillScopeMiddleware + ToolCallLimitMiddleware

第三层：结果边界
数据模型 + AgentOutputValidator + Run 状态机
```

<span style="color:black"><strong>模型负责理解和生成，Skill 负责组织能力，中间件负责限制执行，工具负责提供事实，Validator 负责建立信任，Coordinator 负责完成闭环。</strong></span>

这就是生产级 Harness 全链路打通后的核心结构。

---

<a id="chapter-15"></a>

## 15 · Docker安装指南

> [查看本篇原文](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/Docker%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97.md) · [返回目录](#阅读目录)

### 	Docker环境安装



#### 一、统一虚拟机设置

##### 1、修改VMnet8网卡

**windows10**

![](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/images/2-1697804269629.png)

**windows11**

![](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/images/1-1697804269629.png)



##### 2、确认网卡信息

![](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/images/3-1697804269629.png)



##### 3、修改静态Ip

```sh
网卡所在目录
cd  /etc/sysconfig/networks-scripts

ls
```



![](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/images/4-1697804269630.png)

```sh
# 如上图，则修改ens33即可，下面为模板，
# 需要注意的字段。
#  ONBOOT=yes               IPADDR=192.168.200.128      GATEWAY=192.168.200.2 
#  NETMASK=255.255.255.0    DNS1=114.114.114.114        BOOTPROTO=static
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=ad2a226b-701d-4973-ae20-90d2cefa1a12
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.200.128
GATEWAY=192.168.200.2
NETMASK=255.255.255.0
DNS1=114.114.114.114

### 你的网卡至少可以为
BOOTPROTO=static
DEFROUTE=yes
NAME=ens33
UUID=b8fd5718-51f5-48f8-979b-b9f1f7a5ebf2
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.200.128
GATEWAY=192.168.200.2
NETMASK=255.255.255.0
DNS1=114.114.114.114

```

![](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/docker%E5%AE%89%E8%A3%85/images/5-1697804269630.png)



##### 4、用xshell等工具连接

> 设置完静态Ip后，使用Xshell等连接工具连接即可。



#### 二、安装**Docker**

> 官方文档：https://docs.docker.com/engine/reference/commandline/docker/

##### 1、修改yum源

```sh
yum clean all   # 清除旧缓存
yum makecache   # 构建新缓存
```

##### 2、移除之前的Docker和有关依赖

```sh
sudo yum remove docker *
```

##### 3、安装Docker repository

```sh
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 从国内阿里云镜像站，同步 Docker 官方的所有安装包
```

##### 4、安装Docker Engine

```sh
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
# docker-ce: Docker 后台核心守护进程，负责处理高级业务（如网络、存储）并调度底层
# docker-ce-cli: Docker 命令行工具，负责在终端接收并传达用户输入的命令
# containerd.io: 底层容器运行时，负责真正去执行容器的生命周期管理和镜像存储
# docker-compose-plugin: 多容器编排插件，支持用 docker compose 一键运行多个容器
```

##### 5、开启Docker

```sh
sudo systemctl enable docker --now

#和上面等价
sudo systemctl start docker
systemctl enable docker
```





---

<a id="chapter-16"></a>

## 16 · Docker镜像加速（梯子）

> [查看本篇原文](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F/Docker%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F%EF%BC%88%E6%A2%AF%E5%AD%90%EF%BC%89.md) · [返回目录](#阅读目录)

### Docker镜像加速



要让阿里云服务器或者虚拟机通过你本地电脑的 **Clash Verge** 上网，最稳妥且不需要额外安装复杂软件的方法是使用 **反向SSH 隧道 **。

#### 1. 操作

##### 第一步：配置本地 Clash Verge

**查看端口**：默认通常是 `7895`。

**查看本地局域网 IP**：在你的 Windows 电脑终端输入 `ipconfig`，找到类似 `192.168.x.x` 的 IP（假设为 `192.168.1.5`）。



##### 第二步：建立反向 SSH 隧道

需要从 **Windows 电脑** 连接到阿里云服务器。在 Windows 的 PowerShell 或 CMD 中执行以下命令：

```powershell
# 将本地的 7895 端口映射到阿里云服务器的 7895 端口
ssh -R 7897:127.0.0.1:7895 root@111.228.53.183
# 将本地的 7895 端口映射到虚拟机的 7895 端口
ssh -R 7895:127.0.0.1:7895 root@192.168.200.155

# 替换 root 和 111.228.53.183 为你的实际用户名和公网 IP
```

执行后保持这个窗口不要关闭。此时，阿里云服务器的 `127.0.0.1:7895` 就等同于你本地电脑的代理端口。

##### 第三步：配置 Docker 代理

现在阿里云服务器或者虚拟机已经可以通过隧道访问代理了，接下来配置 Docker 守护进程

1. 创建 Docker 代理目录

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

2. 创建代理配置文件

```bash
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
```

3. 写入以下内容（注意这里使用 127.0.0.1）

```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7895"
Environment="HTTPS_PROXY=http://127.0.0.1:7895"
Environment="NO_PROXY=localhost,127.0.0.1,mirrors.aliyun.com"
```

**告诉 Docker，在下载资源时通过之前搭建的 SSH 通道（127.0.0.1:7895）去借用代理上网，同时避开不需要代理的国内/本地网络。**

4. 重新加载并重启 Docker

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```



##### 第四步：验证与拉取镜像

在阿里云服务器或者虚拟机上测试网络是否通畅：

```bash
# 测试能否访问 Google
curl -I https://www.google.com --proxy http://127.0.0.1:7895

# 拉取 nginx 镜像
docker pull nginx
```





#### 2. 原理图

![](originals/day01_%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D%26%E5%AE%A2%E6%9C%8D%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96/4_other/%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F/images/Docker%E8%AE%BF%E9%97%AE%E5%8E%9F%E7%90%86%E5%9B%BE.png)1

---

<a id="chapter-17"></a>

## 17 · Harness_Engineering概念与原理

> [查看本篇原文](originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86.md) · [返回目录](#阅读目录)

### Harness Engineering 概要

#### 学习目标

阅读本文后，应当能够回答以下问题：

1. Harness Engineering 到底解决什么问题？
2. 为什么使用同一个模型，不同 Agent 产品的实际能力会相差很大？
3. Prompt Engineering、Context Engineering 和 Harness Engineering 有什么关系？
4. 一个朴素 Agent 会出现哪些典型故障？
5. Harness 通过哪些机制保证 Agent 可以长时间稳定运行？
6. 如何评估一个新的 Agent 产品或自研 Agent 系统？

#### 1. 初识驾驭工程

今天我们聚焦的，是Claude Code、Codex、Cursor 这类产品背后共同的工程系统——**驾驭工程（Harness Engineering）**,也是 2025—2026 年 AI 工程领域逐渐形成共识的一门新兴工程学科。

这些产品看起来都只是在“调用大模型”，但实际使用时却表现出明显差异：

- 为什么有的 Agent 可以连续运行很长时间，有的执行几轮就开始偏离目标？
- 为什么使用同一个底层模型，裸 API 与 Claude Code、Codex、Cursor 的交付质量仍然不同？
- 为什么有的产品能够修改几十个文件、执行测试并持续修复，有的只能在对话框中生成一段看似正确的代码？
- 为什么模型明明足够强，真正放进项目后仍然会陷入死循环、忘记进度或错误地宣布完成？

答案通常不只在模型本身，而在模型外部是否存在一套成熟的 Harness。模型负责理解和推理，Harness 则负责把推理转化为受控行动，并为长任务提供上下文、工具、反馈、状态、验证和安全边界。

##### 1.1 三层能力对比

先把裸对话、沙箱执行和完整编码 Agent 放在一起比较：

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/2.%E4%B8%89%E5%B1%82%E8%83%BD%E5%8A%9B%E5%88%86%E6%9E%90.png" style="zoom: 67%;" />



三层产品在“能不能生成代码”上差距并不大，真正的分水岭出现在**执行、反馈、恢复、验证和安全**上。

同一个任务在三种模式下会形成完全不同的结果：

1. **裸对话模式**：模型给出代码，但不会真正创建文件、运行测试或观察失败；
2. **沙箱执行模式**：模型能够临时运行代码，但通常无法完整接入本地项目、Git 历史和跨会话状态；
3. **完整 Agent 模式**：模型在 Harness 的支持下读取项目、修改文件、执行工具、验证结果，并根据失败证据继续迭代。

因此，Agent 的关键能力不是“生成更多内容”，而是形成完整行动闭环：

> **观察环境 → 采取行动 → 获取反馈 → 修正行动 → 验证完成**

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/3.Agent%E5%AE%8C%E6%95%B4%E9%97%AD%E7%8E%AF.png" style="zoom:67%;" />

##### 1.2 Model、Agent 与 Harness

这三个概念分别位于不同层次：

| 概念 | 主要职责 | 自身不负责什么 |
|---|---|---|
| Model | 理解输入、推理并生成下一步内容 | 不直接管理文件、进程、权限和长期状态 |
| Agent | 围绕目标进行决策并调用工具 | 不代表天然具备可靠性和安全性 |
| Harness | 管理 Agent 的上下文、执行、状态、验证和边界 | 不替代模型本身的推理能力 |

可以把三者理解为：

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/4.Model%E3%80%81Agent%20%E4%B8%8E%20Harness%E5%85%B3%E7%B3%BB.png" style="zoom: 67%;" />

Model 提供**大脑**，Agent 表现为**行动主体**，Harness 则为行动提供**轨道、反馈和护栏**。

##### 1.3 Workflow 与 Agent 的区别

Workflow 和 Agent 都能完成多步骤任务，但控制方式不同：

| 对比项 | Workflow | Agent |
|---|---|---|
| 执行路径 | 由开发者预先定义 | 由模型根据当前状态动态决定 |
| 可预测性 | 较高 | 相对较低 |
| 灵活性 | 适合固定流程 | 适合开放性任务 |
| 失败处理 | 通常使用预设分支 | 可能由模型分析错误后调整 |
| **Harness** | 编排、重试和状态机 | 同时需要上下文、工具、安全、验证和预算控制 |

两者并不是互斥关系。一个成熟系统往往用 Workflow 固定关键边界，再让 Agent 在某些节点中进行动态决策。

##### 1.4 Tool Calling 不等于 Harness

Tool Calling 只解决“模型如何表达一次工具调用”，例如让模型输出工具名称和参数。它没有自动解决：

- 工具是否允许执行；
- 参数是否安全；
- 执行失败后如何恢复；
- 多个工具如何持续编排；
- 任务进度如何保存；
- 最终结果如何验证；
- 运行成本如何限制。

因此：

> **Tool Calling 让模型能够行动，Harness Engineering 让行动变得可控、可持续、可恢复、可验证。**

##### 1.5 Framework、Runtime 与 Harness

这三个术语也经常被混用：

- **Framework** 提供模型、工具、检索器等开发组件，相当于“积木”；
- **Runtime** 负责执行状态机、工作流、检查点和中断恢复，相当于“发动机”；
- **Harness** 把上下文、Runtime、工具、验证、安全和成本机制组合成完整运行系统，相当于“整车”。

使用了 LangChain、LangGraph 或其他 Agent 框架，不代表已经拥有成熟 Harness。只有当系统形成受控执行、错误反馈、状态恢复、外部验证和资源边界时，才真正进入 Harness Engineering 的范畴。

##### 1.6 Harness判断标准

判断一个系统是在“调用模型”，还是已经具备 Harness，可以先问五个问题：

1. 它能否在真实环境中执行动作？
2. 动作失败后，错误是否会反馈给模型？
3. 任务中断后，是否可以恢复进度？
4. 模型声称完成后，是否有外部验证？
5. 危险操作和资源消耗是否受到硬性限制？

如果这些问题大多没有答案，那么它更接近一个带工具的模型调用，而不是完整的 Harness。

#### 2. 核心概念

**概念**：harness 这个词本意是"马具"——套在马身上让人驾驭它的皮带、缰绳，通常被翻译成中文**驾驭工程**，也即现在常听的Harness Engineering。

Harness Engineering（驾驭工程）是围绕大模型构建的一套工程系统，用于让 Agent 能够安全、稳定、持续地完成真实任务。

其核心公式是：

> **Agent = Model + Harness**

- **Model** 负责理解、推理与生成；
- **Harness** 负责提供工具、上下文、执行循环、状态管理、验证机制和安全边界。

模型决定 Agent 的能力上限，Harness 决定这些能力能否稳定转化为结果。裸模型通常只能“给出答案”，具备完整 Harness 的 Agent 才能读取项目、修改文件、执行命令、验证结果，并在失败后继续调整。

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/5%E3%80%81Harness%E7%9A%84%E6%A6%82%E5%BF%B5.png" style="zoom: 50%;" />

##### 2.1 F1 赛车类比

可以把 Model 理解为赛车的引擎，把 Harness 理解为底盘、变速箱、刹车、方向盘、仪表盘和维修系统。

- 引擎决定理论速度；
- 底盘决定动力能否稳定传递；
- 刹车决定系统是否能安全停止；
- 仪表盘决定驾驶者能否看到故障；
- 维修和进站系统决定长时间比赛能否继续。

只升级模型，相当于只升级引擎。如果执行循环、工具反馈、状态恢复和安全边界没有同步完善，Agent 仍然很容易在真实任务中失败。

#### 3. 工程层次

三者是逐层扩展的包含关系，而不是互相替代：

1. **Prompt Engineering**：优化单次输入，让模型更准确地理解任务；
2. **Context Engineering**：管理模型当前能看到的信息，如历史消息、项目文件、工具定义和检索结果；
3. **Harness Engineering**：管理 Agent 的完整运行过程，使其能够长时间、多步骤地可靠执行。

可以简单理解为：

> Prompt 管“怎么问”，Context 管“让模型看到什么”，Harness 管“如何让模型持续行动并完成任务”。

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/1.%E4%B8%89%E5%A4%A7%E5%BC%95%E6%93%8E%E5%85%B3%E7%B3%BB.png" style="zoom: 33%;" />

##### 3.1 包含关系

常见说法是“Context Engineering 取代了 Prompt Engineering”或“Harness Engineering 取代了 Context Engineering”。这种理解并不准确。

一个成熟 Agent 在运行时，三个层次通常同时存在：

- System Prompt 规定角色、行为原则与输出格式；
- Context 层选择项目规则、相关文件、工具说明和历史状态；
- Harness 层控制循环、工具执行、进度恢复、权限审批和结果验证。

例如，一个编码 Agent 实现登录功能时：

1. Prompt 层告诉模型它是编程助手，并规定编码规范；
2. Context 层读取项目说明、已有认证代码和工具 Schema；
3. Harness 层拆分任务、修改文件、执行测试、保存进度，并在危险操作前请求确认。

##### 3.2 术语边界

| 术语 | 主要含义 | 与 Harness 的关系 |
|---|---|---|
| Agent | 能调用工具并执行任务的软件实体 | Agent 是最终产品形态，Harness 是其工程运行系统 |
| Scaffolding | Prompt 模板、Few-shot、工具描述等输入装配层 | 通常是 Harness 中偏上下文装配的一部分 |
| Framework | 提供模型、工具、检索器等开发抽象的类库 | Framework 提供积木，Harness 是组装后的运行系统 |
| Runtime | 执行状态机、工作流和检查点的运行引擎 | Runtime 是 Harness 的执行基础，但不等于完整 Harness |
| Harness | 覆盖上下文、执行、验证、恢复、安全和成本的机制组合 | 负责让 Agent 在真实环境中稳定完成任务 |

以 LangChain 生态为例，可以粗略理解为：

- `langchain-core` 提供 Framework 能力；
- LangGraph 提供有状态 Runtime；
- DeepAgents 在运行时之上组合长任务所需的 Harness 机制。

#### 4. Harness 的必要性

一个只有“LLM + Tool Calling + while 循环”的 最小Agent，通常会出现以下问题：

1. **循环失控**：没有明确终止条件，持续调用模型和工具；
2. **上下文溢出**：历史消息不断累积，最终超过上下文窗口；
3. **缓存失效**：频繁修改提示词前缀，无法命中 Prompt Cache；
4. **工具错误被吞掉**：异常和退出码没有结构化返回，Agent 无法识别失败；
5. **状态丢失**：进程中断或会话关闭后，只能从头开始；
6. **缺少权限边界**：危险命令和敏感操作可以直接执行；
7. **缺少独立评审**：Agent 自己生成、自己宣布完成，容易产生自评偏差；
8. **成本失控**：没有迭代次数、Token 或费用预算。

Harness Engineering 的本质，就是为这些故障建立系统化防线。

##### 4.1 最小 Agent

一个最简单的 Tool Calling Agent 往往只有以下逻辑：

```python
messages = [system_message, user_message]

while True:
    response = call_llm(messages, tools)
    messages.append(response)

    if not response.tool_calls:
        break

    for tool_call in response.tool_calls:
        result = run_tool(tool_call)
        messages.append(result)
```

这段逻辑能够演示 Agent 的基本原理，却不适合直接用于生产环境。它没有回答以下问题：

- 循环一直不结束怎么办？
- 工具执行失败，但模型没有察觉怎么办？
- 消息越来越多，超过上下文窗口怎么办？
- 进程被关闭后，任务从哪里恢复？
- 模型准备删除文件或强制推送时，谁来拦截？
- 模型说“完成了”，如何证明它真的完成了？
- 一个任务最多允许消耗多少时间和费用？

Harness 的价值就体现在这些“模型之外的问题”上。

##### 4.2 八大故障

| 故障 | 常见触发方式 | 直接后果 |
|---|---|---|
| 循环失控 | 使用 `while True`，只依赖模型主动停止 | 无限调用、资源占用和不可预测行为 |
| Context 溢出 | 每轮无条件追加全部消息和工具输出 | 请求失败、注意力下降、成本增加 |
| Cache Miss | 在 System Prompt 中加入时间、任务 ID 等动态内容 | 前缀缓存失效，延迟和费用上升 |
| Tool 错误吞 | 捕获异常后返回空字符串，忽略进程退出码 | Agent 在错误结果上继续执行 |
| 状态丢失 | 所有进度只保存在内存和对话历史中 | 中断后从头开始，长任务无法恢复 |
| 缺权限闸 | 所有工具调用自动执行 | 删除数据、泄露信息或破坏 Git 历史 |
| 缺自动评审 | 生成者自行判断任务是否完成 | 测试未通过、需求遗漏却被宣布完成 |
| 成本失控 | 没有迭代、Token、时间和费用上限 | 单任务消耗大量预算 |

##### 4.3 故障链

这些问题通常不是孤立发生的。例如：

1. Agent 没有拆分任务，导致一次处理过多内容；
2. 工具输出不断写入 Context，造成上下文膨胀；
3. 模型因信息过载做出错误判断；
4. 工具错误又被吞掉，模型误以为操作成功；
5. 验证缺失，Agent 提前宣布任务完成；
6. 如果模型没有宣布完成，则可能继续循环并消耗更多预算。

因此，成熟 Harness 不是只添加一个 `max_iterations`，而是建立多层、相互补充的防线。

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/6.%E6%95%85%E9%9A%9C%E9%93%BE.jpg" style="zoom: 50%;" />

#### 5. 核心机制

##### 5.1 Agent Loop

将运行过程拆分为：

> **Gather → Action → Verify → Iterate**

即收集上下文、执行动作、验证结果、继续迭代。同时设置最大迭代次数和明确的停止条件，保证循环一定能够退出。

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/7%E3%80%81Agent%20Loop.jpg" style="zoom:50%;" />

**解决的问题**：循环失控。

**关键设计**：

- 使用最大迭代次数作为硬刹车；
- 设置任务完成、任务失败、用户取消和预算耗尽等明确状态；
- 每轮执行前检查剩余预算；
- 每轮执行后必须进入验证阶段；
- 循环退出时返回可解释的终止原因。

```python
for iteration in range(MAX_ITERATIONS):
    context = gather_context(state)
    action = take_action(context)
    result = execute(action)
    verification = verify(result)
    state = update_state(state, verification)

    if state.is_done or state.is_blocked:
        break
```

这里真正重要的不是 `for` 语法，而是系统拥有高于模型的控制权。即使模型不断要求继续，Harness 也能在达到边界时强制停止。

##### 5.2 Tool Use

使用统一 Schema 描述工具名称、参数和返回值。工具失败时返回结构化错误，例如：

```json
{
  "status": "error",
  "error": "File not found"
}
```

失败信息必须对 Agent 可见，不能用空字符串掩盖异常。

**解决的问题**：工具错误不可见。

**关键设计**：

- 输入参数使用 Schema 校验；
- 返回值必须区分成功与失败；
- Shell 工具应保留 `stdout`、`stderr` 和 `returncode`；
- 设置超时、重试次数和可重试错误类型；
- 工具输出过长时进行截断或保存到文件，避免污染 Context；
- 未知工具和非法参数不能直接执行。

一个更完整的工具结果可以包含：

```json
{
  "status": "error",
  "tool": "run_tests",
  "exit_code": 1,
  "stdout": "3 passed, 1 failed",
  "stderr": "AssertionError",
  "retryable": true
}
```

结构化错误让模型能够区分“命令执行失败”“参数错误”“权限不足”和“暂时性网络故障”，从而选择不同的恢复策略。

##### 5.3 Progress Tracking

把任务状态、已完成步骤和下一步计划保存到文件或数据库，并在重要节点创建 Git 提交。即使进程退出，也能从最近的检查点继续执行。

**解决的问题**：状态丢失。

进度记录至少应包含：

- 原始任务和验收条件；
- 已完成、进行中和待处理的子任务；
- 修改过的文件；
- 已执行的验证及结果；
- 当前阻塞原因；
- 下一步建议动作；
- 时间、费用和 Token 消耗。

长期任务通常需要两类检查点：

1. **逻辑检查点**：进度文件、数据库状态或工作流 Checkpoint；
2. **物理检查点**：Git Commit、生成物快照或可恢复的文件版本。

Context Window 不能代替 Progress Tracking。前者是当前请求的短期工作区，后者才是跨会话、跨进程的长期状态。

##### 5.4 Context Management

持续监测上下文长度，在达到阈值时进行压缩、摘要或重置。同时保持 System Prompt 等稳定前缀不变，以提高缓存命中率。

上下文窗口是“临时工作区”，不是长期记忆；长期状态应持久化到外部存储。

**解决的问题**：Context 溢出与 Prompt Cache 失效。

Context 管理并不是简单删除旧消息，而是决定“下一轮模型真正需要看到什么”。常见策略包括：

- 只加载与当前子任务相关的文件；
- 对历史操作生成结构化摘要；
- 将完整日志保存到外部，只把索引和结论放入 Context；
- 对长工具输出截断，并保留错误附近的关键部分；
- 达到阈值时执行 Compaction；
- 在拥有可靠外部进度记录时重置会话；
- 保持稳定前缀不变，让 Prompt Cache 可以复用。

需要特别避免在 System Prompt 中写入时间戳、用户名、任务编号或实时状态。这些动态信息应进入 User Message 或状态区，否则每轮请求的前缀都会发生变化。

Compaction 和 Reset 各有代价：

- **Compaction** 保留连续性，但摘要可能遗漏细节或引入噪声；
- **Reset** 获得干净上下文，但强依赖外部进度记录才能恢复任务。

##### 5.5 Feature List

把大任务拆分成可验证的小任务，并记录状态：

```json
[
  {"id": 1, "task": "读取测试文件", "status": "done"},
  {"id": 2, "task": "修复缺陷", "status": "in_progress"},
  {"id": 3, "task": "运行测试", "status": "pending"}
]
```

每次只处理一个任务，避免 Agent 同时做太多事情导致上下文和执行过程失控。

**解决的问题**：大任务带来的 Context 膨胀和执行混乱。

好的子任务应具有以下特点：

- 范围足够小，可以在有限轮次内完成；
- 输入和输出明确；
- 有可以执行的验收条件；
- 与其他任务的依赖关系清晰；
- 状态可以持久化；
- 失败后能够单独重试。

Feature List 本质上是 Agent 的外置工作记忆。它不应该只存在于模型生成的一段自然语言计划中，而应保存为 JSON、数据库记录或工作流状态。

推荐的状态流转是：

> `pending → in_progress → verifying → passed / failed / blocked`

只有验证通过后才能标记为 `passed`，不能以“代码已经写完”作为完成依据。

##### 5.6 Verification Loop

Agent 声称“已完成”不等于任务真的完成。必须通过 pytest、Playwright、编译器、静态检查等外部工具验证结果。验证失败则继续修复，验证通过后才更新任务状态。

**解决的问题**：工具执行结果不可靠，以及模型自评偏差。

不同任务需要不同的验证器：

| 任务类型 | 推荐验证方式 |
|---|---|
| Python 后端 | pytest、类型检查、静态检查、接口测试 |
| 前端页面 | 单元测试、构建、Playwright 浏览器验证 |
| 数据任务 | 行数、Schema、空值、分布和业务约束检查 |
| 文档任务 | 链接检查、结构检查、读者测试 |
| 基础设施 | dry-run、配置校验、计划输出和健康检查 |

验证闭环必须防止“为了通过而修改标准”。例如，测试失败时 Agent 应优先修复实现，而不是删除测试、降低断言或跳过用例。

一个任务的真实完成条件应是：

> **产物存在 + 验收条件满足 + 验证证据可追溯**

##### 5.7 Subagents

把复杂子任务交给拥有独立上下文的子 Agent。子 Agent 只向主 Agent 返回最终结论，避免大量中间过程污染主上下文。

**解决的问题**：主 Context 被搜索、试错和大量工具输出污染。

子 Agent 适合处理：

- 独立搜索某个模块的实现；
- 运行一组耗时测试并汇总结论；
- 分析日志或错误原因；
- 对多个候选方案进行并行研究；
- 从独立视角审查代码或文档。

使用 Subagents 时需要控制协调成本：

- 子任务必须边界清晰；
- 返回值应使用统一格式；
- 主 Agent 只接收结论、证据和必要引用；
- 避免多个子 Agent 同时修改同一文件；
- 为子 Agent 设置独立预算和超时；
- 不要把无法拆分的强依赖任务机械并行化。

多 Agent 并不天然优于单 Agent。它用隔离性换取了通信、协调和一致性成本。

##### 5.8 Generator–Evaluator

将职责拆分为三个角色：

- **Planner**：拆分并规划任务；
- **Generator**：执行任务并生成产物；
- **Evaluator**：根据标准独立评审结果。

评审者不依赖生成者的自我解释，从架构上减少“自己给自己打分”的偏差。

**解决的问题**：缺少自动化评审和生成者自评偏差。

三角色之间可以形成如下流程：

1. Planner 根据需求和验收条件拆分任务；
2. Generator 只负责完成当前子任务；
3. Evaluator 不查看生成过程，只检查最终产物和证据；
4. 不通过时返回具体问题和修订标准；
5. Generator 根据反馈重新生成；
6. 达到评分阈值或重试上限后结束。

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/8%E3%80%81Generator%E2%80%93Evaluator.jpg" style="zoom:50%;" />

Evaluator 必须使用明确的评价标准，例如：

- 是否满足全部需求；
- 是否通过测试；
- 是否引入安全问题；
- 是否破坏已有行为；
- 是否存在未经验证的假设；
- 是否可以被维护者理解。

如果评价标准仍然只是“请看看做得好不好”，多角色架构不会自动带来高质量。

此外，生产级 Harness 通常还需要两个重要扩展：

- **Permission Gate**：拦截删除文件、强制推送、访问敏感数据等危险操作；
- **Token Budget**：限制迭代次数、Token 用量、执行时间和费用。

##### 5.9 故障与机制

| 故障模式 | 主要防护机制 | 辅助机制 |
|---|---|---|
| 循环失控 | Agent Loop | Token Budget、人工中止 |
| Context 溢出 | Context Management | Feature List、Subagents |
| Cache Miss | 稳定提示词前缀 | Context 分层 |
| Tool 错误吞 | Tool Use | Verification Loop |
| 状态丢失 | Progress Tracking | Git Checkpoint、Runtime Checkpoint |
| 缺权限闸 | Permission Gate | 沙箱、最小权限工具 |
| 缺自动评审 | Generator–Evaluator | Verification Loop |
| 成本失控 | Token Budget | 最大迭代数、超时、Context 管理 |

这张表说明一条故障通常不应只依靠一道防线。例如，设置最大迭代次数可以阻止无限循环，但如果没有费用预算和上下文控制，Agent 仍可能在达到迭代上限前产生高额消耗。

#### 6. 三大维度

可以从三个维度检查一套 Harness 是否完整：

##### Context Engineering

**Agent 能看到什么：**关注信息选择、任务拆分、上下文压缩、进度记录和子任务隔离。

##### Architectural Constraints

**Agent 能做什么：**关注执行循环、工具契约、权限边界、验证流程和多角色协作。

##### Garbage Collection

**Agent 能运行多久：**关注过期上下文清理、状态回收、资源限制和成本控制。

对应三个检查问题：

> 信息是否准确？动作是否受控？资源是否可持续？

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/9%E3%80%81%E4%B8%89%E5%A4%A7%E7%BB%B4%E5%BA%A6.jpg" style="zoom:50%;" />

##### 6.1 Context Engineering

这一层不只是“塞更多资料”，而是让 Agent 在正确的时间看到正确的信息。它包含：

- 项目规则和固定指令；
- 当前任务和验收条件；
- 相关代码与文档；
- 工具能力说明；
- 已完成步骤和外部进度；
- 历史错误与验证结果；
- Context 压缩、检索和重置策略。

衡量标准不是 Context 有多长，而是其中每一部分是否对下一步决策有用。

##### 6.2 Architectural Constraints

这一层通过代码和架构限制 Agent 的动作空间，例如：

- 只能调用白名单工具；
- 工具参数必须通过 Schema 校验；
- 危险操作必须获得批准；
- 每轮必须经过验证；
- 生成与评审职责分离；
- 子 Agent 只能访问完成任务所需的最小资源。

真正可靠的约束必须由系统实现，而不能只写在 Prompt 中。Prompt 中的“不要执行危险命令”属于软约束，Permission Gate 的拒绝执行才是硬约束。

##### 6.3 Garbage Collection

这里的 Garbage Collection 是课程为了便于理解使用的工程类比，重点是清理过期状态并控制资源：

- 清理无用的历史消息；
- 压缩过长工具输出；
- 回收失效会话和临时文件；
- 限制 Token、时间、迭代和费用；
- 结束无进展或重复执行的任务；
- 定期删除已经被新模型能力替代的 Harness 逻辑。

它回答的核心问题是：Agent 不只是能启动，还能否在有限资源中持续运行并最终收敛。

#### 7. 基本流程

一套最小可用 Harness 可以按以下流程运行：

1. 读取任务、项目规则和已有进度；
2. 将任务拆成可独立验证的子任务；
3. 选择一个待处理任务；
4. 收集完成该任务所需的最小上下文；
5. 调用模型并执行工具；
6. 结构化记录工具结果和错误；
7. 使用测试或其他外部工具验证；
8. 通过后保存进度，失败则带着证据继续迭代；
9. 达到完成条件、预算上限或安全边界时停止。

##### 7.1 完整状态流

```mermaid
flowchart TD
    A["初始化"] --> B["读取规则、任务、预算和已有进度"]
    B --> C["生成或恢复 Feature List"]
    C --> D["选择一个待办任务"]
    D --> E["收集最小必要 Context"]
    E --> F["模型决定下一步动作"]
    F --> G{"权限检查"}

    G -->|"拒绝"| H{"可以请求用户批准？"}
    H -->|"是"| I["请求用户确认"]
    I -->|"批准"| J["执行工具"]
    I -->|"拒绝"| X["保存进度并终止"]
    H -->|"否"| X

    G -->|"允许"| J
    J --> K["结构化记录结果、错误和退出码"]
    K --> L["运行外部验证"]
    L --> M{"验证结果"}

    M -->|"失败"| N["记录失败证据并重新规划"]
    N --> E
    M -->|"阻塞"| O["保存进度并等待外部条件"]
    M -->|"通过"| P["更新任务状态并创建检查点"]

    P --> Q{"全部任务完成？"}
    Q -->|"否"| D
    Q -->|"是"| R["执行最终验证"]
    R --> S{"最终验证通过？"}
    S -->|"否"| N
    S -->|"是"| T["生成交付报告与验证证据"]
```

##### 7.2 最小 Harness

如果从零开始构建 Harness，不需要一次实现所有高级机制。一个最小版本至少应具有：

1. 有上限的 Agent Loop；
2. 结构化工具返回；
3. 可持久化任务清单；
4. 至少一种真实验证方式；
5. 危险操作拦截；
6. 时间或 Token 预算；
7. 明确的完成、失败和阻塞状态。

Subagents、复杂 Compaction 和多角色评审可以根据真实故障数据逐步增加。

#### 8. 产品生态

Harness 不是某一个产品的专有能力。主流实现通常可以分成四种形态：

<img src="originals/day07_%E5%AE%A2%E6%9C%8D%E6%9C%8D%E5%8A%A1%E6%B5%8B%E8%AF%95%E4%B8%8E%E6%80%BB%E7%BB%93%26%26AI%E6%9C%8D%E5%8A%A1%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA/4_other/Harness_Engineering%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%8E%9F%E7%90%86/images/10%E3%80%81%E4%BA%A7%E5%93%81%E5%BD%A2%E6%80%81.jpg" style="zoom:50%;" />



评估一个产品时，不应只看它使用什么模型，而应检查：

- 它如何获取项目上下文；
- 能调用哪些工具；
- 如何限制危险操作；
- 是否支持任务拆解和状态恢复；
- 如何验证任务完成；
- 如何处理长 Context；
- 是否支持预算、超时和人工接管；
- 运行在本地、IDE、云沙箱还是自建 Runtime 中。

#### 9. 工程原则

1. **不要完全信任模型的自我判断**：循环必须有硬性出口，结果必须由外部证据验证。
2. **错误必须显性化**：异常、退出码和失败原因都应结构化返回。
3. **状态必须外置**：重要进度不能只保存在上下文窗口中。
4. **大任务必须拆分**：一次只完成一个可验证的小目标。
5. **安全边界由系统控制**：不能只依赖提示词要求模型“谨慎操作”。
6. **控制成本与资源**：为时间、迭代次数、Token 和费用设置预算。
7. **保持简单并准备删除**：Harness 会随着模型能力变化而过时，应持续评估、裁剪和重构。
8. **失败数据比复杂架构更有价值**：根据真实失败案例增加机制，不为假设中的问题提前堆叠复杂度。

##### 9.1 Start Simple

先实现最小可靠闭环，再根据真实失败扩展。过早加入复杂工作流、多 Agent 协议和大量规则，可能增加故障点，并限制更强模型的能力。

##### 9.2 Build to Delete

Harness 固化了对当前模型能力的假设。当模型升级后，旧的提示、重试逻辑和刚性流程可能从保护机制变成阻碍。

因此，Harness 代码应具备：

- 模块化；
- 可观测；
- 可通过 Benchmark 比较；
- 能够单独关闭某项机制；
- 能够快速删除或重写。

##### 9.3 Harness Is the Dataset

真正重要的资产不只是 Harness 代码，还包括运行过程中积累的失败数据：

- 哪类任务最容易失败；
- 失败发生在哪个步骤；
- 哪些工具错误最常见；
- 哪些 Context 会误导模型；
- 哪种验证能发现真实缺陷；
- 哪些人工干预最终帮助任务恢复。

这些数据既能指导 Harness 迭代，也能用于回归测试、模型选型和评估。

#### 10. 开放争议

##### 10.1 多 Agent 还是单 Loop

- 多 Agent 的优势是 Context 隔离和专业分工；
- 缺点是通信协议、失败重试和一致性管理更复杂；
- 单 Loop 更简单、可观察，但所有上下文压力集中在一个会话中。

选择依据应是任务是否真正可拆分，而不是追求 Agent 数量。

##### 10.2 Compaction 还是 Reset

- Compaction 适合需要保留会话连续性的任务；
- Reset 适合 Context 已被大量试错污染、且外部状态记录可靠的任务；
- 两者都不能替代 Progress Tracking。

##### 10.3 CLI-first 还是 IDE-native

- CLI-first 更适合脚本化、长任务和系统工具编排；
- IDE-native 更适合可视化编辑、局部修改和实时人工反馈；
- 两者服务不同工作流，并不存在适用于所有人的唯一答案。

#### 11. 成熟度检查

可以使用以下问题快速检查一套 Agent 系统：

##### 循环与状态

- 是否存在最大迭代次数？
- 是否有 completed、failed、blocked、cancelled 等明确状态？
- 中断后是否可以从检查点恢复？
- 是否能识别多轮没有进展的循环？

##### 工具与安全

- 工具输入是否经过 Schema 校验？
- 是否保留退出码、标准输出和错误输出？
- 是否设置执行超时？
- 危险操作是否需要审批？
- 工具权限是否遵循最小权限原则？

##### Context

- 是否只加载当前任务需要的信息？
- System Prompt 前缀是否稳定？
- 长输出是否截断或外置？
- 是否有 Compaction 或 Reset 策略？
- 长期状态是否保存在 Context 之外？

##### 验证与质量

- 每个子任务是否有明确验收条件？
- 是否执行真实测试，而不是只依赖模型自评？
- 是否禁止通过删除或弱化测试来制造“通过”？
- 最终交付是否包含验证证据？

##### 成本与可观测性

- 是否限制 Token、时间、迭代和费用？
- 是否记录每轮调用、工具结果和终止原因？
- 是否能够比较不同 Harness 版本的成功率？
- 是否收集并分类失败案例？

#### 12. 总结

Harness Engineering 不是单一框架或工具，而是一套让 Agent 从“能够推理”走向“能够可靠交付”的工程方法。

判断一套 Agent 系统是否具备成熟 Harness，可以检查四点：

- 是否有受控且可退出的执行循环；
- 是否能持久化状态并管理上下文；
- 是否通过外部工具验证真实结果；
- 是否具备权限、成本和资源边界。

最终目标不是让 Agent 无限自主，而是让它在明确边界内持续行动、识别失败、恢复进度，并交付可以验证的结果。

整套知识可以压缩为五个记忆点：

1. **一个公式**：`Agent = Model + Harness`；
2. **八类故障**：循环、Context、缓存、工具、状态、权限、评审、成本；
3. **八大机制**：Loop、Tool Use、Progress、Context、Feature List、Verification、Subagents、Generator–Evaluator；
4. **三个维度**：看见什么、能做什么、能跑多久；
5. **一种工程理性**：从真实失败出发，保持简单，并准备随模型进步删除旧 Harness。

Harness 的目标不是代替模型，也不是堆叠尽可能多的控制逻辑，而是在模型与真实世界之间建立一套可观察、可恢复、可验证、可约束的工程系统。
