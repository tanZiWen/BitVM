#### Synchronize模块
- 代码功能概述
  - 目的：Synchronizer 是内存池的一部分，负责处理共识层发送的 ConsensusMempoolMessage，确保本节点的内存池包含所有需要的批次（Batch）。如果缺失批次，它会向其他节点请求数据，并管理同步的超时和重试逻辑。
  - 主要功能：接收共识层的消息（Synchronize 和 Cleanup）。
  - 处理 Synchronize 消息，向目标节点请求缺失的批次。
  - 如果请求超时，广播请求到随机选择的节点（sync_retry_nodes）。
  - 处理 Cleanup 消息，移除过时的批次请求（基于轮次 Round 和垃圾回收深度 gc_depth）。
  - 使用存储（Store）检查批次是否到达，并通知共识层。

- 技术栈：
  - 异步运行时：tokio 用于异步任务、定时器和通道。
  - 网络通信：SimpleSender（自定义网络模块）发送请求。
  - 存储：Store 用于持久化批次数据。
  - 序列化：bincode 序列化 MempoolMessage。
  - 日志：log 记录调试和错误信息。
  - 并发：FuturesUnordered 管理多个异步等待任务。

#### QuorumWaiter模块
- 代码功能概述
  - 目的：保证多数人能接收到交易的batch信息 2f+1，并添加了一个等待慢节点 500 ms的逻辑

#### BatchMaker模块
- 代码功能概述
  - 目的： 接收交易，并将交易打包成一个batch，设定如果到达batch上线或者等待时间到，则将batch交易广播出去。

#### Helper模块
- 代码功能概述
- 目的：处理其他节点发送的batch请求，从本地拿到digest数据，并发送给请求节点
  
#### Mempool模块
- 代码功能概述
- 目的：1.处理共识消息； 2.处理客户端发来的交易； 3.处理mempool消息
- 主要功能描述：
  - 共识消息处理：通过 Synchronizer 模块处理共识相关的消息。
  - 客户端交易处理：涉及以下模块的协同工作：
    - NetworkReceiver 模块：负责接收客户端发送的交易，并通过一个通道（channel）将交易传递给 BatchMaker 模块。
    - BatchMaker 模块：接收交易后进行批次处理，完成后通过另一个通道发送 QuorumWaiterMessage 消息给 QuorumWaiter 模块。
    - QuorumWaiter 模块：等待至少 2f+1 个节点确认收到消息后，通过通道向 Processor 模块发送消息。
    - Processor 模块：接收消息后，将批次交易信息存储到本地，并将批次完成的消息发送给共识模块。
    - 模块间协作：
      - NetworkReceiver → BatchMaker：通过通道传递消息。
      - BatchMaker → QuorumWaiter：通过通道发送 QuorumWaiterMessage。
      - QuorumWaiter → Processor：通过通道发送确认消息。
      - Processor → 共识模块：发送批次完成消息。
- mempool消息处理
  
