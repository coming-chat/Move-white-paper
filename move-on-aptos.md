
# Aptos 平台上的Move

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

## Move中的关键概念

- 数据应存储在拥有它的帐户中，而不是发布模块的帐户中
- 数据流应该有最小的限制，强调生态系统的可用性
- 通过泛型更喜欢静态类型安全而不是运行时安全
- 除非明确确定，否则应限制签名者限制对帐户添加或删除资产的访问权限

## 数据所有权

数据流应该有最小的限制，强调生态系统的可用性。

可以将资产编程为完全限制在模块内，方法是使任何接口都不会以值形式呈现结构，而是只提供用于操作模块内定义的数据的函数。 这将数据的可用性限制为仅在一个模块内并使其不可导出，这反过来又阻止了与其他模块的互操作性。 具体来说，可以想象一个购买合同，它以一些 Coin<T> 作为输入并返回一个 Ticket。 如果 Coin<T> 仅在模块内定义并且不可导出到外部，则该 Coin<T> 的应用程序仅限于模块定义的任何内容。
  
对比以下两种使用充提币实现转账的功能：
``` rust
public fun transfer<T>(sender: &signer, recipient: address, amount: u64) {
    let coin = withdraw(&signer, amount);
    deposit(recipient, coin);
}
  ```
Coin 可以在模块外限制只能使用的以下接口：
``` rust
fun withdraw<T>(account: &signer, amount: u64): Coin<T>
fun deposit<T>(account: address, coin: Coin<T>)
```

通过添加公共存取接口进行取款和充值，币可以被带出模块，被其他模块使用，并从调用模块中返回：
``` rust
public fun withdraw<T>(account: &signer, amount: u64): Coin<T>
public fun deposit<T>(account: address, coin: Coin<T>)
```
  
## 类型安全

在 Move 中，给定一个特定的结构，比如 A，可以通过两种方式区分不同的实例：
- 内部标识符，例如 [GUID](https://github.com/aptos-labs/aptos-core/blob/main/crates/aptos-rest-client/src/types.rs#L74)s
- 泛型，例如 A<T>，其中 T 是另一个结构
  
内部标识符可以很方便，因为它们简单且易于编程。 然而，泛型提供了更高的保证，包括显式编译或验证时间检查，尽管有一些成本。
  
泛型允许完全不同的类型和资源以及期望这些类型的接口。 例如，订单簿可以声明他们希望所有订单使用两种货币，但其中一种货币必须是固定的，例如，buy<T>(coin: Coin<Aptos>): Coin<T>。 这明确指出用户可以购买任何硬币 <T>，但必须使用 Coin<Aptos> 支付。
  
当需要在 T 上存储数据时，泛型的复杂性就会出现。Move 不支持对泛型进行静态调度，因此在像 create<T>(...) : Coin<T> 这样的函数中，T 必须是 幻像类型，即仅用作 Coin 中的类型参数，或者必须将其指定为 Create 的输入。 即使每个 T 都实现了所述函数，也不能在 T 上调用任何函数，例如 T::function。

除了可能大量创建的结构之外，泛型会导致创建许多与跟踪数据和触发事件相关的新存储和资源，这可以说是一个开发人员不太关心的问题。

正因为如此，我们做出了创建两个“代币”标准的艰难选择，一个用于与货币相关的代币，称为 Coin，另一个用于与资产或 NFT 相关的代币，称为 Token。 Coin 通过泛型利用静态类型安全，但合约要简单得多。 而 Token 通过其自己的通用标识符利用动态类型安全性，并且由于影响其使用的人体工程学的复杂性而避开了泛型。
  
## 数据访问

- 除非明确确定，否则应要求签名者限制对帐户添加或删除资产的访问权限。

在 Move 中，模块可以定义如何访问资源以及修改其内容，而不管帐户所有者的签名者是否存在。 这意味着程序员可能会意外创建允许其他用户从其他用户的帐户中任意插入或删除资产的资源。
  
在我们的开发中，我们有几个例子来说明我们允许访问和阻止访问的地方：
- 一个 Token 不能直接插入另一个用户的账户，除非他们已经有一些该 Token
- TokenTransfers 允许用户显式声明存储在另一个用户资源中的令牌，有效地使用访问控制列表来获得该访问权限
- 在 Coin 中，用户可以直接转移到另一个用户的帐户，只要他们有存储该硬币的资源

对我们的代币不那么严格,可能允许用户将代币直接空投到另一个用户账户中，这将为他们的账户增加额外的存储空间，并使他们成为他们没有首先批准的内容的所有者。
  
作为一个具体的例子，使用withdraw函数回到前面的Coin案例。 如果取而代之的是这样定义的withdraw函数：
  ``` rust
  public fun withdraw<T>(account: address, amount: u64): Coin<T>
  ```
任何人都可以从账户中取出硬币
  
