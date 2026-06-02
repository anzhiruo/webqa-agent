# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

团队共享的 webqa-agent 项目指南。个人偏好和本地配置请放在 `CLAUDE.local.md`（git 不跟踪）。

______________________________________________________________________

## 项目概览

**WebQA Agent** 是一个自主式 Web 浏览器代理，用自然语言目标驱动浏览器完成功能、UX、性能、安全测试。**v0.3.2**（分支 `main`），Python `>=3.11`。

### 三种执行模式（按 `engine` + `gen|run` 子命令组合）

| 模式             | 子命令  | `engine` 配置       | 驱动方式                                | 适用场景                                |
| :--------------- | :------ | :------------------ | :-------------------------------------- | :-------------------------------------- |
| **Flash**（默认）| `gen`   | `flash`（默认即可） | chrome-devtools MCP → Chrome            | 一句话目标、秒级反馈、IDE/MCP 内联调用 |
| **Standard Gen** | `gen`   | `standard`          | Playwright + LangGraph 工作流           | AI 自主探索、深度回归                   |
| **Run**          | `run`   | `standard`（强制）  | Playwright + 顺序执行 YAML 用例         | 可重复回归、CI 固定用例                 |

> Run 模式只支持 `engine: standard`；Flash 是 `gen` 专属。
> 模式细节与配置项以 [`docs/MODES&CLI.md`](docs/MODES&CLI.md) 为准，本文件只描述代码架构。

______________________________________________________________________

## 常用命令

```bash
# 安装与启动
uv sync                                          # 同步依赖（Python>=3.11）
uv run playwright install chromium               # Standard/Run 模式需要
npm install -g chrome-devtools-mcp@latest        # Flash 模式需要（Node 20.19+ LTS）

# CLI
webqa-agent init                                 # 生成 config.yaml 模板
webqa-agent init -m run                          # 生成 Run 模式模板
webqa-agent gen                                  # 默认 Flash 引擎运行
webqa-agent gen -c <config> -w <workers>         # 指定配置 + 并发
webqa-agent run -c <config-or-dir>               # Run 模式（YAML 用例）
webqa-mcp-server                                 # 启动 FastMCP 服务（暴露给 Cursor/Claude Code）

# 测试
uv run pytest tests/                             # 全部测试（含 Flash 子套件 webqa_agent/executor/flash/tests）
uv run pytest tests/test_action_executor.py -v   # 单文件
uv run pytest tests/ -k "cc_mini"                # 仅跑 Flash 相关
uv run pytest tests/test_crawler.py --url https://example.com  # 自定义目标 URL

# 代码质量（统一走 pre-commit）
pre-commit install                               # 一次性安装钩子
pre-commit run --files <files>                   # 检查指定文件
pre-commit run --all-files                       # 全量检查

# 全栈平台（Docker Compose）
cd deploy/docker-compose && cp .env.example .env && ./start.sh
```

______________________________________________________________________

## 架构总览

### CLI 入口

`webqa_agent/cli.py:main()` 解析参数后分发到 `execute_gen_mode()` 或 `execute_run_mode()`：

- `execute_gen_mode()` 读取 `engine` 字段决定走 Flash（`_load_cc_mini_runner` → `executor/flash/runner.py`）或 Standard（`GenExecutor`）。
- `execute_run_mode()` 直接走 `RunExecutor`。

`webqa-mcp-server` 入口在 `webqa_agent/mcp_server/server.py:main()`（FastMCP）。

### 包结构（重点模块）

```
webqa_agent/
├── cli.py                  # CLI 入口、engine 分发
├── config_models/          # Pydantic V2: BrowserConfig / LLMConfig / GenConfig / RunConfig
├── executor/
│   ├── gen_executor.py     # Standard Gen 模式编排
│   ├── run_executor.py     # Run 模式编排
│   ├── flash_executor.py   # Flash → Standard 报告桥接
│   ├── flash_report_adapter.py
│   ├── result_aggregator.py
│   ├── gen/                # Standard Gen 的 LangGraph 工作流
│   │   ├── graph.py        # 工作流主图
│   │   ├── agents/         # 执行 agent（execute_agent.py）
│   │   ├── state/          # 状态 schema
│   │   └── utils/          # CaseRecorder / MessageConverter
│   ├── run/                # Run 模式 case 执行
│   └── flash/              # ⚡ Flash 引擎（独立子包，对应 cc-mini）
│       ├── runner.py       # 入口 run_cc_mini()
│       ├── core/           # engine.py / mcp_client.py / llm.py / tool.py / skill_registry.py
│       ├── skills/         # 内置渐进式 skill：plan / ui-audit / recovery / nuclei-scan / button-check
│       ├── tools/          # 可选工具：upload / download / nuclei / verify / wait_stable / load_skill
│       ├── features/       # 报告渲染等确定性工具
│       └── tests/          # Flash 子套件（pytest 已纳入）
├── browser/                # session.py (BrowserSessionPool) / account_pool.py / context_manager.py
├── actions/                # action_executor / action_handler / click_handler
├── tools/                  # 默认工具 (action/ux/verify) + custom + core + registry.py
├── llm/llm_api.py          # 多 provider LLMAPI（Claude/OpenAI/Gemini 自动判别）
├── prompts/                # test_planning / agent_execution / ui_automation
├── crawler/                # 站点抓取
├── mcp_server/             # FastMCP 服务（暴露给 Cursor/Claude Code）
│   ├── server.py           # 入口 & 工具注册（businesses/executions/files/testing）
│   ├── client.py           # WebQAClient（调用全栈平台 REST API）
│   ├── task_manager.py
│   └── tools/
├── templates/              # config.yaml.example 等
└── utils/                  # 通用工具（find_config_file/load_cookies/test_file_library 等）

backend/                    # 全栈 Web Dashboard 后端（FastAPI + Alembic + PostgreSQL + Redis）
frontend/                   # 全栈 Web Dashboard 前端（Vite + React + nginx）
deploy/                     # docker-compose / k8s 部署清单
skills/webqa/               # OpenClaw / Claude Code Skill 包（SKILL.md + references/）
config/                     # config.yaml.example / config_run.yaml.example
docs/                       # 用户文档（详见末尾）
```

### 关键文件入口

| 角色                  | 位置                                                          |
| --------------------- | ------------------------------------------------------------- |
| CLI 主入口            | `webqa_agent/cli.py:main()`                                   |
| Flash 引擎入口        | `webqa_agent/executor/flash/runner.py:run_cc_mini()`          |
| Standard Gen 工作流   | `webqa_agent/executor/gen/graph.py`                           |
| Run 用例编排          | `webqa_agent/executor/run_executor.py`                        |
| 浏览器会话池          | `webqa_agent/browser/session.py:BrowserSessionPool`           |
| 多 provider LLM       | `webqa_agent/llm/llm_api.py:LLMAPI`                           |
| 工具注册中心          | `webqa_agent/tools/registry.py`                               |
| MCP Server 入口       | `webqa_agent/mcp_server/server.py:main`                       |
| Pydantic 配置模型     | `webqa_agent/config_models/` (`base_config.py`, `gen_config.py`, `run_config.py`) |
| 配置模板              | `config/config.yaml.example`、`config/config_run.yaml.example`|

### 配置 → 执行 数据流

```
config.yaml → cli.py validate_and_build_llm_config()
            → engine 分发
              ├─ flash:     runner.py → core/engine.py → MCP(chrome-devtools) → LLM
              └─ standard:  GenConfig/RunConfig → GenExecutor/RunExecutor
                              → executor/gen/graph.py (LangGraph)  或  CaseExecutor
                              → BrowserSessionPool.acquire() → tools/* → LLMAPI
```

______________________________________________________________________

## Flash 引擎要点（v0.3 默认）

- **驱动方式**：通过 `chrome-devtools-mcp`（stdio MCP）控制本机 Chrome；不走 Playwright。
- **`business_objectives`**：可为字符串（单任务）**或字符串列表**（并发批量），空白条目会被过滤。
- **并发**：CLI `-w` > `target.max_concurrent_tests` > 默认；单任务强制串行。
- **渐进式 Skill 加载**：启动只注入 skill 摘要，命中时再加载详细指令以控制 token；内置 5 个 skill（`plan` / `ui-audit` / `recovery` / `nuclei-scan` / `button-check`），位于 `executor/flash/skills/`，每个目录遵循 `SKILL.md` + 可选 `scripts/` `resources/`。
- **Cookie / 账号注入**：CLI 解析 `accounts:` 顶层配置或 `browser_config.cookies` 兜底，转成 cc-mini extensions；运行期可通过 `switch_account` 切换。
- **测试文件池**：`test_config.test_files_dir`（+ 可选 `test_config.test_files` 白名单）→ `utils/test_file_library.py:TestFileLibrary` → 目录索引注入到 system prompt，agent 通过 `mcp__browser__upload_file` 自主上传。
- **报告路径**：`reports/` 根目录；`save_screenshots` / `save_dataflow` 由 `report:` 段控制。
- **LLM provider**：仅支持 `anthropic` / `openai`；Gemini 走 OpenAI 兼容模式。Anthropic 用户若没显式 `base_url`，CLI 会清除默认的 OpenAI URL（否则会破坏 Anthropic SDK 请求）。
- **Provider 默认 temperature**：OpenAI=0.1，Anthropic/Gemini=1.0。**Extended Thinking 时 `temperature` 强制为 1.0，`max_tokens` 必须 > `budget_tokens`**（系统自动校正但建议合理配置；推荐 `effort: medium` 对应 `max_tokens: 20000-25000`）。

______________________________________________________________________

## Standard 引擎要点

仅当 `engine: standard` 时启用。

### 单 Tab 架构（仅 Standard 模式）

> ⚠️ 这套分层协调机制是 Standard Gen 模式 UI Agent / UX Test 的实现细节；**Flash 模式不适用**，由 chrome-devtools MCP 自行管理 Tab。

- 所有 AI 测试在单一浏览器 Tab 中完成；多 Tab 不支持。
- 分层协调（防止 95%+ 的新 Tab，零冲突）：
  - **Layer 0（基础）** `browser/session.py` — 通过 `add_init_script()` 做 context 级 DOM 预处理和事件监听。
  - **Layer 1（增强）** `actions/action_handler.py` — 点击级增强（历史记录、周期检查、表单处理）。
  - **Layer 2（监控）** `actions/click_handler.py` — 执行监控与结果跟踪。
- **协调机制**：全局 flag 防止重复；session.py 优先，action_handler.py 增强。
- **导航**：`GoBack`（浏览器历史）和 `GoToPage`（直跳 URL），返回 `True`/`False` 表示是否成功；普通 `<a target="_blank">` 链接被改写在当前 Tab 内打开。
- **测试模式**：Click → Verify → GoBack。
- **Basic Test（默认非 AI 模式）允许多 Tab**。

### Browser Session 管理

- 全程使用 `BrowserSessionPool`：`pool.acquire()` / `pool.release()`，不使用单例。
- 每 session 加锁防止竞态；session 创建是 pool 私有特权。
- 历史 `Driver.getInstance()` / `driver.page` 均已迁移到 `pool.acquire()` / `session.page`。

### 工具系统（仅 Standard）

- **默认工具**（始终启用）：`tools/action_tool.py`、`tools/ux_tool.py`、`tools/verify_tool.py`
- **自定义工具**（可选，通过 `test_config.custom_tools.enabled` 开启）：`lighthouse`、`nuclei`、`traverse_clickable_elements`、`detect_dynamic_links`
- **核心实现**：`tools/core/` (`ui_driver.py`、`web_checks.py`、`lighthouse.py`)
- **基类**：`tools/base.py:WebQABaseTool`（扩展点）
- **注册中心**：`tools/registry.py` 负责依赖检测和过滤
- 扩展指南：[docs/CUSTOM_TOOL_DEVELOPMENT.md](docs/CUSTOM_TOOL_DEVELOPMENT.md) / [AI 版](docs/CUSTOM_TOOL_DEVELOPMENT_AI.md)

______________________________________________________________________

## 错误处理（统一 Tag 系统）

工具回复必须使用以下 tag 之一，executor / recovery 链据此决策：

- `[SUCCESS]` — 成功
- `[FAILURE:root_cause]` — 可恢复失败
- `[CRITICAL_ERROR:root_cause]` — 不可恢复，必须中止
- `[WARNING]` — 非阻塞问题
- `[CANNOT_VERIFY]` — assertion 前置条件未满足

**失败分类**：`ELEMENT_NOT_FOUND` / `NAVIGATION_FAILED` / `PERMISSION_DENIED` / `PAGE_CRASHED` / `NETWORK_ERROR` / `SESSION_EXPIRED` / `UNSUPPORTED_PAGE` / `VALIDATION_ERROR`。

**自适应恢复**（`dynamic_step_generation.enabled = true` 时）：

- `ELEMENT_NOT_FOUND` 双层恢复（retry → LLM replanning）
- 其他失败由 LLM 驱动恢复（GoBack / timeout / permission 等）
- **循环检测**：同一错误模式重复 2+ 次直接中止
- 策略：`retry_modified` / `skip` / `abort`

**自动处理**：JS 对话框（alert/confirm/prompt）自动接受；critical error 自动中止以节省资源。

______________________________________________________________________

## 研究和规划（CRITICAL）

**对于复杂任务或涉及第三方依赖,必须先做调研**：

1. **Context7 MCP 工具** — `resolve-library-id` + `query-docs` 获取最新官方文档。
2. **WebSearch** — 查找最佳实践和常见陷阱。
3. **调研场景**：集成新三方库（Playwright/LangChain/FastAPI/chrome-devtools-mcp 等）、复杂异步/并发、不熟悉的 Python 特性、复杂工具链、安全相关功能。
4. **流程**：识别难点 → 拉文档 → 查最佳实践 → 制定计划 → 实现。

❌ 不要想当然认为知道某个库的用法；不要跳过调研直接写代码；不要基于旧版本知识实现。

______________________________________________________________________

## 代码质量要求（CRITICAL）

### 规范性

- **类型注解**：严格模式，所有函数必须有类型注解
- **错误处理**：显式 try-except + 日志
- **日志级别**：生产 `info`，调试 `debug`

### 去冗余

- ❌ 不要创建重复的类/方法/变量；不要复制粘贴；不要保留废弃代码（Git 有历史）
- ✅ 重构时整合相似功能；用继承/组合减少重复

### 命名

- 类 `PascalCase`、函数/方法 `snake_case`、常量 `UPPER_SNAKE_CASE`、私有 `_leading_underscore`
- 避免模糊命名（`temp` / `tmp` / `data`）

详细规范见 `.claude/rules/python-quality.md`（如已建立）。

______________________________________________________________________

## 兼容性和稳定性（CRITICAL）

避免非必要的 breaking changes：

- ❌ 不要随意改公共 API 签名 / 改已有行为 / 删正在使用的代码
- ✅ 新功能向后兼容；废弃用 deprecation 警告而非直接删；API 变更需迁移指南
- **新特性设计**：零学习成本优先、默认可选、渐进增强、文档完善
- **重构前 checklist**：是否影响公共 API？是否改变用户可见行为？是否需要改配置？是否更新文档？是否测了兼容性？上下游是否要适配？
- **兼容性策略**：配置/API/数据/行为四个维度都保留旧路径，新路径作为增强
- **何时允许 breaking**：major 版本（v0.2→v0.3）、安全漏洞、数据损坏、长期 deprecated 项

______________________________________________________________________

## 代码审查清单（团队标准）

所有代码提交前必须通过：

- [ ] 类型注解
- [ ] 错误处理 + 日志
- [ ] 测试编写/更新并通过
- [ ] 文档更新
- [ ] 清理调试 print / 注释代码
- [ ] 无重复实现
- [ ] 向后兼容
- [ ] 配置变更可选 + 有文档
- [ ] 所有 pre-commit hooks 通过（flake8/isort/codespell/pylint/gitleaks/mdformat 等，配置见 `.pre-commit-config.yaml`）

______________________________________________________________________

## 测试

- `tests/conftest.py` — 共享 fixtures（支持 `--url` 覆盖）
- `tests/mocks/` — JSON mock 数据
- `tests/test_pages/` — 本地 HTML 测试页
- `tests/test_cc_mini_*.py` + `webqa_agent/executor/flash/tests/` — Flash 引擎测试（已纳入 `pytest.ini_options.testpaths`）
- `tests/test_mcp_server/` — MCP Server 测试

```bash
uv run pytest tests/ -v -l                       # 详细 + 局部变量
uv run pytest tests/ --cov=webqa_agent           # 覆盖率
uv run pytest tests/ -s                          # 显示 print
```

______________________________________________________________________

## MCP Server（暴露 WebQA 给 IDE）

`webqa-mcp-server` 命令启动 FastMCP 服务（`webqa_agent/mcp_server/server.py`），让 Cursor / Claude Code 通过自然语言触发 WebQA 测试。

- **环境变量**：`WEBQA_API_URL`（全栈平台地址）、`WEBQA_API_KEY`（平台 API Key）
- **工作流**：`run_test` → 每 10s 轮询 `get_test_status` → `get_test_report`（典型耗时 2–10 min）
- **工具模块**：`mcp_server/tools/` 下 `businesses` / `executions` / `files` / `testing`
- **客户端**：`WebQAClient` (`mcp_server/client.py`) 调全栈平台 REST API
- 配置参考：[docs/MCP_SERVER.md](docs/MCP_SERVER.md)

______________________________________________________________________

## 全栈 Web 平台（可选）

为团队提供持久化 Dashboard、测试管理、调度、历史：

- `backend/` — FastAPI + Alembic + PostgreSQL + Redis（架构见 `backend/BACKEND_ARCHITECTURE.md`）
- `frontend/` — Vite + React + nginx
- `deploy/docker-compose/` — 一键 `./start.sh`
- `deploy/k8s/` — 生产 K8s

部署详见 [deploy/README.md](deploy/README.md)。

> 平台目前仅有中文界面。

______________________________________________________________________

## 输出目录

| 路径                            | 说明                       |
| ------------------------------- | -------------------------- |
| `reports/`                      | HTML 测试报告（根目录）    |
| `logs/`                         | 应用日志与 trace（根目录） |
| `webqa_agent/logs/`             | 包级日志                   |
| `webqa_agent/reports/`          | 包级报告                   |
| `tests/actions_test_results/`   | 测试执行产物 + 截图        |
| `tests/crawler_test_results/`   | Crawler 截图               |

______________________________________________________________________

## 用户文档（`docs/`）

- [MODES&CLI.md](docs/MODES&CLI.md) — 模式与 CLI 权威参考（中文版 `MODES&CLI_zh-CN.md`）
- [MCP_SERVER.md](docs/MCP_SERVER.md) — MCP Server 工具参考
- [CUSTOM_TOOL_DEVELOPMENT.md](docs/CUSTOM_TOOL_DEVELOPMENT.md) — 自定义工具开发（Standard 模式）
- [CUSTOM_TOOL_DEVELOPMENT_AI.md](docs/CUSTOM_TOOL_DEVELOPMENT_AI.md) — AI 增强工具
- `webqa_agent/executor/flash/README.md` — Flash 引擎细节
- `webqa_agent/executor/flash/skills/README.md` — Flash skill 编写规范
- `skills/webqa/SKILL.md` — OpenClaw / Claude Code Skill 包

______________________________________________________________________

## 快速故障排查

```bash
# Playwright 未安装
uv run playwright install chromium

# Flash 模式 chrome-devtools-mcp 缺失
npm install -g chrome-devtools-mcp@latest

# API Key
export OPENAI_API_KEY="..."         # 或 ANTHROPIC_API_KEY / GEMINI_API_KEY

# 配置定位
webqa-agent init                                  # 生成模板
webqa-agent run -c /path/to/config.yaml           # 指定路径

# 开启 debug 日志
# config.yaml:
#   log:
#     level: debug
```
