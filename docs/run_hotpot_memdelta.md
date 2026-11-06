这个脚本是 M-1 主实验的\*\*本地零成本验证工具\*\*，不是论文正式结果的复现器。它把同一批问题分别放入 5 种记忆条件，比较 F1/EM 的\*\*条件间差值\*\*，并验证 merge 审计日志确实被触发。

## 1. 整体执行流程

入口在 [run_memdelta_local.py](file:///home/cxy/meld-experiments/scripts/run_memdelta_local.py#L109-L176)：

1. 把 `src/` 加入 `sys.path`，因此不需要先 `pip install` 本包（第 28–29 行）。
2. 读取环境变量（第 114–121 行）：
   - `SMP_LLM=ollama|oracle`，默认 `ollama`
   - `SMP_REALMODEL=qwen2.5:7b`
   - `SMP_ENDPOINT=http://localhost:11434/v1/completions`
   - `SMP_N=10`、`SMP_OVERLAP=0.5`
   - `SMP_MODES=fixed_recall`，可设为 `fixed_recall,agentic`
   - `SMP_AGENTIC_N=2`
3. 固定使用 `encoder="hashing"`、`nli="lexical"`，并加载阈值（第 122 行）。
4. 用 [load_hotpot_fixture](file:///home/cxy/meld-experiments/src/meld/agent/hotpot_loader.py#L206-L212) 生成 n 条确定性 fixture。fixture 本身只有 6 条题，`n=10` 时会循环复用（task_id 加盐区分）。
5. 对 `mode × condition × task` 三重循环执行 [run_one](file:///home/cxy/meld-experiments/src/meld/agent/run_hotpot_memdelta.py#L79-L127)，收集行数据。
6. 用 [summarize](file:///home/cxy/meld-experiments/src/meld/agent/run_hotpot_memdelta.py#L160-L182) 汇总均值、bootstrap 置信区间和 merge 审计。
7. 执行三条断言，决定退出码。

## 2. 五种记忆条件

定义在 [MEMORY_CONDITIONS](file:///home/cxy/meld-experiments/src/meld/agent/mem_conditions.py#L67-L70)：

| 条件 | 构造方式 | 作用 |
| --- | --- | --- |
| `none` | 空上下文 | F1 地板 |
| `full_context` | 所有段落直接内联进 prompt，无 brain | 上限/天花板 |
| `centralized` | 所有段落写入一个 MELD brain，再从该 brain recall | 单库基线 |
| `distributed_merge` | 段落分给 3 个 sovereign brain，各自 publish delta，全局 brain 执行 `merge_on_receive` | MELD 主路径 |
| `naive_union` | 同样分片，但每个 delta 直接 raw insert，绕过合并判定 | 消融基线 |

关键实现细节：

- `distributed_merge` 与 `naive_union` 的分片不是完全独立的：金标 supporting sentence 会被复制到多个 brain，并做轻量改写，以确保触发 `overlaps` / MERGE，否则 merge 永远不会发生。见 [_build_distributed_merge](file:///home/cxy/meld-experiments/src/meld/agent/mem_conditions.py#L343-L370) 和 [_build_naive_union](file:///home/cxy/meld-experiments/src/meld/agent/mem_conditions.py#L373-L397)。
- `naive_union` 通过 [_raw_insert](file:///home/cxy/meld-experiments/src/meld/agent/mem_conditions.py#L400-L411) 直接构造 INSERT Patch，因此 `overlap_merges` 按设计恒为 0。

## 3. 两种答题模式

1. **`fixed_recall`（默认，也是 headline）**
   - 从 brain 做确定性 top-k 图遍历 recall，拼成单次 prompt 调 LLM。
   - 去掉了 agent 策略方差，条件间差异更干净。
   - 每次还会计算 `recall_at_k` 和 `MRR`（仅对有 brain 的条件）。

2. **`agentic`**
   - 用 `react` agent + `RecallTool` + `FinalAnswerTool`，最多 6 步。
   - 只在 `SMP_AGENTIC_N` 条问题上跑，因为更慢、噪声更大。

`llm_factory` 在 [第 126–131 行](file:///home/cxy/meld-experiments/scripts/run_memdelta_local.py#L126-L131)：

- `oracle`：返回 `MockLLM([FINAL ANSWER: gold])`，无视上下文。
- 默认：`OpenWeightLLM(model, transport="vllm-openai", endpoint, max_new_tokens=256, temperature=0.0)`。

## 4. 每行记录与汇总指标

`run_one` 返回的 row 包含（[第 105–126 行](file:///home/cxy/meld-experiments/src/meld/agent/run_hotpot_memdelta.py#L105-L126)）：

- `prediction`、`gold`、`em`、`f1`
- `evidence`、`prompt`：完整持久化，便于审计和零构建重放
- `stored_units` / `live_units`：存储与活跃单元数
- `n_merge` / `n_relate` / `n_conflict` / `n_overlap_merges` / `n_total_patches`
- 有 brain 时额外记录 `recall_at_k`、`mrr`

`summarize` 输出：

- 每条件的 F1/EM/recall/MRR 均值
- 三条 headline delta（[HEADLINE_DELTAS](file:///home/cxy/meld-experiments/src/meld/agent/mem_delta_metrics.py#L136-L140)）：
  - `distributed_merge - naive_union`
  - `distributed_merge - centralized`
  - `full_context - none`
- 配对 bootstrap CI（默认 2000 次）
- merge 审计与 footprint 聚合

## 5. 校验逻辑

在 [第 160–175 行](file:///home/cxy/meld-experiments/scripts/run_memdelta_local.py#L160-L175)，只有 `fixed_recall` 存在时才校验：

1. 各条件 F1 均值不完全相同（`distinct`）
2. `distributed_merge` 至少触发 1 次 overlap merge
3. `naive_union` 的 overlap merge 必须为 0

三者都满足才输出 `LOCAL M-1 VALIDATION: OK` 并返回 0，否则返回 1。

注意：如果 `SMP_MODES=agentic`，`fr` 为 `None`，整个校验块被跳过，脚本会直接返回 OK。这是一个校验漏洞。

## 6. `SMP_MODELS` 分支

设置 `SMP_MODELS` 后会进入 [_model_axis_smoke](file:///home/cxy/meld-experiments/scripts/run_memdelta_local.py#L40-L106)，而不是正常 delta 流程：

- 对每个模型轴 key 调 `open_model()`，验证能解析到精确 HF id
- 用 `MockLLM` 跑完整条件循环
- 检查 row 里是否存在 `model_key`、`evidence`、`prompt`
- 检查三条 headline delta 都已填充

这个分支只验证“新增模型尺寸轴”所需的接线，不产生真实 F1，也不检查 overlap merge。

## 7. 实测行为

| 命令 | 结果 |
| --- | --- |
| 默认 `qwen2.5:7b` | HTTP 404，本机未安装该模型 |
| `SMP_REALMODEL=qwen3:8b SMP_N=10` | exit 0，`LOCAL M-1 VALIDATION: OK` |
| `SMP_LLM=oracle` | exit 1，`distinct=False` |
| `SMP_REALMODEL=qwen3:8b SMP_N=1` | exit 1，样本太少、条件 F1 全相同 |

`oracle` 必然失败，因为 MockLLM 无条件输出 gold，所有条件都是 F1=1.0；脚本注释也说明它只验证管线，不是能力结论。

## 8. 代码层面的几个注意点

1. **阈值没有按 encoder 校准**  
   [第 122 行](file:///home/cxy/meld-experiments/scripts/run_memdelta_local.py#L122) 调用的是 `_calibrated_cfg()`，没有传 `encoder="hashing"`。因此实际加载的是默认 `thresholds.yaml`（0.85/0.55），而不是 `thresholds-hashing.yaml`（0.49/0.29）。这与 [run_hotpot_gate.py](file:///home/cxy/meld-experiments/src/meld/agent/run_hotpot_gate.py#L45-L58) 里“per-encoder calibrated”的注释意图不一致，可能影响 merge 触发率。

2. **fixture 不是真实 HotpotQA**  
   `load_hotpot_fixture` 只有 6 条手写题，`SMP_N=10` 会循环复用；真实语料要走 `load_hotpot_real`，需要 `datasets` 和网络/缓存。

3. **本地结果不能当论文结论**  
   本地是 `hashing + lexical + 小模型`，论文 M-1 是 `minilm + 真实模型 + CSC`。这次实测中 `naive_union` 甚至高于 `distributed_merge`，说明小样本噪声很大。

4. **`SMP_MODES` 只含 `agentic` 时校验被跳过**  
   会无条件返回成功，不建议把它当作有效验证。

5. **每条 cell 都重新构造 brain**  
   `run_one` 内部重新 `build_condition`，5 条件 × 10 题 = 50 次构建，本地开销主要在这里，而不是 LLM 调用。