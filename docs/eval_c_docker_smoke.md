# eval-c Docker Smoke 测试解析

> 脚本：[scripts/eval_c_docker_smoke.py](file:///home/cxy/meld-experiments/scripts/eval_c_docker_smoke.py)
> 对应论文章节：§7.4（Convergence / partition heal）+ Table 1 底部
> 服务端点：[src/meld/sync/service.py](file:///home/cxy/meld-experiments/src/meld/sync/service.py)
> 正式多主机实验：[edge/run_eval_c.py](file:///home/cxy/meld-experiments/edge/run_eval_c.py)

## 1. 一句话定位

这是一个**容器级的 wire 层 CRDT 收敛 smoke 测试**：启 3 个独立的 `meld.sync.service` 容器，让一条 claim 在分区期间收到相互矛盾的 overrule link（状态分化），愈合后让每个节点同时收到两条 link，验证 **Status CRDT 收敛到同一个最终状态，顺序无关**。

它不是性能实验，不是大规模分区，也不是真实的网络 tc 注入——它只验证一件核心的事：**CRDT join 算子在 HTTP wire 上正确工作，不会像 last-writer-wins 那样丢失最终状态**。

## 2. 为什么用容器，不用 `BrainNode` 直接构造

仓库里已经有两个纯进程内的 CRDT 收敛测试：

| 测试 | 方式 | 位置 |
| --- | --- | --- |
| `test_status_crdt_converges_order_independently_across_nodes` | `BrainNode` 直接构造、内存里 apply status | tests/test_brain_node.py |
| `run_eval_c_inprocess.py` | 多 seed、多顺序，纯进程内 | scripts/run_eval_c_inprocess.py |

但这两者都绕过了真实的网络边界——没有 HTTP 序列化/反序列化、没有独立进程的内存隔离、没有 service 层的路由/端点。eval_c_docker_smoke 的存在理由是：

1. 验证 `BrainService.handle()` 的 `POST /status` 和 `GET /metrics` 端点确实把 CRDT add/join 暴露到了网络层。
2. 验证进程隔离不会破坏 CRDT 的交换律/幂等性（两个独立 BrainNode 实例 apply 同一组 link，结果应相同）。
3. 作为 edge/run_eval_c.py 的**前置烟雾测试**——正式多主机实验要 SSH + sudo tc 注入真实分区，成本高、排错难，先过掉这个 smoke 再跑大规模。

## 3. 测试脚本逐段解析

### 3.1 固定布局

```python
URL = {"A": "http://127.0.0.1:8801",
       "B": "http://127.0.0.1:8802",
       "C": "http://127.0.0.1:8803"}
CLAIM = "the canonical policy is X"
LINKS = [("A", "La", "DEPRECATED"),
         ("C", "Lb", "OVERRULED")]   # 两条 link 互相矛盾
```

- 三个固定端口，没有 docker-compose 文件——脚本假设用户自己 `docker run` 起三个容器。
- 两条 link 是**相互矛盾**的：一条把 CLAIM 置为 DEPRECATED，另一条置为 OVERRULED。最终 CRDT 的 lattice join 应该取两者的交叠（两者都是 ACTIVE 的覆盖/废弃操作，OVERRULED ≥ DEPRECATED），得到一致的最终状态。
- 两条 link 被分到不同侧：`La` 只发给 A，`Lb` 只发给 C，B 什么都不发——这就是"分区"。

### 3.2 等待容器就绪

```python
for d in range(40):
    try:
        if all(urllib.request.urlopen(URL[n] + "/healthz", timeout=3).status == 200
               for n in URL):
            break
    except Exception:
        time.sleep(2)
```

最多 40 × 2 = 80 秒等待 `/healthz` 就绪。`/healthz` 端点返回 `{"status": "ok"}`（见 service.py#L151-L152），纯存活探针，不检查任何脑状态。

### 3.3 共同 ingest 初始 claim

```python
for n in URL:
    _post(URL[n] + "/ingest", CLAIM.encode())
```

每个容器都先 POST `/ingest`，把 CLAIM 作为一个新 observation 合并进各自的 brain（走 `BrainNode.ingest` → `SMPMemory.write` → `merge_on_receive` → `Brain.apply(PATCH_INSERT)`）。此时三节点状态一致，都是 ACTIVE。

### 3.4 分区阶段（让状态分化）

```python
for node, lid, eff in LINKS:
    _post(URL[node] + "/status",
          json.dumps({"claim": CLAIM, "link_id": lid, "effect": eff}).encode())

during = _agreement()
```

这里"分区"不是真的网络断线，而是**有选择地只把 status link 发给部分节点**：

| 节点 | 收到的 link | 对 CLAIM 的有效状态 |
| --- | --- | --- |
| A | `La: DEPRECATED` | DEPRECATED |
| B | （无） | ACTIVE（没变） |
| C | `Lb: OVERRULED` | OVERRULED |

三节点状态 = `{DEPRECATED, ACTIVE, OVERRULED}`，模态占比最多 1/3（取决于具体实现），`agreement` < 1.0。脚本断言：

```python
if during >= 1.0:
    print("FAIL: expected divergence during the partition")
    sys.exit(1)
```

如果分区阶段 agreement 已经是 1.0，说明"分区"没有生效，或者三节点被某种意外同步过了。

### 3.5 愈合阶段（让 CRDT join 收敛）

```python
for n in URL:
    for _node, lid, eff in LINKS:
        _post(URL[n] + "/status",
              json.dumps({"claim": CLAIM, "link_id": lid, "effect": eff}).encode())

healed = _agreement()
```

现在**每个节点都收到两条 link**，每条 POST `/status` 都会调 `BrainNode.apply_status` → `StatusState.add(link_id, effect)`。Status CRDT 内部用 (link_id, effect) 作为唯一键，幂等——重复 apply 同一条 link 不会改变状态。

三个节点内部做的 lattice join：

```
A: apply(La, DEPRECATED) → apply(Lb, OVERRULED)
B: apply(La, DEPRECATED) → apply(Lb, OVERRULED)
C: apply(La, DEPRECATED) → apply(Lb, OVERRULED)
```

三条路径虽然顺序不同，但 CRDT 的 join 是结合律 + 交换律的，所以结果必须完全相同。脚本断言：

```python
if healed != 1.0:
    print("FAIL: status-CRDT did not reconverge after heal")
    sys.exit(2)
```

## 4. CRDT 的状态格是什么样子

`meld/merge/status_crdt.py` 定义的 Effect lattice：

```
ACTIVE
  │
  ├── DEPRECATED     ← 废弃
  │
  └── OVERRULED      ← 覆盖（比 DEPRECATED "更强"）
```

每个 claim 维护一个 `StatusState`，存所有对它的 link（link_id → effect）。`effective_status()` 取所有 effect 的**最大上界**：

- 只有 ACTIVE → ACTIVE
- 有 DEPRECATED 没有 OVERRULED → DEPRECATED
- 有 OVERRULED → OVERRULED（因为 OVERRULED > DEPRECATED）

所以无论 `La` 和 `Lb` 以什么顺序到达同一个节点，只要两条都到了，最终 effective status 就是 OVERRULED。这就是 CRDT 的顺序无关性。

## 5. 关键 HTTP 端点

| 端点 | 方法 | 作用 | smoke 里用在哪 |
| --- | --- | --- | --- |
| `/healthz` | GET | 容器存活 | 就绪等待 |
| `/ingest` | POST | 合并一条 local observation 进 brain | 初始状态 |
| `/status` | POST | 对某个 claim apply 一个 status link | 分区注入 + 愈合注入 |
| `/metrics` | GET | 返回 node_id、live claims、所有 claim 的当前 effective status | 读 agreement |
| `/publish` | POST | publish-once delta 到 peers（需要 transport seam） | smoke 没用 |
| `/patch` | POST | 接收 peer 的 signed typed delta（wire merge） | smoke 没用 |
| `/resync` | POST | 反熵全量重同步 | smoke 没用 |
| `/reset` | POST | 清空 brain + latency meter | smoke 没用 |

eval_c_docker_smoke **只用了** `/healthz`、`/ingest`、`/status`、`/metrics` 四个端点。`/publish`、`/patch`、`/resync` 是给 §7.3 的真实 peer-to-peer 同步用的，这个 smoke 不跑 peer-to-peer。

## 6. 和正式多主机实验 `edge/run_eval_c.py` 的对比

| 维度 | eval_c_docker_smoke | edge/run_eval_c.py |
| --- | --- | --- |
| 拓扑 | 3 容器（本地端口） | 多主机（SSH + IP 表） |
| 分区方式 | 不发 cross-side `/status`（delivery 级） | `tc` 丢包 100%（网络级），或同样 delivery 级 |
| 分区持续时间 | 瞬时（注入完立即愈合） | 可配置 `partition_s` |
| seed 数量 | 1 | 30 |
| 交付顺序 | 固定 | seeded 随机顺序 × per-brain |
| 度量 | 只有 agreement 1.0 vs 不是 | CRDT 收敛 + 分析性 last-writer-wins 反事实 |
| 依赖 | 本地 Docker | SSH + sudo tc + testbed.yaml + brain 栈 |
| 失败影响 | 小——只是 wire smoke | 大——正式论文数字 |

`run_eval_c_inprocess.py` 又比 smoke 更纯：完全不启 HTTP 服务，直接在内存里用 `BrainNode` 构造多节点，唯一差别是没有独立进程隔离。

所以三层测试的依赖成本是：**inprocess（最低）→ docker smoke（中等）→ edge testbed（最高）**，覆盖范围互补。

## 7. 怎么跑

### 7.1 前置条件

- Docker（版本 ≥ 20），`docker buildx` 可选
- 端口 8801、8802、8803 未被占用（脚本写死了这三个）
- 仓库路径：`/home/cxy/meld-experiments`

**镜像构建的一个坑**：`edge/docker/Dockerfile.meld` 里 `COPY edge/docker/requirements-smoke.txt .`，但这个文件**没有随仓库发布**——仓库里只有 `Dockerfile.meld-minilm` 和它需要的 `requirements-minilm.txt`。构建时会直接报错：

```
COPY failed: file not found in build context or excluded by .dockerignore
```

两种修法任选其一：

**方案 A（最快）**：直接在 Dockerfile 里把 COPY + RUN 行注释掉，用容器基础镜像自带的 pip 空跑——因为 smoke 只用到 Python 标准库（stdlib http.server、json、urllib），不需要 `sentence-transformers`、`requests` 等三方包。

```bash
cd /home/cxy/meld-experiments
# 临时改 Dockerfile（注释掉 requirements COPY + RUN）
sed -i '/^COPY edge\/docker\/requirements-smoke.txt/,/^RUN pip install --no-cache-dir -r requirements-smoke.txt/d' edge/docker/Dockerfile.meld
docker build -f edge/docker/Dockerfile.meld -t meld-brain:smoke .
```

**方案 B（更干净）**：创建一个空的 `requirements-smoke.txt`（文件存在，pip install 无依赖），不改 Dockerfile。

```bash
cd /home/cxy/meld-experiments
touch edge/docker/requirements-smoke.txt
docker build -f edge/docker/Dockerfile.meld -t meld-brain:smoke .
rm edge/docker/requirements-smoke.txt   # 用完可以删
```

### 7.2 构建镜像

```bash
cd /home/cxy/meld-experiments
docker build -f edge/docker/Dockerfile.meld -t meld-brain:smoke .
```

预期输出尾部：

```
Successfully tagged meld-brain:smoke
```

镜像大小约 200MB（python:3.11-slim + iproute2 + curl + 仓库 src/）。

### 7.3 启动三个容器

**serve_main 必需的 4 个环境变量（无静默默认，缺一个容器启动就会 ValueError）**：

| 变量 | 说明 | smoke 怎么设 |
| --- | --- | --- |
| `SMP_NODE_ID` | 节点身份，写入 BrainNode.node_id | `A`、`B`、`C` |
| `SMP_PORT` | 容器内服务端口 | 固定 `8800`（Dockerfile 里已 ENV） |
| `SMP_ENCODER` | 相似度编码器档位 | smoke 用 `hashing`（快，无模型）；正式 campaign 用 `minilm` |
| `SMP_NLI` | NLI 信号档位 | `lexical`（纯词法，无 cross-encoder） |

**可选但建议显式设置的**：

| 变量 | 默认值 | smoke 怎么设 |
| --- | --- | --- |
| `SMP_ROUTING_MODE` | `broadcast` | `broadcast`（smoke 不跑 publish 也不影响） |
| `SMP_PEERS` | 空串 | 空（smoke 不跑 `/publish`，不跨节点） |
| `SMP_SIGMA_LO` | `0.0` | 用默认 |
| `SMP_TOP_K` | 未设 | 不设 |
| `SMP_HTTP_TIMEOUT` | `10.0` | 用默认 |

关键观察：smoke 脚本**完全不碰** `/publish`、`/patch`、`/resync` 端点，所以不需要 `SMP_PEERS`、`SMP_TRANSPORT`、真实 Kafka broker、也不需要 `minilm` 编码器。三个容器之间的"分区"完全由脚本通过有选择地发 `/status` 实现。

**一键启动命令**：

```bash
# 先确保没有残留容器
docker rm -f brainA brainB brainC 2>/dev/null || true

# 启动
docker run -d --name brainA -p 8801:8800 \
  -e SMP_NODE_ID=A -e SMP_PORT=8800 -e SMP_ENCODER=hashing -e SMP_NLI=lexical \
  -e SMP_ROUTING_MODE=broadcast meld-brain:smoke

docker run -d --name brainB -p 8802:8800 \
  -e SMP_NODE_ID=B -e SMP_PORT=8800 -e SMP_ENCODER=hashing -e SMP_NLI=lexical \
  -e SMP_ROUTING_MODE=broadcast meld-brain:smoke

docker run -d --name brainC -p 8803:8800 \
  -e SMP_NODE_ID=C -e SMP_PORT=8800 -e SMP_ENCODER=hashing -e SMP_NLI=lexical \
  -e SMP_ROUTING_MODE=broadcast meld-brain:smoke
```

三个 `-p 端口映射` 把容器内 8800 分别映射到宿主机 8801/8802/8803，匹配脚本里硬编码的 `URL = {"A": "http://127.0.0.1:8801", ...}`。

**验证容器健康**：

```bash
# 看容器是否在运行
docker ps --filter name=brain

# 看容器启动日志（serve_main 是否正常 serve_forever）
docker logs brainA     # 预期无报错，静默等待请求

# 直接 curl 端点
curl -s http://127.0.0.1:8801/healthz   # 预期 {"status":"ok"}
curl -s http://127.0.0.1:8802/healthz
curl -s http://127.0.0.1:8803/healthz
```

如果容器没起来，`docker logs` 里大概率会是 `ValueError: SMP_NODE_ID is required (no silent default)` 或类似缺 env 报错——4 个必需变量一个都不能少。

### 7.4 跑 smoke

```bash
cd /home/cxy/meld-experiments
PYTHONPATH=src python scripts/eval_c_docker_smoke.py
```

预期输出：

```
  during partition: agreement=0.33 ({'A': 'DEPRECATED', 'B': 'ACTIVE', 'C': 'OVERRULED'})
  after heal:       agreement=1.00 ({'A': 'OVERRULED', 'B': 'OVERRULED', 'C': 'OVERRULED'})
EVAL-C SMOKE OK (CRDT reconverged over the wire)
```

### 7.5 手动检查容器内部状态（可选）

smoke 跑完后可以直接 curl `/metrics` 看最终状态字段，验证 CRDT 收敛：

```bash
# 三个节点的 status 字段应该全是 OVERRULED
curl -s http://127.0.0.1:8801/metrics | python -m json.tool | grep "the canonical policy"
curl -s http://127.0.0.1:8802/metrics | python -m json.tool | grep "the canonical policy"
curl -s http://127.0.0.1:8803/metrics | python -m json.tool | grep "the canonical policy"
```

也可以 `docker exec` 进容器直接调 Python：

```bash
docker exec -it brainA python -c "
import os; os.environ.setdefault('PYTHONPATH', '/app/src')
from meld.sync.node import BrainNode
print('node already running as HTTP service via serve_main, Python console here is idle')
"
```

### 7.6 清理

```bash
# 停并移除容器（保留镜像）
docker rm -f brainA brainB brainC

# 彻底清理（可选，同时删镜像）
docker rm -f brainA brainB brainC && docker rmi meld-brain:smoke
```

### 7.7 常见启动失败

| 现象 | 原因 | 排查 |
| --- | --- | --- |
| 容器起来就退出，`docker logs` 报 `ValueError: SMP_NODE_ID is required` | 4 个必需 env 缺一个 | 检查 `docker run -e` 参数拼写 |
| 容器运行中但 `curl /healthz` 连接拒绝 | `-p` 映射端口冲突，或容器内服务端口被改 | `docker ps` 看映射；容器内 `cat /proc/net/tcp` 确认监听 |
| smoke 卡在"等待健康检查"40 次循环 | 容器没起来或启动报错 | `docker logs brainA` 看 traceback |
| `BUILD` 时报 `COPY failed: file not found` | `requirements-smoke.txt` 不存在 | 用 7.1 节的方案 A 或 B |
| smoke 通过但 `metrics` 里三个节点 status 不同 | 分区愈合阶段 `/status` 没发全 | 看 smoke 输出，退出码 1 表示分区阶段就没分化，退出码 2 表示愈合后没收敛 |

## 8. 已知局限

1. **没有真实网络分区**："分区"只是选择性地不发 `/status`，不验证 CRDT 在真丢包/乱序下的行为。想做这个要跑 `edge/run_eval_c.py` 加 `partition.mode: tc`。
2. **只有一条 claim + 两条 link**：规模太小，不覆盖多 claim 并发、merge 时带 status、或 link 本身 OVERRULED 另一个 link（link-of-link）的情况。
3. **不跑 `/publish` 或 `/patch`**：smoke 完全没测 publish-once delta 跨节点交付，也没测 signed typed delta 的验签 admission。这两个端点是给 §7.3 的 sync-latency 用的，eval_c 系列只关心 status CRDT。
4. **没有 docker-compose 文件**：每次手动 `docker run` 很繁琐，也容易把端口/环境变量配错。仓库没有提供 compose 模板，可能是为了保持 "dependency-lean" 的 public-bound posture。
5. **固定端口/节点名/claim/link**：全部写死在脚本里，不支持命令行覆盖；要换 claim 或 effect 组合得改源码。

## 9. 退出码含义

| 退出码 | 含义 |
| --- | --- |
| 0 | 愈合后 agreement = 1.0，CRDT 收敛 |
| 1 | 分区阶段 agreement 已经是 1.0，分区没生效 |
| 2 | 愈合后 agreement 仍 < 1.0，CRDT 没有收敛 |

三种退出码覆盖了"分区注入故障"、"CRDT 故障"、"正常通过"三种情况。
