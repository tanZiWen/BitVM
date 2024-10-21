# BitVM2:将比特币桥接到二层网络

作者：Robin Linus<sup>1</sup>, Lukas Aumay<sup>2</sup>, Alexei Zamyatin<sup>3</sup>, Andrea Pelosi<sup>2,4,5</sup>, Zeta Avarikioti<sup>2,6</sup>, Matteo Maffei<sup>2</sup>

<sup>1</sup>ZeroSync <sup>2</sup>Tu Wien <sup>3</sup>BOB <sup>4</sup>University of Pisa <sup>5</sup>University of Camerino <sup>6</sup>Common Perfix

## 摘要

BitVM2 是一种新型范式，能够在比特币网络中实现任意程序执行，从而将图灵完备的表达能力与比特币共识的安全性结合在一起。BitVM2 的核心在于利用乐观计算，假设操作员是诚实的，除非挑战者通过欺诈证明（fraud proofs）证明其不诚实。此外，BitVM2 还利用 SNARK 证明验证脚本，这些脚本被拆分为子程序，并在比特币交易中执行。因此，BitVM2 只需要三笔链上交易就能确保程序的正确性。相比之前的 BitVM 设计，BitVM2 提出了无许可的挑战机制，并通过减少链上交易的复杂性和数量，首次实现了更高效的争议解决。我们的构建方案无需对比特币的共识进行任何更改。

BitVM2 使比特币中全新类别的应用成为可能。我们通过 BitVM Bridge 展示了这一点，这是一种协议，旨在改进之前的比特币跨链桥，降低了对存款安全的信任假设，从多数诚实 (t-of-n) 转变为存在性诚实 (1-of-n) 的假设。为了保证活性，我们只需要一个活跃的理性操作员（即使其他人可能是恶意的）。任何用户都可以充当挑战者，从而实现对协议的无许可验证。

> 安全性假设，n个多签地址中至少有一个诚实；n个operator中至少有一个诚实
>
> 活性假设，至少有一个operator活跃

## 1 引言

比特币与其他第一层区块链（Layer 1）类似，面临着显著的可扩展性挑战，这促使了第二层解决方案（Layer 2）的需求，以提升网络性能。在比特币的情况下，L2 还可以扩展其脚本功能，以匹配像以太坊这样更具表达性的区块链。虽然其他 L1 区块链（尤其是以太坊）成功集成了 L2 解决方案，但比特币的实现却受限于其脚本语言的局限性。尽管与社区中的一些误解相反，我们可以在比特币脚本中表达任何有界的计算，但由于本地可用操作非常有限的低效性，这些程序很快就会超出比特币的区块大小和堆栈限制。

比特币跨链桥（Bridges）是将BTC从比特币链上转移到其他区块链并作为“wrapped”资产使用，同时，也是第二层架构（Layer 2）的关键组成部分。这些跨链桥需要管理存入资产和提取资产至L1。每一个桥接机制的核心问题是托管——即如何在其他区块链上使用wrapped的资产时的同时安全地存储 L1上的BTC。目前，几乎所有比特币跨链桥都依赖于多重签名或阈值签名方案，在这些方案中，一组 t-of-n 签名者负责安全保管 BTC。尽管一些跨链桥通过抵押经济安全性来加强保障，但这些设计因资本要求过高而面临可扩展性挑战，因此在实际应用中采用率有限。

**贡献**：本文带来了两个根本性的贡献：一是设计了 **BitVM2**，一种新型范式，能够在比特币中执行任意程序；二是基于此设计，提出了 **BitVM Bridge**，一个将 BTC 桥接到比特币第二层系统的创新协议。

- 与 Truebit[21]和 Arbitrum[10]类似，**BitVM2** 利用乐观计算，假设计算方（操作员）是诚实的，直到通过所谓的欺诈证明（fraud proofs）证明其不诚实。在 BitVM2 中，一组操作员在链上承诺（commit）正确执行某个任意程序的 SNARG[5]验证器（off-chain），这样任何观察方（挑战者）都可以在预定义的挑战期内驳斥错误计算。我们在比特币脚本[26]中实现了 Groth16 SNARK[7]验证器程序，并将其拆分为顺序的子程序，每个子程序足够小，可以在比特币区块中执行。如果操作员被挑战者质疑，必须在链上公开这些子程序的中间结果，以及 SNARK 验证器的输入和输出状态。按照设计，如果操作员试图作弊，发布的某个中间状态将不正确，即与操作员承诺的子程序结果不一致。为了驳斥操作员的虚假声明，挑战者只需在比特币交易中执行相应的子程序，展示计算与操作员声明的不匹配。根据这些设计原则，BitVM2 确保任何人都可以在仅需 3 笔链上交易的情况下挑战并驳斥一个错误的操作员。

  **BitVM2** 受 Linus[14]提出的 BitVM 范式启发，并在计算和通信复杂性以及安全性方面取得了重大改进。在 BitVM 和类似设计中，只有一组固定的操作员可以发起挑战，而 BitVM2 首次支持无需许可的挑战，即任何拥有比特币全节点的用户都可以质疑错误的操作员。此外，原始的 BitVM 设计需要多达 70 笔链上交易（在实践中可能需要数周甚至数月）来驳斥一个错误的操作员，而 BitVM2 只需要 3 笔链上交易，可以在 1-2 周内完成，从而实现了与以太坊第二层解决方案类似的实际效果。

- 我们通过展示 **BitVM Bridge**，一个将 BTC 桥接到比特币第二层系统的创新协议，来说明 BitVM2 所启发的突破性设计机会。**BitVM Bridge** 相较现有的比特币桥接设计有了显著改进，将信任假设从大多数签名者诚实降低到存在性诚实（existential honesty），同时只需要一个活跃的理性操作员和理性挑战者来维护协议的安全性。

**组织结构**：我们首先在第2节中介绍一个（非正式的）模型、协议目标和协议概述。接着在第3和第4节中介绍必要的背景知识、符号和构建模块。然后我们将这些构建模块结合起来，在第5节中介绍 **BitVM2**，这是一个在比特币上安全运行的通用函数验证协议。随后，在第6节中，我们概述了基于 BitVM2 的桥接协议，称为 **BitVM Bridge**。在当前草稿中，第7节和第8节分别概述了我们计划进行的安全性分析以及协议的局限性与扩展部分，这些将在完整版本中完成。

##  2 模型与协议概述

在本节中，我们介绍我们的模型和假设，并提供基于 BitVM2 的桥接协议的高级概述。BitVM2 的安全属性可能具有独立的研究价值，并将在安全分析中讨论和证明，作为桥接协议安全性的基础。

### 2.1 假设

区块链或分布式账本协议的输入是用户提供的交易，输出是最终包含所有提供交易的不可更改的交易顺序。我们假设比特币和一个侧系统维护了具有持久性和活性的可靠公共交易账本，如文献[4]中定义。我们将 **∆<sub>L</sub>** 表示为活性参数，即诚实参与者将一笔交易包含在基础区块链中的最长时间上限。

我们假设侧系统利用比特币的共识，例如是一个所谓的roll-up系统，通过发布到比特币区块链的数据承诺来获得对交易顺序的共识，并伴随某种形式的状态转换验证。协议示意图如图5所示。

我们假设网络是同步的，即所有广播到网络的消息将在已知的时间范围内传递给所有参与者。我们还采用常见的密码学假设：参与者的计算能力是有限的，并且存在加密安全的通信通道、哈希函数、签名和加密方案。我们将区块链的概念扩展用于表示其交易账本。

### **2.2 系统模型与目标**

在我们的设置中，桥接协议涉及两个主要操作：1) **Peg-In**，即用户 Alice 在比特币网络上存入资金 uB，这些资金随后在侧链系统中表示为“wrapped BTC”（Bs）；2) **Peg-Out**，即另一名用户（如 Bob）从侧链系统中提取这些资金 u，并将其取回比特币区块链。

除了存款人（Alice）和受益人（Bob）的角色外，在我们的桥接协议中，还存在以下参与者角色：

- **操作员 (O1, ..., Om)**：操作员负责执行预先商定的程序 ***f***；在我们的桥接协议中，这对应于 **Peg-Out** 过程。假设操作员是理性的，即他们是以利润最大化为目标的代理。

- **挑战者**：挑战者负责确保 **Peg-Out** 过程的安全性，在操作员行为不当时发起挑战。任何人都可以充当挑战者，包括操作员。假设挑战者也是理性的。

- **签名者 (S1, ..., Sn)**：这是一个由 n 名签名者组成的委员会，负责正确设置一个用于预定程序 ***f*** 的 BitVM2 实例。假设签名者中有一名是诚实的（存在性诚实）。如我们在第4.2节中解释的，如果底层区块链支持协定（covenants），则这种假设（即委员会及其假设）是不必要的。

我们假设所有参与者的计算能力是有限的，即密码学原语是安全的。我们首先证明，假设签名者委员会具有存在性诚实，桥接协议的设置是安全的。在此假设下，我们证明当所有挑战者是理性的且至少一个活跃的操作员是理性时（其余可以是恶意的），BitVM2 桥接协议是安全的。值得强调的是，尽管操作员是预定义的实体（有权限的），任何人都可以充当挑战者（无需许可的），因此任何希望退出侧链系统的参与者都可以作为挑战者，确保协议的安全性。

**协议目标** 以下内容涉及桥接协议，并涵盖了其安全性和活性：

- **余额安全性**：任何在侧链系统中持有 v 个代币的用户可以销毁这些代币，并且只有在销毁后，最终才能在主区块链上申领 v - f<sub>O</sub> 个代币，其中 f<sub>O</sub> ≥ 0 是用于支付桥接操作成本的费用。

### 2.3 BitVM2协议概述

BitVM2 的核心理念是通过乐观的方式进行任意计算，即假设计算方（操作员）是诚实的，除非有挑战者通过所谓的欺诈证明证明其不诚实。这种设计选择源于比特币协议所提供的有限存储和计算能力。我们在 BitVM2 中使用了一些技巧来实现这一目标，同时继承了比特币的安全性：

1. 我们首先使用 SNARKs 压缩程序。在 BitVM2 中，我们并不验证程序的执行本身，而是验证 SNARK 验证器正确地验证了程序执行的证明。
2. 我们在比特币脚本中实现了 SNARK 验证器。程序的大小（例如 Groth16 需要 3GB）远远超出了比特币区块的大小限制。
3. 我们将验证器分割成多个子程序块，每个块的大小不超过 4MB。这些子程序可以在比特币交易或区块中执行。这些子程序是顺序执行的：第 2 个程序的输入是第 1 个程序的输出，依此类推。
4. 操作员在设置过程中承诺正确执行程序（即 SNARK 验证器）。通过预签名精心设计的比特币交易和 Taproot 树，可以确保操作员只能以一种方式提取资金，如果行为不当，可以通过挑战来质疑其提取行为。
5. 当操作员想从 BitVM2 中提取资金（例如作为桥接协议的 Peg-Out 过程的一部分提取 BTC）时，他们必须将 SNARK 验证器的输出发布到链上。
6. 任何人都可以根据已知的公开输入（不考虑数据可用性问题）检查该数据与其本地的 SNARK 验证器执行结果是否一致。如果结果不同，任何人都可以在链上发起挑战，迫使操作员公开更多的计算数据。
7. 如果被挑战，操作员必须在链上交易中公开所有子程序的中间输出结果。
8. 现在，任何人都可以找到错误的中间结果，并通过在链上执行相应的子程序块证明操作员作弊，展示结果不匹配。这将使操作员的 BitVM2 提款交易失效，并扣押（部分）抵押品，以补偿链上的费用。

总结来说，BitVM2 允许任何人在 3 笔链上交易内挑战并扣押不诚实的操作员，延迟不超过 2-3 周。图1提供了 BitVM2 协议的概述。

![](https://private-user-images.githubusercontent.com/8327633/375940475-cb33d159-4278-4b8d-b58a-d3ebd89eadf8.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTkzNjQsIm5iZiI6MTcyOTQ5OTA2NCwicGF0aCI6Ii84MzI3NjMzLzM3NTk0MDQ3NS1jYjMzZDE1OS00Mjc4LTRiOGQtYjU4YS1kM2ViZDg5ZWFkZjgucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDgyNDI0WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ZmViMGE2NzUwYWQ2YThhNTg5MzcyZTdlN2Y2MDVhZjUzMjMzN2UzODQ2ZDBlMjhmYjM4ZDAwZTA3MjdjN2IyMyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.VLvFmUTIh5UPM3shkl2G7_sW6xrbvZ2KNjPiY6ZWneE)

**图 1. BitVM2 交易流程的简化概述**  
SNARK 验证器程序 f(x) = y 被分割为 f<sub>1</sub>, f<sub>2</sub>, ..., f<sub>k</sub>，并且具有中间结果 z<sub>0</sub>, ..., z<sub>k</sub>。如果操作员被挑战，他们必须在 Assert 交易中公开这些中间状态。这使得挑战者可以通过在比特币交易中执行有争议的子程序 f<sub>i</sub> 来证明操作员的虚假声明不成立。

## 3 背景和符号说明

### 3.1 数字签名

数字签名方案 $\Sigma\$ 是由以下三个算法组成的一个元组：***KeyGen***（密钥生成）、***Sign***（签名）和 ***Vrfy***（验证）。

- $\(pk, sk) \leftarrow \Sigma.\text{KeyGen}(\lambda)\$ 是一个概率多项式时间（PPT）算法，接受安全参数 $\\lambda\$ 作为输入，并返回一对密钥，分别是私钥 sk 和公钥 pk。

- $\sigma \leftarrow \Sigma.\text{Sign}(sk, m)\$ 是一个 PPT 算法，接受私钥 sk 和消息 $m \in {0, 1}^*$ 作为输入，并输出一个认证标签或签名 $\sigma\$。

- \{True, False} $\leftarrow \Sigma.\text{Vrfy}(pk, \sigma, m)\$ 是一个确定性多项式时间（DPT）算法，接受公钥 pk、签名 $\sigma\$ 和消息 $m \in \{0, 1\}^*$ 作为输入，当且仅当 $\sigma\$ 是由 pk 对应的私钥 sk 生成的有效签名时，输出 True，即 $(pk, sk)\$ 是由 $\Sigma.\text{KeyGen}$ 生成的密钥对。

为了与比特币脚本保持一致，我们将此验证过程称为 $\\text{CheckSig}_{pk}(\sigma)\$，消息即为交易。

在本研究中，我们使用的签名方案是抗适应性选择消息攻击的（EUF-CMA）安全签名方案。

### 3.2 简洁的非交互论证 (SNARGs)

对于这个定义，我们紧密参考了 [5,7]。设 R $\leftarrow$ R(λ) 是一个关系生成器，它接受一个安全参数 λ 作为输入，并返回一个多项式时间可判定的二元关系 R。我们将 ϕ 记作声明，将 w 记作证人，其中 (ϕ, w) 属于关系 R。我们定义一个有效的公共可验证非交互式论证为一组三个 PPT 算法的元组：***Setup***、***Prove*** 和 ***Vrfy***。

- crs $\leftarrow$ SNARG.Setup(q) 以一个关系 R 作为输入，并输出一个公共参考字符串 crs。
- π $\leftarrow$ SNARG.Prove(R, crs, ϕ, w) 以公共参考字符串 crs 以及 (ϕ, w) $\in$ R 作为输入，并返回一个论证 π。
- \{True, False} $\leftarrow$ SNARG.Vrfy(R, crs, ϕ, π) 以公共参考字符串 crs、声明 ϕ 和论证 π 作为输入，返回 True 或 False（非正式地），具体取决于 π 是否为有效论证。

如果(***Setup***, ***Prove***, ***Vrfy***) 满足完美的完整性和计算上的正确性，它就是 R 的一个非交互式论证，如 [7,5] 中定义的那样。从高层次上看，完美的完整性意味着，对于任何真实的声明 ϕ，一个诚实的证明者可以以压倒性的概率说服一个诚实的验证者；计算上的正确性意味着如果没有证人存在，则不可能证明一个虚假的声明，也就是说，以压倒性的概率无法说服验证者。

最后，一个非交互式论证，如果验证者在 λ + |ϕ|的多项式时间内运行，且证明大小在 λ 的多项式范围内，则被称为（预处理的）简洁非交互式论证（SNARG）。需要注意的是，尽管我们稍后会使用 [7] 中的实现，但我们并不关心或使用零知识属性，也不使用更强的简洁非交互式知识论证（SNARKs）。

### 3.3 [Lamport数字签名方案](https://github.com/tanZiWen/BitVM/blob/master/Lamport%20%E7%AD%BE%E5%90%8D.md)

设 $\( h : X \rightarrow Y \)$ 为一个单向函数，其中 $\( X := \{0, 1\}^*\) ， \( Y := \{0, 1\}^\lambda \)$ ，其中 $\( \lambda \)$ 是一个给定的安全参数。设 $\( m \in \{0, 1\}^\ell \)$ 为一个 $\(\ell\)$ 位的消息，其中 $\( \ell \in \mathbb{N} > 0\)$。Lamport数字签名方案 [12] Lamp 由一组三个算法（KeyGen、Sig 和 Vrfy）组成：

- $\( (pk_M, sk_M) \leftarrow Lamp.KeyGen(\ell) \)$ (算法 1) 是一个 PPT 算法，它以正整数 $\( \ell \)$ 为输入，并返回一个密钥对，由秘密密钥 $\( sk_M \) 和公钥 \( pk_M \)$ 组成，可用于一次性签署一个 $\(\ell\)$ 位的消息。为了易读性， $\( M = \{0, 1\}^\ell \) 是 \(\ell\)$ 位消息空间的别名。

- $\( c_m \leftarrow Lamp.Sig_{sk_M}(m) \)$ (算法 2)，是一个参数化为秘密密钥$ \( sk_M \)$ 的 DPT 算法，它接受一个消息 $\( m \in M \)$ 作为输入，并输出签名 $\( c_m \)$，我们也称其为Lamport承诺。

- $\( \{True, False\} \leftarrow Lamp.Vrfy_{pk_M}(m, c_m) \) $(算法 3)，是一个参数化为公钥 $\( pk_M \)$ 的 DPT 算法，它以消息 $\( m \)$ 和签名 $\( c_m \)$ 为输入，当且仅当 $\( c_m \)$ 是由与 $\( pk_M \)$ 对应的秘密密钥 $\( sk_M \)$ 生成的有效签名时，输出 True，即 $\( (pk_M, sk_M) \)$ 是由 Lamp.KeyGen 生成的密钥对。

Lamport签名是一种安全的一次性签名。我们使用 $\( sk_M \)$ 和 $\( pk_M \)$ 来表示与消息空间 $\( M \)$ 相关联的秘密密钥和公钥。该密钥对可用于签署 $\( M \)$ 中的任何消息，但一旦创建了签名 $\( c_m \)$，该密钥对就被绑定到一个特定的消息 $\( m \in M \)$。换句话说，只要单个密钥对仅签署一个消息 $\( m \in M \)$，任何多项式界限内的对手都无法以非可忽略的概率伪造该密钥对的另一条消息 $\( m' \neq m \)$ 的签名。

更具体地说，当一方使用Lamport签名方案签署消息时，他们会为消息的每一位 $\( m[i] \)$ 公开两个前像 $\( x[0, i] \)$ 和 $\( x[1, i] \)$，其中 $\( i = 0, \dots, \ell - 1 \)$。这意味着签名者声明 $\( m[i] \)$ 要么是 0，要么是 1。注意，承诺一个 $\(\ell\)$ 位消息 $\( m \)$ 实际上就是对 1 位消息（每个比特对应一个）的 $\(\ell\)$ 个承诺。

关于一次性安全性的正式定义及其证明，可以参考 [2]。Lamport签名，尤其是算法 3，可以在 Bitcoin Script 中实现；[26] 提供了一个示例实现。由于我们利用Lamport签名使得某方能够承诺一个比特（因此也能承诺一个消息），从现在开始，我们将算法 2 称为 LampComm 而非 Lamp.Sig，将算法 3 称为 CheckLampComm 而非 Lamp.Vrfy。
![](https://private-user-images.githubusercontent.com/8327633/378338239-c7c4414f-2e84-423f-9ea4-0ecdb735c749.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTk5MjQsIm5iZiI6MTcyOTQ5OTYyNCwicGF0aCI6Ii84MzI3NjMzLzM3ODMzODIzOS1jN2M0NDE0Zi0yZTg0LTQyM2YtOWVhNC0wZWNkYjczNWM3NDkucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDgzMzQ0WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9YjE0NTk2MDMzYmI0YWU3NzNjMThiZmE1ODJhNTg2NjViMzk2N2QzOGNjZDljZDgyMTM4M2QyZjg5MmMyN2IyZiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.Ju6GvtWOvYAzu6mA29LytE0BKN4XHzETjLQZs-sqeDI)

![](https://private-user-images.githubusercontent.com/8327633/378338325-9016982c-b975-4199-977e-5f5c57175c60.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTk5NDAsIm5iZiI6MTcyOTQ5OTY0MCwicGF0aCI6Ii84MzI3NjMzLzM3ODMzODMyNS05MDE2OTgyYy1iOTc1LTQxOTktOTc3ZS01ZjVjNTcxNzVjNjAucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDgzNDAwWiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NjUxNWU3OTBjNGE4YjIwYmFiOTU2YTdiNGIzOGJkOGI4YThlNjE1M2U1ZmIyZTEyYmUzNzY2NWJlODI3MTA4ZSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.HGvp0SDAAqYv6_2-uioeUHLVpHIK4G84hdRNarab-GI)

![](https://private-user-images.githubusercontent.com/8327633/378338425-f754a7ee-ad98-4095-b26e-bdfab2c294b8.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTk5NTUsIm5iZiI6MTcyOTQ5OTY1NSwicGF0aCI6Ii84MzI3NjMzLzM3ODMzODQyNS1mNzU0YTdlZS1hZDk4LTQwOTUtYjI2ZS1iZGZhYjJjMjk0YjgucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDgzNDE1WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NjA2MmIwMDJjMDg1OTU1OTE2NzU1MjdkMjU1ODJhYzI5MjIwNGEzMzRkMmY4ODYzOTMzMzI4NDA2NGNhYzllZSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.tr1BqVs1JuIwVMNSo8PIxzsFBvDUv5VLC-RT8qUxc_0)

### 3.4 UTXO模型中的交易  

我们通过签名方案 $\Sigma\$ 中的密钥对 (pk_U, sk_U) 在账本 L 上 证明 一个用户 U 对代币的所有权。我们用 $\sigma_U(m)\$ 表示用户 U 对消息 m $\in$ {0, 1}* 的数字签名。如果明确签署了什么消息，有时我们使用 $\sigma_U$ 作为简写。  

在未花费交易输出 UTXO 模型中，每个交易输出都与一个代币值（以 B 计价）相关联。输出被定义为一个属性元组 out := (a_B, lockScript)，即它包含一个币值 out.a $\in \mathbb{R}_{\geq 0}$ 的代币 \(B\) 以及在什么条件下可以花费该币值的条件 out.lockScript。一个交易 Tx将一组非空的现有未花费输出映射到一组非空的新的 Tx.outputs。为了区分它们，我们将前者称为交易的 Tx.inputs。一个输入 in := (PrevTx, outIndex, lockScript) 唯一标识一个现有输出，方法是引用交易 PrevTx 以及一个输出索引 outIndex，并为了方便重复输出的花费条件 lockScript。  

形式上，我们定义交易为 Tx := [inputs, witnesses, outputs]，除了前述的 Tx.inputs := [in_1, $\dots$, in_n] 和 Tx.outputs := [out_1, $\dots$, out_m]，还包含见证数据 Tx.witnesses := [w_1, $\dots$, w_n]，这是满足交易输入花费条件的见证数据列表，每个输入对应一个见证。锁定脚本由账本的脚本语言表达，并以相应的见证作为脚本输入执行。如果执行结果为 False，则交易无效；如果执行结果为 True，则满足了花费条件。  

要使交易有效，所有见证必须满足它们相应输入的锁定条件；所有交易输入必须为未花费的；输出的总价值必须小于或等于输入的总价值。如果输出总价值小于输入总价值，则差额会给予矿工。

**交易的花费条件** 。我们特别关注比特币，它有一个基于堆栈的脚本语言。现在我们描述本文中使用的比特币支持的部分花费条件。以下每个条件都可以使用逻辑运算符 ^ （与）和  v（或）组合，以创建更复杂的花费条件。

– **签名锁**。一个使用 CheckSig<sub>pk<sub>U</sub></sub> 锁定的输出，只有在花费交易使用与密钥对 (sk<sub>U</sub>, pk<sub>U</sub>) 对应的秘密密钥签名时才能花费。

– **多重签名锁**。要满足多重签名的花费条件，需提供 n 个签名中的 k 个。例如，对于用户 A 和 B，2-of-2 的多重签名花费条件记为 CheckMultiSig<sub>pk<sub>A,B</sub></sub>，相应的签名为 σ<sub>A,B</sub>。

– **时间锁**。时间锁会将交易输出锁定到未来的某个指定时间（绝对时间锁）或直到交易上链后的特定时间（相对时间锁）。我们将前者表示为 AbsTimelock(Δ)，后者表示为 RelTimelock(Δ)。在后续描述中，我们将时间锁与其他花费条件结合使用。例如，如果 UTXO Tx.out1 的锁定脚本为 lockScript := RelTimelock(Δ) ^ CheckSig(pkU)，则用户 U 可以在 Tx.out1 发布上链后，经过一定时间 T 之后花费该 UTXO。

– **Taproot 树 [23]**，或称为 Taptrees，使得 UTXO 可以通过满足多种花费条件之一来花费。这些花费条件是 Merkle 树的 (Tap) 叶子。为了花费具有 Taptree 作为锁定脚本的 UTXO，用户需要提供某个叶子的证明，以及该叶子在 Taptree 中的 Merkle 包含证明。接下来，我们将 Taptree 锁定脚本的 Tap 叶子表示为 {leaf1, ..., leafr}；当用户满足脚本 leaf i 以解锁交易 Tx 的第 j 个 UTXO 时，我们将对应的输入写为 (Tx, j, {leaf<sub>i</sub>})。每当用户通过 Taptree 的 Tap 叶子花费 UTXO 时，我们假设用户已提供了该 Tap 叶子的有效 Merkle 包含证明。

– **其他条件**。我们用 True 表示始终满足的条件，用 False 表示永远无法满足的条件。在后一种情况下，代币无法赎回，它们将被销毁。

**组合花费条件**。当我们需要表达较长的花费条件时，会明确给出伪代码，将前面介绍的花费条件与其他可在 Bitcoin Script 中表示的标准语言结构结合起来，例如 if-then-else 结构。具体而言，当我们在长脚本 LongScript 中为某个子花费条件（例如 script）附加 Verify 关键字时，其返回值可以是 True 或 False。我们这样做的目的是模拟 Bitcoin OP_VERIFY 操作码的行为：如果 script 返回 True，则从堆栈中弹出 True，并继续执行剩余脚本；如果返回 False，则标记该交易无效（从而无法满足 LongScript）。

**SIGHASH 标志**。SIGHASH 标志指定了在签名锁的一部分签名中包含哪些交易数据。这些标志主要用于帮助多个用户协作创建和签署交易。

– **ALL**: 所有的输入和输出都被签名。交易只有在当前状态下有效。

– **NONE**: 所有输出被签名，但没有输入被签名。可以使用任意数量的输入来为这笔交易提供资金。

– **SINGLE**: 所有输入和仅一个输出被签名，其他输出可以随意添加。

此外，`ANYONECANPAY` 标志只签名一个输入，可以与其他标志组合，以创建更高级的结构。

## 4 BitVM2构建块

**在本节中，我们展示了一组 Bitcoin 脚本原语，这些原语用于 BitVM2 设计中。**

### 4.1 通过一次性签名公钥硬编码实现有状态的 Bitcoin 脚本

尽管 Bitcoin 脚本语言是无状态的，通过巧妙使用一次性数字签名方案（例如 Lamport 签名），可以在不同的 Bitcoin 交易中保留状态。请考虑以下示例。假设用户 U 拥有与消息空间 M 关联的 Lamport 密钥对 (sk<sub>M</sub>, pk<sub>M</sub>)，其中 M 是所有 v 位消息的集合。我们可以将 M 视为可以保存任意 v 位字符串的变量。使用这一思维模型，U 可以通过生成承诺 c<sub>m</sub> $\leftarrow$  LampComm<sub>M</sub>(m)，为 M 分配消息 m 的值。

通过在多个输出的锁定脚本中硬编码 CheckLampComm<sub>pkM</sub>（针对公钥 pkM），不仅可以检查该变量赋值，还可以将其从一个输出传递到另一个输出，从而在 Bitcoin 中实现全局状态。这是通过从一个输出的解锁脚本中读取 m 和 cm，并将它们通过解锁脚本传递给另一个输出来完成的。

### 4.2 使用预签名交易模拟契约

在 Bitcoin Script 中，**契约**（如文献[16,19,8]所述）是一类拟议的支出约束，允许交易的锁定脚本对未来如何花费 UTXO 中的币施加限制。截至撰写本文时，契约尚未被添加到比特币中。如果契约被添加到 Bitcoin Script 中，它将允许对后续交易的输出进行限制。契约的一个简单示例是，限制一个由 Bob 可花费的 UTXO，使得它只能在花费交易中为 Alice 分配 5 BTC。契约除了其他功能外，还可以通过一系列不同的交易实现状态的存储和状态机的执行。这将提供更具表现力的智能合约功能，并能在比特币上实现更复杂的应用程序。

在 BitVM2 中，我们需要限制 UTXO 的花费方式，以便运营者从 BitVM2 中花费时可以被挑战者挑战。在没有契约的情况下，我们可以通过一个由 n 个签名者组成的委员会 S1, ... Sn 来模拟其功能，其中至少有一个是诚实的。实际上，这个想法可以通过以下方式实现为了防止服务拒绝攻击，只要用户活跃，他们就可以加入委员会（不活跃的用户将被踢出委员会）。因此，诚实的用户可以通过加入委员会来验证这种存在性的诚实假设[3]。具体而言，在每个 BitVM2 实例的设置阶段，我们引入了以下步骤：

1. 每个签名者生成一对新的密钥对。
2. 对于每个只能以特定方式使用的交易输出，我们引入了额外的 n-of-n 多重签名花费条件 CheckMultiSig<sub>C</sub>，即所有签名者必须协作生成签名 σ<sub>C</sub> 才能花费 UTXO。
3. 对于这些 UTXO，签名者预签署了应用于花费输出的特定交易。每个操作员都会收到一组专门针对他们的预签名交易，可以在特定条件下由他们花费。
4. 最后，签名者删除他们的密钥。

该机制确保只要有一个签名者诚实并删除了其密钥，UTXO 就只能通过预签名交易之一进行花费，即按照协议的预期方式进行花费。我们还可以使用签名聚合方案来提高设置效率，减少链上占用。此外，如果所有协议参与方事先已知，例如在两方协议或许可协议中，他们可以组成委员会。

为提高可读性，并突出引入契约（Covenants）可以消除委员会的需求，我们对这种模拟进行了抽象化，并且从此以后使用 CheckCovenant 来表示输出的花费条件。我们的交易图和正式的交易定义展示了哪些交易可以花费这些输出：除了我们定义的交易外，没有其他交易可以花费它们。换句话说，每当我们写 CheckCovenant（例如在图 2 中），这可以替换为 n-of-n 多重签名花费条CheckMultiSigVerify<sub>pk<sub>C</sub></sub>(σ<sub>C</sub>)， 与签名委员会预签的交易或未来在 Bitcoin 中引入的实际契约配对使用。我们在交易的见证中使用 Covenant 来表示满足了契约的条件。

### **4.3连接器输出**（Connector Outputs）

连接器输出是一种确保一组比特币交易中只有一个是有效的，并且可以被包含在比特币区块链中的技术。具体而言，我们为多个交易创建相同的输入，即引用相同的连接器输出。我们通过为每个交易引入一个附加的输入（引用该连接器输出），并设置 **SIGHASH ALL** 标志来实现这一点，要求所有输入都必须存在才能使交易有效。

一旦其中一个交易被包含在比特币区块链中并使用了该连接器输出，其他交易将自动变为无效。通过这种方式，连接器输出可以指定多种自定义的花费条件。

在 **BitVM2** 中，我们使用连接器输出来使有问题的运营者在挑战者成功发起挑战后企图从 BitVM2 实例中花费资金的行为失效。

## **5 BitVM2: 基于比特币的通用函数验证**

在本节中，我们展示了如何使用比特币脚本（Bitcoin Script）在不需要共识更改的情况下，在比特币上计算任意函数。我们将此应用于构建基于 **BitVM2** 的跨链桥协议，但也指出这种构建模块本身作为独立的构造是很有趣的。假设一方 **O**，即操作员（作为证明者），想要在比特币上证明，对于用比特币脚本编写的给定程序 **f**，输入 **x** 和输出 **y** 属于可能的多个有效输出集 **V**，并且断言 **f(x) = y** 成立。

与社区中的普遍看法相反，比特币脚本（Bitcoin Script）实际上支持通用计算，即我们可以在比特币脚本中表示任何程序（尽管可能非常大）。然而，比特币的共识规则对有效区块的大小施加了限制，进而限制了脚本的大小。由于 **f** 是比特币脚本程序，执行 **f**（或任何子程序 **f<sub>i</sub>**，其中 **i** $\in$  {1, ..., k}）意味着要在一个堆栈中操作数据，这个堆栈可以存储固定数量的元素。在这种术语下，**f** 的输入 **x** 和输出 **y** 分别是程序执行的初始状态和最终状态。

这种表示方式利用了比特币脚本中基于堆栈的数据处理模型，其中输入和输出通过堆栈的状态转移来实现，使得比特币脚本能够执行更复杂的程序逻辑，但受限于共识规则对区块和脚本大小的限制。

在 **BitVM2** 中，某一方需要质押一笔金额为 **dB**（其中 **d ≥ 0**）的押金，操作方在成功验证后可以领取这笔押金。根据函数 **f** 的使用场景，押金 **d** 可以有以下两种用途：

- **第三方的押金**：例如，一个第三方请求计算任务，成功执行后，押金将支付给操作方作为报酬。
- **操作方的担保押金**：操作方质押押金，成功执行后可以收回押金。这种情况下，计算的支付是在 **BitVM2** 之外处理的，例如在第6节描述的 **BitVM Bridge** 协议中。

这两种押金机制为不同的应用场景提供了灵活的激励和担保手段，确保操作方能够按预期执行计算或提供验证。

### 5.1 本地的函数验证

在这部分中，我们利用 **Lamport 签名** 和 **Taptrees** 来使操作方 **O** 在比特币网络上断言 **f(x) = y**，由于编码该程序的脚本无法完全放入一个区块中。为此，我们设计了一种乐观的验证机制，该机制允许任何用户（挑战者）在操作方提供的断言不正确的情况下进行反驳。

这种机制的主要思想是，如果操作方试图提交不正确的计算结果，挑战者可以通过提供证据来证明错误。这样，虽然 **Bitcoin Script** 有块大小限制，但通过将断言分解为多个验证步骤，并利用挑战者的监督，我们依然可以实现复杂的函数验证。

**1.初始化设置**。首先，我们将程序 **f** 拆分为 **k** 个子程序 **f₁, f₂, ..., fₖ**，其中每个子程序都小于脚本大小限制，也就是说，这些子程序都可以在一个比特币区块中执行。定义执行这些子程序后的中间状态 **z<sub>i</sub>**，我们有以下状态转换过程：

- **z₁ = f₁(z₀)**
- **z₂ = f₂(z₁)**
- …
- **zₖ = fₖ(zₖ₋₁)**

其中，**z₀ = x** 为输入，**zₖ = y** 为输出。对于集合 **{z₀, z₁, ..., zₖ}** 中的每个元素，操作方 **O** 创建一个新的 **Lamport** 密钥对 **(sk<sub>z₀</sub>, pk<sub>z₀</sub>), (sk<sub>z₁</sub>, pk<sub>z₁</sub>), ..., (sk<sub>zₖ</sub>sub>, pk<sub>zₖ</sub>)**。

接下来，操作方创建了三个硬编码这些公钥的交易，分别为 **Assert**、**Disprove** 和 **Payout**，它们在下方的方程式 (1) 至 (3) 中定义，并在 **图2** 中展示。通过 **Covenants**（约束），我们限制了这些输出的花费方式，即只能通过这些特定的交易进行花费。正如在 **4.2** 节中所解释的，我们通过在设置阶段由签名者委员会预签 **Assert**、**Disprove** 和 **Payout** 来模拟 **Covenants**，并将这些交易分享给相关方，确保他们验证这些签名的有效性。

**2. 执行**。操作方 **O** 在链下执行子程序 **f₁, f₂, ..., fₖ**，给定输入 **x = z₀**，并在此过程中生成中间状态 **z₁, z₂, ..., zₖ₋₁** 以及最终状态 **y = zₖ**。最终输出 **zₖ** 应该属于有效输出集 **V**，例如，**V** 可以是 **{True}**。

通过链下执行，操作方逐步计算这些状态，并确保最终输出符合预期，这样就可以证明程序的执行是正确的。

**3. 承诺以及挑战期**。 操作方随后发布 Assert 交易（如方程 (1) 中所述）到链上，承诺 z₀, z₁, ..., zₖ 并声称程序正确执行。**Assert** 交易的输入可以是任何持有 **dB** 的输出（用 **\*** 表示），该输出必须满足以下至少几个条件才能被花费：

1. **契约（Covenant）的要求**：即只能通过 **Assert** 交易来花费；
2. **操作方的签名**；
3. 每个公钥 **pkz₀, ..., pkzₖ** 对应的有效的 **Lamport** 承诺 **c₀, ..., cₖ**。

这种机制允许操作方提交其声称的正确计算结果，随后进入挑战期，其他用户可以质疑计算是否正确。这种方式将用于后文的构建中。

```javascript
Assert := (input = [(*)],
					 witness = [(*, Covent, z0,...,zk, Cz0,...,czk)],
					 output = [(dB, <DisproveScript1,...,DisproveScriptk,
					 RealTimelock(∆A)^CheckCovenant^CheckSigpkO)])，
```

在 **Assert** 交易中，创建了一个包含 \( k + 1 \) 个 Tapleaves（Taproot 叶子）的 Taptree（Taproot 树），这些叶子定义了不同的花费条件，具体取决于执行路径是成功（称为 "Happy Path"）还是被质疑（称为 "Unhappy Path"）：

1. **Happy Path（支付路径）**：
   - 第 \( k + 1 \) 个叶子允许操作员在预定的质疑期  $\Delta$<sub>A</sub>) 结束后花费该输出。
   - 花费条件包括：
     - 满足 **Covenant**（契约）条件。
     - 提供 **操作员的签名**。

2. **Unhappy Path（质疑路径）**：
   - 前  k 个叶子对应争议场景。每个叶子 i （其中 \( i $\in$ \{1, 2, ..., k\} ）包含以下内容：
     - 子程序  f<sub>i</sub> 、输入程序的公钥 pk<sub>z<sub>i-1</sub></sub>对应输入状态 z<sub>i-1</sub>以及输出程序的公钥 pk<sub>z<sub>i</sub></sub>对应输出状态 z<sub>i</sub> 。
   - 花费此类叶子需要：
     1. 满足 **Covenant**（契约）条件。
     2. 提供子程序  f<sub>i</sub> 的输入和输出公钥的正确承诺。
     3. 证明  z_i $\neq$ f<sub>i</sub>(z<sub>i-1</sub>) ，即操作员提交了错误的中间状态 z<sub>i</sub> 对应于子程序 f<sub>i</sub> 。

详细的争议处理逻辑在 **DisproveScript** 中呈现，描述了质疑操作员声明的具体步骤。

- 如果操作员作出了错误的声明，即对于某个 i $\in$ \{1, ..., k\} ，存在 z<sub>i</sub> $\neq$ f<sub>i</sub>(z<sub>i-1</sub> \) 的情况，任何观察方可以在质疑期 $\Delta$<sub>A</sub> 内进行质疑。
- 他们可以发布 **Disprove**（反驳）交易来展示不一致性并对操作员的执行提出异议。

4. **(i) 支付路径（Payout）：**  如果在质疑期 $\Delta$<sub>A</sub> 结束后，没有人发布 **Disprove**（反驳）交易，操作员可以发布 **Payout** 交易并将资金转移给自己。具体定义如下：

**Payout** := (*input* = [(**Assert**, 0, <**RealTimelock($\Delta$<sub>A</sub>)** ^ **TakeScript**>)],

​		   witness = [(**Coventant**, σO)],

​		    output = [(d_B, **CheckSig<sub>pko</sub>**)])

(ii) **反驳路径（Disprove）**：  
使用操作员在 **Assert** 交易的见证中对 z<sub>0</sub>, $\dots$, z<sub>k</sub> \) 的承诺，任何人都可以反驳操作员对程序执行中间状态的错误声明，即如果某个 z<sub>i</sub> $\neq$ f<sub>i</sub>(z<sub>i-1</sub>)，其中 \( i $\in$ \{1, $\dots$, k\} \)，就可以通过发布 **Disprove** 交易来实现反驳，该交易将花费 **Assert** 交易的输出。具体定义如下：

**Disprove** := (*input* = [(**Assert**, 0, <DisproveScript<sub>i</sub>)],

​			*witness* = [(Covenant, z<sub>i-1</sub>, z<sub>i</sub>, c<sub>zi-1</sub>, c<sub>zi</sub>)],

​			*output* = [(b_B, **False**), (a_B, **True**)]),

该交易的输出决定操作员提供的押金 d_B 的处理方式。具体来说，**Disprove** 交易必须创建一个新的输出，用来烧毁部分押金 b_B ，其中 \( 0 $\leq$ b $\leq$ d \)。通过设置 **SIGHASH** 标志为 **SIGHASH SINGLE**，上述的输入和输出变为不可变。这允许任何用户发布 **Disprove** 交易，同时添加进一步的输出，比如他们可以领取剩余的 a_B 作为赏金或用来支付交易费用，其中 \( a = d - b \)。为了简化，上述定义中的 (a_B, True) 表示这个操作。这一机制为用户提供了发布此交易的激励，确实允许任何人执行该操作并领取相应的赏金。

图 2 展示了比特币上的函数验证机制所涉及的交易。该机制主要包含以下关键交易：

![](https://private-user-images.githubusercontent.com/8327633/377767179-7cc000f8-1ee5-4d23-a080-a8a7a9700a1a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzY5NTgsIm5iZiI6MTcyOTIzNjY1OCwicGF0aCI6Ii84MzI3NjMzLzM3Nzc2NzE3OS03Y2MwMDBmOC0xZWU1LTRkMjMtYTA4MC1hOGE3YTk3MDBhMWEucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDczMDU4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9Y2RlYTVlOTNkMzkxMDQ1YTk0M2U3NjAyNzBmYzgyMzg3ZjcyODNmNWY1MjQ3Y2M4OWMxMGI3MTUzNjE0OTYwMiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.4GY9BgG3svSl6ig643PureKwAPn4Ma0gf57T6lp8EGk)

图 2. Assert、Disprove 和 Payout 交易的示意图。Assert 交易的输入可以是任何 UTXO，用“*”表示，但我们要求花费脚本为 Covenant 并包含对 z<sub>0</sub>, z<sub>1</sub>, \dots$, z<sub>k</sub> 的承诺。为了增加可读性，我们引入了以下颜色编码：**灰色**圆角矩形表示交易。我们通过**绿色**矩形表示锁定在交易输出中的 BTC，通过**橙色**矩形表示交易输入花费的 BTC。**蓝色**箭头表示操作员采用的支出路径，**红色**箭头表示其他人采用的支出路径。在箭头上方，我们标注了花费箭头起点输出的条件。灰色虚线矩形围绕一些交易的输入和输出，表示在使用不同于 SIGHASH ALL 的 SIGHASH 标志时，将这些部分哈希并预签名。

![](https://private-user-images.githubusercontent.com/8327633/377772373-546593ca-b68e-4b53-8f82-b346299e9360.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzc4MTYsIm5iZiI6MTcyOTIzNzUxNiwicGF0aCI6Ii84MzI3NjMzLzM3Nzc3MjM3My01NDY1OTNjYS1iNjhlLTRiNTMtOGY4Mi1iMzQ2Mjk5ZTkzNjAucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDc0NTE2WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ZDVmMTczMjFiNzdhYzE5NjNmZWUwYzI5MjQyYWIxZTQzNTE5MzMwYjU5OWZjODI5ZTNiYjA1NmMxYmQxMThkNyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.4VFd2GLpvrsO8qYufHvME-l2fxxcOQ-7-dp2svNXrF0)

### 5.2 成本优化的乐观函数验证  

在当前的设计中，**Assert** 和 **Disprove** 交易的规模都较大，导致发布到链上时的交易费用很高。我们通过扩展构建到乐观模型来解决这一问题，从而在诚实操作的情况下大幅减少链上开销。因此，最初操作员只需提交程序 f 的输入 x \，只有在被挑战者质疑时才必须公开输出和中间状态。为此，我们在**提交和挑战期**以及**支付期**引入了额外的交易。**设置**和**执行**部分保持不变。改进后的构建如图 3 所示。

1. **设置**。如果使用委员会来模拟 Covenant，则还需要创建并由签名委员会预先签署 **PayoutOptimistic** 交易。否则，该交易保持不变。

2. **执行**。保持不变。

3. **提交和挑战期**。在提交程序 **f** 的执行时，操作员首先发布一笔 **Claim** 交易（在公式 (4) 中定义），表明对于某个预定义集合 **V** 中的 **y = z<sub>k</sub>**，他们知道某个 **x = z<sub>0</sub>**，使得 **f(x) = y**。操作员可以通过两种方式说服潜在的挑战者他们知道这样的 **x**：将其发布到链上或通过其他通信渠道（如使用第三方数据可用性层）发送给他们。

![](https://private-user-images.githubusercontent.com/8327633/377779047-236cabdc-5a58-4a9b-bb7b-540e6c7eb6dc.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzg5NzUsIm5iZiI6MTcyOTIzODY3NSwicGF0aCI6Ii84MzI3NjMzLzM3Nzc3OTA0Ny0yMzZjYWJkYy01YTU4LTRhOWItYmI3Yi01NDBlNmM3ZWI2ZGMucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDgwNDM1WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ZTQ0Y2JiNDM1MmRmMjIwNTE0ZDg4ODBhMTdhZjEyM2Y1NTRkZmViZmU0MDE1Njk1NzVlM2JjZWNlMzQ5OTI3MSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.ItvDIaFnFyIP2jg3GHCoGvyW68wFXFUAmA7oO-pdju0)

![](https://private-user-images.githubusercontent.com/8327633/377779047-236cabdc-5a58-4a9b-bb7b-540e6c7eb6dc.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzg5NzUsIm5iZiI6MTcyOTIzODY3NSwicGF0aCI6Ii84MzI3NjMzLzM3Nzc3OTA0Ny0yMzZjYWJkYy01YTU4LTRhOWItYmI3Yi01NDBlNmM3ZWI2ZGMucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDgwNDM1WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ZTQ0Y2JiNDM1MmRmMjIwNTE0ZDg4ODBhMTdhZjEyM2Y1NTRkZmViZmU0MDE1Njk1NzVlM2JjZWNlMzQ5OTI3MSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.ItvDIaFnFyIP2jg3GHCoGvyW68wFXFUAmA7oO-pdju0)

再次需要注意的是，如果 **x** 已通过链下共享，则可以从见证数据中省略 **x**。**AssertScript** 的定义如下：

```css
AssertScript := CheckCovenant ^ CheckLampComm(pkz0) ^ ⋯ ^ CheckLampComm(pkzk);
```

在接下来的挑战期 **∆B** 中，任何人都可以通过发布一笔 **Challenge** 交易（在公式 (6) 中定义）来对该声明提出质疑，这会迫使操作员发布 **Assert** 交易，而 **Assert** 交易可以进一步被推翻。**Claim** 交易的第二个输出充当所谓的连接输出：它必须保持未花费状态，以便操作员在没有被挑战的情况下使用 **PayoutOptimistic** 从 **BitVM2** 中提取资金。

4. **(i) 支付路径（Payout）**。如果操作员的声明没有受到质疑，在 **∆B** 期过后，操作员可以发布 **PayoutOptimistic** 交易（在公式 (5) 中定义），以花费 **Claim** 交易的两个输出。通过这种方式，操作员从 **BitVM2** 中领取资金，并禁用争议逻辑。

![](https://private-user-images.githubusercontent.com/8327633/377782487-71b74240-d55f-4aa5-a29a-2209d18713ec.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzk1NTMsIm5iZiI6MTcyOTIzOTI1MywicGF0aCI6Ii84MzI3NjMzLzM3Nzc4MjQ4Ny03MWI3NDI0MC1kNTVmLTRhYTUtYTI5YS0yMjA5ZDE4NzEzZWMucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDgxNDEzWiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NDRjZTM1YWM4MzA5YjQ2MWFmNjZjZTEwMTE0ZDdkMGNkNDVjYzViNTRiYjk1YTViZTg1YjBlNmRmNzRiNWY0MiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.MncH4Y3TBAt1TcqEYePmVgRZ1uRWKzGPe6aHjBryqVU)

(ii) **挑战与反驳（Challenge and Disprove）**。任何挑战者都可以通过发布 **Challenge** 交易（在公式 (6) 中定义）来质疑声明，该交易花费 **Claim** 交易的第二个（连接器）输出，并禁用 **PayoutOptimistic** 交易，迫使操作员进入争议阶段。此时，操作员只能通过 **Payout** 交易访问 **BitVM2** 资金 **dB**，而这需要发布 **Assert** 交易。如第 5.3 节所讨论的，如果操作员的声明有误，**Assert** 交易可以被反驳。

![](https://private-user-images.githubusercontent.com/8327633/377783224-b7315b58-8847-4e22-9530-0aad685d9d8a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzk2ODMsIm5iZiI6MTcyOTIzOTM4MywicGF0aCI6Ii84MzI3NjMzLzM3Nzc4MzIyNC1iNzMxNWI1OC04ODQ3LTRlMjItOTUzMC0wYWFkNjg1ZDlkOGEucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDgxNjIzWiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9OWFjMGRiMTRkOGE3ODlmYWQ4ZDIwMGZmNmI0NWZjZDhiOGNkODE4ZTU0MGE4Y2RkMzkyOGNjOWJlN2Q5ZDdmMyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.W1j-LWSyIDMr6wOrKwh6Ll5hblg8iWKd27W6wCSg18A)

### **5.3 BitVM2：基于比特币的 SNARG 验证器**

我们可以使用该技术在比特币上编码一个 **SNARG** 验证器（参见第 3.2 节），特别是 **Groth16** 验证器（[7]），其甚至符合更强的 **SNARK** 验证器标准。其目标是使交易的执行取决于提供有效的 **SNARG** 证明。为了实现这一点，**Assert** 交易使用 **f** 作为 **SNARG** 验证器。该验证器被拆分为 **k** 个块，并按照第 5 节中的解释作为 **Tap leaves** 设置。操作员只能在提供正确验证的证明时执行 **Payout** 交易。否则，任何人都可以发布 **Disprove** 交易。

在 [26] 中提供了一个用比特币脚本编写的 **Groth16** 验证器的示例实现，表明在比特币上验证 **SNARKs**（因此也包括较弱的 **SNARGs** 概念）是可行的。第 4 图展示了最终 **BitVM2** 协议的概述。

![](https://private-user-images.githubusercontent.com/8327633/377784336-785e6550-5360-4422-b0e7-d2a178a8b307.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyMzk4NjYsIm5iZiI6MTcyOTIzOTU2NiwicGF0aCI6Ii84MzI3NjMzLzM3Nzc4NDMzNi03ODVlNjU1MC01MzYwLTQ0MjItYjBlNy1kMmExNzhhOGIzMDcucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDgxOTI2WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NmMxZWMwYWZhMTZkZWIxMTVjZGUxZmUyY2IxYWRlZWNlMjYwNjhkYWVlMGUxMGUxZGQ1MWQwYmQ2OWJhZjhlNyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.FQ6LM2ZzyyHhUQ_B6M6oEw4iARA_Csm2D4BKdGkYxKs)

### **5.4 通过抵押保护免受恶意挑战者攻击**

目前，任何用户都可以随时挑战操作员，迫使其公布 **Assert** 交易，这会增加链上费用，即使操作员是诚实的。为了防止这种针对操作员的恶意行为（称为 **griefing** 攻击），我们要求挑战者提供一定的抵押物 **cB**。该抵押物 **cB** 应根据链上 **Assert** 交易的费用进行参数化，以确保挑战者能覆盖相关费用。

**众筹挑战抵押物**。虽然抵押物可以保护操作员，但如果对个体用户来说挑战成本过高，可能会阻止他们发起挑战。我们可以通过众筹挑战抵押物来缓解这个问题。通过将 **Challenge** 交易的 **SIGHASH** 标志设置为 **SIGHASH SINGLE|ANYONECANPAY**，操作员可以预先签署交易的第一个输入和输出，并广播此签名，从而允许挑战者添加更多的输入。结果是，不再是单个挑战者需要独自承担挑战的前期成本，而是可以将抵押物的费用分摊到多个用户之间，从而摊薄成本。每个参与的挑战者可以添加自己的输入，并将交易发送出去，直到足够的输入覆盖 **cB**。

## **6 BitVM 桥：信任最小化的比特币桥**

BitVM2 的一个主要实际应用是用于在比特币（以及类似的区块链）和其他区块链网络、侧系统（sidesystems）之间实现信任最小化的桥接，特别是与所谓的 Layer 2 网络的桥接。在下文中，我们将这些网络统称为 **侧系统**。正如第 2 节所概述的，桥接的目标是创建比特币在侧系统上的“包装”（wrapped）表示（称为 **Bs**），可以在未来以 1:1 的比例赎回为比特币（**B**），这被称为“**peg-out**”（赎回）。

![](https://private-user-images.githubusercontent.com/8327633/377794281-2f7fc6f2-98c2-4624-afd7-af472a69f465.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkyNDE0MzMsIm5iZiI6MTcyOTI0MTEzMywicGF0aCI6Ii84MzI3NjMzLzM3Nzc5NDI4MS0yZjdmYzZmMi05OGMyLTQ2MjQtYWZkNy1hZjQ3MmE2OWY0NjUucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAxOCUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMThUMDg0NTMzWiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NzE4ZTRmY2JmZTY3MjI3ODQ0OWY3NzczMzY0OGNjNDVmYTA1MDMxYmUzMTAyZjQwOGY3N2ViODJhMTk1MmRmMCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.QpuNG5uBVZCJtfBhjiTARrWe6chax85FVgvGIxv26Sk)

图 4. BitVM2 协议的概述。首先，将 SNARG（或 SNARK）验证器（在一般情况下，这是一个任意程序）实现为比特币脚本，然后将其拆分为子程序 \(f<sub>1</sub>, $\ldots$, f<sub>k</sub>\)，使得每个子程序都可以在比特币区块中执行。当关闭 BitVM2 实例时，操作员承诺程序结果。如果受到质疑，操作员需要揭示起始、中间和结束状态 \(z<sub>0</sub>, $\ldots$, z<sub>k</sub>)。因此，任何人都可以证明 \z<sub>1</sub> = f<sub>2</sub>(z<sub>1</sub>)，从而驳斥有问题的操作员的声明。

跨链桥的设计需要一个可信的第三方【24】，并且其构建本质上非常复杂，这也是为什么如今所有比特币跨链桥都依赖于多签名设计和诚实多数假设的原因。接下来，我们展示如何将附加的信任假设（在比特币和侧链系统安全的基础上）减少到只需存在一个活跃的理性操作员和理性的挑战者，以实现平衡的安全性。我们注意到，以下假设基于正确的设置（CheckCovenant），如第4.2节所述，即要求签名委员会的存在性诚实性，以模拟契约。特别是，我们描述了一个所谓的轻客户端跨链桥协议，具体如下，并在图5中进行了说明。

**1. PegIn**: 用户 Alice (A) 通过 PegIn 交易（在等式 (7) 中指定）将 \(vB\) 存入 BitVM2 实例。该交易的单一输出使用 CheckCovenant 作为花费条件，确保它只能通过任何挑战者可以质疑的交易进行花费。侧链系统通过比特币轻客户端功能验证该 PegIn 交易是否已被包含在比特币区块链中，并在侧链系统中为 Alice 的账户铸造 \(v\) 个包装的比特币（Bs）。侧链系统的共识机制（例如通过智能合约）强制执行这一验证和铸币逻辑。

现在，Alice 可以自由地在侧链系统中使用这些 Bs。为了模拟真实场景，假设 Alice 将她的 \(vBs\) 转移给了另一位用户 Bob (B)，因此，Bob 在侧链系统中持有 \(vBs\)。

![](https://private-user-images.githubusercontent.com/8327633/378265870-9b48d770-a855-4ed6-86af-4d242e274cd8.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0ODIzMjksIm5iZiI6MTcyOTQ4MjAyOSwicGF0aCI6Ii84MzI3NjMzLzM3ODI2NTg3MC05YjQ4ZDc3MC1hODU1LTRlZDYtODZhZi00ZDI0MmUyNzRjZDgucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDM0MDI5WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MzdjZWNhMGFkNGFhM2IzZmFlOTJiM2FkZDA3OGZiYjBkMzBhNjA3YzcwZjI5ZWZjOGM0ODFmMTgxNWNiZmM0OCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.5zBah_siWt7hlB0gAi4w_SvIPmxIqPqDljGivE-o4bk)

**2. PegOut**: Bob 想要将他的 \(vBs\) 提回比特币。为此，他在侧链系统中使用一个 Burn 交易来销毁 \(vBs\)，使这些 Bs 无法再被使用。

**理想功能**: 在理想情况下，Bob 现在应该能够通过 PegOutideal 交易，使用之前 Alice 通过 PegIn 交易锁定的 \(vBs\)。这个理想的 peg-out 交易应当验证 Burn 交易的正确性，并确认它已被侧链系统包含，使用 SidesystemLCScript 来实现侧链的轻客户端功能。

然而，当前这一理想功能无法直接在比特币脚本中实现，原因之一是当 PegIn 交易发生时，执行 PegOut 的用户（即 Bob）还不确定，因此无法强制将 PegIn 交易与 PegOutideal 交易之间建立可执行的关联。

### **6.1   稻草人桥接协议

在本节中，我们概述了一个 BitVM2 桥的初步设计，它通过乐观的 SNARG 验证器来模拟理想的 PegOut 功能（PegOutideal）。在这里，我们假设侧链系统作为一个 rollup 运行，并依赖比特币进行共识。具体来说，侧链系统使用发布到比特币区块链的数据承诺来确定交易顺序，并结合状态转换验证的方法。这一设计通过只专注于验证包含在比特币区块链中的交易来简化验证过程。因此，BitVM2 只需要实现一个比特币轻客户端，而不需要同时实现侧链系统和比特币轻客户端。

我们在图 5 中展示了这一初步协议。一个完全运作的桥包括多个 BitVM2 实例，每个实例都通过 PegIn 交易存入了预定义数量的 \(vB\)。按照设计，不同 BitVM2 实例中的包装 BTC (Bs) 是可互换的，也就是说，Bob 执行 PegOut 时，不需要特定选择某一个 BitVM2 实例，侧链系统可以决定选择哪个实例。

![](https://private-user-images.githubusercontent.com/8327633/378267310-c5e113d1-e2bc-4cb5-bf5c-c1b621de74d2.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0ODI4NzgsIm5iZiI6MTcyOTQ4MjU3OCwicGF0aCI6Ii84MzI3NjMzLzM3ODI2NzMxMC1jNWUxMTNkMS1lMmJjLTRjYjUtYmY1Yy1jMWI2MjFkZTc0ZDIucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDM0OTM4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9YTJiYmM4ZmUyNjk3NzY2NmUyNWE5YWYzOTM5YmI1MDY1NzAzNzQzMDk2OGQwZGVhNGM0NzMyNzUwNzcwNWM5MSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.K_q5HUkIgxuXiKMAwKN5vMBxBybr01uQlYoXE6rkRwo)

图5所示。展示了稻草人桥接协议的说明。在我们的桥接协议中，侧系统轻客户端为比特币轻客户端。

**初始化 SNARG**  回顾第 3.2 节，SNARG 是通过关系 R\（通过算法 SNARG.Setup）定义的。在我们的例子中，我们关心的关系 R涉及对 ($\varphi$, w) 的配对，其中声明 $\varphi$ 包含两个区块哈希，见证 w 是一个区块链。声明 $\\varphi$ 是正确的，即 ($\varphi$, w) $\in$ R\，如果存在一个区块链 w，并且满足以下条件：

1. 区块链的第一个和最后一个区块哈希是 $\varphi$；
2. 区块链是有效的，即每个区块都引用它的父区块，并且其哈希值小于设定的目标；
3. 区块链包含操作员 O 的 PegOut 交易，即 O 向 Bob 支付 PegIn 交易的金额（可能在 OP_RETURN 输出中引用 PegIn 交易的哈希）；
4. 区块链的某个区块中包含侧链系统的状态，其中侧链系统包含 Bob 的 Burn 交易。

更正式地，设 H : \{0, 1\}<sup>$\ast$</sup> $\to$ \{0, 1\}<sup>$\lambda$</sup> 为哈希函数（建模为随机预言机）， $\varphi$ $\in$ \{0, 1\}<sup>$\lambda$</sup> $\times$ \{0, 1\}<sup>$\lambda$</sup> 为两个区块哈希，B $\in$ \{0, 1\}<sup>bl</sup> 为表示一个区块的比特串，区块包含字段 B.parent $\in$ \{0, 1\}<sup> $\lambda$ </sup>，w $\in$ (\{0, 1\}<sup>bl</sup>)<sup>$\ast$</sup> 为区块列表（区块链）。我们定义 R := \{($\varphi$, w) : $\Pi$($\varphi$, w) = True\}，其中 $\Pi$ 定义在算法 5 中。注意，我们用函数 SidesystemState(w) 抽象了从区块链 w 中提取侧链系统状态的过程，该函数返回侧链系统的状态，记录在比特币区块链上。实际的实现依赖于侧链系统的操作方式。

![](https://private-user-images.githubusercontent.com/8327633/378271625-2270d994-a0b9-429a-9f9d-186f104a7cbf.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0ODQ1NDgsIm5iZiI6MTcyOTQ4NDI0OCwicGF0aCI6Ii84MzI3NjMzLzM3ODI3MTYyNS0yMjcwZDk5NC1hMGI5LTQyOWEtOWY5ZC0xODZmMTA0YTdjYmYucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDQxNzI4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NTRiOTE4ODRmNTBiYWQ0ODJmYzA5NzY3NDgxMzI3NzM0ZTkwZjcwNmI1NTQ3Mjk1NDY1YjA4NTQyNWI2OTI2MCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.wvZ5cmXcQnqYU7ZJCGhWS1uTVETp8Dg27rD07OAsOu0)

**通过 OP_BLOCKHASH 模拟比特币轻客户端**   桥的一个关键组成部分是能够在 BitVM2 实例中检查和验证比特币的状态。这种验证对于确认 PegOut 交易是否正确执行并与侧链系统的 Burn 交易保持一致至关重要。由于比特币目前没有内置对这种功能的支持，因此需要在 BitVM2 中实现一个比特币轻客户端。我们强调，使用连接输出来确保这两个步骤的原子性，而不是依赖轻客户端，是不可行的。特别是，将连接输出添加到 PegOut 交易并将其用作 Payout 交易的输入是不可能的，因为在桥接的场景中，无法提前知道提取用户Bob的身份，因此在设置过程中无法确定所需的交易哈希。

为了简化操作，我们从一个假设的方案开始，使用虚拟的比特币脚本操作码 OP_BLOCKHASH，并在第6.2节展示了一个无需额外操作码的实际比特币轻客户端构建。我们假设 OP_BLOCKHASH 从比特币脚本执行栈中获取一个元素，将其解释为一个数字，然后获取该高度的区块并将其哈希值放入栈中。

在这个初步方案中，操作员在 Assert 交易的输入中包括一个发生在 PegOut 交易之后的区块高度 h<sub>end</sub>，并同时提交对 z<sub>0</sub>, $\dots$, z<sub>k</sub>的承诺（因此，是等式 (1) 的略微修改版）。通过 OP BLOCKHASH 获取与 h<sub>end 对应的区块哈希 H(B<sub>end</sub>)，然后检查 H(B<sub>_end</sub>) 是否属于 z<sub>0</sub>。这确保了 SNARG 的参数化是正确的，从而可以验证 PegOut 交易被包含的区块确实属于比特币区块链。回想一下，对 z<sub>0</sub>的承诺在 AssertScript 和 DisproveScript1 中得到验证。因此，我们使用 OP_BLOCKHASH 确保属于 z<sub>0</sub> 的区块哈希 H(B<sub>end</sub>) 并输入到 SNARG.Vrfy 中的确是比特币区块链的一部分。

创世区块 B<sub>start</sub>的高度 h<sub>start</sub> := 0（或桥设置过程中约定的比特币区块链中的其他区块）可以在设置时硬编码。这样我们有 $\\varphi$ = H(B<sub>start</sub>, H(B<sub>end</sub>))。注意，SNARG 只有在提供的声明 $\\varphi$ 正确时才会返回 True。因此，如果结果 z<sub>k</sub> 为 True，这意味着 Burn 和 PegOut 交易确实发生了。

**初步协议**   我们继续描述初步桥接协议的各个阶段。通过 BitVM2 模拟理想的 PegOut<sub>ideal</sub>功能，并使用操作员和挑战者确保 peg-out 正确处理。协议概述如下，依赖关系见图 6 中的交易可视化。

1. **设置阶段 (Peg-In)**  按照 BitVM2 的设计，SNARG.Vrfy 代表了被分割成子程序的程序 f。回顾一下，SNARG.Vrfy 的输入为 \(pR, crs, $\varphi$, $\pi$)。由于关系 R 和公共参考字符串 crs 在设置时已知，它们可以硬编码，从而将输入形式简化为 z<sub>0</sub> = ($\varphi$, $\pi$)，其中包括声明 $\\varphi$和 SNARG 证明 $\pi$。除此之外，设置步骤与标准 BitVM2 实例保持一致（见第 5.2 节）。用户 Alice (A) 通过 PegIn 交易（如等式 (7) 所示）存入 \(vB\)。

2. **执行阶段 (Peg-Out)**  
   首先，Bob 在侧链系统上发布 Burn 交易。接着，Bob 创建一个未完成的 PegOut 比特币交易（如等式 (9) 所示），其输入来自 Bob 且值为零，输出将 v - f<sub>O</sub> 分配给他自己，其中 f<sub>O</sub> 是操作员预先商定的提取手续费。Bob 为该交易设置了 SIGHASH SINGLE|ANYONECANPAY 标志，允许其他用户通过提供尚未支付的 vB 来完成交易。

   在这一阶段，操作员开始介入：其中一个操作员从自己的资金中提前支付 vB。因此，所有操作员都竞争发布 PegOut 交易以赚取相关费用，但只有其中一笔交易会被比特币区块链接受。成功协助 peg-out 的操作员随后通过花费 PegIn 交易的输出来回收预付款的 vB。由于每个 BitVM2 实例的 m 个操作员集是预定义的，我们可以通过 CheckCovenant 来强制执行回收过程。在缺少必要的比特币操作码的情况下，CheckCovenant 通过一个委员会 n-of-n 多重签名（参见第 4.2 节）来预签交易以进行仿真。操作员随后将 PegOut 交易发布到比特币区块链，完成对 Bob 的提取。![](https://private-user-images.githubusercontent.com/8327633/378310669-57e43c64-4f15-4c45-92f7-4a0e437f4223.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTQ5NTgsIm5iZiI6MTcyOTQ5NDY1OCwicGF0aCI6Ii84MzI3NjMzLzM3ODMxMDY2OS01N2U0M2M2NC00ZjE1LTRjNDUtOTJmNy00YTBlNDM3ZjQyMjMucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDcxMDU4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9YmViYTViZWExOTFmY2FmYTJkZDEwZDA5YjY0OWUxMzNkZjQwYTUxNWQwZjY0YzQ0NTJhMmU4YTA1NDg2ZWMwMyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.ssqK__lvhgYz5Td9-J3ot_hXByF6arobbNYTbGPu9fk)

3. **承诺与挑战期**  
   随后，操作员发起索赔以从 BitVM2 中收回预支付的 vB（即 PegIn 输出）。这是通过发布 Claim 交易（如等式 4 所示）来完成的。在操作员能够花费这些资金之前，必须验证该操作员是否正确处理了 Bob 的 peg-out，即与侧链系统上的 Burn 交易相对应的 PegOut 交易已包含在比特币区块链中。所有挑战者（包括其他操作员）在本地机器上执行此验证，并在有争议时在超时 $\Delta$<sub>B</sub> 内发布 Challenge 交易。这确保了 (i) 只有在侧链系统上发生 Burn 交易时 peg-out 才会发生，并且 (ii) 防止操作员窃取桥中锁定的比特币。

4. **支付或证明无效**  
   在挑战期结束后，操作员通过发布 PayoutOptimistic 完成提取。争议的处理方式与其他 BitVM2 程序相同：挑战者发布 Challenge 交易，操作员必须通过发布 Assert 交易作出回应，挑战者则可以使用 Disprove 交易对操作员进行惩罚（或诚实的操作员可以在超时$ \$\Delta$<sub>A</sub> 后使用 Payout 来领取 \(vB)。

![](https://private-user-images.githubusercontent.com/8327633/378312549-fe182e19-5cde-4173-bb16-9b444cbe785e.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTUzNTYsIm5iZiI6MTcyOTQ5NTA1NiwicGF0aCI6Ii84MzI3NjMzLzM3ODMxMjU0OS1mZTE4MmUxOS01Y2RlLTQxNzMtYmIxNi05YjQ0NGNiZTc4NWUucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDcxNzM2WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ZjM3ZTI0MGI1MjMxNjk1NmUzMzAzNjVkMzdkMzU1OTNjZjFlNzhhY2IwNTRmNDE4MzI0MWUxZjc3ZGIwNzgwMiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.PuyW-3daYWdbJ5JJyBhAdFt-pZHl2fhfpvBPtU7WaAY)

图 6. 使用假设的操作码 OP_BLOCKHASH 实现交易验证。输入 z<sub>0</sub> := ($\varphi$, $\pi$) 包含声明和 SNARG 证明。前者 $\varphi$ := (H(B<sub>start</sub>), H(B<sub>end</sub>)\) 由两个区块哈希组成。在 Assert 交易中，操作员还承诺一个高度 h<sub>end</sub>，并且检查 OP_BLOCKHASH(h<sub>end</sub>)是否等于 H(B<sub>end</sub>)。

### 6.2 BitVM 桥：兼容比特币的桥协议

在本节中，我们展示如何使用 BitVM2 的乐观 SNARG 验证器实现一个比特币轻客户端。通过将该轻客户端与我们在 6.1 节中描述的 Strawman 桥设计相结合，创建了第一个与现今比特币兼容的、信任最小化的桥。  
为了确保桥的设计安全性，我们需要重新审视比特币轻客户端的必要功能。我们需要验证两点关键内容：(i) 确认 Burn 交易在侧链系统中正确包含，并 (ii) 确认相应的 PegOut 交易在操作员 O 通过 Claim 交易请求退款前，已包含在比特币区块链中。这确保 Bob 烧毁的 vBs 数量与 O 在比特币上发送给 Bob 的 vB 数量相匹配（减去操作员的费用 f<sub>O</sub>）。  
我们的轻客户端面临的主要挑战是验证 PegOut 对应的区块 \(B_{\text{end}}\)，其哈希作为见证 \(w\) 的一部分被传递给 SNARG 验证器，属于与 Assert 交易相同的区块链。换句话说，我们需要模拟假设的操作码 OP_BLOCKHASH。SNARG 验证器可以检查给定的链是否有效，且所关心的交易是否是链的一部分，但它不能进行区块链内省。因此，恶意操作员可能会尝试挖掘包含 PegOut 交易的私人比特币分叉，并生成有效的 SNARG 证明。然后，该证明可用于通过 Assert 和 Payout 交易取走资金，而不在主链上发布 PegOut。

**PowPV：使用超级区块的轻客户端**  我们提出了一个实际的轻客户端模型，利用所谓的超级区块，即超过最小难度目标的比特币区块。

**超级区块**  回顾比特币，每个难度周期（大约两周内的 2016 个区块）必须超出一定的难度目标才能算有效。区块的哈希值前导零越多，该区块的工作量证明难度就越高。结果是，我们可以根据区块的哈希值来对比特币区块进行排序。在给定的时间段内 \( \Delta C \)，将有唯一的区块，即哈希值最小的“最重”区块（假设碰撞抵抗哈希函数，这种情况除非极小概率事件发生）。挖到该区块的概率与比特币矿工的相对算力（相对于网络的其余部分）成正比。

**替代 OP_BLOCKHASH**  由于比特币缺乏 OP_BLOCKHASH 操作码，我们需要确保在 SNARG.Vrfy 函数的输入中提供的区块哈希 H(B<sub>start</sub>) 和 H(B<sub>end</sub>) 属于与操作员在 Assert 交易中提交的同一链条上的区块。证明 Bend 位于最长链上是足够的，这确保如果正确的 PegOut 未包含在主链中，SNARG.Vrfy 函数将输出 False，从而让挑战者轻松否定私有分叉提交。由于启动区块 B<sub>start</sub> 在 BitVM2 设置期间硬编码，例如创世区块，因此它肯定在主链上。我们主要关注运行时操作员在 Assert 交易中提供的 h<sub>end</sub>。

我们通过确保 B<sub>end</sub> 是在操作员发起从 BitVM2 取回资金过程后挖出的区块来解决此问题。该区块的选择必须是确定性的，同时操作员不能轻易预测或操纵它。这正是我们使用超级区块的原因。

**轻客户端协议**  我们略微修改了我们的 SNARG 设置，使用修改后的关系 R_1，并在 BitVM 桥协议的“承诺和挑战期”阶段插入逻辑。

1. 操作员通过在比特币上发布新的 KickOff 交易（参见 Eq. 10）来启动回收流程。

   ![](https://private-user-images.githubusercontent.com/8327633/378320704-ba3d54f3-5c0c-4af3-9337-56ffb6ed182b.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY4MjQsIm5iZiI6MTcyOTQ5NjUyNCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMDcwNC1iYTNkNTRmMy01YzBjLTRhZjMtOTMzNy01NmZmYjZlZDE4MmIucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0MjA0WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MTY4ODg2ODFhOWFiYWU3N2YxOGEwZjcxZjBjNWZmYzUwMmIxZmU0ZGExZTkyMjkwNGRiNWI2N2E1MTZhM2ExNCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.DwZ2n0xf5AKwhDc0H3_nex5dMwgkARFPlXGmCYmNu28)

2. 紧接着，操作员必须发布 StartTime 交易，承诺当前时间 TS。此时间 TS 标记超级区块测量周期的开始，该周期持续时间为 ΔC（例如 2000 个区块），在此期间操作员必须观察主链上的所有区块，并识别出最重的超级区块 SB。

   ![](https://private-user-images.githubusercontent.com/8327633/378320959-0e041d0c-fb1a-4722-8078-576b7a7133d2.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY4NjQsIm5iZiI6MTcyOTQ5NjU2NCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMDk1OS0wZTA0MWQwYy1mYjFhLTQ3MjItODA3OC01NzZiN2E3MTMzZDIucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0MjQ0WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MjMxNjE3NTAyNmNmZjU5MzU3OTlmODg2MmIwMjlhYmFiZDZjMjRjZTkwMGE0NDc5ZmU5YjI3YzRkNmExYTM2OSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.YRhiKJ-lhurFoQp1FE6GSRoX4Bo50KoxZxR1wyyQL0s)

3. KickOff 交易的两个连接输出可以通过三种方式支出。

​	3.a 在 ΔL 之后，如果操作员未发布 StartTime 交易。此时，挑战者可以通过 TimeTimeout 交易（参见 Eq. 12）支出两个输出。这将使 Claim 交易无效，确保操作员无法在测量之前预挖私有链。

​	![](https://private-user-images.githubusercontent.com/8327633/378321073-5a5384d9-8d97-4080-9bdb-623282b5df4e.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY4ODUsIm5iZiI6MTcyOTQ5NjU4NSwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMTA3My01YTUzODRkOS04ZDk3LTQwODAtOWJkYi02MjMyODJiNWRmNGUucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0MzA1WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9OGJiYjY4ODUzZDZiYzNiNGE4YjA2YTQzNmM0YjZlMzNhZTc1ZGExNDE4YTVlYmRlMWI5YWQ1YmY2MzgzNjc1NiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.UwvMhjiCbgSmRjrUq7oOMvCu9VG8Y1ZzYNa9ufl4BOI)

​	3.b 在 ΔC 之后但在 ΔC + ΔL 之前。假设操作员正确发布了 StartTime 交易，他们现在发布 ClaimLC 交易（参见 Eq. 13，一个修改版本的 Claim），在此交易中他们承诺认为是最重的超级区块 SBO。

​	![](https://private-user-images.githubusercontent.com/8327633/378321177-4cbd6b31-cd79-4e35-8b68-0b007e518974.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY5MDQsIm5iZiI6MTcyOTQ5NjYwNCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMTE3Ny00Y2JkNmIzMS1jZDc5LTRlMzUtOGI2OC0wYjAwN2U1MTg5NzQucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0MzI0WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NTkzM2JlZTEyZDhhNjc1NjQ3YmY0YWRlMmQxZmQyMzAyZGUxY2YzODU1NmY3YWRlN2FmZjZjMTdmYzMwNmEyZCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.sCsJJGidhyC2GvH8i79RbDcPtTTN30UQ2MOWnfJUCFU)

​	3.c 在 ΔC + ΔL 之后，如果操作员未发布 ClaimLC 交易。此时，挑战者通过 ClaimTimeout 交易（参见 Eq. 14）支出第一个输出。这将使 ClaimLC 交易无效，并确保恶意操作员最多只有与诚实矿工相同的时间 ΔC（加上活跃参数 ΔL）来尝试挖掘私有分叉。

​	![](https://private-user-images.githubusercontent.com/8327633/378321355-dc31d216-0183-45ff-b0e0-7b181096a161.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY5MzgsIm5iZiI6MTcyOTQ5NjYzOCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMTM1NS1kYzMxZDIxNi0wMTgzLTQ1ZmYtYjBlMC03YjE4MTA5NmExNjEucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0MzU4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ZDExOWM0YjYyZjI2NzI2ZjE2MGE5M2IxMmRmZmY3NTFlYWU0NzQ4ZjFlNzgwOGE5NDFjNDgxOWIwODJlMTg3NyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.nb0T7ejaZJNtGbKxFu0VvKYBOMCcMtHxGw5AENFT0jw)

4. 一旦 ClaimLC 交易被包含在比特币区块链中，挑战者将审查超级区块 SBO，并将其与他们观察到的最重的超级区块 SBV 进行比较。任何挑战者都可以在 SBO ≠ SBV 且 SBO.weight < SBV.weight 的情况下，使用 DisputeChain 交易对操作员的声明提出异议，即操作员承诺的区块的重量低于在 ΔC 期间挖出的比特币主链上的最重超级区块。ClaimLC 交易引入了相对时间锁 ΔL，确保 Assert 交易只能在 ΔL 时间过去后发布，为挑战者提供时间以发布 DisputeChain 交易。为了保护诚实的操作员免受恶意挑战者错误争议的影响，挑战者只能在提供了在 TS 和 TS + ΔC 之间挖掘的区块 SBV 的情况下发布 DisputeChain 交易。

   ![](https://private-user-images.githubusercontent.com/8327633/378321495-944dd154-4b6a-499e-96a0-944239975172.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY5NjgsIm5iZiI6MTcyOTQ5NjY2OCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMTQ5NS05NDRkZDE1NC00YjZhLTQ5OWUtOTZhMC05NDQyMzk5NzUxNzIucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0NDI4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MjdmNjRlZmE3OGU0MzYxNDM5MmIxY2E1N2NmZmM2MzMyMTRiMDQyOGY5YzFkM2JlZWM2ZjM4OTI2MDVkMjkyNyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.tX89B_jqpHctDRPL3bq0jVLN0N4ACtTOplgur0IMh0A)

5. 如果挑战者不对超级区块承诺提出异议，我们将按照通常的协议路径继续，即挑战者可以通过发布 Challenge 交易对桥的执行提出挑战，或者操作员在 ΔB 后使用 PayoutOptimistic 交易完成从 BitVM2 领取资金。我们注意到，输入 z0 = (ϕ, π) 传递给 SNARG 验证器，由声明和证明组成，其中 ϕ = (H(B_start), H(B_end), TS, SBO) 包含 TS 和 SBO。操作员在 Assert 交易中承诺 z₁₀，这与 z₀ 相同，但缺少 TS 和 SBO。DisproveScript1 经过略微修改：它检查 z₁₀、SB 和 TS 的公共密钥的 Lamport 承诺。然后，z₀ 从 z₁₀、SB 和 TS 中重建，执行 f₁(z₀)。

为了攻击这个轻客户端协议，恶意操作员必须在 ΔC + ΔL 的期间内挖掘出一条比主比特币区块链更长的链，并找到最重的超级区块。成功的概率与操作员的哈希率成正比。测量期 ΔC 越长，成功概率越精确，因为方差减少。直觉上，这为经济安全提供了足够的保障，以劝阻攻击者，因为他们必须在非主链的分叉上浪费大量哈希率，从而放弃他们的区块奖励。

**处理比特币难度调整**  为了简单起见，我们的 PowPV 轻客户端设计目前假设比特币难度目标 T 是恒定的。我们注意到，Π₁ 可以轻松修改以编码区块链 w 的复杂难度重调整。安全的关键部分是在 TS 和 TS + ΔC 之间的区块难度。首先，建议操作员选择 TS，使得 TS 和 TS + ΔC 之间没有难度变化。其次，我们无法提前知道 TS 和 TS + ΔC 之间的难度，因为在设置时间时，我们只知道当时的难度目标 T<sub>setup</sub>。

为了解决这个问题，我们可以将 T<sub>setup</sub>（可能通过 R≥0 中的一些常数缩放）硬编码到 Π₁ 中，以适用于 TS 和 TS + ΔC 之间的区块。选择更高或更低的难度始终是安全性（较低的难度使得操作员更容易伪造证明）和活跃性（高于实际难度的难度使得操作员在难度足够高之前无法完成提款）之间的权衡。我们正在撰写更全面的论文，其中提出（i）一种基于历史难度变化和设置时间与 TS 之间的时间差估计未来难度的方法，以及（ii）一种从超级区块 SB 中读取难度的方法。

## 7 分析

请注意，这项工作仍在进行中，尚未形成正式分析。作为初稿，我们仅概述了当前针对这篇论文更广泛版本所进行的分析和证明策略。

在我们的模型下，主要存在两种情况下操作员或挑战者可能会偏离协议规范。我们的分析将主要集中在操作员和挑战者拥有计算能力的情况，即他们是矿工或与矿工串通，因为当其中一方拥有算力时，证明安全性相对简单。我们可以忽略恶意用户 Bob（与操作员串通）试图通过 Payout 交易从 BitVM2 实例中提款，而没有在辅助系统上发布正确的 Burn 交易的攻击。这是因为我们假设辅助系统的状态已记录在比特币区块链中，因此会导致 SNARG.Vrfy 函数执行失败，进而引发挑战。

![](https://private-user-images.githubusercontent.com/8327633/378321682-adfc6329-cee4-47de-9eba-4c7fb8eebf46.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTY5OTgsIm5iZiI6MTcyOTQ5NjY5OCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyMTY4Mi1hZGZjNjMyOS1jZWU0LTQ3ZGUtOWViYS00YzdmYjhlZWJmNDYucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDc0NDU4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MGRkNmU4MTdmOGM5NDVkN2FlZWM5YmU1OWNmYmU5MTZkMmJmNTY2ODQ5YzRjZDE2YjVlNWFjMmViZGMyYzExMiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.87hvqaQH1NAFf_9RqVaj7c67cBloAqR4JFWEIvuS4Ac)

**偏离的操作员** 操作员可以尝试窃取资金 v，如果他们在未发布 PegOut 交易的情况下设法发布 Payout 或 PayoutOptimistic 交易（本应将资金给 Bob）。我们在这里提出，理性的挑战者会通过发布 Challenge 交易来阻止操作员发布 PayoutOptimistic。对此感兴趣的一方包括 Bob（否则他会损失资金）和其他在桥中质押了资金的用户。剩下的情况是操作员试图发布 Payout，这只有在没有挑战者发布 Disprove 交易的情况下才有可能，即操作员能够发布 Assert 交易而不被挑战，尽管他们没有发布 PegOut 交易。

由于我们没有 OP_BLOCKHASH 指令可用，6.2 节中提出的轻客户端引入了一些风险。具体而言，操作员必须在 TS 和 TS + ΔC 期间找到最长链的区块 SBO，并在相同的时间段内挖掘尽可能多的具有足够难度的区块。在完整版本中，我们将量化这种风险，并进一步提供经济分析，即确定攻击者需要花费多少钱来执行此类攻击，并将其与可能的收益进行比较。我们最终将通过展示理性的操作员会正确遵循协议规范来结束分析。

![](https://private-user-images.githubusercontent.com/8327633/378328293-8c33bf77-4cca-4525-a1e9-55c632203c7e.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0OTgxNzgsIm5iZiI6MTcyOTQ5Nzg3OCwicGF0aCI6Ii84MzI3NjMzLzM3ODMyODI5My04YzMzYmY3Ny00Y2NhLTQ1MjUtYTFlOS01NWM2MzIyMDNjN2UucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAyMSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMjFUMDgwNDM4WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9ODE3MjY1MGY5NmYxNTU2YzBmNWI2YmEwOTIyMmUxM2EwOTdjY2Y5NDRjMWRiNjQwMThmMTlkOWZhNTliN2E0OCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.XnJ6thF0Hf6hlxWMsUPTkZoqF_R4Y-FusIzqpRHAT8I)

**偏离的挑战者** 原则上，挑战者可以通过发布 DisproveChain 交易来惩罚一个诚实的操作员，即使操作员已发布 PegOut 交易，也能烧掉操作员的大部分押金 d，并获得少量的费用 a。在这种情况下，操作员还会损失来自 PegOut 交易的 v - fO 硬币。为了做到这一点，挑战者需要在 TS 和 TS + ΔC 期间生成一个伪造的最长链区块，其重量超过该时间段内诚实挖出的最长链区块。同样，我们将量化这一风险，并分析攻击者需要花费多少资金来进行攻击。值得注意的是，攻击者只能获得非常少量的 a，这使得此类攻击主要是“悲痛攻击”。我们最终将通过展示任何理性的挑战者都会正确遵循协议来结束分析。
