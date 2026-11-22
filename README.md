# engineering-notes

> yangzeelong 的工程笔记：性能、平台差异、以及“我们量过什么、怎么量的”。

这里只放**代码变了依然成立**的东西：复盘、方法论、跨项目对比。
会随代码一起变的参考手册放在各自仓库里（例如
[NexusFlow docs/](https://github.com/yangzeelong/NexusFlow/tree/master/docs)）。

## 文章

| 日期 | 标题 | 主题 |
|---|---|---|
| 2026-11-22 | [线程池之前和之后，到底差多少](posts/2026-11-22-pool-before-after.md) | 池化 vs 串行、三个口径、WSL2 的两态陷阱 |
| 2026-11-21 | [为什么 Windows 上“睡 10 微秒”其实睡 2 毫秒](posts/2026-11-21-windows-timer-granularity.md) | 定时器粒度、自旋 vs 睡眠、一个 36% 的吞吐 bug |
| 2026-11-20 | [两个 C++ DAG 框架的同形对比：NexusFlow vs ai-pipe](posts/2026-11-20-nexusflow-vs-ai-pipe.md) | 口径统一、复现方法、10.8× 差距的归因 |

## 写作约定

- 每个数字都能追到一次运行：给命令、给口径、给重复次数。
- 不确定的标“未确认”，不猜。
- 平台事实（版本、拓扑、工具可用性）单独写一节，因为它们是会过期的。

## 链接

- 个人站点（项目介绍）：待建
- NexusFlow：https://github.com/yangzeelong/Nexusflow
