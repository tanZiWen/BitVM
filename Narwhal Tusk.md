Narwhal 和 Tusk 是基于有向无环图（DAG）的内存池（mempool）共识算法，旨在提升区块链系统的事务处理性能。以下是对 Narwhal 和 Tusk 共识算法的简要概述：
### Narwhal 内存池协议
核心思想：Narwhal 将事务（transaction）的可靠传播与事务排序分开，以实现高性能的 DAG 共识。事务传播由 Narwhal 内存池负责，而排序则交给共识协议。
设计特点：
1. DAG 结构：Narwhal 使用基于轮次的 DAG 结构存储事务的因果历史，每个节点（验证者）维护一个本地 DAG，记录事务及其依赖关系。
2. 高吞吐量：通过多个工作节点（workers）并行处理事务，Narwhal 能够线性扩展吞吐量，实验表明其在广域网（WAN）上可达 600,000 tx/sec（事务每秒）而无显著延迟增加。
3. 异步网络适应性：Narwhal 能容忍异步网络环境，即使在网络故障或延迟情况下仍保持高性能。
4. 容错性：假设系统中节点总数为 ( N )，其中最多 f < N/3个节点为拜占庭节点，Narwhal 仍能保证事务的可靠传播。
5. 性能：与部分同步共识协议（如 HotStuff）结合（称为 Narwhal-HotStuff），Narwhal 在 WAN 上可实现 170,000 tx/sec，延迟约 2.5 秒，而单独的 HotStuff 仅为 1,800 tx/sec，延迟 1 秒。
### Tusk 共识协议
核心思想：Tusk 是 Narwhal 的异步共识扩展，旨在解决 Narwhal-HotStuff 在异步网络中可能出现的活跃性（liveness）丢失问题。Tusk 通过零消息开销的机制实现高效的异步共识。
设计特点：
1. 基于随机币机制：Tusk 在 Narwhal 的 DAG 结构上增加随机币（random coin）机制，用于确定事务的全局排序。每个验证者在块中包含随机信息，通过分布式随机性生成协议（如阈值签名）达成一致。
2. 三轮波次（Wave）：Tusk 将 DAG 划分为波次，每波次包含三轮：
  1. 提议轮：验证者提议自己的块，包含事务及其因果历史。
  2. 投票轮：验证者通过在自己的块中引用其他提议来投票。
  3. 随机性轮：验证者生成随机性以选举一个领导者块，确定排序。
  4. 零消息开销：Tusk 的共识过程完全基于本地 DAG 的解释和共享随机性，无需额外通信。
  5. 低延迟与高吞吐量：Tusk 在 WAN 上可达 160,000 tx/sec，延迟约 3-4 秒，相比其他异步协议（如 DAG-Rider）性能提升约 20 倍。
  6. 理论基础：Tusk 基于 DAG-Rider 协议，改进了其实用性和延迟表现，同时继承了其安全性保证。DAG-Rider广播完整的区块信息，Tusk广播区块头信息。
### Narwhal 和 Tusk 的结合
1. Narwhal 提供数据可用性：Narwhal 负责高效的事务传播和存储，确保所有诚实验证者最终看到相同的 DAG。
2. Tusk 提供排序：Tusk 在 Narwhal 的 DAG 上运行，通过随机性机制实现事务的全局排序，适合完全异步环境。
3. 容错与性能：在存在故障（如拜占庭节点或网络异步）的情况下，Narwhal 和 Tusk 均能维持高吞吐量。Narwhal-HotStuff 在部分同步环境中延迟可能略高，而 Tusk 在异步环境中表现更稳定。
**应用与开源**
1. Sui 区块链：Narwhal 和 Tusk 被用于 Sui 智能合约平台，其中 Narwhal 作为内存池，结合 Bullshark（Tusk 的部分同步变体）或 Mysticeti（基于 Narwhal-Tusk 优化的新协议）作为共识引擎。
2. 开源贡献：Mysten Labs 于 2022 年 4 月宣布 Narwhal 和 Tusk 开源，代码托管在 GitHub，采用 Apache 2.0 许可证，鼓励 Web3 社区复用其技术。

**性能数据（基于实验）**
1. Narwhal-HotStuff：130,000-170,000 tx/sec，延迟 < 2-2.5 秒
2. Tusk：140,000-160,000 tx/sec，延迟约 3-4 秒。
3. 扩展性：增加工作节点后，吞吐量可线性提升至 600,000 tx/sec。
4. 对比：优于传统协议（如 HotStuff 或其他异步协议）数倍，尤其在高负载和故障场景下。

**局限性**
1. 延迟权衡：Tusk 的异步设计在高吞吐量下可能导致较长的确认延迟（3-4 秒），相比部分同步协议（如 HotStuff 的 1 秒）。
2. 复杂性：DAG 结构的维护和垃圾回收（garbage collection）需额外优化，以避免内存占用过高。
3. 公平性：Tusk 的 DAG 清理机制可能导致较慢节点的块被丢弃，影响块级公平性。

**后续发展**
1. Bullshark：Narwhal 和 Tusk 的部分同步变体，优化了延迟和性能，广泛应用于 Sui 区块链。
2. Mysticeti：Sui 最新引入的基于 DAG 的 BFT 共识协议，继承 Narwhal-Tusk 的优势，进一步降低延迟（< 1 秒）并提升吞吐量（测试中达 400,000 tx/sec）。
### 总结
Narwhal 和 Tusk 通过分离事务传播与排序，结合 DAG 结构和零消息开销的异步共识，显著提升了区块链共识的吞吐量和容错能力。Narwhal 提供高吞吐量的事务传播，Tusk 则保证异步环境下的活跃性和排序效率。其开源实现和在 Sui 等平台的应用展示了其在 Web3 领域的潜力，但延迟和公平性仍需进一步优化。
如需更详细的技术分析或代码实现，请参考以下资源：
论文：《Narwhal and Tusk: A DAG-based Mempool and Efficient BFT Consensus》（arXiv:2105.11827）

GitHub 仓库：https://github.com/mystenlabs/narwhal[](https://github.com/Zclim/consensus)

Sui 文档：https://docs.sui.io/[](https://docs.sui.io/devnet/learn/architecture/consensus)
