# 我为什么看好Move 语言

## 我是谁？
我的技术称号曾用名：gguoss, 光华，郭大侠，Gavin Guo。 我是一名区块链技术工程师。
- 长期从事过区块链底层框架开发， 从我的github记录可以去追踪， 在EOS没有能跑通时，我给他最先跑通过，之前有个EOSC的仓库。 在没有substrate框架时， 我首先从polkadot框架分离出了ChainX，最早上线的substrate链。
- 5年前有做过很多的技术框架型布道，在当今区块链技术上的预测也依然成立。[github博客](https://gguoss.github.io/)。

![](https://github.com/coming-chat/Move-white-paper/blob/main/gguoss%20%E9%A2%84%E6%B5%8B.jpeg)

那我为什么 5年内没做技术框架博客的更新了呢？ 因为区块链的5年内，虽然Dapp出现了很多杀手级别的应用，如Uniswap/aavax/maker dao等。 但底层框架技术一直没太大的迭代。
我没啥可写的，自己也在怀疑自己之前追寻的以太坊杀手：EOS/Cardano/Cosmos/Polkadot。 但还是被以太坊躺赢了，所以在无尽的自我怀疑中，2年前，我去做了[ComingChat](https://coming.chat/)。 灵感来源于 Web2 的QQ/微信。 微信是我见过web2中最好用的入口。 我抱着想做一个像微信/QQ一样好用的web3的入口的梦想，做了ComingChat。

## 隐约看到了第三代公链框架技术的出现。
直至Sui/Aptos的出现，我是和大家一样的想法， 为什么全球加密圈最顶级资本要花那么多钱投资Move项目，难道是要割韭菜吗？ 怀着学习和证伪的态度，我深刻的去学习Move。这次让我大有收获。
我觉得我5年前放弃的区块链底层框架技术变革的追求，又回来了。 现在的区块链我分为2大类，
- 第一代区块链是 专注于 加密货币 的区块链架构 比特币。
- 第二代区块链是 专注于 金融领域 的智能合约的区块链架构 以太坊。

比特币的局限性是： 非图灵完备的脚步语言难以编程Dapp。 
以太坊的局限性是： 1， 以树型结构组织的账户系统难以并行运行，限制了TPS。2 以EVM/Solidity配套的草根型合约编程，难以大规模协调编程，以及花费大量的时间在安全性上。

从Sui/Move的出现，我看到了Sui 能成为第三代区块链技术架构的潜能。以table数据结构为存储模型支持的并发系统。 以殿堂级的区块链开发语言Move作为基础。 以 GameFi 领域作为未来场景的主要应用。
因为如此，我们Team作出了两个 迎合这个时代的产品对接。
- [MoveChina](https://move-china.com/)。 旨在做最大Move语言中文社区。为什么起名叫MoveChina, 因为China和Chain字母完成相同，意味着做汉语届Move链及其生态的帮助工作。
- Sui 上游戏NFT交易平台和类似微信小程序的链游stdio。 这都依赖于ComingChat现有的SocialFI模块（去中心化数字身份CID代表“人”，omniwallet 代表“钱”，chat+dao朋友圈代表“social”。comingchat上的这三大模块基本已经上线），结合进ComingChat中，做一个完整的类似于web2的QQ体系。

## 为什么看好 Move 语言。
我是在翻译了很多Move相关的技术文档，浏览了一下代码得出的结论。文档基本发布了[move中文社区](https://move-china.com/)。
部分精彩文档如：
- [Move 语言白皮书](https://move-china.com/topic/67)。
- [Move 语言和Solana rust的对比](https://move-china.com/topic/108)。
- [Move 源代码](https://github.com/move-language/move)。

得出的结论是：Move语言是未来区块链智能合约编程大趋势。
理由如下：
- Move 语言是Facebook的Libra团队，最顶级的专业人士投入了大量的钱和人力认真分析了当下智能合约编程的优缺点。有完整的规划和目的。如Move 语言有完整的详细的白皮书。
- 面向资产编程，在Rust语言的启发（语言层面来保证内存管理的安全，让编译器和规则来约束程序员，从而避免糟糕的程序工程师来管理内存）下。 Move语言同样从 语言层面来管理资产安全，避免了糟糕的智能合约开发人员来 书写合约代码。 以Solidity为例，像Solidity 之父gavin wood 都曾两次用solidity语言写的智能多签合约出了严重的安全事故。
- 没有动态调用。 类似于RUST避免了 C++ 的运行时 内存分配和管理。 把代码从静态层面，运行前确定合约的安全。
- 形式化证明。 spec代码和move程序代码在一起，可以比以前写的测试用例 强一个数量级的安全验证保证。
- 大规模协同编程。 move语言的模块化结构清晰，可以适用于大规模人员的协同开发。
- Module和Script 结合。 一个用于做专门的累积木代码和数据库。另外一个专门用于做Dapp的执行程序。 让Move链可以随着开发代码和数据的增多而增加整个区块链系统的丰富度。类似于人工智能训练系统，输入的数据越多，系统的价值越大。



