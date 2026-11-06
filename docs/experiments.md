对照 PDF 的 §7.1–7.8、Table 8 和补充材料 Appendix A–N，脚本对应关系如下。注意：代码注释里用的是内部草稿编号（如 `7-c`、`7-h`、`§7-r`），以论文正式章节为准。

## §7.1 Recall under decentralized merge

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [hpc/submit.sh](file:///home/cxy/meld-experiments/hpc/submit.sh) → `run_memdelta.sbatch` → [run_memdelta.py](file:///home/cxy/meld-experiments/scripts/run_memdelta.py#L1) | 五模型 HPC 主实验（Qwen/Mistral/Gemma） | 产出 headline JSONL；对应 Figure 4 与 0.630 vs 0.619 |
| [run_memdelta_local.py](file:///home/cxy/meld-experiments/scripts/run_memdelta_local.py#L1) | 同一套 memory-condition 的本地验证 | 小模型/本地端点，不是论文正式数字 |
| [run_r2_lme.py](file:///home/cxy/meld-experiments/scripts/run_r2_lme.py#L1) | LongMemEval 第二语料（Appendix E / Table 4） | 需下载 LongMemEval 语料 |
| [run_r2_overlap_k.py](file:///home/cxy/meld-experiments/scripts/run_r2_overlap_k.py#L1) | overlap × k 敏感性与 exact-dedup 基线（Appendix K） | 验证增益来自语义合并而非缩库 |
| [run_order_shuffle.py](file:///home/cxy/meld-experiments/scripts/run_order_shuffle.py#L1) | 交付顺序对图结构的影响（Appendix M） | Table 8 的“order-sensitivity”行 |
| [fig_m1_hotpot.py](file:///home/cxy/meld-experiments/scripts/fig_m1_hotpot.py#L1) | 渲染 Figure 4（M-1 headline 图） | 读取 `data/csc-m1-archive/`，归档未随仓库发布 |

## §7.2 Merge classification quality

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [run_ablation_metrics.py](file:///home/cxy/meld-experiments/scripts/run_ablation_metrics.py#L1) 的 M-2 部分 | 104-pair gold、macro-F1、false-merge、ROC、baseline | 需 `sentence-transformers`；无则退化为 hashing 并明确告警 |
| [run_r3_contradiction_field.py](file:///home/cxy/meld-experiments/scripts/run_r3_contradiction_field.py#L1) | NLI 在自然分布上的 contradiction-miss 率 | §7.2 “Field contradiction-miss rate” |
| [run_r5_signal_sweep.py](file:///home/cxy/meld-experiments/scripts/run_r5_signal_sweep.py#L1) | encoder × NLI 信号敏感性（Appendix L） | 对应 macro-F1 0.765–0.845、false-merge ≤0.013 |
| [run_candidate_scaling.py](file:///home/cxy/meld-experiments/scripts/run_candidate_scaling.py#L1) | 候选选择成本随 store size 的扩展（Appendix M / Table 10） | 对应 3.8 ms / 1,000 claims |
| [run_ablation_metrics.py](file:///home/cxy/meld-experiments/scripts/run_ablation_metrics.py#L230) 的 E2(b) | embedding 噪声 / NLI 损坏鲁棒性 | §7.2 “Robustness and signal generality” |
| [run_meld_eval.py](file:///home/cxy/meld-experiments/src/meld/eval/run_meld_eval.py#L234) `--exp M2` | 轻量 SIM 版本 M-2 | 用 LongMemEval 形状的 fixture，非正式 104-pair gold |

## §7.3 Sync latency, throughput, fan-out

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [edge/run_campaign.py](file:///home/cxy/meld-experiments/edge/run_campaign.py#L1) 的 eval-s | RTT × fan-out 延迟网格（Table 7） | 需多主机 edge testbed |
| [fig_eval_s_latency.py](file:///home/cxy/meld-experiments/scripts/fig_eval_s_latency.py#L1) | 渲染 Figure 5（p50 latency surface） | 注释写“Section 7.4”，正式论文是 §7.3，属于草稿编号漂移 |
| [run_m1_throughput.py](file:///home/cxy/meld-experiments/scripts/run_m1_throughput.py#L1) | 单 sink 饱和吞吐膝点（≈54 merges/s） | 文件名里的 `m1` 是内部吞吐标签，不是 M-1 recall |
| [fig_scaling.py](file:///home/cxy/meld-experiments/scripts/fig_scaling.py#L1) 面板 (a) | 吞吐曲线 small-multiples | 读取运行结果 JSON |

## §7.4 Convergence / partition heal

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [run_eval_c_inprocess.py](file:///home/cxy/meld-experiments/scripts/run_eval_c_inprocess.py#L1) | C-1 顺序无关性 + C-1b reject-gate 位置 | 200 种交付顺序，CRDT=1.000 vs LWW=0.085 |
| [run_ablation_metrics.py](file:///home/cxy/meld-experiments/scripts/run_ablation_metrics.py#L230) 的 E2(a) | partition duration × reorder rate 扫描 | §7.4 “robustness sweeps” |
| [run_anti_entropy_loss.py](file:///home/cxy/meld-experiments/scripts/run_anti_entropy_loss.py#L1) | 丢包下的 anti-entropy digest（Appendix J / Table 9） | 10% loss 时 agreement 0.117 → 1.000 |
| [edge/run_eval_c.py](file:///home/cxy/meld-experiments/edge/run_eval_c.py#L1) | 多主机 30-seed partition→heal（Table 1 bottom） | 真实 `tc` 分区注入 |
| [eval_c_docker_smoke.py](file:///home/cxy/meld-experiments/scripts/eval_c_docker_smoke.py#L1) | 容器栈上的 wire-level CRDT 复现 smoke | 需自行启动 compose 栈 |

## §7.5 Routing ablation

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [edge/run_campaign.py](file:///home/cxy/meld-experiments/edge/run_campaign.py#L1) 的 eval-r | semantic vs broadcast 的 Table 1 上半部分 | 5GTN 多主机 |
| [cross-tier/routing_ablation.py](file:///home/cxy/meld-experiments/cross-tier/routing_ablation.py#L1) | broadcast / random / round-robin / semantic 四策略 | 对应 §7.5 的可分离三域 in-process probe |
| [fig_eval_s_latency.py](file:///home/cxy/meld-experiments/scripts/fig_eval_s_latency.py#L1) | 按 routing × fan-out 分组的延迟曲线 | semantic 平/降，broadcast 随 N 上升 |

## §7.6 Node-scaling

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [run_scale_sweep.py](file:///home/cxy/meld-experiments/scripts/run_scale_sweep.py#L1) | 每 claim 交付成本 vs N（Table 5，S-3） | in-process 到 N=128 |
| [run_sink_load.py](file:///home/cxy/meld-experiments/scripts/run_sink_load.py#L1) | sink live store vs N（Table 6，S-4） | merge 保持 26 units，union 增长 26N |
| [fig_scaling.py](file:///home/cxy/meld-experiments/scripts/fig_scaling.py#L1) 面板 (b)(c) | 上述两条 scaling 曲线 | 与吞吐面板并列 |

## §7.7 Additional ablations

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [run_ablation_metrics.py](file:///home/cxy/meld-experiments/scripts/run_ablation_metrics.py#L28) | A-1 gate necessity、T-1 conflict/authority、A-2 echo safety、A-3 token economics | 对应 Table 2 的确定性探针；纯仿真、无模型 |
| [run_meld_eval.py](file:///home/cxy/meld-experiments/src/meld/eval/run_meld_eval.py#L211) `--exp A1` | 轻量版单 gate 消融（去掉 NLI gate） | 非正式四门探针 |

## §7.8 Cross-tier deployment & per-delta cost

| 脚本 | 对应内容 | 说明 |
| --- | --- | --- |
| [cross-tier/orchestrate.sh](file:///home/cxy/meld-experiments/cross-tier/orchestrate.sh) → [driver.py](file:///home/cxy/meld-experiments/cross-tier/driver.py#L1) + [brain.py](file:///home/cxy/meld-experiments/cross-tier/brain.py#L1) | relay 上的三层次 live 部署与 11–13 ms RTT | 需 SSH 反向隧道 |
| [cross-tier/kafka_orchestrate.sh](file:///home/cxy/meld-experiments/cross-tier/kafka_orchestrate.sh) → [kafka_xtier.py](file:///home/cxy/meld-experiments/cross-tier/kafka_xtier.py#L1) | Kafka 上的单次跨层可行性验证 | 需 Kafka broker |
| [cross-tier/kafka_partition_heal_wan.sh](file:///home/cxy/meld-experiments/cross-tier/kafka_partition_heal_wan.sh) → [kafka_partition_heal.py](file:///home/cxy/meld-experiments/cross-tier/kafka_partition_heal.py#L1) | 10-trial Kafka WAN partition-heal（Table 3 / Appendix D） | 10/10 reconverge，p50 ≈ 3.0 s |
| [run_m2_edge_cost.py](file:///home/cxy/meld-experiments/scripts/run_m2_edge_cost.py#L1) | 每 delta admission 成本分解 | CPU 23.5 ms；V100 9.8 ms（embed+NLI） |

## 不属于论文实验章节的脚本

| 脚本 | 性质 |
| --- | --- |
| [run_gaia_gate_smoke.py](file:///home/cxy/meld-experiments/scripts/run_gaia_gate_smoke.py#L1)、`run_gaia_gate_smoke_multibaseline.py`、`run_gaia_realmodel_extsmoke.py` | GAIA 被论文 §9.2 明确列为 deferred，不在 §7 实验矩阵中 |
| [run_local_smoke.py](file:///home/cxy/meld-experiments/scripts/run_local_smoke.py#L1) | 本地端到端复现 smoke，不产出论文数字 |
| [run_hotpot_realmodel_extsmoke.py](file:///home/cxy/meld-experiments/scripts/run_hotpot_realmodel_extsmoke.py#L1) | 真实模型 harness 连通性 smoke，不是 M-1 结果 |
| [cross-tier/probe_client.py](file:///home/cxy/meld-experiments/cross-tier/probe_client.py#L1)、`probe_broker.py` | SSH 隧道可达性探测脚手架 |
| `scripts/fig_*.py` 中除上述三个图之外的辅助脚本 | 渲染/汇总，不直接对应实验章节 |

## 两个容易混淆的点

1. **`M1` 命名冲突**：[fig_m1_hotpot.py](file:///home/cxy/meld-experiments/scripts/fig_m1_hotpot.py#L1) 的 “M-1” 是 recall 主实验；[run_m1_throughput.py](file:///home/cxy/meld-experiments/scripts/run_m1_throughput.py#L1) 的 “M1” 是 sink 吞吐实验（§7.3/§7.6），二者不是同一实验。
2. **注释章节号有漂移**：`run_eval_c_inprocess.py` 写 “Section 7-c”、`fig_m1_hotpot.py` 写 “7-h”、`routing_ablation.py` 写 “§7-r”，都是内部草稿编号；正式对应关系以上表和 PDF §7.1–7.8 为准。