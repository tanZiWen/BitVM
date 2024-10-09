# BitVM:在比特币上计算任何东西

作者：Robin Linus
邮箱：robin@zerosync.org

时间：2023.12.12

## 摘要

BitVM是一种表达图灵完备比特币合约的计算范式。它不需要改变网络的共识规则，不是在比特币上执行计算，而是验证，类似于**Optimistic Rollups**。证明者声称给定函数对某些特定输入求值为某些特定输出。如果该声明是错误的，那么验证者可以执行简单的欺诈证明并惩罚证明者。使用这种机制，任何可计算的函数都可以在比特币上进行验证。

在**Taproot**地址中提交大型程序需要大量的链下计算和通信，然而，由此产生的链上足迹是最小的。只要双方合作，他们就可以执行任意复杂的、有状态的链下计算，而不会在链中留下任何痕迹。只有在发生争议的情况下才需要链上执行。

## 引言

通过设计，比特币的智能合约功能被简化为基本操作，例如**签名**、**时间锁**和**哈希锁**。BitVM为更具表现力的比特币合约和链下计算创造了一个新颖的设计空间。潜在的应用包括国际象棋、围棋或扑克等游戏，特别是比特币合约的有效性证明验证。此外，它还可以将BTC跨链到其他链、建立一个预测市场，或者模拟新的操作码。这里提出的模型的主要缺点是它仅限于具有证明者和验证者的两方设置。另一个限制是，对于证明者和验证者来说，执行程序都需要大量的链下计算和通信。然而，这些问题似乎有可能通过进一步的研究来解决。在这篇文章中，我们专注于**两方BitVM**的关键组成部分。

## 架构

与**Optimistic rollup**[1]和**MATT**提案(Merkelize All the Things)[2]类似，我们的系统是基于欺诈证明和挑战响应协议。然而，BitVM不需要改变比特币的共识规则。底层的原理相对简单。它主要基于**哈希锁**、**时间锁**和大型**Taproot**树。

证明者需要逐步提交程序，但要验证所有这些计算链上的成本太高，因此验证者执行一系列精心设计的挑战，以简洁地反驳证明者的错误声明。证明者和验证者共同预先签署了一系列挑战-响应交易，他们以后可以使用这些交易来解决任何争议。

该模型旨在简单地说明这种方法允许在比特币上进行通用计算。对于实际应用，我们应该考虑更有效的模型。

该协议很简单:首先，证明者和验证者将程序编译成一个巨大的二进制电路。证明者在**Taproot**地址中提交电路，该地址具有电路中每个逻辑门的叶子脚本。此外，它们预先签署一系列交易，从而在证明者和验证者之间实现挑战-响应游戏。现在他们已经交换了所有必需的数据，因此他们可以将链上存款到生成的**Taproot**地址。

这激活了合约，他们可以开始交换链下数据，以触发电路中的状态变化。如果证明者做出了任何错误的声明，验证者可以拿走他们的押金。这保证了攻击者总是会损失他们的存款。

## 比特承诺

比特承诺是系统中最基本的组成部分。它允许证明者将特定位的值设置为“0”或“1”。特别的，它允许证明者跨不同的脚本和**UTXO**设置变量的值。这是关键，因为它通过将比特币虚拟机拆分为多笔交易来扩展其执行运行时。

与**Lamport**签名[3]类似，承诺包含两个哈希值，**hash0**和**hash1**。稍后，证明者通过显示*preimage0*（hash0的原像）将比特的值设置为“0”，或者证明者通过显示*preimage1*（hash1的原像）将比特的值设置为“1”，如果在某一点上，它们同时显示了原像*preimage0*和*preimage1*，然后验证者可以将其作为欺诈证明，并收取证明者的押金。这就是所谓的模棱两可。能够惩罚模棱两可是使承诺具有约束力的原因—这是一种“基于激励的承诺”。将比特承诺与时间锁结合起来，验证者可以强制证明者在给定的时间范围内决定特定比特的值。

![](https://private-user-images.githubusercontent.com/8327633/374852168-1dbbf718-2b92-4700-90a6-8dd00445b670.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjg0NTkwODMsIm5iZiI6MTcyODQ1ODc4MywicGF0aCI6Ii84MzI3NjMzLzM3NDg1MjE2OC0xZGJiZjcxOC0yYjkyLTQ3MDAtOTBhNi04ZGQwMDQ0NWI2NzAucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAwOSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMDlUMDcyNjIzWiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MjliYmEyYzllZTE2ZmE5Zjk0MDExOWE0Mjk0Yjk5NmI2NTdhN2QxNTMxZTZlMjU0ZWI5OWFiMTZlMTEwYjcwMSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.qduVM-tTPNMk6Zr19fQDnKi2BvPyTZ_8hweUB2aIvnQ)

**图1:**1位承诺的具体实现。为了解锁这个脚本，证明者必须显示**hash0**或**hash1**的原像。在这个示例执行中，证明者显示了**hash1**，并将该位的值设置为“1”。我们可以拥有该承诺的副本，以便在不同的脚本中强制执行特定的值。

为简单起见，从这里开始，我们假设有一个操作码**OP_BITCOMMITMENT**，它是上面脚本的简写。操作码消耗两个哈希值和其中一个哈希值的原像。它在堆栈上放一个位值，根据这个值来匹配原像的哈希值。

## 逻辑门承诺

任何可计算函数都可以表示为布尔电路。**NAND**门是一种通用逻辑门，因此可以由它们组成任何布尔函数。为了保持我们的模型简单，我们展示了我们的方法适用于简单的**NAND**门。此外，我们还展示了如何任意组合门。总之，这表明**BitVM**可以表示任何电路。

**NAND**门承诺的实现很简单。它包含代表两个输入的两位承诺和代表输出的第三位承诺。脚本计算两个输入的**NAND**值，以确保它与提交的输出位匹配。

![](https://private-user-images.githubusercontent.com/8327633/374852313-c0f7d0e0-9aa7-4402-b53e-0315517ba75a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjg0NTkwOTYsIm5iZiI6MTcyODQ1ODc5NiwicGF0aCI6Ii84MzI3NjMzLzM3NDg1MjMxMy1jMGY3ZDBlMC05YWE3LTQ0MDItYjUzZS0wMzE1NTE3YmE3NWEucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAwOSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMDlUMDcyNjM2WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MTE5NDQxN2RiMWJiNTY4NWQxYzI0YmJmZWJhZDdlZWIxY2EyMDlhNGEyNWZjMWY2YWQ2M2E4OGFkNzBjN2RlYyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.pAT980TvUoMcLC3bl6N2R0AWJN7mHQ9ZNae6kHsX2Bo)

**图2:** **NAND**操作的逻辑门承诺。执行此脚本需要显示位承诺A、B和C的值，以便`A NAND B = C`保持不变。

(这里，为了简单起见，我们假设存在用于操作**NAND**的操作码。实际上是这样的
不存在，但是，可以使用**OP_BOOLAND**和**OP_NOT**轻松实现。)

## 二进制电路承诺

在前一节中，我们定义了**NAND**门的承诺。我们可以通过组合门承诺来表示任何电路。执行的每一步都是在**tappleaf**中承诺的。它们都被组合到同一个**Taproot**地址中，这样证明者就可以执行电路中的任何门。执行门需要证明者打开相应的门承诺并为其输入和输出位设置值。**Taptree**可能会变得庞大，拥有10亿个**Taptree**脚本，但它在链上的足迹很小。

![](https://private-user-images.githubusercontent.com/8327633/374852376-2f1a1cec-ee58-4fe2-89ff-42a60024c861.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjg0NTkxMDcsIm5iZiI6MTcyODQ1ODgwNywicGF0aCI6Ii84MzI3NjMzLzM3NDg1MjM3Ni0yZjFhMWNlYy1lZTU4LTRmZTItODlmZi00MmE2MDAyNGM4NjEucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAwOSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMDlUMDcyNjQ3WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NmI2M2Q0NDA3NDIxZTVkYWE2Y2Y4YmYxOGNiNjk4ZmRlZmYxOTYxNDAwYWU5ZTEzOTlkYjRjMTlhNGE3ZGYzYyZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.L2lzjZXLPPc48sLxKV0TLQypyKm5h6icBEg2JuxEb2s)

**图3:**一个随机示例电路，它有8个不同的**NAND**门和4个输入A,B,C, D。使用数十亿个门，我们基本上可以定义任何函数。

![](https://private-user-images.githubusercontent.com/8327633/374852434-7ce8c5b4-fdee-4799-8b14-92a2896643f4.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjg0NTkxMTksIm5iZiI6MTcyODQ1ODgxOSwicGF0aCI6Ii84MzI3NjMzLzM3NDg1MjQzNC03Y2U4YzViNC1mZGVlLTQ3OTktOGIxNC05MmEyODk2NjQzZjQucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAwOSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMDlUMDcyNjU5WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9NjllMjRkNzQzY2Q4N2NmNTY0MDMyODE2NzA5NWNkZjExODQzNjE0NDRiYmRjZGQ5NTFkNDUzZDk0OWIzMTI5ZCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.yqzpyJfqRGkgLRzbLnzkEck07kspf5HVTk-APhxxUHw)

**图4:**对于每个门，证明者的**Taproot**地址包含一个带有相应门承诺的叶子脚本。这允许证明者在以后的任何时间点设置电路输入的值(这里是A、B、C和D)。

## 挑战与响应

承诺一个电路是不够的。为了反驳一个不正确的提取，验证者必须能够质疑证明者的声明。这可以通过在设置过程中对一系列交易进行预签名来实现。交易是像挑战→响应→挑战→响应→.. ..这样链接的，如果其中一方停止参与，那么在一段时间后，另一方赢得了挑战，可以拿走两笔存款。只要双方合作，他们可以共同解决任何2-of-2签名的合约。

只有在欺诈的情况下才需要以下机制。

![](https://private-user-images.githubusercontent.com/8327633/374852488-da0e58a1-f2f5-4b92-8ece-d40a130cf840.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjg0NTkxMjksIm5iZiI6MTcyODQ1ODgyOSwicGF0aCI6Ii84MzI3NjMzLzM3NDg1MjQ4OC1kYTBlNThhMS1mMmY1LTRiOTItOGVjZS1kNDBhMTMwY2Y4NDAucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAwOSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMDlUMDcyNzA5WiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MTg3YzJlY2Q5YWFmMGYyNDNlYmM0ZGVkNDMzNWM3YjJlMzkwNGJmNTE1ZjEwMjZjMDIwYTQzYWJjYjNlZmNkZSZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.zLHTniAm9fdoyI1-0QSrxks-z8yvt7XPdxxl0Ysj-Bg)

图5:执行多轮挑战-响应的预先签名的交易序列。这个序列是在设置过程中生成的。

Vicky通过打开她的**Tapscript**叶子中的一个**hashlocks**来选择一个挑战。这将为Paul解锁一个特定的**Tapscript**并强制他执行它。这个脚本将迫使Paul揭露被Vicky挑战的门承诺。通过对几轮查询重复此过程，任何不一致的声明都可以迅速被推翻。

如果证明者停止与链下的验证者合作，验证者需要一种方法来强制他上链。验证者通过解锁哈希锁来实现这一点: 由于验证者持有原像，证明者的**UTXO**中的每个**NAND** **Tapleaves**能被花费。因此，验证者可以通过显示其输入和输出来证明给定的**Tapleaf**能被正确执行，从而“解锁”它。应用二进制搜索，验证者可以在几轮挑战和响应后快速识别证明者的错误。

![](https://private-user-images.githubusercontent.com/8327633/374852539-9af79dbd-670e-4d62-943f-979ad7114d24.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjg0NTkxNDAsIm5iZiI6MTcyODQ1ODg0MCwicGF0aCI6Ii84MzI3NjMzLzM3NDg1MjUzOS05YWY3OWRiZC02NzBlLTRkNjItOTQzZi05NzlhZDcxMTRkMjQucG5nP1gtQW16LUFsZ29yaXRobT1BV1M0LUhNQUMtU0hBMjU2JlgtQW16LUNyZWRlbnRpYWw9QUtJQVZDT0RZTFNBNTNQUUs0WkElMkYyMDI0MTAwOSUyRnVzLWVhc3QtMSUyRnMzJTJGYXdzNF9yZXF1ZXN0JlgtQW16LURhdGU9MjAyNDEwMDlUMDcyNzIwWiZYLUFtei1FeHBpcmVzPTMwMCZYLUFtei1TaWduYXR1cmU9MTkyY2MzYmU1Mjg3YzEwZGVlMjE1ZTI5YzQ2MjZlMWYwYmE4MTc1NzE1Yzg2Y2FlZTk3NTRiZjBkMTRkMjBiZiZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QifQ.ec6jvAEBFMroCiBlraSSiTEZPPa_ph3hzWjANzYHJnY)

**图6:** 在每个回应之后，Vicky可以惩罚作恶。如果Paul为一个变量揭示了两个相互冲突的值，那么Vicky立即赢得了挑战，并被允许拿走他的存款。Vicky通过揭示他的任何一点承诺来证明Paul的作恶。

## 输入和输出

证明者可以通过显示相应的比特承诺来设置输入。理想情况下，他们在链下披露承诺，以最大限度地减少链上足迹。在非合作的情况下，验证者可以强迫证明者在链上透露他们的输入。通过预先交换加密数据来处理大量数据是可能的。这样，证明者可以在稍后的时间点显示解密密钥。多方输入也是可能的。门可以从双方得到一点承诺。

## 局限性与展望

在简单的**NAND**电路中表达函数是低效的。通过使用更高级的操作码，程序可以更有效地表达。例如，比特币脚本支持添加32位数字，因此我们不需要二进制电路。我们也可以有更大的位承诺，例如，可以在单个哈希中承诺32位。此外，脚本的大小可以达到大约4 MB。因此，我们可以在每个叶子脚本上实现比单个**NAND**指令更多的指令。

这里提出的模型仅限于两方。然而，有可能有双向通道，并将它们链接起来形成一个类似闪电的网络。探索两方设置可能会产生有趣的概括可能性。例如，我们可以探索网络的1对n星型拓扑。另一个研究问题是，我们是否可以将我们的模型应用于n-n设置，并创建更复杂的渠道工厂。此外，我们可以将我们的系统与不同的链下协议结合起来，例如**闪电网络**或**rollup**。

其他研究方向包括跨应用内存，如何对嵌入到链中的任意数据进行声明，或链下可编程电路，即链下VM。也有可能应用更复杂的采样技术，类似于**STARKs**，在单轮中检查电路。

下一个重要的里程碑是完成具体的 **BitVM** 和 **Tree++** 的设计和实现，Tree++是一种编写和调试比特币合约的高级语言。

## 结论

比特币是图灵完备的，因为在大型**Taptrees**中编码欺诈证明允许验证任何程序的执行。这里概述的模型的一个主要约束是它仅限于两方设置。希望在以后的工作中可以推广。

## 致谢

特别感谢Super Testnet和Sam Parker，他们一直拒绝相信比特币不会是图灵完备的。

## 参考文献

> Ethereum Research. Optimistic rollups. https://ethereum.org/en/developers/ docs/scaling/optimistic-rollups/, 2022.
>
> Salvatore Ingala. Merkleize all the things. https://lists.linuxfoundation.org/ pipermail/bitcoin-dev/2022-November/021182.html, 2022.
>
> Jeremy Rubin. CheckSigFromStack for 5 Byte Values. https://rubin.io/blog/ 2021/07/02/signing-5-bytes, 2021.
