# 为什么 Windows 上“睡 10 微秒”其实睡 2 毫秒

> 一个 36% 的吞吐损失，根因是一行 `sleep_for`。记录症状、测量、机制和修复。

## 症状

给 NexusFlow 加了一个可选的模块线程池（v1.1 的 L2：多个模块共享一条线程，
模块固定绑定，计数器仍是单写者）。同一台机器：

| 配置 | 吞吐 |
|---|---|
| 每模块一线程（p0） | 3.267 M msg/s |
| 4 条池线程（p4） | 2.087 M msg/s |

p4 只有 p0 的 **68%**，但 CPU 占满 4 核。第一反应是“池的调度开销”，
于是量了一圈：轮次、turns、空转，最后发现**线程有 5% 的轮次在睡眠，每次睡 2 毫秒**。

## 测量

800 行 C++ 的 microbenchmark，跑四组：

| 调用 | 实测平均 |
|---|---|
| `Sleep(1)`（有 `timeBeginPeriod(1)`） | **1.907 ms**（min 1.003 / max 2.508） |
| `sleep_for(10µs)`（无 period） | **15.374 ms**（min 7.308 / max 16.451） |
| `sleep_for(10µs)`（有 period） | **1.907 ms** |
| `sleep_for(50µs)`（有 period） | 1.907 ms |

同一台机器在 WSL2 里，`time.sleep(10e-6)` 是 **68.3 µs**——差了 28 倍。

## 机制：三层，缺一层都不成立

1. **计价单位**。Win32 只有 `Sleep(DWORD milliseconds)`，没有亚毫秒参数。
   MSVC 的 `std::this_thread::sleep_for` 先 `chrono::ceil<milliseconds>(d)`，
   再 `_Thrd_sleep_for(ms)` → `::Sleep(ms)`。**任何正的亚毫秒时长都变成 `Sleep(1)`。**
2. **tick 对齐**。`Sleep(1)` 不在 1 ms 后醒来，而是按系统 tick 醒来。
   默认定时器分辨率 **15.625 ms**，`timeBeginPeriod(1)` 之后是 1 ms。
3. **没有唤醒源**。池线程停泊后只有超时能叫醒它（`push` 不唤醒池线程），
   所以一次“以为 10 µs”的停泊 = 这一级流水线停一个 tick。

前两层是平台事实，第三层是我自己的设计缺陷：退避阶梯用了时间单位。

## 修复

空闲退避从“时间”改成“轮数”：

```cpp
// ThreadPool::Loop(), idle 分支
if (spins < 4096)        { ++spins; Pause(); }   // PAUSE，约 50–70 µs
else if (spins < 8192)   { ++spins; std::this_thread::yield(); }
else                     { std::this_thread::sleep_for(1ms); }
```

为什么是 8192 轮：整链 3 M msg/s、每消息过 3 条边 → 单边到达间隔约 0.33 µs；
突发间隔在亚微秒到几微秒。64 轮（约 0.5 µs）会掉进 sleep，8192 轮（约 50–70 µs）
比稳态间隔大两个数量级，又能在真正空闲时把核让出去。

结果（5 s × 3 中位，MSVC）：

| 配置 | 修复前 | 修复后 |
|---|---|---|
| L4cap16，p4 | 2.221 M | **3.019 M**（+36%） |
| L4cap16，p4/p0 | 68.0% | **93.6%** |
| L16cap8，p4 | 0.980 M | **1.411 M**（+44%） |
| 周期/消息（L4cap16-p4） | 5,526 | **4,485** |

## 为什么不用条件变量

第一反应是“让生产者唤醒消费者”。做了，吞吐腰斩：spin 2.069 M → cv 1.087 M。
池里一次交接间隔几十纳秒到几微秒，挂起/唤醒的内核往返比自旋贵，而且唤醒源
本身要加原子和通知。**在这里，自旋是正确选择，只是不能睡。**

## 教训

1. **“睡 10 微秒”这种话必须实测**。语言/STL 的单位是毫秒，不是微秒。
2. **空闲计数器要数轮数，不要数时间**。我最初看到 idle 只占 5.3% 就排除了它——
   但 5.3% 的轮次每次付 2 ms，相当于 10% 的 CPU 时间在停泊。
3. **off-CPU 火焰图只有一根柱时，那根柱就是答案**。我第一眼读成“正常退避”，
   直到把它和 wall 吞吐对上才改口。
4. **平台差异要沉淀成事实文档**，不要每次重新发现：
   [docs/platform/windows.md](https://github.com/yangzeelong/Nexusflow/blob/master/docs/platform/windows.md)。
5. 微优化之前先确认宏观瓶颈。我先砍了三刀（`OutputRoom` 扫描、平表、batch scratch），
   收益都在 ±5% 噪声里；真正的一刀在这里。

## 复现

NexusFlow 仓库（`perf/pool-contiguous` 之后的 `bugfix/pool-idle-backoff`）：

```bash
# Windows / MSVC
build/benchmarks/Release/pipeline_benchmark.exe \
  --topology linear --modules 4 --capacity 16 --pool-size 4 \
  --cpus 8 --duration 5 --repeat 3
```

修复前的对照二进制留在 `pb_dispatch_before.exe`，修复后是 `pb_fixed.exe`，
A/B 用交替顺序跑，避免机器状态漂移。
