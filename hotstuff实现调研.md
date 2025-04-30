### hotstuff原作者版本
1. github: https://github.com/hot-stuff/libhotstuff c++实现
2. 论文：https://arxiv.org/pdf/1803.05069v5 作者：Maofan Yin
3. 问题：C++版本后面不好改造
4. 好处：hotstuff原作者
### diem项目方，原Libra团队，facebook
1. github: https://github.com/asonnino/hotstuff rust实现
2. 论文：https://arxiv.org/pdf/2106.10362 作者：Jolteon and Ditto
3. 问题：diem项目方已经停止运营，代码停留在20年，不是hoststuff原版，是基于hotstuff的改进版
4. 好处：相对简单便于理解
### aptos项目方
1. https://github.com/aptos-labs/aptos-core/tree/main/consensus rust实现
2. 论文：Jolteon作者加入Aptos基于diem论文的改进，https://medium.a41.io/aptos-series-2-aptosbft-consensus-algorithm-for-scalability-and-security-9c6adffc76b3
3. 问题：和Apots项目本身有一些特性的耦合
4. 好处：实际线上运行，可行性进过论证
