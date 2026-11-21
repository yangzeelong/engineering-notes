# 两个 C++ DAG 框架的同形对比：NexusFlow vs ai-pipe

> 同一台机器、同一条 6 节点链、同一个容量、同一个外部生产者。
> 结论先给：**送达吞吐差 10.8×，端到端 p50 差 5×**；但更值得记的是三条测量方法论。

## 方法

- 机器：i7-14700KF（8 P + 12 E，28 线程），WSL2 Ubuntu 20.04，GCC 11.4，Release。
- 形状：source → 4 stage → sink，容量 64，主线程做外部生产者（等价于两边的
  “外部 push” API）。
- 口径：**吞吐一律取墙钟送达**（sink 实际收到的条数 ÷ 运行墙钟）。
- 取样：NexusFlow 5 s ×（1 预热 + 3 次）取中位；ai-pipe 用我写的同形探针跑 5 s。

## 结果

| 指标 | NexusFlow | ai-pipe | 比值 |
|---|---|---|---|
| 送达吞吐 | **3,800,668 msg/s** | **353,075 帧/s**（丢 2.5%） | **10.8×** |
| 端到端 p50 | 61.25 µs | 306 µs | 5.0× |
| 端到端 p99 | 149 µs | 543 µs | 3.6× |
| 每帧 CPU | 1.52 µs | 0.65 µs（其 harness 口径） | 0.4× |
| 运行态 RSS | 4.62 MB | 31.2 MB | 6.8× |
| 丢帧 | 0（阻塞背压） | 2.5%（满队列丢最老） | — |

ai-pipe 的并发曲线是**倒 U**：2 worker 175,741 帧/s（丢 56.5%）、4 worker 353,075、
6 worker 232,307、8 worker 78,888（−78%）。NexusFlow 没有回头：6 模块 5.77 核，
16 模块 12.1 核、吞吐 1.83 M。

## 三条方法论

### 1. `items_per_second` 是 CPU 时间口径

Google Benchmark 的 `items_per_second` = items ÷ **cpu_time**。同一次运行：

```
Time 15.5 ms | CPU 1.20 ms | iterations 1000 | items_per_second = 831.169k/s
```

墙钟只有 1000 / 15.5 ms = **64.5K 帧/s**，与 831K 差 12.9×。
ai-pipe 报告里的 922K frames/s 就是这么来的；换算成墙钟要除以 1.8–2.1。

### 2. 吞吐要在“交付”端计数

ai-pipe 的 harness 报生产者提交数；满队列丢帧不计入分母。
同形探针里 1,811,693 pushed 只有 1,765,574 delivered。NexusFlow 的 `cov` 行
把 pushed/popped/discarded/residual 全部对账，`nothingLost` 才算 ok。

### 3. 平台会把“测量方法”放大成“框架缺陷”

Diamond 用 `sleep_for(50 µs)` 模拟负载。Windows 上它是 1.9 ms（定时器粒度），
Linux 上是 108 µs——同一份代码，Windows 的 Diamond 是 1.9 ms，WSL2 是 202 µs。
所以“Windows 上 2 worker 比 1 worker 慢”“922K 复现不出”“深度 8 死锁”这三条，
都是平台/工具问题，不是 ai-pipe 的设计问题。报告本身在 Linux 上可复现：
深度 1→32 的墙钟 47.8→252 µs，与报告的 88.8→546 µs 差 1.8–2.2×，正好对应
3.4/2.3 GHz 主频与 14/6 物理核。

## 优缺点（只挂证据）

**NexusFlow 强在**：墙钟吞吐与延迟（10.8× / 5×）、真背压不丢消息、
内存小（4.6 vs 31.2 MB）、深链仍能扩（16 模块 12.1 核）。
**NexusFlow 弱在**：可观测性开关极贵（`--timings` 让 3.385 M 掉到 1.235 M，
−63.5%）、融合拓扑只有非融合的 49.6%、停止要排空（cap 64 + 2 ms 慢支路 188.7 ms）、
没有丢帧语义、没有 profiler、依赖要联网取。

**ai-pipe 强在**：每帧 CPU（0.65 vs 1.52 µs）、批处理深度扩展（1→32 节点 5.3×）、
低深度延迟可接受（23–70 µs）、依赖 vendored 离线可构建、丢帧语义适合实时采集。
**ai-pipe 弱在**：流式并发负扩展（8 worker −78%）、**“高扩展”的说法不成立**、
报告没有区分两种吞吐口径。

## 教训

1. 跨框架比较先统一口径，否则差 2× 的“性能”可能只是除数不同。
2. 复现别人的 benchmark 时，先复现**形状**（趋势、拐点、放大倍数），
   再看绝对值；绝对值受主机影响，形状不受。
3. 把“平台事实”和“设计结论”分开记，否则同一条异常会在三个月后被重新发现一次。

## 复现

- NexusFlow：`--topology inject --modules 6 --capacity 64 --duration 5 --repeat 3`
- ai-pipe：`benchmarks/match/match_probe.cpp`（我加的同形探针）
- 完整报告（含 262 行逐项表）：NexusFlow 仓库 `benchmarks/PERFORMANCE-desktop.md`
