### 1. 背景与目标
HotStuff 的设计目标是解决传统 BFT 协议（如 PBFT）和区块链共识协议（如 Tendermint、BFT-SMaRt）的以下局限性：
1. 高通信复杂度：PBFT 的正常情况通信复杂度为O(n^2)，在大型网络中开销大。
2. 高延迟：许多 BFT 协议需要多轮消息交换，导致提交延迟较高。
3. 视图切换复杂性：传统协议的视图切换（领导者更换）可能需要复杂的状态同步，影响性能。
4. 区块链适配性：早期 BFT 协议未针对区块链场景优化，无法高效支持高吞吐量和动态参与。
   
**HotStuff 的核心贡献是：**
1. 线性通信复杂度：在正常情况下（无故障）和视图切换时均保持 O(n)通信复杂度。
2. 低延迟：通过流水线设计，优化提交延迟。
3. 简单性与鲁棒性：结合 PBFT 的鲁棒性和 Tendermint 的简单性，支持部分同步网络。
4. 区块链适用性：支持区块链的链式结构，适合许可（permissioned）区块链。

### 2. HotStuff 的核心设计原理

HotStuff 的设计围绕以下关键机制展开：

**(1) 部分同步网络模型**

假设：HotStuff 运行在部分同步网络中，即存在一个未知的全局稳定时间（GST），在 GST 之后消息延迟有界（不超过 Δ）。
1. 系统包含n个节点，最多 f<3/n个节点是拜占庭节点（恶意或故障）。
2. 通信通道可靠且经过认证，恶意节点无法伪造诚实节点的签名。
   
**意义：**

1. 部分同步模型接近现实网络（如互联网），比完全同步模型更实用。
2. 保证安全性（不提交冲突区块）和活性（在 GST 后进展）。
   
**(2) 领导者驱动的共识**

1. 领导者角色：HotStuff 采用领导者驱动模型，每个视图（view）由一个领导者负责提议区块。
2. 领导者通过轮询机制确定（例如，leader=view mod n），确保公平性。
3. 视图（View）：时间划分为多个视图，每个视图有一个唯一的视图编号（view number，单调递增）。
4. 视图切换发生于领导者故障或超时，节点进入新视图并选举新领导者。
   
**(3) 流水线（Pipelining）设计**

1. 核心思想：HotStuff 将共识过程分解为多个阶段（提议、投票、提交），并通过流水线并行处理多个区块。
2. 每个区块的投票不仅认证当前区块，还为前一个区块的提交提供支持，形成链式依赖（chaining）。
   
**三阶段协议：**

HotStuff 的单区块确认分为三个逻辑阶段（但通过流水线合并为一轮通信）：
1. Prepare：领导者提议区块，节点验证并投票，形成 Prepare QC（法定人数证明，包含 2f+1个签名）。
2. Pre-commit：节点收到 Prepare QC，投票形成 Pre-commit QC。
3. Commit：节点收到 Pre-commit QC，投票形成 Commit QC，区块被提交。
4. 流水线化：每个阶段的投票通过单轮消息交换完成，下一区块的 Prepare 阶段与当前区块的 Pre-commit/Commit 阶段并行。

### 核心组件

HotStuff 的实现依赖以下核心组件：
- 节点（Replica）：系统中运行共识协议的实体，每个节点有唯一标识（ID）。
- 领导者（Leader）：负责提议区块并协调投票，领导者在视图（View）中轮换。
- 视图（View）：共识的一个轮次，由视图编号（View Number）标识，领导者更换时视图切换。
- 仲裁证书（Quorum Certificate, QC）：由至少 n−f个节点（( n ) 为节点总数，( f ) 为最大可容忍故障节点数）的签名集合组成，用于证明某一阶段的投票达成一致。
- 区块（Block）：包含交易数据和元数据的提议单元，链接形成区块链。
- Pacemaker：负责视图同步和领导者轮换的模块，确保系统活性。
- 安全规则（SafeNode）：用于验证提案的安全性，防止恶意领导者破坏一致性。
- 消息类型：包括提案（Proposal）、投票（Vote）和超时（Timeout）消息。

#### Pacemaker 的功能

Pacemaker 的主要职责包括：
- 视图管理：维护当前视图编号（View Number），并在需要时推进到下一视图。
- 领导者轮换：根据视图编号选择领导者（通常通过确定性函数，如 hash(view) % n）。
- 超时检测：监控领导者的活跃性，若领导者在规定时间内未推进共识，触发视图切换。
- 状态同步：确保所有节点在视图切换时同步最高仲裁证书（Quorum Certificate, QC），避免分叉或状态不一致。
- 活性保证：即使存在最多 ( f ) 个故障节点（包括拜占庭节点），Pacemaker 也能通过视图切换推动系统进展。

#### Prepare 到 PreCommit 的关联机制
Prepare 阶段和 PreCommit 阶段通过以下步骤关联：
- Prepare 阶段输出
   - Prepare QC：Prepare 阶段结束时，领导者收集至少 n−f个 Prepare 投票，形成 Prepare QC：
```python
prepare_qc = QuorumCertificate(
    view = current_view,
    block_hash = block.hash,
    phase = "Prepare",
    signatures = [vote.signature for vote in votes]
)
```
- 广播：领导者将 Prepare QC 广播给所有节点

**PreCommit 阶段输入**
- 接收 Prepare QC：每个节点接收 Prepare QC，作为 PreCommit 阶段的触发条件。
-节点验证 Prepare QC：
   - 视图编号是否匹配 current_view。
   - QC 包含至少 n−f个有效签名。
   - 关联区块 ( B )（通过 prepare_qc.block_hash）存在于本地区块树。
  ```python
  verify_qc(qc):
    return (qc.view == self.current_view and
            len(qc.signatures) >= (self.n - self.f) and
            all(verify_signature(sig, qc.block_hash) for sig in qc.signatures) and
            self.block_tree.contains(qc.block_hash)) ```

**PreCommit 阶段投票**
- 生成 PreCommit 投票：
 - 若 Prepare QC 验证通过，节点为同一区块 ( B ) 生成 PreCommit 投票：
```python
vote = Vote(
    voter_id = self.id,
    view = self.current_view,
    block_hash = prepare_qc.block_hash,
    phase = "PreCommit",
    signature = sign(prepare_qc.block_hash, self.private_key)
)
```

- 节点更新 locked_qc，记录 Prepare QC 作为锁定点：
```python
self.locked_qc = max(self.locked_qc, prepare_qc, key=lambda qc: qc.view)
```
- 锁定确保后续提案不会与 ( B ) 冲突，除非基于更高视图的 QC。
- 领导者收集 PreCommit 投票
   - 领导者收集至少 n−f个 PreCommit 投票，形成 PreCommit QC：
```python
precommit_qc = QuorumCertificate(
    view = current_view,
    block_hash = block.hash,
    phase = "PreCommit",
    signatures = [vote.signature for vote in votes]
) 
```


