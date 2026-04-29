# hermes-agent 架构速读

[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 出的自迭代 agent 框架(Python,MIT,123k★)。

> 基于 commit [`6e9691f`](https://github.com/NousResearch/hermes-agent/commit/6e9691ff12605cee570b7543138b9cec2950dba6) (2026-04-29)。

## 入口(三个,都汇到同一个 agent core)

- **`cli.py`** — 终端 TUI
- **`gateway/run.py`** — 长跑 messaging 进程,18 个平台 adapter(Telegram/Discord/Slack/WhatsApp/Signal/Email/Matrix/Feishu/QQ/微信/WeCom/Yuanbao/Home Assistant/...全在 `gateway/platforms/`)
- **`acp_adapter/`** — 给 IDE 用的 Agent Client Protocol 适配

## 核心(`run_agent.py` 一个大文件)

`run_agent.py` 单文件 **694 KB**(约 1 万行),里面是 `AIAgent` 类,负责 provider 选型、prompt 拼装、tool 调度循环、retry、持久化。同步式 orchestration,不是 graph/DAG 那种风格。

围绕它的 `agent/` 目录拆成职能模块:

- `prompt_builder.py` + `prompt_caching.py` — 拼 system prompt(personality + memory + skills + context files),应用 Anthropic cache breakpoints
- `context_compressor.py` / `context_engine.py` / `context_references.py` — 上下文压缩、引用
- `transports/` — provider 抽象(anthropic / bedrock / chat_completions / codex)
- 一堆 adapter:`anthropic_adapter / bedrock_adapter / gemini_*_adapter / codex_responses_adapter / google_code_assist`
- `credential_pool.py` + `credential_sources.py` — 多账号轮转
- `memory_manager.py` / `memory_provider.py` — agent-curated memory
- `trajectory.py` — 训练数据导出

## 工具系统

- **`tools/registry.py`** 中心化注册,84 个文件 / 47 个 tool / 19 个 toolset
- **6 个 terminal backend**(在 `tools/environments/`):`local / docker / ssh / daytona / singularity / modal` — 这就是 README 说的 "$5 VPS 到 GPU 集群" 的来源,Daytona/Modal 是 serverless 休眠
- 各种特化 tool:`browser_*`(camofox + browserbase + firecrawl 三供应商)、`mcp_tool`(MCP 接入)、`delegate_tool`(子 agent 并行)、`code_execution_tool`、`memory_tool`、`cronjob_tools`、`image_generation_tool`、`neutts_synth`(语音)

## 持久化与状态

- **`hermes_state.py`** — SQLite + **FTS5** 全文检索,负责 session、状态、跨会话搜索
- 平台隔离 + lineage 追踪(同一对话跨平台续聊)

## 调度与扩展

- **`cron/`** — 内置 cron 调度器,job 触发时**起一个全新 AIAgent 实例**(skill 注入 + 跑完投递),不复用主进程上下文
- **`plugins/`** — 三种 discovery 来源(tools / hooks / CLI commands)
- **`skills/`** + `optional-skills/` + `agent/skill_*.py` — agentskills.io 标准,可在使用中自我改进
- **`environments/`** — RL 训练环境(`hermes_base_env.py`、`agent_loop.py`、Atropos benchmarks: tblite / terminalbench_2 / yc_bench / hermes_swe_env)
- **`tui_gateway/`** + **`ui-tui/`** — TUI 实现拆分
- **`web/`** + **`website/`** — docs 站点

## 一句话总结

**单大类 orchestrator (`AIAgent` in `run_agent.py`) + 注册表式 tool/transport/platform 插件 + SQLite-FTS5 状态**。卖点是把"messaging 多平台前端"、"6 种 terminal 后端(含 serverless)"、"自我演化的 skill/memory" 这三件事塞在同一个进程模型里。架构上比较"经典 monolith agent"——不是 LangGraph/CrewAI 那种 DAG 编排,而是一个大循环调一堆注册过的工具,靠目录拆分维持可读性。
