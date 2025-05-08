#### Synchronizer模块
- 概述：主要用于从本地获取父区块和主父区块，如果本地没有父区块，则从其他节点获取，由于会存在多个异步请求通过FuturesUnordered管理，如果拿到通知共识，如果超时，从其他节点获取
  - pending：管理请求区块列表是一个HashSet
  - requests：管理请求的父区块列表
#### Proposer模块
- 概述：监听mempool消息，并将区块Digest放入buffer，制作区块，清空buffer缓存。制作区块主要是制作好一个新的区块，并将该区块广播给其他共识节点，等待收集满2/3的签名。
#### Messages模块
- 概述：定义核心的数据结构，包括 Block（区块）、Vote（投票）、QC（投票认证）、Timeout（超时消息）和 TC（超时证明）。
- 验证区块合法性：
  - 区块里面的author有权重stake
  - 验证该区块经过author签名
  - QC合法性校验通过
  - TC合法性校验通过
- QC数据结构
  - hash 认证的区块摘要。
  - round 认证的轮次。
  - votes 投票者的公钥和签名。
- QC合法性校验：
  - 投票没有重复
  - 每个投票者都有投票权
  - 总投票数达到2/3的阈值
  - 签名验证通过
- TC数据结构
  - round 超时的轮次。
  - votes 超时投票（投诉），包含投票者、签名和最高 QC 轮次。
- TC合法性校验
  -  投票者无重复。
  -  每个投票者有投票权
  -  总投票权达2/3阈值。
  -  每个投票的签名有效（基于 round 和 high_qc_round）。
- Timeout数据结构
  - high_qc 节点知道的最高QC  
  - round 超时的轮次。
  - author 投诉者的公钥。
  - signature 签名。
- Timeout验证
  - 投诉者有投票权。
  - 签名有效。
  - high_qc 有效（非创世）

