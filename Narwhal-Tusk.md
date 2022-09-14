# Narwhal 和 Tusk：基于 DAG 的内存池和高效的 BFT 共识

George Danezis Mysten Labs & UCL.  Alberto Sonnino Mysten Labs

Lefteris Kokoris-Kogias IST Austria. Alexander Spiegelman Aptos

## 摘要

我们建议将可靠的交易传播任务与交易排序分开，以实现高性能的拜占庭容错基于仲裁的共识。
我们设计并评估了一个 mempool 协议 Narwhal，该协议专门用于高吞吐量可靠传播和交易因果历史的存储。 
Narwhal 可以容忍异步网络并在出现故障的情况下保持高性能。 Narwhal 的设计目的是在每个验证节点上使用多个工作线程轻松扩展，我们证明我们可以实现的吞吐量没有可预见的限制。

将 Narwhal 与部分同步的共识协议 (Narwhal-HotStuff) 组合在一起，即使在存在故障或由于异步导致的间歇性失去活力的情况下，也能产生明显更好的吞吐量。
但是，失去活性会导致更高的延迟。为了在发生故障时获得整体良好的性能，我们设计了 Tusk，一个零消息开销的异步共识协议，与 Narwhal 一起工作。我们展示了它在各种配置和故障下的高性能。

作为结果总结，在 网络上，Narwhal-Hotstuff 在不到 2 秒的延迟下实现了超过 130,000 次/秒的传输，而 Hotstuff 在 1 秒的延迟下实现了 1,800 次/秒的传输。
额外的工作人员将吞吐量线性增加到 600,000 tx/秒，而没有任何延迟增加。
Tusk 以大约 3 秒的延迟达到 160,000 tx/秒。在故障情况下，两种协议都保持高吞吐量，但 Narwhal-HotStuff 的延迟增加。

