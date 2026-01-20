# RD-Agent 项目基本架构（代码阅读摘要）

本文基于对仓库目录结构与关键代码（`rdagent/` 下 core/workflow/scenarios/components 等）的阅读，总结项目的基本架构：它是一个可配置的“R&D（Research & Development）闭环”框架，在不同数据驱动场景（量化、Kaggle、通用数据科学、论文模型复现等）下，通过配置把 **R(提案/假设生成)** 与 **D(实现/运行/反馈)** 组装为可执行、可恢复、可并行的迭代流程。

---

## 1. 总体分层（从入口到能力）

项目整体可以理解为如下分层：

- **App/CLI 入口层**：提供命令行入口与 demo/scenario 启动方式。
- **Core 抽象层**：定义场景、实验、开发、提案、演进等核心接口与数据结构（场景无关）。
- **Workflow 编排层**：提供可恢复/可并行的步骤式循环调度器，并组装标准 R&D loop。
- **Components 能力层**：实现可复用的 coder、知识管理、agent 适配、loader、benchmark 等通用组件。
- **Scenarios 场景落地层**：面向具体领域（qlib、kaggle、data_science、general_model）实现 Scenario/Proposal/Developer 等具体类。
- **LLM 与运行环境支撑层**：统一 LLM 后端与缓存/重试策略；提供 Docker/本地/Conda 环境执行能力。
- **日志与 UI**：记录 trace/session 并通过 Streamlit UI 回放。

---

## 2. 命令行入口与应用层（`rdagent/app`）

### 2.1 CLI 入口

- 入口文件：`rdagent/app/cli.py`
- 使用 `typer` 注册命令：
  - `fin_factor / fin_model / fin_quant / fin_factor_report`
  - `data_science`
  - `general_model`
  - `ui / server_ui`（日志可视化）
  - `health_check / collect_info`

该入口在最开始执行：
- `load_dotenv(".env")`

确保 Pydantic Settings 初始化之前加载环境变量（避免配置读取时机不一致）。

### 2.2 各场景的应用主程序

例如 Qlib 因子闭环：`rdagent/app/qlib_rd_loop/factor.py`
- 创建 Loop（如 `FactorRDLoop`）
- `asyncio.run(loop.run(...))`
- 支持从 `__session__` 断点恢复继续跑（`LoopBase.load(...)`）

---

## 3. Core 框架层（`rdagent/core`）：关键抽象与对象模型

`rdagent/core` 提供场景无关的抽象接口，定义“自动化 R&D 迭代”所需的对象与协作方式。

### 3.1 场景抽象：`Scenario`

- 文件：`rdagent/core/scenario.py`
- 典型接口：
  - `background`
  - `source_data` / `get_source_data_desc(task)`
  - `get_scenario_all_desc(task, ...)`
  - `get_runtime_environment()`

作用：把“领域背景、数据描述、运行环境信息”等从通用框架中隔离，作为 Prompt/Agent 的输入基础。

### 3.2 实验与任务抽象：`Task / Workspace / Experiment`

- 文件：`rdagent/core/experiment.py`
- 关键对象：
  - `Task`：任务（含 `user_instructions`）
  - `Workspace`：任务实现载体（可执行、可 copy、可 checkpoint）
  - `FBWorkspace`：文件夹式工作区（注入代码到目录后运行）
  - `Experiment`：一次实验 = 多个 `sub_tasks` + `sub_workspace_list` + `running_info/result` + `prop_dev_feedback`

作用：把“LLM 生成代码”落地为可运行/可回滚/可追踪的 workspace 结构。

### 3.3 Proposal（Research/R）：假设、实验生成、反馈总结

- 文件：`rdagent/core/proposal.py`
- 关键抽象：
  - `HypothesisGen`：生成假设
  - `Hypothesis2Experiment`：把假设转为实验（拆成 tasks）
  - `Experiment2Feedback`：对执行结果生成反馈（是否接受、原因、下一步方向等）
  - `Trace`：保存历史实验与反馈（支持 DAG parent、selection 等）

作用：标准化“提出想法 → 组织实验 → 反馈总结”的 R 侧过程。

### 3.4 Developer（Development/D）：实现与运行

- 文件：`rdagent/core/developer.py`
- 抽象接口：`Developer.develop(exp)`

通常会有两类实现：
- **coder**：把任务实现成代码（写入 workspace）
- **runner**：执行 workspace，收集指标/日志/产物

---

## 4. Workflow 编排层（`rdagent/utils/workflow` + `rdagent/components/workflow`）

### 4.1 工作流引擎：`LoopBase`

- 文件：`rdagent/utils/workflow/loop.py`
- 核心能力：
  - step 列表驱动的循环调度
  - 并行控制（`RD_AGENT_SETTINGS.step_semaphore`）
  - session 持久化（每个 step dump 到 `__session__`）
  - 断点续跑、checkout、超时终止
  - 异常策略：skip/withdraw
  - 需要时可把 step 放入 subprocess 执行（并行模式下）

它相当于“R&D 迭代的调度器 + 可恢复执行引擎”。

### 4.2 通用 R&D Loop：`RDLoop`

- 文件：`rdagent/components/workflow/rd_loop.py`
- 定义标准 steps：
  1. `direct_exp_gen`：提案+生成实验（内部调用 `_propose` 和 `_exp_gen`）
  2. `coding`：调用 coder.develop
  3. `running`：调用 runner.develop
  4. `feedback`：调用 summarizer.generate_feedback，并写入 `Trace.hist`

场景通过配置注入 `scen/hypothesis_gen/coder/runner/summarizer` 等组件，即可复用闭环。

### 4.3 配置注入：`BasePropSetting`

- 文件：`rdagent/components/workflow/conf.py`
- 字段大多是类路径字符串：
  - `scen / hypothesis_gen / hypothesis2experiment / coder / runner / summarizer ...`

场景配置实例（如 Qlib）：`rdagent/app/qlib_rd_loop/conf.py`。

---

## 5. Components（`rdagent/components`）：可复用能力模块

这一层提供通用能力实现，被多个 scenario 复用：

- `components/workflow/*`：通用 RDLoop 与配置
- `components/coder/*`：编码与自我改进策略
  - `coder/CoSTEER/*`：核心的“协作式演进编码”实现（多任务、反馈驱动、可多进程）
  - `coder/data_science/*`、`coder/factor_coder/*`、`coder/model_coder/*`：面向不同任务的 coder/评估器/模板
- `components/knowledge_management/*`：向量库/图谱等知识管理（RAG/记忆）
- `components/agent/*`：不同 agent 形态适配（rag/context7/mcp 等）
- `components/loader/*`：task/experiment loader

总体是“把 core 抽象落地的一组可插拔实现”。

---

## 6. Scenarios（`rdagent/scenarios`）：领域/任务落地层

`scenarios` 目录是面向具体领域的实现集合：

- `scenarios/qlib/*`：量化金融
  - `experiment/`：Qlib 场景 Experiment/Workspace/运行逻辑
  - `proposal/`：factor/model/quant 的假设生成与实验映射
  - `developer/`：coder/runner/feedback summarizer
  - `docker/`：Qlib docker 环境

- `scenarios/kaggle/*`：Kaggle 竞赛（含模板与知识管理）
- `scenarios/data_science/*`：通用数据科学/MLE-bench 风格任务
- `scenarios/general_model/*`：论文/模型结构抽取与实现
- `scenarios/shared/*`：运行时信息等共享工具

理解方式：Scenario 提供领域描述与运行环境；Proposal/Developer 将 core 接口具体化；再由 app/loop 组装运行。

---

## 7. LLM 与执行环境支撑

### 7.1 LLM 适配层：`rdagent/oai`

- `oai/backend/*`：后端适配（默认 LiteLLM，另有 deprecated 等）
- 典型能力：
  - chat/embedding 统一接口
  - 自动重试、缓存（SQLite）、json 解析修复策略
  - 自动续写（finish_reason=length 时补全）

这是整个系统与外部模型交互的底座。

### 7.2 执行环境：`rdagent/utils/env.py`

- 抽象 `Env`，实现：
  - `LocalEnv / CondaEnv`
  - `DockerEnv`（Qlib/Kaggle/DataScience/MLEBench 等配置）

与 `FBWorkspace.execute/run()` 配合：
- “生成代码 → 在隔离环境中运行 → 收集 stdout/exit_code/time”。

---

## 8. 日志与 UI（`rdagent/log`）

- 统一 logger：带 tag/对象记录
- session dump：`__session__` 目录用于断点续跑/回放
- `rdagent/log/ui/*`：Streamlit UI
- CLI：`rdagent ui ...` 启动日志回放

---

## 9. 典型运行链路（以 `rdagent fin_factor` 为例）

1. CLI：`rdagent/app/cli.py` 注册命令 → 调用 `rdagent/app/qlib_rd_loop/factor.py:main`
2. 读取配置：`FACTOR_PROP_SETTING`（`rdagent/app/qlib_rd_loop/conf.py`）提供类路径字符串：
   - `scen / hypothesis_gen / hypothesis2experiment / coder / runner / summarizer`
3. `RDLoop.__init__` 使用 `import_class()` 动态导入实例化，构造 `Trace`
4. `LoopBase.run()` 调度 steps：
   - `direct_exp_gen`：Hypothesis → Experiment(sub_tasks)
   - `coding`：coder.develop → 产出 workspace 代码
   - `running`：runner.develop → env 中运行 → 收集结果
   - `feedback`：summarizer.generate_feedback → 写入 trace.hist
5. 每步完成 dump 到 `log/.../__session__/...`，可断点续跑，并用 UI 回放。
