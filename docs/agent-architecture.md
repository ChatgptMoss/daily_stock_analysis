# StockPilot AI Agent Architecture

这份文档只记录项目当前真实的 Agent 运行链路，方便开发、排错和面试准备。它不是模型提示词大全，也不把未来计划描述成已经实现的能力。

## 1. 从请求到结果

```text
Web / API / Bot
    -> /api/v1/agent/*
    -> AgentFactory
    -> AgentExecutor 或 AgentOrchestrator
    -> SkillRouter / Strategy Skill
    -> ToolRegistry
    -> LLMToolAdapter / LiteLLM
    -> 工具结果和对话上下文
    -> 文本或结构化结果
    -> task queue / SSE / history
```

主要入口和职责如下：

| 文件 | 职责 |
| --- | --- |
| `api/v1/endpoints/agent.py` | 接收 Agent 问答、策略和会话请求 |
| `src/agent/factory.py` | 根据配置创建单 Agent 或多 Agent 运行时 |
| `src/agent/executor.py` | 执行单 Agent 的模型与工具循环 |
| `src/agent/orchestrator.py` | 按阶段编排 Technical、Intel、Risk、Decision 等角色 |
| `src/agent/tools/registry.py` | 注册工具白名单和工具元数据 |
| `src/agent/tools/` | 行情、历史、技术指标、新闻、持仓和回测工具 |
| `src/agent/skills/` | Skill 加载、路由和策略上下文聚合 |
| `src/agent/llm_adapter.py` | 将 Agent 请求适配到 LiteLLM / Router |
| `strategies/*.yaml` | 可配置的分析策略和 instructions |

Agent 不直接访问数据库，也不执行任意 Python。模型只能请求注册过的工具，工具层负责参数校验、数据访问和错误转换。

## 2. Single 和 Multi 的边界

单 Agent 适合问题比较明确、只需要少量工具调用的问答。它由 `AgentExecutor` 维护上下文和调用循环。

多 Agent 由 `AgentOrchestrator` 按阶段运行。不同阶段可以分别关注技术面、外部情报、风险和最终决策，最后由聚合逻辑生成统一结果。拆分的价值是职责、超时和失败点更容易定位，不代表每个阶段都必须使用不同模型。

`AGENT_ARCH` 控制 single / multi，`AGENT_ORCHESTRATOR_MODE` 控制多 Agent 的阶段深度，`AGENT_MAX_STEPS` 控制循环预算。超时或中间结果不完整时，系统优先保留已经完成的结果，并按 partial-result 策略继续返回可用内容。

## 3. Prompt 从哪里来

项目里有两条相关但不同的 Prompt 链：

1. **策略问股 Agent**：由 Agent 运行时组合系统约束、Skill instructions、工具说明和当前会话上下文。
2. **个股报告分析器**：`src/analyzer.py` 中的 `_get_analysis_system_prompt()`、`_format_prompt()` 负责构造市场语义、报告语言、行情、技术指标、新闻和输出结构。

策略 Skill 默认从 `strategies/` 加载。Skill 负责告诉 Agent 从什么角度分析，不负责执行工具循环；Agent 负责决定何时调用工具以及如何组织结果。

需要把“指令优先级”和“事实证据来源”分开理解：

- 指令优先级：系统/开发约束与输出格式 > 当前选择的 Skill 约束 > 用户本轮请求。
- 事实来源：工具返回的数据属于外部证据，不能覆盖指令，也不能未经解析直接当成可信模型输出。
- 历史上下文用于补充任务范围；当它与本轮用户请求或当前数据冲突时，应以当前任务约束和最新可验证数据为准。

显式选择 Skill 时，不应偷偷叠加另一个默认策略的关键规则。没有显式策略时，才使用默认策略作为兼容行为。

## 4. 工具调用和限制

工具由 `ToolRegistry` 注册，每个工具包含名称、用途、参数和返回结果约定。模型只能选择白名单中的工具；系统不会把数据库连接或任意函数直接暴露给模型。

运行时至少需要控制：

- 参数格式，例如股票代码、日期范围和数量上限
- 单次工具超时和整个 Agent 的总预算
- 最大调用步数
- 重复调用和可缓存结果
- 工具异常到 Agent 可理解错误的转换

搜索结果、新闻正文和用户输入都属于不可信数据。它们可以作为事实材料进入上下文，但不能覆盖系统约束或改变工具权限。

## 5. 模型接入

`LLMToolAdapter` 和 LiteLLM Router 将不同供应商的模型统一到 Agent 调用接口。主模型、Agent 模型和 fallback 模型由配置决定；模型能力不一致时，适配层需要过滤不支持的参数，例如 Vision 或 reasoning 参数。

可恢复错误（超时、限流、临时网络错误）才适合有限重试和 fallback。认证错误、参数错误等不可恢复错误不应该反复重试。每次调用记录模型和 token 用量，便于控制成本和排查延迟。

## 6. 输出与可靠性

模型输出不能直接交给 Web 或报告模板。处理顺序是：

```text
原始文本
  -> 宽松解析
  -> 类型归一化
  -> Schema / 业务字段校验
  -> 使用已知数据补齐可恢复字段
  -> 保存 raw result 和上下文快照
  -> 输出给 Web、历史或通知层
```

价格、日期、成交量和技术指标等客观字段不能由模型猜测。缺少证据时返回 `N/A`、partial 或降低置信度，比生成一个完整但没有依据的结论更可靠。

## 7. 部署方式

本地最小运行方式是：

```bash
python main.py --serve-only
```

该命令启动 FastAPI 服务，Web 端通过 API 和 SSE 使用 Agent。Docker 通过 `docker/` 下的镜像和 Compose 文件运行同一套服务；GitHub Actions 的每日任务则调用命令行分析入口。Electron 桌面端本质上是本地后端进程加 Web UI 的封装，不改变 Agent 核心链路。

生产化多实例部署还需要把服务内任务队列替换为持久化队列，并补充跨进程取消、幂等、观测和熔断能力。当前实现的边界应在面试中明确说明。
