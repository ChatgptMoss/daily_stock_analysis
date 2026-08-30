# Daily Stock Analysis

一个面向个人自选股复盘的 AI 分析工作台。项目把行情与新闻检索、确定性技术指标、LLM 分析、Agent 工具调用、结构化报告和异步任务流串成一条可回放的链路。

这个仓库用于学习和实践 AI 应用工程。它基于 [ZhuLinsen/daily_stock_analysis](https://github.com/ZhuLinsen/daily_stock_analysis) 进行整理和改造，保留原项目的 MIT 许可证与来源信息。下面的说明重点放在 Agent、Prompt、模型接入、数据 fallback 和任务可靠性等模块。

## 功能概览

- **实时数据检索**：统一接入行情、新闻、基本面等外部数据，并处理超时、字段缺失和数据源 fallback。
- **Agent 分析**：根据用户问题选择策略 Skill 和白名单工具，支持单 Agent 与多 Agent 编排。
- **Prompt 分层**：系统约束、市场语义、策略 Skill 和动态数据上下文分别构建，避免默认策略污染显式选择。
- **结构化输出**：LLM 结果经过宽松解析、字段归一化和业务校验后，再交给报告和 Web 层。
- **异步任务**：长时间分析通过后台任务执行，Web 端使用 SSE 接收进度，最终结果以任务状态和历史记录为准。
- **可回放**：保存分析上下文、原始输出、模型用量和结果，便于排查和评估。

## 使用方式

### 命令行分析

适合定时任务、批量分析和本地排查：

```bash
python main.py                       # 按配置执行日常分析
python main.py --stocks 600519,000858 # 临时指定股票
python main.py --dry-run             # 只检查流程，不发送通知
python main.py --market-review       # 生成市场复盘
python main.py --schedule            # 启动定时调度
```

### Web 工作台

```bash
python main.py --serve-only
```

访问 `http://127.0.0.1:8000`，可以提交分析任务、查看异步进度、进行策略问答、浏览历史报告、运行回测和维护自选股。自选股输入支持代码、名称和拼音匹配，设置页还支持 CSV/Excel 导入；图片识别需要配置 Vision 模型。前端源码位于 `apps/dsa-web/`，生产构建使用：

```bash
cd apps/dsa-web
npm ci
npm run build
```

### 策略问答

Agent 支持围绕股票代码和策略进行多轮提问。策略可以通过 `AGENT_SKILLS` 选择，YAML 配置位于 `strategies/`；没有显式选择时使用默认策略兼容旧调用。长任务通过任务 ID 和 SSE 返回进度，页面切换不会中断后端任务。

### 其他入口

- FastAPI 服务：`uvicorn server:app --reload --host 0.0.0.0 --port 8000`
- Bot：通过 `bot/commands/` 下的命令接入聊天平台，复用同一套分析和历史服务。
- 通知：在环境变量中配置通知渠道后，命令行、Web 和 Bot 可以复用统一的发送服务。
- Docker / GitHub Actions：复用 `docker/` 和 `.github/workflows/` 中的服务、定时分析与发布流程。

## Agent 调用链

```text
Web / API / Bot
    -> api/v1/endpoints/agent.py
    -> src/agent/factory.py
    -> AgentExecutor 或 AgentOrchestrator
    -> ToolRegistry 中的白名单工具
    -> LLMToolAdapter / LiteLLM Router
    -> Prompt + Skill + 外部数据上下文
    -> 结构化结果、SSE 事件和历史记录
```

实现细节见 [Agent 架构说明](docs/agent-architecture.md)。

## 核心实现拆解

### Agent 运行时

`api/v1/endpoints/agent.py` 接收请求，`src/agent/factory.py` 根据配置创建 `AgentExecutor` 或 `AgentOrchestrator`。单 Agent 维护一次对话中的上下文和工具循环；多 Agent 按 Technical、Intel、Risk、Portfolio、Decision 等阶段聚合结果。`AGENT_MAX_STEPS` 和超时预算限制循环成本，部分阶段失败时保留已完成结果并返回最小可用摘要。

### Prompt 与 Skill

Agent Prompt 由系统约束、Skill instructions、工具说明、当前用户请求和会话历史组合而成；股票报告链路则由 `src/analyzer.py` 独立构造市场语义、行情、指标、新闻和输出格式。`src/agent/skills/` 负责加载和路由 Skill，`strategies/*.yaml` 只描述分析角度，不负责执行工具循环。这样可以避免显式选择策略时被默认策略的关键规则污染。

### 工具白名单与数据 fallback

`src/agent/tools/registry.py` 是工具的统一注册入口，负责参数 Schema、工具元数据和可用工具集合；行情、K 线、技术指标、新闻、基本面和回测能力分别位于 `src/agent/tools/`。数据源适配集中在 `data_provider/`，调用失败时按优先级、缓存或降级 payload 返回状态，不让单个免费数据源的限流拖垮整条分析链路。

### 结构化报告与历史

模型原始文本先经过宽松 JSON 解析、类型归一化和 `src/schemas/report_schema.py` 校验，再交给 `src/services/report_renderer.py` 和 Web 层。价格、日期、成交量等客观字段优先取代码或数据源结果；缺失证据时保留 `N/A` 或 partial 状态。`src/repositories/analysis_repo.py` 和 `src/services/history_service.py` 保存原始输出、上下文快照和可回放记录。

### 异步任务与流式进度

`src/services/task_service.py`、`src/services/task_queue.py` 负责后台任务状态和去重，API 返回 `task_id` 后由前端 `apps/dsa-web/src/hooks/useTaskStream.ts` 订阅 SSE。任务状态、进度事件和最终报告分开处理，便于页面刷新、状态恢复和历史回看。

### 模型适配

`src/agent/llm_adapter.py` 和 LiteLLM Router 把不同供应商统一到 Agent 调用接口。主模型、Agent 模型和 fallback 模型由环境变量决定；适配层过滤不兼容参数，并区分超时/限流等可恢复错误与认证/参数错误，避免无意义重试。

## 界面预览

![Web 工作台示意](sources/fastapi_server.png)

此截图用于展示当前 Web 工作台界面。

## 目录速览

| 目录 | 职责 |
| --- | --- |
| `src/agent/` | Agent 执行器、编排器、工具、Skill 和模型适配 |
| `src/core/` | 分析 Pipeline、任务调度、回测和配置管理 |
| `src/services/` | 分析、历史、任务队列、报告和系统配置服务 |
| `data_provider/` | 行情和基本面数据源适配、标准化与 fallback |
| `api/` | FastAPI 应用、认证、Agent 和分析接口 |
| `apps/dsa-web/` | React + TypeScript Web 工作台 |
| `strategies/` | 可配置的策略 Skill YAML |
| `templates/` | Markdown / 通知报告模板 |
| `tests/` | 后端回归测试 |

## 环境配置

Python 3.10+ 是最低要求。先安装依赖并准备环境变量：

```bash
pip install -r requirements.txt
cp .env.example .env
```

至少配置一个 LLM 和一组股票代码，例如：

```env
OPENAI_API_KEY=your-key
OPENAI_BASE_URL=https://api.deepseek.com/v1
OPENAI_MODEL=deepseek-chat
STOCK_LIST=600519,000858
```

## Agent 配置

```env
AGENT_ARCH=single
AGENT_SKILLS=ma_golden_cross
AGENT_MAX_STEPS=10
```

常用配置含义：

- `AGENT_ARCH=single` 使用单 Agent；`multi` 使用编排器。
- `AGENT_SKILLS` 指定策略 Skill，策略文件默认位于 `strategies/`。
- `AGENT_MAX_STEPS` 限制工具调用和推理循环，防止任务无限运行。
- `AGENT_LITELLM_MODEL` 可为 Agent 单独指定模型，否则继承主模型。

模型配置和部署入口以 `.env.example`、`docker/` 及工作流文件为准；本 README 只保留本地最小运行路径。

## 可靠性边界

模型不是事实源。行情、时间、价格和技术指标由代码和外部数据源提供，LLM 负责解释和组织结论。系统会对外部调用设置超时，对非核心数据采用 fail-open，对模型输出进行 Schema 和字段校验，并保留原始结果用于排查。

当前项目仍有明确限制：免费数据源可能限流，服务内任务队列不适合多实例部署，新闻检索属于实时上下文增强而不是完整的向量数据库 RAG，历史回测也不能证明实盘收益。

## 验证

后端修改后运行：

```bash
python -m py_compile main.py server.py
python -m pytest -m "not network"
```

完整门禁：

```bash
./scripts/ci_gate.sh
```

## License

[MIT License](LICENSE)。本项目基于开源项目进行学习和改造，使用或二次开发时请保留许可证和来源说明。
