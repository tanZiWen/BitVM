## DAG算法总结
1. 基于可靠广播的3种算法：DAG-Rider，Tusk，Bullshark;不是基于可靠广播的2种算法：Mysticeti, Mahi
2. 按照可靠广播的定义，基于可靠广播的算法一个节点在一个round只会产生一个区块
3. 基于可靠广播的算法由于节点在同一个round不会产生两个区块，所以理论上只需要f+1的引用数就可以终局。DAG-Rider是一个特例，算法上要求2f+1的引用但实际f+1个引用应该就可以了
4. 不是基于可靠广播的算法，Mysticeti和Mahi一个节点在同一个round可能会产生两个不同的区块，所以需要解决分叉的问题，通过r轮，r+1轮有2f+1轮引用r轮的区块，r+2轮引用r+1轮的2f+1个区块都满足2f+1的引用。
5. Hahi相比较与Mysticeti来说的改进：1. Mysticeti基于半同步的网路假设，Mahi基于异步的网络假设 2. Hahi是一个wave 5轮，而Mysticeti一个wave是3轮 3. Hahi有wave叠加的的设计，而Mysticeti没有 4. Mysticeti是一个wave的round 1里面只要满足两轮2f+1就都能作为Leader节点终局前面的区块，而Mahi是产生一个随机数，再按照两轮2f+1的算法选出Leader
6. 同步网路是指消息在规定的时间内可以达到，异步网路没有时间的限制，异步网路都是通过随机数选择一个Leader
7. Narwhal&Tusk优化了DAG-Rider的可靠广播，DAG-Rider广播的是完整区块，Narwhal&Tusk是广播区块头信息，交易可以异步的拉去或广播
8. Bullshark分为同步版本和异步版本，同步版本基于同步网路假设，两个round一轮，异步版本基于异步网路和DAG-rider一样
9. Mysticeti 基于半同步网路，
10. 延迟： DAG-RDIER 10+, Tusk 3-4s, BullShark 2-3s, hotstuff 1-2s Mysticeti 0.4s 
11. 异步网路假设：DAG-RIDER, TUSK, BullShark, Mahi 半同步：BullShark，Mysticeti
12. 代码实现：TUSK, BullShark, Mysiceti 主网
13. TPS： Narwhal-HotStuff：30000-70000 tx/sec，延迟约 1-2 秒，Tusk：100,000-160,000 tx/sec，延迟约 3-4 秒，Bullshark约 100,000-130,000 tx/sec 的吞吐量，延迟为 2-3 秒，Shoal 0.5–1秒 100,000–160,000 TPS，Mysticeti 300,000-400000 tx/sec 延迟0.4s，Mysticeti-FPC模式 延迟降低至 0.25s
14. hoststuff 在 4–10 个节点的测试环境中，HotStuff 的吞吐量可接近 20,000 TPS（简单交易，带宽充足）, 1s




