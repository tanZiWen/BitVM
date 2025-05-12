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
#### Helper
- 功能： 响应其他节点的区块请求，并返回Propose消息
#### Mempool
- 功能： 和mempool功能相关，从其他节点获取缺失的区块，等待结果并清除拿到结果的线程，反馈给共识拿到的结果
#### Core
- 功能：构建proposal，发送给其他节点投票签名生成QC，处理超时的消息，并生成TC。
- 流程：
  - 构建proposal消息
    - leader节点才能构造
    - propose模块处理，接收到ProposerMessage::Make消息
    - 构建区块，从buffer中拿到所有打包好的digest，向其他节点发送ConsensusMessage::Propose消息，并等待返回
    - 需要接收到2/3的共识节点消息
  - 处理proposal消息
    - 其他节点接收到Propose消息，开始处理proposal消息
    - 验证propose消息是不是leader节点发出
    - 验证区块合法性，签名验证
    - 处理QC消息
    - 处理TC消息
    - 验证本地是否包含所有的交易信息，和Mempool模块关联，如果不包含则向leader节点拿。
    - 进一步处理区块信息
  - 处理区块信息
    - 获取父区块和爷爷区块，判断是否满足 b0 <- |qc0; b1| <- |qc1; block|条件，如果没有从leader节点获取，
    - 存储区块到本地
    - 从buffer清除b0,b1,block的digest信息
    - 如果满足 b0.round + 1 == b1.round，则可以提交b0，清除所有小于b0的区块同步请求
      - 如果last_committed_round大于block.round，直接返回，该区块已经提交过，不能重复提交
      - 如果last_committed_round+1<block.round, 证明本地缺少历史区块，从其他节点获取所有历史区块
      - tx_commit处理缺失？？
    - 确保block.round = self.round，恶意领导者可能故意生成带有不正确轮次（如非常大的未来轮次）的区块，试图干扰系统的正常运行。
    - 对区块进行投票
      - block.round > self.last_voted_round，需要满足当前区块的round大于最后投票的round，防止节点为旧轮次的区块投票（避免重复投票或回退）。
      - block.qc.round + 1 == block.round，当前区块的qc的round+1等于当前区块的round，这确保区块是在正确的轮次顺序上提议的，防止跳跃或伪造轮次。或者，tc.round + 1 == block.round，确认新提案接在超时之后，并且 block.qc.round >= *tc.high_qc_rounds().iter().max().expect("Empty TC"); 不能引用一个太老的QC，是超时TC里面记录的最大的QC，防止引用老的QC导致分叉。
  - 处理投票消息
    - 获取下一轮的leader，如果是leader节点
      - 如果投票的round小于本地round直接返回
      - 验证投票的合法性
      - 聚合投票
      - 升级本地的QC
      - 生成新的proposal
    - 如果不是leader节点，向其他节点发送ConsensusMessage::Vote消息
  - 处理超时消息
    - 超时的round需要大于等于本地round
    - 验证超时消息的合法性
    - 升级本地round和high qc
    - 向其他节点发送TC信息
    - 如果是Leader节点开启新的round
  - 处理TC消息
    - 验证TC信息的合法性
    - 升级本地的round
    - 如果是leader节点开启新的round
 
#### 问题：
1. 既然所有节点都会收到 Timeout 并通过 handle_timeout 生成 TC 和推进轮次，为什么还需要 handle_tc 来再次处理广播的 TC 和调用 advance_round？是否可以省略 handle_tc，让 handle_timeout 完成所有超时相关的逻辑？
 - 在 handle_timeout 中，某个节点可能收到 2/3 以上 Timeout 消息，生成 TC 并推进轮次（advance_round）。但其他节点可能因为网络延迟，尚未收到足够 Timeout 消息，未能生成 TC，仍在旧轮次。
 - handle_tc 处理广播的 TC，确保 所有节点（包括那些没生成 TC 的节点）同步到新轮次（advance_round(tc.round)）。TC 是 2/3 以上节点的共识证明，广播后通过 handle_tc 让落后节点“赶上”正确轮次。
2. 如果我（某个节点）先生成了 TC 并广播给其他人，我的轮次（round）已经加1（advance_round），而其他节点也生成了 TC 并广播给我，会不会导致 handle_tc 再次执行，重复推进轮次或触发不必要的操作？
 - handle_tc 中包含了 tc.round < self.round return 的逻辑，我的round如果已经被推进了，那其他人广播给我的round就低，不会继续推进round
3. 




