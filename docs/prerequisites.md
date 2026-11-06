# 运行前置条件

本文说明从零开始运行本仓库代码所需的软件环境、Python 依赖、模型权重、数据集和外部基础设施。内容基于当前工作树的实际验证；验证环境为 Linux + Python 3.10。

## 1. 场景速查

| 运行目标 | 必需前置 | 可跳过 |
| --- | --- | --- |
| 单元测试、确定性消融、进程内收敛实验 | Python 3.10+、核心依赖、`requests`、`hypothesis`、`PYTHONPATH=src` | GPU、模型权重、数据集、网络 |
| M-2 稠密分类、order-shuffle、吞吐实验 | 上述基础 + `sentence-transformers`、`torch`、MiniLM/NLI 权重 | 数据集、外部服务 |
| 本地真实模型 smoke | 上述基础 + 一个 OpenAI 兼容端点（Ollama 或 vLLM） | GPU 非必需（小模型可 CPU） |
| HPC 分布式/集中式对比 | SLURM、A100 级 GPU、Ollama、模型权重、HotpotQA 缓存 | Edge/跨层基础设施 |
| Edge 多主机实验 | 多台可 SSH 主机、Docker、`tc`/netem、`pyyaml` | GPU |
| 跨层 continuum 实验 | SSH、`rsync`、远程 `tmux`、反向隧道、可选 Kafka | GPU |

## 2. 基础运行环境

- **Python 3.10 或更高版本**。仓库 README 建议 3.11，容器镜像也基于 `python:3.11-slim`；实测 Python 3.10 可运行核心测试与确定性实验。
- **git**。`tests/test_repo_hygiene.py` 通过 `git ls-files` 扫描受版本控制的文件；若仓库没有 `.git`，该测试会失败。部分脚本也会读取 `git rev-parse` 作为 provenance。
- **bash / 常规 Linux 工具**。`tests/test_hpc_config.py`、`tests/test_edge_config.py` 在缺少 `bash` 时会跳过。
- **无需安装本仓库为 Python 包**。仓库没有 `setup.py` / `pyproject.toml`，所有运行方式都要求把 `src/` 放进 `PYTHONPATH`。

推荐使用 conda 初始化环境：

```bash
cd /path/to/2026-MELD
conda create -n meld python=3.11 -y
conda activate meld
python -m pip install -U pip
pip install -r requirements.txt
```

后续命令均在 `conda activate meld` 之后执行。若站点要求使用指定 Python 版本，把 `python=3.11` 换成对应版本即可。

之后所有命令都需要在仓库根目录执行，并带上 `PYTHONPATH=src`（`pytest` 由 `tests/conftest.py` 自动注入，脚本内部也会自行加入 `src/`）。

## 3. Python 依赖

### 3.1 核心依赖

[requirements.txt](../requirements.txt) 声明了：

| 包 | 用途 |
| --- | --- |
| `numpy` | 向量/哈希编码器与相似度计算 |
| `pyyaml` | 阈值配置、`testbed.yaml` |
| `protobuf` | `src/meld/proto` 的协议绑定 |
| `matplotlib` | smoke/图表脚本（无显示环境，使用 Agg 后端） |
| `pytest` | 测试套件 |
| `kafka-python-ng` | 仅 Kafka 传输路径需要；缺失时相关测试会跳过 |

```bash
pip install -r requirements.txt
```

**protobuf 版本注意**：`requirements.txt` 写作 `protobuf>=4.25`，但随仓库附带的生成代码是按 protobuf 6.31.1 生成的。安装 4.x 会直接报错：

```text
ImportError: cannot import name 'runtime_version' from 'google.protobuf'
```

请显式使用 6.31 以上版本（实测 7.36.2 可用）：

```bash
pip install "protobuf>=6.31"
```

### 3.2 测试套件的补充依赖（未在 requirements.txt 中声明）

完整测试还需要两个未声明的包：

```bash
pip install requests hypothesis
```

- `requests`：`tests/test_wire_integration.py` 在收集阶段直接 `import requests`；`src/meld/sync/wire.py` 的 HTTP 发送实现也依赖它。缺失时 pytest 会中断收集。
- `hypothesis`：`tests/test_status_crdt.py` 用它做属性测试。

### 3.3 可选：稠密语义依赖

以下功能需要 `sentence-transformers` + `torch`：

- M-2 稠密分类与 ROC、阈值标定
- `scripts/run_order_shuffle.py`、`scripts/run_m1_throughput.py`、`scripts/run_m2_edge_cost.py`
- `tests/test_recall_sweep.py`、`tests/test_recall_sweep_cache.py`（共 3 个用例，缺少依赖时会失败而非跳过）
- HPC 的 M-1 运行（`SMP_ENCODER=minilm`）

```bash
pip install "sentence-transformers>=3.0" "torch>=2.0"
```

未安装时，`scripts/run_ablation_metrics.py` 会自动退化为 hashing 路径，并明确打印“dense rung 不可用”，不会伪造 ROC。

### 3.4 可选：数据集与其它路径

```bash
# 加载 HotpotQA / GAIA 真实数据
pip install datasets

# 下载 LongMemEval 语料
pip install huggingface_hub

# 可选的 smolagents 基线测试（未安装时 tests 自动 skip）
pip install smolagents
```

## 4. 模型权重

| 模型 | 用途 | 默认获取方式 |
| --- | --- | --- |
| `sentence-transformers/all-MiniLM-L6-v2` | 稠密编码器（`encoder="minilm"`） | 首次调用时从 Hugging Face 下载 |
| `cross-encoder/nli-deberta-v3-small` | 交叉编码 NLI（`nli="mnli-cross-encoder"`） | 首次调用时从 Hugging Face 下载 |
| `qwen2.5:1.5b` / `:7b` / `:32b`、`mistral:7b`、`gemma3:12b` 等 | 真实 LLM 回答生成 | Ollama 模型仓库或本地 vLLM 服务 |

离线或受限网络环境下，需要提前把权重放入 Hugging Face 缓存，并设置：

```bash
export HF_HOME=/path/to/hf_home
export HF_HUB_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
```

HPC 的 sbatch 脚本已经默认设置上述离线变量，并要求模型缓存事先就位。

## 5. 数据与配置文件

仓库**不包含**任何数据集或实验输出。以下路径均被 `.gitignore` 排除，需要按需准备：

| 路径 / 环境变量 | 作用 | 是否需要手工准备 |
| --- | --- | --- |
| `data/benchmark_provenance.json` | 基准身份门禁（G0）读取的来源核验记录 | 需要；缺失时 4 个测试失败 |
| `data/longmemeval/longmemeval_s` | `scripts/run_r2_lme.py` 的默认 LongMemEval 语料 | 需要（仅运行该脚本时） |
| HotpotQA 缓存 | `load_hotpot_real()` / HPC M-1 运行 | 需要联网预取或预先填充 HF 缓存 |
| `site.env`（`hpc/`、`cross-tier/`） | 站点/主机配置 | 从对应 `site.example.env` 复制并填写 |
| `testbed.yaml`（`edge/`） | 多主机拓扑与 netem 网格 | 从 `testbed.example.yaml` 复制并填写 |
| `src/meld/config/thresholds*.yaml` | 已标定阈值 | 已随仓库提供，无需修改 |

### 5.1 基准身份门禁

`tests/test_benchmark_identity.py` 会读取 `data/benchmark_provenance.json`，并校验 LongMemEval / GAIA 的名称、arXiv 编号和身份哈希。该文件属于运行期核验产物，未随仓库发布；缺失时这 4 个测试会以 `FileNotFoundError` 失败。期望值存放在 [benchmark_identity_expected.json](../src/meld/eval/benchmark_identity_expected.json)。

在未准备该文件时，可先排除这组门禁测试（见 6.1 节），或按团队流程重新核验来源后生成该 JSON。

### 5.2 真实数据集来源

- **HotpotQA**：`hotpotqa/hotpot_qa`，`distractor` 配置，`validation` 划分；开放、无需 token。首次加载需要联网，离线运行前先在可联网节点预取到 `HF_HOME`。
- **LongMemEval**：从 `xiaowu0162/longmemeval` 下载 `longmemeval_s` 或 `longmemeval_oracle`，放到 `data/longmemeval/`。
- **GAIA**：受门控，需要 `HUGGING_FACE_HUB_TOKEN` 或 `HF_TOKEN`，并在 Hugging Face 上接受数据集条款。

## 6. 运行命令

### 6.1 测试套件

```bash
# 完整套件
PYTHONPATH=src pytest -q

# 若尚未安装稠密依赖、也没有 data/benchmark_provenance.json，
# 先排除这三个已知需要外部输入的测试文件：
PYTHONPATH=src pytest -q \
  --ignore=tests/test_benchmark_identity.py \
  --ignore=tests/test_recall_sweep.py \
  --ignore=tests/test_recall_sweep_cache.py
```

在“仅核心依赖 + requests + hypothesis”的全新环境中，完整命令预计有 7 个失败：4 个来自基准身份门禁，3 个来自 MiniLM recall-sweep 用例。其余测试通过或按条件跳过。

### 6.2 确定性实验

```bash
# 门控必要性、冲突/权威、回声安全、token 经济学、
# M-2（无稠密依赖时自动退化）、E2 扰动扫描
PYTHONPATH=src python scripts/run_ablation_metrics.py

# 进程内状态 CRDT 收敛：顺序无关性 + 拒绝门位置
PYTHONPATH=src python scripts/run_eval_c_inprocess.py

# 本地 smoke：完整仿真 + 渲染图（写入 figures/）
PYTHONPATH=src python scripts/run_local_smoke.py
```

这些实验只依赖核心 Python 包，不需要 GPU、网络或数据集。输出写入被忽略的 `data/`、`figures/`、`outputs/` 目录。

### 6.3 真实模型 smoke

需要一个 OpenAI 兼容的 `/v1/completions` 端点。默认配置：

```bash
# Ollama（默认端口 11434）
ollama serve
ollama pull qwen2.5:7b

PYTHONPATH=src python scripts/run_memdelta_local.py
PYTHONPATH=src python scripts/run_hotpot_realmodel_extsmoke.py

# 或指向 vLLM / 其它兼容服务
export SMP_ENDPOINT=http://localhost:8000/v1/completions
export SMP_TRANSPORT=vllm-openai
```

常用环境变量：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `SMP_ENDPOINT` | `http://localhost:11434/v1/completions` | OpenAI 兼容端点 |
| `SMP_TRANSPORT` | `vllm-openai` / `ollama` | 传输类型 |
| `SMP_REALMODEL` | `qwen2.5:7b` | Ollama 模型标签 |
| `SMP_MODEL_KEY` | `qwen2.5-7b` | 模型轴键名 |
| `SMP_OUT` | 无默认值 | M-1 runner 的持久化 JSONL 路径，必需 |
| `SMP_PER_LEVEL` | `easy:20,medium:60,hard:60` | HotpotQA 难度抽样 |
| `SMP_ENCODER` | `minilm` | 编码器档位 |
| `SMP_KAFKA_BOOTSTRAP` | `localhost:9092` | Kafka broker 地址 |
| `SMP_TESTBED` | `edge/testbed.yaml` | Edge 拓扑配置路径 |

## 7. 外部基础设施

### 7.1 HPC（`hpc/`）

前置条件：

- SLURM 集群与可用的 GPU 分区；站点示例假设单张 A100 40GB。
- 集群上安装 Ollama（`$SMP_OLLAMA_ROOT/bin|lib`），并把 `SMP_MODELS` 中的模型标签拉取到其模型库。
- 一个安装了本仓库依赖的 Python（至少 `numpy`、`sentence-transformers`；运行真实 HotpotQA 还需 `datasets`），通过 `SMP_PY` 指定；若使用 conda，可填 `SMP_PY=/path/to/conda/envs/meld/bin/python`。
- 仓库检出到 `$SMP_REPO`，可写的 `$SMP_SCRATCH`。
- HotpotQA 已预取到 `HF_HOME`：sbatch 默认设置 `HF_HUB_OFFLINE=1`，不会在线下载。

运行方式：

```bash
cd hpc
cp site.example.env site.env
$EDITOR site.env
./submit.sh
```

### 7.2 Edge 多主机测试床（`edge/`）

前置条件：

- 多台可通过 SSH 互达的主机（裸机、虚机或容器），且 orchestrator 节点能免交互 SSH 到其它节点。
- 每台主机安装 Docker，并具备 `tc` / netem；`deploy.sh` 使用 `sudo tc`，需要可用的 sudo 权限。
- orchestrator 节点上的 Python 安装 `pyyaml`，且本仓库可导入（`PYTHONPATH=src`）。
- 每台主机有监听 `watchdog_port`（默认 8090）的 watchdog 服务。
- 已填写 `edge/testbed.yaml`（节点、IP、端口、镜像标签、RTT/路由网格）。

```bash
cd edge
cp testbed.example.yaml testbed.yaml
$EDITOR testbed.yaml
./deploy.sh push
./deploy.sh start broadcast hashing
python run_campaign.py      # eval-s + eval-r
python run_eval_c.py        # partition -> heal -> CRDT
```

镜像构建上下文是仓库根目录：

```bash
docker build -f edge/docker/Dockerfile.meld -t meld-brain:smoke .
docker build -f edge/docker/Dockerfile.meld-minilm -t meld-brain:minilm .
```

**已知缺口**：两个 Dockerfile 都 `COPY edge/docker/requirements-smoke.txt` / `requirements-minilm.txt`，但这两个文件当前不在仓库中，直接构建会失败。需要自行补齐对应依赖清单（smoke 镜像至少需要 `numpy`、`protobuf`、`pyyaml`、`requests`；minilm 镜像另需 `sentence-transformers`）。仓库也未提供 docker-compose 文件与 watchdog 守护脚本，`scripts/eval_c_docker_smoke.py` 需要你自行在 8801/8802/8803 启动三个 brain。

### 7.3 跨层 continuum（`cross-tier/`）

前置条件：

- 本机作为 hub，可运行 `python3` 与 `cross-tier/probe_broker.py`（仓库内置的轻量 TCP broker，不依赖 MQTT 服务）。
- 到 HPC / edge 节点的免交互 SSH；远程需要 `python3 >= 3.10` 且安装 `pyyaml`（或通过站点模块加载）。
- 本地需要 `rsync`；远程需要 `tmux`；`autossh` 可选（存在时用于保持反向隧道）。
- 远程允许 SSH 反向隧道；隧道将远端的 `TUN_PORT` 映射到本机 `BROKER_PORT`。
- 已填写 `cross-tier/site.env`。

```bash
cd cross-tier
cp site.example.env site.env
$EDITOR site.env
./orchestrate.sh
```

Kafka 路径额外需要：

- 一个可达的 Kafka broker（默认 `localhost:9092`，可用 `SMP_KAFKA_BOOTSTRAP` 覆盖）。
- `kafka-python-ng`（已在 `requirements.txt` 中）。无 broker 时相关测试会跳过。

## 8. 已知缺口与排错

| 症状 | 原因 | 处理 |
| --- | --- | --- |
| `ImportError: cannot import name 'runtime_version'` | 安装了 protobuf 4.x | `pip install "protobuf>=6.31"` |
| pytest 收集阶段 `No module named 'requests'` | 依赖未声明 | `pip install requests` |
| `No module named 'hypothesis'` | 依赖未声明 | `pip install hypothesis` |
| 3 个 recall-sweep 测试 `No module named 'sentence_transformers'` | 稠密依赖缺失 | 安装 `sentence-transformers` + `torch`，或排除这两个测试文件 |
| 4 个 benchmark 测试 `FileNotFoundError` | 缺少 `data/benchmark_provenance.json` | 重新核验来源后生成，或排除 `tests/test_benchmark_identity.py` |
| `edge/docker/requirements-*.txt` 不存在 | 镜像依赖清单未随仓库提供 | 自行补齐后再构建镜像 |
| `fig_m1_hotpot.py` 等图表脚本找不到 `data/csc-m1-archive-*` | 原始运行归档未发布 | 仅在拥有对应运行结果时使用；确定性的 `fig_eval_s_latency.py` 不依赖外部数据 |
| `testbed config not found` / `site.env` 缺失 | 未复制示例配置 | 按 7.2 / 7.3 节复制并填写 |
| HPC 加载 HotpotQA 失败 | `HF_HUB_OFFLINE=1` 但缓存未预取 | 先在可联网节点预取到 `HF_HOME` |

## 9. 最小可运行清单

1. 用 conda 创建 Python 3.10+ 虚拟环境：`conda create -n meld python=3.11 -y && conda activate meld`。
2. `pip install -r requirements.txt`，再 `pip install requests hypothesis`，并确认 protobuf ≥ 6.31。
3. 从仓库根目录运行 `PYTHONPATH=src pytest -q`（按 6.1 节排除依赖外部输入的测试）。
4. 运行 `scripts/run_ablation_metrics.py` 与 `scripts/run_eval_c_inprocess.py` 复现确定性结果。
5. 如需稠密结果，安装 `sentence-transformers` + `torch` 并准备模型缓存。
6. 如需真实数据或外部实验，按第 5、7 节准备数据集、`site.env`、`testbed.yaml` 和对应硬件。
