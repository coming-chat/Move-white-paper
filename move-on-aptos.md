
# Aptos 平台上的Move？

Aptos 区块链由运行共识协议的验证节点组成。 当在Move虚拟机 (MoveVM) 上执行过程中，共识协议就交易的顺序及其输出达成一致。 每个验证节点将交易与当前区块链账本状态一起转换为 VM 的输入。 MoveVM 处理此输入以生成变更集合或存储增量作为输出。 一旦达成共识并承诺输出，它就会公开可见。 在本指南中，我们将向您介绍核心 Move 概念以及它们如何应用于 Aptos 上的开发。

## 什么是Move？

Move 是一种安全可靠的 Web3 编程语言，强调稀缺性和访问控制。 Move 中的资产可以由资源类型表示和存储。 默认情况下会强制执行稀缺性，因为不能复制结构。 只有在字节码层明确定义为副本的结构才能被复制。 ***稀缺性*** 是指 Move的结构类型 不能被复制，表现得很稀缺。 只有在字节码层明确定义为副本的结构才能被复制。

***访问控制***是约束不同帐户对不同模块的访问权限。Move 中的模块是一段表达创建、存储或传输资产的程序或者库。Move 确保只有公共模块功能可以被其他模块访问。除非结构具有公共构造函数，否则它只能在定义它的模块内构造。同样，结构中的字段只能在其模块内或通过公共访问接口和设置接口访问和更改。

在 Move 中，交易的发送者由签名者代表，签名者是特定帐户的经过验证的所有者。 签名者在 Move 中拥有最高级别的权限，并且是唯一能够将资源添加到帐户中的实体。 此外，模块开发人员可以要求签名者通过了了验签后才能访问资源或修改帐户中存储的资产。

## 与其他虚拟机的比较
|     | Aptos/Move | Solana / SeaLevel | EVM |
| :---: | :---:| :---: | :---: |
|数据存储|存储在所有者的帐户中|存储在与程序关联的所有者帐户中|存储在与智能合约关联的账户中|
|并行化|运行时推断并行化|需要在交易中指定所有访问的帐户和程序| 没有并行|
|交易安全|序列号|交易唯一性+记忆交易|nonces，类似于序列号|
|类型安全|模块结构和泛型|程序结构|合约类型|
|函数调用|不在泛型上的静态调度|静态调度|动态调度|

## Aptos 的Move 特性

MoveVM 的每个部署都能够通过适配器层使用附加功能扩展核心 MoveVM。 此外，MoveVM 具有支持标准操作的框架，就像计算机具有操作系统一样。

Aptos Move 适配器功能包括：
- 细粒度存储:解耦存储在账户中的数据量，这些数据量会影响与该账户相关的交易的 Gas 费用
- 在帐户中大规模存储键、值对的[表](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/table.move)
- 通过 [Block-STM](https://medium.com/aptoslabs/block-stm-how-we-execute-over-160k-transactions-per-second-on-the-aptos-blockchain-3b003657e4ba) 实现并行化，无需用户输入即可并发执行交易

Aptos 框架附带了许多有用的库：
- 一种[代币标准](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-token/sources/token.move)，可以在不发布智能合约的情况下创建 NFT 和其他丰富的代币。
- 一种[硬币标准](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/coin.move)，可以通过发布一个简单的模块来创建类型安全的硬币
- 允许遍历表中所有条目的[可迭代表](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/iterable_table.move)
- 质押和代理框架
- 用于在运行时识别给定类型的地址、模块和结构名称的 [type_of](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-stdlib/sources/type_info.move) 服务
- 允许多个签名者实体的多签名者框架
- 一个[时间戳服务](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/timestamp.move)，它提供一个单调递增的时钟，映射到当前的实际 unixtime

更多即将推出。

