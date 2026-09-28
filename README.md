# Harness Assistant · 电商智能客服

基于 FastAPI、LangChain、Vue、PostgreSQL 和 Redis 的电商客服项目。用户发送消息后，由后台 Worker 组织对话轮次、调用受工具范围与输出校验约束的 Agent，再通过实时事件返回结果；同时支持人工客服接单和回复。

快速了解项目：[图解项目说明与技术栈](docs/PROJECT_GUIDE.md)。包含整体架构、消息链路、模块分工和源码阅读入口。

课程资料：[Day01–Day11 顺序索引](docs/course-materials/README.md) · [完整合并阅读版](docs/course-materials/COURSE_ALL.md)。全部文档见 [docs 目录](docs/README.md)。

本说明依据当前源码整理。启动步骤尚未进行数据库、Redis 和真实模型的端到端验证；当前项目包含演示数据和未完成模块，不应视为已具备生产部署条件。

## 项目结构

| 目录 | 职责 | 本地端口 |
| --- | --- | --- |
| `backend/ecommerce-service/` | 商品、订单、物流、售后和演示登录 API | 8001 |
| `customer-service/` | 会话、消息、Turn、人工转接、WebSocket 和 Outbox | 8000 |
| `ai-service/` | Agent 创建、技能选择、工具调用、证据校验和 Run 记录 | 8002 |
| `frontend/user-frontend/` | 用户聊天、商品订单与业务操作页面 | 5173 |
| `frontend/admin-frontend/` | 人工客服工作台、指标与 Run 查看 | 5174 |
| `.agents/skills/harness-project-learner/` | 仅适用于本项目的源码学习技能 | — |

`customer-service` 和 `ai-service` 都使用 `atguigu` 包名，必须在各自服务目录、各自 Python 环境中运行，不能把两个服务的包混装进同一环境。

## 核心链路

1. 用户端通过 HTTP 发送消息，Customer Service 保存消息并合并短时间内的连续输入。
2. AI Worker 从 PostgreSQL 领取到期 Turn，携带历史消息和输入版本调用 AI Service。
3. Agent 通过 `load_skill` 选择业务领域，按动态工具范围查询电商 API 或政策知识。
4. 工具结果记录为证据；服务端校验结构化回答、业务事实和页面动作，并对可纠正错误有限重试。
5. Customer Service 检查输入版本，丢弃过期结果，将有效消息和 Outbox 事件在本地事务中保存。
6. Outbox Worker 发布 Redis 事件，WebSocket 转发给前端；转人工时由客服工单流程接管。

Redis 在这里承担实时发布订阅，AI 任务领取来自数据库。WebSocket 推送完成后的业务消息，不是模型 token 流式输出。Agent 给出的取消订单、售后等页面入口，由用户确认后通过前端调用业务写接口执行。

## 本地运行

### 1. 环境和配置

准备 Python 3.12+、uv、Node.js（建议 22.12+，具体以锁定 Vite 的 engines 要求为准）、npm、PostgreSQL 和 Redis。先创建三个 PostgreSQL 数据库，例如 `ecommerce`、`customer_service`、`ai_customer_service`，并为连接用户授予建表和读写权限。

三个 Python 服务分别读取自己的 `.env`。以下是需要设置的关键变量，示例中的占位值应替换为本地配置；已有 `.env` 时只修改需要的字段，不要直接覆盖。

**`backend/ecommerce-service/.env`**（也可参考该目录已有 `.env.example`）：

```dotenv
ECOM_DATABASE_URL=postgresql+psycopg://USER:PASSWORD@127.0.0.1:5432/ecommerce
JWT_SECRET=REPLACE_WITH_SHARED_LOCAL_SECRET
JWT_ALGORITHM=HS256
```

**`customer-service/.env`**：

```dotenv
DATABASE_URL=postgresql+psycopg://USER:PASSWORD@127.0.0.1:5432/customer_service
REDIS_URL=redis://127.0.0.1:6379/0
AI_SERVICE_URL=http://127.0.0.1:8002
INTERNAL_SERVICE_TOKEN=REPLACE_WITH_SHARED_INTERNAL_TOKEN
JWT_SECRET=REPLACE_WITH_SHARED_LOCAL_SECRET
JWT_ALGORITHM=HS256
API_PORT=8000
```

**`ai-service/.env`**：

```dotenv
AI_DATABASE_URL=postgresql+psycopg://USER:PASSWORD@127.0.0.1:5432/ai_customer_service
ECOMMERCE_BASE_URL=http://127.0.0.1:8001/api/v1
INTERNAL_SERVICE_TOKEN=REPLACE_WITH_SHARED_INTERNAL_TOKEN
JWT_SECRET=REPLACE_WITH_SHARED_LOCAL_SECRET
JWT_ALGORITHM=HS256
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=https://YOUR_PROVIDER_HOST/v1
LLM_MODEL=YOUR_MODEL_NAME
LLM_API_KEY=YOUR_API_KEY
API_PORT=8002
```

三个服务的 `JWT_SECRET` 和算法必须一致；Customer Service 与 AI Service 的 `INTERNAL_SERVICE_TOKEN` 必须一致。AI 配置中的 `LLM_BASE_URL` 是必填项。模型需支持当前 Agent 使用的工具调用和结构化输出。

配置定义分别见各服务的 `common/config.py`。部分源码默认数据库/Redis 地址指向局域网地址，应显式设置为自己的地址。

前端默认连接本机的 8000/8001/8002 端口。需要覆盖时，在各前端目录的 `.env` 中配置 `VITE_CUSTOMER_SERVICE_URL`、`VITE_ECOMMERCE_SERVICE_URL`；管理端还支持 `VITE_AI_SERVICE_URL`。`VITE_*` 会暴露给浏览器，不能放入模型密钥或内部服务令牌。

### 2. 安装与初始化

以下 PowerShell 命令均从项目根目录执行。两个 `init_db` 模块负责建表，不负责创建 PostgreSQL 数据库；电商服务启动时会建表并填充演示数据。

```powershell
Push-Location backend/ecommerce-service
uv sync --locked
Pop-Location

Push-Location ai-service
uv sync --locked
uv run python -m atguigu.infrastructure.init_db
Pop-Location

Push-Location customer-service
uv sync --locked
uv run python -m atguigu.infrastucture.init_db
Pop-Location

Push-Location frontend/user-frontend
npm ci
Pop-Location

Push-Location frontend/admin-frontend
npm ci
Pop-Location
```

注意：客服服务的目录名确实是 `infrastucture`，命令中不要自动更正拼写。若 `uv sync --locked` 报锁文件与依赖声明不一致，应先核对差异，再决定是否更新锁文件。

### 3. 启动服务和 Worker

分别打开终端，切换到表中工作目录并运行命令。所有长驻进程都需要保持运行。

| 工作目录 | 命令 |
| --- | --- |
| `backend/ecommerce-service` | `uv run python -m ecommerce_service.server` |
| `ai-service` | `uv run python -m atguigu.main` |
| `customer-service` | 使用下方客服 API 命令 |
| `customer-service` | `uv run python -m atguigu.worker.ai.worker` |
| `customer-service` | `uv run python -m atguigu.worker.realtime` |
| `frontend/user-frontend` | `npm run dev` |
| `frontend/admin-frontend` | `npm run dev` |

客服服务现有 `atguigu/main.py` 引用的是 `app.app:app`，与从服务根目录运行时的包路径不一致。可使用下面的显式入口，复用项目的 SelectorEventLoop 设置，兼容 Windows 下的异步数据库调用：

```powershell
# 在 customer-service 目录执行
uv run python -c 'import uvicorn; from atguigu.common.event_loop import run_async; from atguigu.common.config import get_settings; s=get_settings(); run_async(uvicorn.Server(uvicorn.Config("atguigu.app.app:app", host=s.api_host, port=s.api_port, loop="none")).serve())'
```

前端现有 `start.ps1` 在缺少 `.env` 时会复制 `.env.example`，但当前两个前端目录没有该模板；这里使用 `npm run dev` 启动。

### 4. 检查与停止

- 打开用户端 `http://localhost:5173` 和管理端 `http://localhost:5174`。
- 查看三个 API 的 `/docs`；电商服务另有 `http://127.0.0.1:8001/health`。
- 用演示用户发送订单查询，检查 AI Worker、AI Service、Outbox Worker 日志和前端返回消息，再验证人工转接。
- 各终端按 `Ctrl+C` 停止。`reset-data.ps1` 会重置演示数据，日常启动不需要执行。

## 学习当前项目

专用技能入口：[harness-project-learner/SKILL.md](.agents/skills/harness-project-learner/SKILL.md)。它结合 Day01–Day11 的 14 篇主讲义、3 篇补充文档与当前源码，按小节讲解、逐题检查，并区分课程阶段示例与当前实现。详见 [讲义与源码对应地图](.agents/skills/harness-project-learner/references/course-map.md)。

在此项目中打开新的 Codex 任务，使用：

可自由选择 [路线 A：Day01–Day11 原课顺序](.agents/skills/harness-project-learner/references/course-map.md) 或 [路线 B：先看完整链路再深入机制](.agents/skills/harness-project-learner/references/recommended-outline.md)。支持随时切换，按实际掌握的知识点衔接进度，不必从头重学。

```text
$harness-project-learner 带我从一次用户消息的完整链路开始学习。
$harness-project-learner 继续学习 Turn 与 AgentRun 的区别，每次只问一个问题。
$harness-project-learner 带我读懂 SkillScopeMiddleware 和输出校验。
$harness-project-learner 按 Day01 的文章开始学，一次讲一个小节。
$harness-project-learner 学习 Day04，重点对照讲义与当前 confirm_run 的区别。
$harness-project-learner 按你的推荐路线，从 B01 开始。
$harness-project-learner 切换到课程顺序，已掌握的内容简短回顾即可。
```

若当前会话尚未发现新技能，可直接说“读取当前项目 `.agents/skills/harness-project-learner/SKILL.md`，按它开始学习”。技能只位于当前项目内，不安装到全局目录，也不修改原 project-learner。默认不写学习进度；明确要求保存时使用 `.codex/project-learning.md`，该个人文件已忽略。

## 当前能力边界

- 知识查询目前返回固定政策数据，尚不是完整 RAG；管理前端的知识库/评测页面不代表对应后端已实现。
- 事实校验是有限规则和工具证据匹配，不能保证所有自然语言事实正确。
- confirm/cancel 属于结果发布流程，不能等同于数据库两阶段提交；资源归属和状态迁移仍有改进空间。
- Worker 租约和 Outbox 不等于已经证明恰好一次执行/送达。
- 登录接口使用演示身份签发令牌，尚非生产认证机制。
- 当前目录没有覆盖主链路的完整自动化测试套件；Redis 实验脚本不等于端到端测试。

详细源码分析见 [PROJECT_COMPARISON.md](PROJECT_COMPARISON.md)，其中包含与旧电商客服的对比；学习当前项目时以当前源码为准。

## 版本管理

`.gitignore` 忽略真实 `.env`、虚拟环境、依赖目录、缓存、日志及运行时状态，同时保留配置模板、`uv.lock`、`package-lock.json` 和项目技能。添加此说明时根目录尚未初始化 Git；忽略规则会在使用 Git 后生效，且不会自动移除已被跟踪的文件。
