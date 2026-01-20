# Customize Qlib Datasets in RD-Agent

本文基于对 RD-Agent 代码的阅读，说明 **RD-Agent 与 Qlib 的配合机制**、当前默认使用的 **Qlib 模型/数据组件**，以及当你要 **定制数据源（dataset/provider）** 时需要修改哪些地方、注意哪些同步点。

> 适用范围：
> - `rdagent fin_factor`（因子迭代）
> - `rdagent fin_model`（模型迭代）
> - 以及两者结合的 quant 场景（因子+模型联合）

---

## 1. RD-Agent 与 Qlib 的配合方式（端到端链路）

### 1.1 分工：RD-Agent 生成/组织实验；Qlib 负责训练/回测/打分

在 Qlib 场景下，RD-Agent 的闭环大致是：

1. **R（proposal）**：LLM 生成新的因子/模型假设
2. **D（coder）**：把假设实现为可运行代码（如 `model.py` 或 factor 实现代码）
3. **D（runner）**：调用 Qlib 的 `qrun conf*.yaml` 运行训练/回测
4. **summarizer**：从 Qlib recorder/日志中读指标并形成反馈，进入下一轮

Qlib 在这里扮演“统一实验执行器/回测框架”的角色；RD-Agent 主要负责：
- 产出和维护 workspace（代码 + 运行配置 + 生成的特征文件）
- 驱动迭代流程（proposal → coding → running → feedback）

---

## 2. Qlib 执行入口：`QlibFBWorkspace.execute()`

Qlib 的执行发生在 workspace 目录内，通过 `qrun` 调起。

- 文件：`rdagent/scenarios/qlib/experiment/workspace.py`
- 类：`QlibFBWorkspace(FBWorkspace)`

关键逻辑：

- 根据配置选择运行环境
  - docker：`QTDockerEnv()`
  - conda：`QlibCondaEnv(conf=QlibCondaConf())`
- 在 workspace 中执行：
  - `qrun {qlib_config_name}`
  - `python read_exp_res.py`（读取最新 recorder 的 metrics 与收益曲线）

`read_exp_res.py` 的作用：
- `qlib.init()` 后通过 `qlib.workflow.R` 找到最新 recorder
- 将 metrics 写到 `qlib_res.csv`
- 将回测收益曲线写到 `ret.pkl`

对应文件：
- `rdagent/scenarios/qlib/experiment/model_template/read_exp_res.py`
- `rdagent/scenarios/qlib/experiment/factor_template/read_exp_res.py`

---

## 3. 为什么配置文件里有 `{{train_start}}` 这种变量？

RD-Agent 会在运行时把时间段/超参等通过环境变量注入 Qlib config 模板。

例如：
- `train_start/train_end/valid_start/valid_end/test_start/test_end`
- `n_epochs/lr/batch_size/early_stop/weight_decay`

对应代码：
- 模型 runner：`rdagent/scenarios/qlib/developer/model_runner.py`
- 因子 runner：`rdagent/scenarios/qlib/developer/factor_runner.py`

它们都会构造 `env_to_use` 并传给：
- `exp.experiment_workspace.execute(qlib_config_name=..., run_env=env_to_use)`

因此：
- Qlib yaml 文件不是纯静态配置，而是 RD-Agent 可参数化的模板。

---

## 4. RD-Agent(Qlib) 默认使用的 Qlib 模型是什么？

### 4.1 因子迭代（`fin_factor`）的基线模型：`LGBModel`

因子 loop 的默认配置使用 LightGBM 作为稳定、低成本的评估器：

- `rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml`
- `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml`

关键字段：
- `task.model.class: LGBModel`
- `task.model.module_path: qlib.contrib.model.gbdt`

含义：
- RD-Agent 重点进化“因子/特征”，用一个固定的 GBDT 模型打分并做策略回测。

### 4.2 模型迭代（`fin_model`）的核心：`GeneralPTNN` + 注入的 PyTorch 网络结构

模型 loop 使用 Qlib 的 PyTorch 通用 wrapper：

- `task.model.class: GeneralPTNN`
- `task.model.module_path: qlib.contrib.model.pytorch_general_nn`

并通过：
- `pt_model_uri: "model.model_cls"`

把具体网络结构从 workspace 的 `model.py` 中注入。

对应模板：
- `rdagent/scenarios/qlib/experiment/model_template/conf_baseline_factors_model.yaml`
- `rdagent/scenarios/qlib/experiment/model_template/conf_sota_factors_model.yaml`

`model.py` 的注入逻辑：
- `rdagent/scenarios/qlib/developer/model_runner.py` 会把 coder 产出的 `model.py` 写入 experiment workspace。

### 4.3 常用数据组件：`Alpha158 / DataHandlerLP + DatasetH/TSDatasetH`

常见配置：
- baseline handler：`Alpha158`（`qlib.contrib.data.handler`）
- 合并因子/自定义因子加载：`NestedDataLoader + StaticDataLoader(config: combined_factors_df.parquet)`
- dataset：
  - Tabular：`DatasetH`
  - TimeSeries：`TSDatasetH`（并注入 `step_len/num_timesteps`）

模型 loop 会根据 task 的 `model_type` 切换 dataset（见 `model_runner.py`）。

---

## 5. 定制数据源：你需要改哪些地方？

“定制数据源”建议拆成三层理解：

1. **Qlib 真正训练/回测用的数据**（`provider_uri`）
2. **因子实现阶段提供给 LLM 的样例数据文件夹**（导出的 `daily_pv.h5`）
3. **数据 schema 与 handler/dataset 的匹配**（是否能算 Alpha158、label 计算是否成立等）

下面给出三条可操作路线。

---

## 6. 路线 A：尽量不改代码（推荐）

### 做法

把你的数据导入/转换成 Qlib 支持的数据格式后，放到默认路径：
- `~/.qlib/qlib_data/cn_data`

此时通常不需要修改任何模板 yaml。

### 优点
- 最省事
- docker/conda 两种环境都能用（docker 默认挂载 `~/.qlib`）

---

## 7. 路线 B：修改 `provider_uri`（数据不放在 `~/.qlib`）

### 7.1 你必须同时处理三件事

#### 1) 修改 Qlib 模板中的 `qlib_init.provider_uri`

涉及文件：
- 因子 loop：
  - `rdagent/scenarios/qlib/experiment/factor_template/conf_baseline.yaml`
  - `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors.yaml`
  - `rdagent/scenarios/qlib/experiment/factor_template/conf_combined_factors_sota_model.yaml`
- 模型 loop：
  - `rdagent/scenarios/qlib/experiment/model_template/conf_baseline_factors_model.yaml`
  - `rdagent/scenarios/qlib/experiment/model_template/conf_sota_factors_model.yaml`

将：
- `qlib_init.provider_uri: "~/.qlib/qlib_data/cn_data"`

改为你希望使用的数据目录。

> 注意：这里写的是“运行时可见路径”。如果用 docker 跑，必须是 **容器内路径**。

#### 2) 修改导出 factor 样例数据的脚本 `generate.py`

因子 coder 会给 LLM 展示 debug 数据文件夹（`daily_pv.h5`），默认从 Qlib provider 导出。

文件：
- `rdagent/scenarios/qlib/experiment/factor_data_template/generate.py`

将：
- `qlib.init(provider_uri="~/.qlib/qlib_data/cn_data")`

改为你的 provider。

否则会出现数据错位：
- Qlib 训练使用新数据
- 但 LLM 仍“看着旧数据/或生成失败”来写因子，导致反馈噪声增大。

#### 3) 如果使用 docker：确保新数据路径挂载进容器

Qlib docker 默认只挂载宿主机 `~/.qlib`：
- `rdagent/utils/env.py` → `QlibDockerConf.extra_volumes`

如果你的数据在宿主机 `/data/qlib_cn`，容器默认不可见。

两种方案：
- **B1（更推荐）**：把数据同步放到 `~/.qlib/...`（回到路线 A）
- **B2**：扩展 docker volumes，把 `/data/qlib_cn` 挂到容器目录（例如 `/data/qlib_cn`），然后模板里 `provider_uri` 写容器内路径。

---

## 8. 路线 C：深度定制（换 market/region/instruments/handler/label 等）

当你要的不只是换路径，还包括：
- 换市场（`csi300 → csi500/all/自定义股票池`）
- 换 region（`cn/us`）
- 换特征体系/字段
- 换 label（预测目标）
- 换 handler / data_loader

主要修改点仍在 **Qlib 模板 yaml**。

### 8.1 定制股票池与基准

模板默认：
- `market: csi300`
- `benchmark: SH000300`

你可以按需修改为你想要的 instruments 与 benchmark。

### 8.2 定制 handler / dataset

RD-Agent(Q) 默认大量使用：
- baseline：`Alpha158` / `Alpha158DL`
- 合并因子：`NestedDataLoader + StaticDataLoader` + `DataHandlerLP`
- dataset：`DatasetH` / `TSDatasetH`

如果你的数据 schema 不满足 Alpha158（例如缺 `$volume`），你需要：
- 把数据规整成能支持 Alpha158 的基础字段（更兼容）
- 或改模板 yaml，换成你自定义的 handler/data_loader（并保证在运行环境可 import）。

### 8.3 定制 label

模板常见 label：
- `Ref($close, -2) / Ref($close, -1) - 1`

你可以改成未来 N 日收益、超额收益或其他目标；但要注意策略与评估可能也需调整。

---

## 9. RD-Agent 侧的两个“容易漏掉”的同步点

### 9.1 因子实现提示数据不直接读 provider，而是读导出的 h5

因子 scenario 的 source_data 描述来自：
- `rdagent/scenarios/qlib/experiment/utils.py:get_data_folder_intro()`

如果 `FACTOR_CoSTEERSettings.data_folder(_debug)` 不存在，会自动触发从 Qlib 导出 `daily_pv*.h5`。

路径配置：
- `rdagent/components/coder/factor_coder/config.py`
  - `data_folder`（默认 `git_ignore_folder/factor_implementation_source_data`）
  - `data_folder_debug`（默认 `git_ignore_folder/factor_implementation_source_data_debug`）

建议：
- 你也可以自己准备 `daily_pv.h5` 和 README，放到上述目录，并用环境变量指定路径，避免每次自动生成。

### 9.2 合并因子 parquet 是 runner 注入的，模板通过 StaticDataLoader 读取

当存在“合并因子”时，runner 会写：
- `combined_factors_df.parquet`

并且模板里通过：
- `StaticDataLoader(config: "combined_factors_df.parquet")`

读取。

若你修改了 handler/data_loader 逻辑，需要同步保证：
- 文件仍存在且格式正确
- 或你改模板读取你新的格式/文件名。

---

## 10. 建议的验证方式（快速自检）

1. 在你修改后的 provider 上单独运行一次 qrun（进入某个 workspace/template 目录执行）
2. 确认 `read_exp_res.py` 能产出：
   - `qlib_res.csv`
   - `ret.pkl`
3. 再启动 `rdagent fin_factor / fin_model`，观察：
   - Qlib 执行日志（`Qlib_execute_log`）
   - metrics 是否合理

---

## 11. 你可能需要提供的信息（以便选择最佳方案）

为了更精确地给出“改哪些文件/怎么挂载”的操作清单，建议你明确：

1. 你用 **docker** 还是 **conda** 跑 Qlib？（影响 provider 路径的可见性）
2. 你的数据是否已经是 Qlib provider 格式？
3. 你只想换 `provider_uri`，还是要连 `region/instruments/handler/label` 一起改？

