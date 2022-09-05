# Move 语言中文白皮书
## 翻译自： [Move language whiter paper](https://github.com/coming-chat/Move-white-paper/blob/main/Move%202020-05-26.pdf)

## Move: 一种使用可编程**resources**的语言

Sam Blackshear, Evan Cheng, David L. Dill, Victor Gao, Ben Maurer, Todd Nowacki, Alistair Pott, Shaz Qadeer, Rain, Dario Russi, Stephane Sezer, Tim Zakian, Runtian Zhou * 合著

译者：GG@ComingChat

读者须知：本报告在协会发布白皮书 v2.0 之前发布，其中包括对 Libra 支付系统的多项关键更新。已删除过时的链接，但除此之外，本报告尚未修改以包含更新，应在该上下文中阅读。


## 摘要
我们为 Libra 区块链[1][2] 提供了一种安全且灵活的编程语言，它的名字叫做Move。Move也是一种可执行的字节码语言，用于实现自定义交易和智能合约。

Move 的关键特性是能够使用受线性逻辑启发的语义定义自定义**resources**类型 [3]：***resources***永远不能被复制或隐式丢弃，只能在程序存储位置之间移动。这些安全保证由 Move 的类型系统静态强制执行。虽然有这些特殊保护，但不会局限***resources***这一类型的功能，它可以像传统的类型一样，可以存储在数据结构中，也可以作为参数传递。

***First-class resources***是一个非常笼统的概念，程序员不仅可以使用它来实现安全的数字资产，还可以编写正确的业务逻辑来包装资产以及对数值不同权限的访问和操作。
Move 的安全性和表现力使我们能够在 Move 中实现 Libra 协议的重要部分，包括 Libra 币、交易处理和验证者管理。

## 1.介绍

互联网和移动宽带的出现连接了全球数十亿人，提供了获取知识、免费通信和广泛的低成本、更便捷的服务。
这种连通性也使更多人能够访问金融生态系统。然而，尽管取得了这些进展，但对于最需要金融服务的人来说，获得金融服务的机会仍然有限。

Libra的使命是改变这种状况 [1]。 在本文中，我们介绍了 Move，这是一种新的编程语言，用于在 Libra 协议中实现自定义交易逻辑和智能合约 [2]。为了介绍move，我们分以下例举的几个章节来说明：

1. 描述在区块链上表示数字资产的挑战（第 2 节）。
2. 解释我们的 Move 设计如何应对这些挑战（第 3 节）。
3. 对 Move 的关键特性和编程模型（第 4 节）给出一个面向示例的概述。
4. 深入研究语言和虚拟机设计的技术细节（第 5 节、第 6 节和附录 A）。
5. 最后总结我们在 Move 上取得的进展，描述我们的语言发展计划，并概述我们在 Libra 区块链上支持第三方 Move 代码的路线图（第 7 节）。

本文适用于两种不同的读者：
- 可能不熟悉区块链系统的编程语言研究人员。 我们鼓励这些读者从头到尾阅读这篇论文，但我们警告说，我们有时可能会在没有为不熟悉的读者提供足够背景的情况下提及区块链概念。 在深入研究本文之前阅读 [2] 会有所帮助，但这不是必需的。
- 可能不熟悉编程语言研究但有兴趣了解 Move 语言的区块链开发人员。 我们鼓励这些读者从第 3 节开始。我们警告第 5 节、第 6 节和附录 A 包含一些可能不熟悉的编程语言术语和形式化。

## 2. 在区块链上管理数字资产
我们将首先在抽象层面简要解释区块链，以帮助读者理解像 Move 这样的“区块链编程语言”所扮演的角色。 此讨论有意省略了区块链系统的许多重要细节，以便专注于从语言角度来看相关的功能。

### 2.1. 区块链 概要
区块链是一个复制的状态机 [4][5]。 系统中的复制器称为验证节点。 系统用户将交易发送给验证者。 每个验证节点都了解如何执行交易以将其内部状态机从当前状态转换为新状态。
验证节点利用他们对交易执行的共同理解来遵循共识协议来共同定义和维护复制状态。
- 验证节点从相同的初始状态开始。
- 验证节点选择执行相同的交易顺序。
- 执行交易产生确定性的状态转换。

验证阶节点执行以上步骤后，会就下一个状态达成一致。 重复应用此方案允许验证节点在继续就当前状态达成一致的同时处理交易。

请注意，共识协议和状态转换组件对彼此的实现细节不敏感。 只要共识协议确保交易之间的总顺序并且状态转换方案是确定性的，组件就可以和谐地交互。

### 2.2. 在一个开放的系统中进行数字资产编码

像 Move 这样的区块链编程语言的作用是决定如何表示转换和状态。为了支持丰富的金融基础设施，Libra 区块链的状态必须能够在给定的时间点对数字资产的所有者进行编码。此外，状态转换应该允许资产转移。
区块链编程语言的设计还必须考虑另一个考虑因素。与其他公有链一样，Libra 区块链是一个开放系统。 任何人都可以查看当前的区块链状态或向验证节点提交交易（即提议状态转换）。传统web2行业，用于管理数字资产的软件（例如银行软件）在具有特殊管理控制的封闭系统中运行。 在公有链中，所有参与者都是平等的。 参与者可以提出她喜欢的任何状态转换，但系统并不应该允许所有状态转换。 例如，Alice 可以自由提议转移 Bob 拥有的资产的状态转换。 状态转换函数必须能够识别这个状态转换是无效的并且拒绝它。

在开放软件系统中选择编码数字资产所有权的转换和状态表示具有挑战性。 特别是，实物资产有两个属性难以在数字资产中编码：
- 稀缺性：控制系统中的资产供应， 禁止复制现有资产，创建新资产为特权操作。
- 访问控制：系统中的参与者应该能够通过访问控制策略保护她的资产。

为了直观表现，我们将在一系列关于状态转换表示的 strawman 提案 实例中看到这些问题是如何出现的。我们将假设一个区块链跟踪称为 StrawCoin 的单一数字资产。区块链状态 G 被构造为一个键值存储，它将用户身份（用加密公钥表示）映射到对每个用户持有的 StrawCoin 进行编码的自然数值。提案由一个交易脚本组成，该脚本将使用给定的评估规则进行评估，生成一个更新以应用于全局状态。我们将写 G[𝐾] := 𝑛 来表示用值 𝑛 更新存储在全局区块链状态中键 𝐾 处的自然数。

每个提案的目标是设计一个系统，该系统具有足够的表达能力，允许 Alice 将 StrawCoin 发送给 Bob，但又受到足够的约束，以防止任何用户违反稀缺性或访问控制属性。 这些提案并没有试图解决重放攻击等安全问题 [6]，这些问题很重要，但与我们关于稀缺性和访问控制的讨论无关。

Scarcity. 最简单的提案是直接在交易脚本中对状态的更新进行编码：

   |交易脚本格式      |         评估规则 |
   |:-------:       | :------------:  |
   |<K,n>           |      G[K] := n  |
   
这种表示可以编码从 Alice 向 Bob 发送 StrawCoin。 但它有几个严重的问题。 一方面，这个提议并没有强制StrawCoin的Scarcity。 通过发送交易⟨Alice, 100⟩，Alice 可以“凭空”给自己尽可能多的 StrawCoin。 因此，Alice 发送给Bob的StrawCoin实际上毫无价值，因为Bob可以很容易地为自己创造这些代币。

稀缺性是有价值的实物资产的重要属性。 像黄金这样的稀有金属自然是稀缺的，但数字资产并不存在固有的实物稀缺性。 编码为某些字节序列（例如 G[Alice] → 10）的数字资产在物理上并不比另一个字节序列（例如 G[Alice] → 100）更难生成或复制。相反，评估规则必须以编程方式强制执行稀缺性。

让我们考虑考虑稀缺性的第二个提案：
 |交易脚本格式       |     评估规则 |
 | :---:           |    :-----:  | 
 |<Ka, n, Kb>      |  if G[Ka] >= n then G[Ka] := G[Ka] - n; G[Kb] := G[Kb] + n; |
 
在此方案下，交易脚本指定发送者 Alice 的公钥 𝐾𝑎 和接收者 Bob 的公钥 𝐾𝑏。 评估规则现在在执行任何更新之前检查存储在𝐾𝑎下的StrawCoin数量是否至少为𝑛。 如果检查成功，则评估规则从存储在发送者密钥下的StrawCoin中减去𝑛，并将𝑛 添加到存储在接收者密钥下的StrawCoin中。 在这种方案下，执行有效的交易脚本通过保存系统中的StrawCoin数量来强制稀缺性。 Alice 不能再凭空创造 StrawCoin——她只能从她的账户中扣除给予 Bob StrawCoin。

访问控制. 尽管第二个提案解决了稀缺性问题，但它仍然存在一个问题：Bob 可以发送花费属于 Alice 的 StrawCoin 的交易。 例如，评估规则中的任何内容都不会阻止 Bob 发送交易⟨Alice, 100, Bob⟩。 我们可以通过添加基于数字签名的访问控制机制来解决这个问题：

  | 交易所脚本格式| 评估规则 |
  |  :-----:    | :-----:  |
  | SKa(<Ka, n, Kb>) | if verify_sig(SKa(<Ka, n, Kb>)) && G[Ka] >= n then G[Ka] := G[Ka] - n ; G[Kb] := G[Kb] + n|
  
该方案要求 Alice 用她的私钥签署交易脚本。 我们写 𝑆𝐾(𝑚) 用于使用与公钥 𝐾 配对的私钥对消息 𝑚 进行签名。 评估规则使用 verify_sig 函数来检查与 Alice 的公钥 𝐾𝑎 的签名。 如果签名未验证，则不执行更新。 这条新规则通过使用数字签名的不可伪造性来防止 Alice 从她自己的账户以外的任何账户中借记 StrawCoin，从而解决了之前提案的问题。

顺便说一句，请注意在第一个strawman 提案中实际上不需要评估规则——提案的状态更新直接应用于键值存储。 但随着我们对提案的推进，执行更新的先决条件与更新本身之间出现了明显的分离。 评估规则通过评估脚本来决定是否执行更新以及执行什么更新。 这种分离是基本的，因为执行访问控制和稀缺策略不可避免地需要某种形式的评估——用户提出状态更改，并且必须执行计算以确定状态更改是否符合策略。 在一个开放的系统中，参与者不能被信任来执行链下策略并直接向状态提交更新（如在第一个提案中）。 相反，访问控制策略必须由评估规则在链上执行。

### 2.3. 已存在的区块链语言

StrawCoin 是一种玩具语言，但它试图捕捉比特币脚本 [7][8] 和以太坊虚拟机字节码 [9] 语言（尤其是后者）的精髓。 尽管这些语言比 StrawCoin 更复杂，但它们面临着许多相同的问题：

1. ***资产的间接代表***。 资产使用整数编码，但整数值与资产不同。 事实上，没有代表比特币/以太币/稻草币的类型或值！ 这使得编写使用资产的程序变得笨拙且容易出错。 诸如将资产传入/传出程序或将资产存储在数据结构中等模式需要特殊的语言支持。
2. ***稀缺性不可扩展***。 语言只代表一种稀缺资产。 此外，稀缺性保护直接硬编码在语言语义中。 希望创建自定义资产的程序员必须在没有语言支持的情况下小心地重新实现稀缺性。
3. ***访问控制不灵活***。 该模型实施的唯一访问控制策略是基于公钥的签名方案。 与稀缺性保护一样，访问控制策略深深嵌入语言语义中。 如何扩展语言以允许程序员定义自定义访问控制策略并不明显。

***比特币脚本。*** Bitcoin Script 有一个简单而优雅的设计，专注于表达用于花费比特币的自定义访问控制策略。 全局状态由一组未使用的交易输出 (UTXO) 组成。 比特币脚本程序提供满足其使用的旧 UTXO 的访问控制策略的输入（例如，数字签名），并为它创建的新 UTXO 指定自定义访问控制策略。 由于比特币脚本包含用于数字签名检查的强大指令（包括多重签名 [10] 支持），因此程序员可以对各种访问控制策略进行编码。

然而，Bitcoin Script 的表达能力从根本上是有限的。 程序员不能定义自定义数据类型（因此，自定义资产）或程序，并且语言不是图灵完备的。 合作方可以通过复杂的多交易协议 [11] 执行一些更丰富的计算，或者通过“彩色硬币”[12][13] 非正式地定义自定义资产。 然而，这些方案通过将复杂性推到语言之外来工作，因此不能实现真正的可扩展性。

***以太坊虚拟机字节码*** 以太坊是一个开创性的系统，它展示了如何将区块链系统用于支付以外的用途。 以太坊虚拟机 (EVM) 字节码程序员可以发布智能合约 [14]，与以太币等资产交互并使用图灵完备语言定义新资产。 EVM 支持比特币脚本不支持的许多特性，例如用户定义的程序、虚拟调用、循环和数据结构。

然而，EVM 的表现力为代价高昂的编程错误打开了大门。 与 StrawCoin 一样，以太币在语言中具有特殊地位，并且以强制稀缺性的方式实施。 但是自定义资产的实现者（例如，通过 ERC20 [15] 标准）不会继承这些保护（如 (2) 中所述）——他们必须小心不要引入允许复制、重用或资产丢失的错误。 由于 (1) 中描述的间接表示问题和 EVM 的高度动态行为相结合，这是具有挑战性的。 特别是，将 以太转移到智能合约涉及动态调度，这导致了一类新的错误，称为重入漏洞 [16]。 备受瞩目的漏洞利用，例如 DAO 攻击 [17] 和 Parity Wallet 黑客攻击 [18]，使攻击者能够窃取价值数百万美元甚至更多的加密货币。

## 3. Move 的设计目标

Libra 的使命是打造一个简单的全球货币和金融基础设施，为数十亿人提供支持 [1]。Move 语言旨在提供一个安全、可编程的基础，可以在此基础上构建这一愿景。Move 必须能够以精确、可理解和可验证的方式表达 Libra 货币和治理规则。 从长远来看，Move 必须能够对构成金融基础设施的各种资产和相应的业务逻辑进行编码。

为了满足这些要求，我们在设计 Move 时考虑了四个关键目标：first-class 资产、灵活性、安全性和可验证性。

### 3.1. First-Class Resources

区块链系统允许用户编写直接与数字资产交互的程序。 正如我们在 2.2 节中所讨论的，数字资产具有将它们与传统编程中使用的值（如布尔值、整数和字符串）区分开来的特殊特征。 使用资产进行编程的强大而优雅的方法需要保留这些特征的表示。

Move 的关键特性是能够使用受线性逻辑启发的语义定义自定义Resources类型 [3]：Resources永远不能被复制或隐式丢弃，只能在程序存储位置之间移动。 这些安全保证由 Move 的类型系统静态强制执行。 尽管有这些特殊保护，Resouces也可以作为普通的程序值——它们可以存储在数据结构中，作为参数传递给 程序（函数），等等。 First-Class Resources是一个非常笼统的概念，程序员不仅可以使用它来实现安全的数字资产，还可以编写正确的业务逻辑来包装资产和执行访问控制策略。
  
Libra 币本身是作为普通的 Move Resouces实现的，在语言中没有特殊状态。由于 Libra 代币代表由 Libra 储备管理的现实世界资产 [19]，Move 必须允许创建Resources（例如，当新的现实世界资产进入 Libra 储备时）、修改（例如，当数字资产更改所有权时） ) 和销毁（例如，当支持数字资产的实物资产被出售时）。Move程序员可以使用模块保护对这些关键操作的访问。 Move 模块类似于其他区块链语言中的智能合约。模块声明resource类型和程序，这些程序对创建、销毁和更新其声明的resource的规则进行编码。模块可以调用其他模块声明的程序并使用其他模块声明的类型。然而，模块强制执行强大的数据抽象——一个类型在其声明模块内部是透明的，而在其外部是不透明的。此外，对resource类型 T 的关键操作只能在定义 T 的模块内执行。

### 3.2. 灵活性

Move 通过交易脚本为 Libra 增加了灵活性。 每个 Libra 交易都包含一个交易脚本，该脚本实际上是交易的主要程序。 交易脚本是包含任意Move代码的单个程序，允许自定义交易。 一个脚本可以调用区块链中发布的模块的多个程序，并对结果进行本地计算。 这意味着脚本可以执行表达性的一次性行为（例如支付一组特定的收件人）或可重用行为（通过调用封装可重用逻辑的单个程序）。

Move模块通过安全但灵活的代码组合实现了不同类型的灵活性。 在高层次上，Move 中的模块/Resources/程序之间的关系类似于面向对象编程中的类/对象/方法之间的关系。 但是，有一些重要的区别——Move 模块可以声明多种Resource类型（或零Resource类型），而 Move 程序没有 self 或 this 值的概念。 Move模块最类似于 ML样式[20]模块的受限版本。

### 3.3. 安全

Move 必须拒绝不满足关键属性的程序，例如Resource安全、类型安全和内存安全。 我们如何选择一个可执行的表示，以确保在区块链上执行的每个程序都满足这些属性？ 两种可能的方法是：
(a) 使用带有检查这些属性的编译器的高级编程语言
(b) 使用低级无类型汇编并在运行时执行这些安全检查。

Move 采取了介于这两个极端之间的方法。 Move 的可执行格式是一种类型化的字节码，它比汇编高级，但比源语言低。 字节码由字节码验证器在链上检查Resource、类型和内存安全性，然后由字节码解释器直接执行。 这种选择允许 Move 提供通常与源语言相关的安全保证，但无需将源编译器添加到受信任的计算库或将编译成本添加到交易执行的关键路径中。

### 3.4. 可验证性

理想情况下，我们将通过链上字节码分析或运行时检查来检查 Move 程序的每个安全属性。不幸的是，这是不可行的。 我们必须仔细权衡安全保证的重要性和普遍性与计算成本和通过链上验证执行保证的增加的协议复杂性。

我们的方法是尽可能多地对关键安全属性执行轻量级的链上验证，但设计 Move 语言以支持先进的链下静态验证工具。 我们做出了一些设计决策，使 Move 比大多数通用语言更适合静态验证：
1. ***没有动态调用***。 每个程序的调用点都可以静态确定。 这使得验证工具可以轻松准确地推断程序调用的效果，而无需执行复杂的调用图构造分析。 
2. ***有限的可变性***。 Move每个值的变化都是通过引用发生的。 引用是必须在单个交易脚本范围内创建和销毁的临时值。 Move 的字节码验证器使用类似于 Rust 的“借用检查”方案来确保在任何时间点最多存在一个对值的可变引用。 此外，该语言确保全局存储始终是树而不是任意图。 这允许验证工具模块化推理写操作的影响。
3. 模块化。 Move模块强制执行数据抽象并本地化对resouces的关键操作。 模块启用的封装与 Move 类型系统强制执行的保护相结合，确保为模块类型建立的属性不会被模块外部的代码违反。我们希望这种设计能够通过孤立地查看模块而不考虑外部调用者来对重要的模块不变量进行详尽的功能验证。

静态验证工具可以利用 Move 的这些属性来准确有效地检查是否存在运行时故障（例如，整数溢出）和重要的程序特定功能正确性属性（例如，最终可以由参与者来声明锁定在支付渠道中的resources）。 我们将在第 7 节中分享有关我们功能验证计划的更多细节。

## 4. Move概览

我们通过简单的点对点支付所涉及的交易脚本和模块来介绍 Move 的基础知识。该模块是真实的Libra 代币实现的简化版本。 示例交易脚本演示了模块外的恶意或粗心程序员不能违反模块resources的关键安全不变量。 示例模块展示了如何实现利用强数据抽象来建立和维护这些不变量的资源。

本节中的代码片段是用 Move 中间代码（IR) 的变体编写的。Move IR 足够高级，可以编写人类可读的代码，但又足够低级，可以直接转换为 Move 字节码。我们在 IR 中展示代码是因为基于堆栈的 Move 字节码更难阅读，我们目前正在设计 Move 源语言（参见第 7 节）。我们注意到，在执行代码之前，Move 类型系统提供的所有安全保证都在字节码级别进行了检查。

### 4.1. 点对点支付脚本

```rust
public main(payee: address, amount: u64) {
  let coin: 0x0.Currency.Coin = 0x0.Currency.withdraw_from_sender(copy(amount));
  0x0.Currency.deposit(copy(payee), move(coin));
}
```

该脚本有两个输入：付款接收者的帐户地址和一个无符号整数，表示要转移给接收者的代币数量。 执行此脚本的效果很简单：代币金额将从交易发送方转移到收款方。 这发生在两个步骤中。
第一步，发送者从存储在 0x0.Currency 的模块中调用一个名为withdraw_from_sender 的程序。 正如我们将在 4.2 节中解释的那样，0x0 是存储模块的帐户地址，Currency是模块的名称。 该程序返回的价值硬币是类型为0x0.Currency.Coin的资源值。 
第二步，发送方通过resours类型的代币的值转移到 0x0.Currency 模块的存款程序中，将资金转移给收款人。

这个例子很有趣，因为它非常巧妙。Move 的类型系统将拒绝可能导致不良行为的相同代码的小变体。特别是，类型系统确保resources永远不会被复制、重用或丢失。例如，脚本的以下三个更改将被类型系统拒绝：
- ***通过将 move(coin) 更改为 copy(coin) 来复制代币***。请注意，示例中变量的每次使用都包含在 copy() 或 move() 中。 Move 遵循 Rust 和 C++，实现了 move 语义。 每次读取Move变量 x 都必须指定用法是将 x 的值移出变量（使 x 不可用）还是复制该值（使 x 可继续使用）。 u64 和地址等不受限制的值都可以复制和移动。 但是resources值只能移动。 尝试复制resources值（例如，在上面的示例中使用 copy(coin)）将导致字节码验证时出错。
- ***通过两次写入 move(coin) 来重用代币***。 在上面的示例中添加行 0x0.Currency.deposit(copy(some_other_payee), move(coin)) 将使发送者“花费”两次代币——第一次使用 payee，第二次使用 some_other_payee。 这种不良行为对于实物资产是不可能的。 幸运的是，Move 会拒绝这个程序。 变量coin在第一次移动后变得不可用，第二次移动将触发字节码验证错误。
- ***因忽视移动（代币）而损失代币金额***。 Move 语言实现了必须移动一次的线性 [3][23] resources。 未能移动resources（例如，通过删除上面示例中包含 move(coin) 的行）将触发字节码验证错误。 这可以保护 Move 程序员免于意外或故意丢失resources的跟踪。 这些保证超出了纸币等实物资产的可能范围。

我们使用术语resources安全来描述 Move resources永远不会被复制、重用或丢失的保证。 这些保证非常强大，因为 Move 程序员可以实现也享有这些保护的自定义resources。 正如我们在 3.1 节中提到的，即使是 Libra 代币也是作为自定义resources实现的，在 Move 语言中没有特殊状态。

## 4.2. Currency 模块

在本节中，我们将展示上述示例中使用的 Currency 模块的实现如何利用resources安全性来实现安全的可替代资产。 我们将首先解释一下运行 Move 代码的区块链环境。

***入门：Move执行模型***。 正如我们在第 3.2 节中解释的那样，Move 有两种不同的程序：
- 交易脚本，如第 4.1 节中概述的示例
- 模块，如我们将很快介绍的Currency模块。

像示例这样的交易脚本包含在每个用户提交的交易中，并调用模块的程序来更新全局状态。 执行交易脚本是个原子性的执行逻辑，只有全部代码执行成功才会交易脚本执行完成，否则交易脚本内的代码全部不执行。并且脚本执行的所有写入都提交到全局存储，要么执行因错误而终止（例如，由于断言失败或超出 gas错误），并且没有任何事情发生。 交易脚本是一段一次性的代码——在它执行之后，它不能被其他交易脚本或模块再次调用。

相比之下，模块是在全局状态下发布的一段长期存在的代码。 上面示例中使用的模块名称 0x0.Currency 包含发布模块代码的帐户地址 0x0。 全局状态被构造为从帐户地址到帐户的映射。

每个帐户可以包含零个或多个模块和一个或多个resources值。 例如，地址为 0x0 的账户包含模块 0x0.Currency 和类型为 0x0.Currency.Coin 的resources值。 地址 0x1 的账户有两个resources和一个模块； 地址 0x2 的帐户有两个模块和一个资源值。
相比之下，模块是在全局状态下发布的一段长期存在的代码。 上面示例中使用的模块名称 0x0.Currency 包含发布模块代码的帐户地址 0x0。 全局状态被构造为从帐户地址到帐户的映射。
![figure1](https://github.com/coming-chat/Move-white-paper/blob/main/Figure1.png)

帐户最多可以包含一个给定类型的resources值和最多一个具有给定名称的模块。 不允许地址 0x0 的帐户包含额外的 0x0.Currency.Coin resources或另一个名为 Currency 的模块。 但是，地址 0x1 的帐户可以添加一个名为 Currency 的模块。 在这种情况下，0x0 也可以拥有 0x1.Currency.Coin 类型的resources。 0x0.Currency.Coin 和 0x1.Currency.Coin 是不同的类型，不能互换使用； 声明模块的地址是类型的一部分。

请注意，帐户中最多允许给定类型的单个resource不是限制性的。 此设计为顶级帐户值提供了可预测的存储架构。 程序员仍然可以通过定义自定义包装resource（例如，resource TwoCoins { c1: 0x0.Currency.Coin, c2: 0x0.Currency.Coin }）在帐户中保存给定resource类型的多个实例。

***声明 Coin 资源***。 在解释了模块如何适应 Move 执行模型之后，我们可以开始看 Currency 模块的内部：
```rust
module Currency {
  resource Coin { value: u64 }
  // ...
}
```
以上代码声明了一个Currency 的模块，里面包含了一个 名为Coin的 resource 类型。Coin 是一种结构类型，具有 u64 类型的单个字段值（64 位无符号整数）。 Coin 的结构在 Currency 模块之外是不透明的。 其他模块和交易脚本只能通过模块公开的公共程序写入或引用值字段。 同样，只有 Currency 模块的程序可以创建或销毁 Coin 类型的值。 该方案支持强大的数据抽象——模块作者可以完全控制其声明resource的访问、创建和销毁。 在 Currency 模块公开的 API 之外，另一个模块可以对 Coin 执行的唯一操作是移动。 resource安全禁止其他模块复制、破坏或双重移动resource。

***deposit程序的实现***。 让我们研究上一节中交易脚本调用的 Currency.deposit 程序是如何工作的：
```rust
public deposit(payee: address, to_deposit: Coin) {
  let to_deposit_value: u64 = Unpack<Coin>(move(to_deposit));
  let coin_ref: &mut Coin = BorrowGlobal<Coin>(move(payee));
  let coin_value_ref: &mut u64 = &mut move(coin_ref).value;
  let coin_value: u64 = *move(coin_value_ref);
  *move(coin_value_ref) = move(coin_value) + move(to_deposit_value);
}
```
面向代码阅读者者的可读性代码逻辑。此程序将 Coin resource作为输入，并将其与存储在收款人帐户中的 Coin resource相结合。 它通过以下方式实现：
1. 销毁输入的 Coin 并记录它的值。
2. 获取收款人下面的唯一 resource类型的coin 引用。
3. 加上传递给程序的 Coin 的值来更新到收款人的 Coin 的值。

此程序的面向机器语言的某些方面值得解释。 绑定到 to_deposit 的 Coin 资源归存款程序所有。 要调用该程序，调用者需要将绑定到 to_deposit 的 Coin 移动到被调用者（这将阻止调用者重用它）。

在第一行调用的 Unpack 程序是用于操作模块声明的类型的几个内置模块之一。 Unpack<T> 是删除 T 类型resource的唯一方法。它将 T 类型的resource作为输入，销毁它，并返回绑定到resource字段的值。 像 Unpack 这样的内置模块只能用于当前模块中声明的resource。 在 Unpack 的情况下，此约束防止其他代码销毁 Coin，这反过来又允许 Currency 模块为销毁 Coin resource设置自定义的前提条件（例如，它可以选择只允许销毁0值的Currency）。

第三行调用的 BorrowGlobal 程序也是一个内置模块。 BorrowGlobal<T> 将地址作为输入并返回对在该地址下发布的 T 的唯一实例的引用。 这意味着上面代码中的 coin_ref 类型是 &mut Coin,表面这是对 Coin resource的可变引用，而不是拥有Coin resource的 本身。 下一行移动绑定到 coin_ref 的参考值，以获取 Coin 值字段的参考 coin_value_ref。 该程序的最后几行读取收款人 Coin resource的先前值，并改变 coin_value_ref 以反映存款金额。

我们注意到 Move 类型系统无法捕获模块内的所有实现错误。 例如，类型系统不会确保所有存在的coin的总量通过存款调用来保存。 如果程序员在最后一行写错了 *move(coin_value_ref) = 1 + move(coin_value) + move(to_deposit_value)，类型系统将毫无疑问地接受该代码。 这表明了明确的职责分工：在模块范围内为 Coin 建立适当的安全不变量是程序员的工作，而类型系统的工作是确保模块外的 Coin 客户端不会违反这些不变量。

***实现withdraw_from_sender程序***。 在上面的实现中，通过deposit程序存入资金不需要任何授权——任何人都可以调用存款。 相比之下，从帐户中取款必须受到授予货币resource所有者独占权限的访问控制策略的保护。 让我们看看我们的点对点支付交易脚本调用的withdraw_from_sender 程序是如何实现这个授权的：

``` rust
public withdraw_from_sender(amount: u64): Coin {
  let transaction_sender_address: address = GetTxnSenderAddress();
  let coin_ref: &mut Coin = BorrowGlobal<Coin>(move(transaction_sender_address));
  let coin_value_ref: &mut u64 = &mut move(coin_ref).value;
  let coin_value: u64 = *move(coin_value_ref);
  RejectUnless(copy(coin_value) >= copy(amount));
  *move(coin_value_ref) = move(coin_value) - copy(amount);
  let new_coin: Coin = Pack<Coin>(move(amount));
  return move(new_coin);
}
```
这个程序几乎是deposit的逆程序，但也不全是。 如下：
1. 获取对发送者帐户下发布的 Coin 类型的唯一resource的引用。
2. 将引用的 Coin 的值减少输入数量。
3. 创建并返回一个具有对应发送金额的新Coin。

此程序执行的访问控制检查有些微妙。 withdraw程序允许调用者指定传递给 BorrowGlobal 的地址，但withdraw_from_sender 只能传递 GetTxnSenderAddress 返回的地址。 此程序是允许Move代码从当前正在执行的交易中读取数据的几个交易内置程序之一。 Move 虚拟机在交易执行之前验证发送者地址。 以这种方式使用 BorrowGlobal 内置确保交易的发送者只能从她自己的 Coin resource中提取资金。

与所有内置模块一样，BorrowGlobal<Coin> 只能在声明 Coin 的模块内调用。 如果 Currency 模块没有公开返回 BorrowGlobal 结果的程序，则 Currency 模块之外的代码无法获取对全局存储中发布的 Coin resource的引用。

在减少交易发送者的 Coin resource的值之前，该程序使用 RejectUnless 指令断言代币的价值大于或等于输入的数额。 这确保了发件者不能提取超过她所拥有的金额。 如果此检查失败，当前交易脚本的执行将停止，并且它执行的任何操作都不会更新于全局状态。

最后，该程序将发送者的 Coin 的值按数量减少，并使用 Unpack 的逆操作（内置的 Pack 模块）创建一个新的 Coin resource。 Pack<T> 创建一个类型为 T 的新resource。与 Unpack<T> 一样，Pack<T> 只能在resource T 的声明模块内部调用。这里，Pack 用于创建一个类型为 Coin 的resource new_coin 并移动它 给 调用者。 调用者现在拥有此 Coin resource，并且可以将其移动到任何她喜欢的地方。 在我们第 4.1 节的示例交易脚本中，调用者选择将 Coin 存入收款人的账户。

## 5. Move 语言 详解
在本节中，我们给出了 Move 语言、字节码验证器和虚拟机的半正式描述。 附录 A 详细列出了所有这些组成部分，但没有任何附带的散文。 我们在这里的讨论将使用附录中的摘录，并且偶尔会引用其中定义的符号。

***Global state.***
```
  Σ ∈ GlobalState = AccountAddress ⇀ Account
  Account = (StructID ⇀ Resource) × (ModuleName ⇀ Module)
```
Move 的目标是使程序员能够定义全局区块链状态并安全地实现更新全局状态的操作。 正如我们在 4.2 节中解释的那样，全局状态被组织为从地址到帐户的部分映射。 帐户包含resource数据值和模块
代码值。 帐户中的不同resource必须具有不同的标识符。 帐户中的不同模块必须具有不同的名称。

***Modules.***
```
Module = ModuleName × (StructName ⇀ StructDecl) × (ProcedureName ⇀ ProcedureDecl)
ModuleID = AccountAddress × ModuleName
StructID = ModuleID × StructName
StructDecl = Kind × (FieldName ⇀ NonReferenceType)
```

模块由名称、结构声明（包括resource，我们稍后将解释）和程序声明组成。 代码可以使用由模块的帐户地址和模块名称组成的唯一标识符来引用已发布的模块。 模块标识符用作命名空间，用于限定其结构类型的标识符和模块外部代码的程序。

Move模块支持强大的数据抽象。 模块的程序对创建、写入和销毁模块声明的类型的规则进行编码。 类型在其声明模块内部是透明的，在外部是不透明的。但Move 可以通过以下几个特定程序获得不同权限

- 通过 MoveToSender 指令强制发布帐户下的resource
- 通过 BorrowGlobal 指令获取对帐户下resource的引用
- 通过 MoveFrom 指令从帐户中删除resource的先决条件。

模块使 Move 程序员可以灵活地为resource定义丰富的访问控制策略。 例如，一个模块可以定义一个只有在其f字段为零时才能被销毁的resource类型，或者一个只能在某些帐户地址下才能被发布的resource。

***类型***
```
PrimitiveType = AccountAddress ∪ Bool ∪ UnsignedInt64 ∪ Bytes
StructType = StructID × Kind
𝒯 ⊆ NonReferenceType = StructType ∪ PrimitiveType
Type ::= 𝒯 |&mut𝒯 |&𝒯
```
Move 支持基本类型，包括布尔值、64 位无符号整数、256 位帐户地址和固定大小的字节数组。 结构是由模块声明的用户定义类型。 通过使用resource种类标记结构类型，将其指定为resource。 所有其他类型，包括非resource结构类型和原始类型，都称为无限制类型。
   
resource类型的变量是resource变量； 不受限制类型的变量是不受限制的变量。 字节码验证器对resource变量和resource类型的结构字段实施限制。 resource变量无法复制，必须始终Move。 resource变量和resource类型的结构字段都不能重新分配 - 这样做会破坏先前保存在存储位置中的resource值。 此外，不能取消对resource类型的引用，因为这会产生底层resource的副本。 相比之下，不受限制的类型可以被复制、重新分配和取消引用。
   
最后，不受限制的结构类型可能不包含具有resource类型的字段。此限制确保 (a) 复制不受限制的结构不会导致嵌套resource的复制，并且 (b) 重新分配不受限制的结构不会导致嵌套resource的破坏。
   
引用类型可以是可变的或不可变的； 不允许通过不可变引用进行写入。 字节码验证器执行引用安全检查，以强制执行这些规则以及对resource类型的限制（参见第 5.2 节）。

***值：***
```
Resource = FieldName ⇀ Value
Struct = FieldName ⇀ UnrestrictedValue
UnrestrictedValue = Struct ∪ PrimitiveValue
𝑣 ∈ Value = Resource ∪ UnrestrictedValue
g ∈ GlobalResourceKey = AccountAddress × StructID
𝑎𝑝 ∈ AccessPath ::= 𝑥|𝑔|𝑎𝑝.𝑓
𝑟 ∈ RuntimeValue ::= 𝑣 |ref𝑎𝑝
```
除了结构和原始值之外，Move 还支持引用值。参考不同于其他Move值，因为应用值是瞬态的。字节码验证器不允许引用**类型**的字段。该要求会约束引用只能在交易脚本执行期间被创建，并在该交易脚本结束之前释放引用。

对结构值形状的限制确保全局状态始终是树而不是任意图。状态树中的每个存储位置都可以使用其访问路径 [24] 进行规范表示 - 从存储树中的根（局部变量𝑥或全局resource键𝑔）到由字段序列标记的后代节点的路径标识符𝑓。

该语言允许引用原始值和结构，但不允许引用其他引用。 Move程序员可以使用 BorrowLoc 指令获取对局部变量的引用，使用 BorrowField 指令获取结构的字段，使用 BorrowGlobal 指令获取在帐户下发布的全局resource。 后两个构造只能用于当前模块内声明的结构类型。
   
***程序和交易脚本***：
![](https://github.com/coming-chat/Move-white-paper/blob/main/figure2.jpeg)
程序签名由可见性、类型化的形式参数和返回类型组成。 程序声明包含签名、类型化的局部变量和字节码指令数组。 程序可见性可以是公开的或内部的。 内部程序只能由同一模块中的其他程序调用。 公共程序可以由任何模块或交易脚本调用。

区块链状态由交易脚本更新，该脚本可以调用当前在帐户下发布的任何模块的公共程序。 交易脚本只是一个没有关联模块的程序声明。

一个程序可以由它的模块标识符和它的签名来唯一标识。 调用字节码指令需要一个唯一的程序 ID 作为输入。 这确保了 Move 中的所有程序调用都是静态确定的——没有函数指针或虚拟调用。 此外，模块之间的依赖关系在构造上是非循环的。 一个模块只能依赖于线性交易历史中较早发布的模块。 非循环模块依赖图和不能动态分派两者的组合决定了确定性执行逻辑：属于模块中的程序的所有堆栈帧必须是连续的。 因此，Move 模块中没有以太坊智能合约的重入 [16] 问题。
  
在本节的其余部分，我们将介绍字节码操作及其语义（第 5.1 节），并描述字节码验证器在允许执行或存储模块代码之前执行的静态分析（第 5.2 节）

### 字节码解释器
```
𝜎 ∈ InterpreterState = ValueStack × CallStack × GlobalRefCount × GasUnits
𝑣𝑠𝑡𝑘 ∈ ValueStack ::= []|𝑟::𝑣𝑠𝑡𝑘
𝑐𝑠𝑡𝑘 ∈ CallStack ::= []|𝑐::𝑐𝑠𝑡𝑘
𝑐 ∈ CallStackFrame = Locals × ProcedureID × InstrIndex
Locals = VarName ⇀ RuntimeValue
```
移动字节码指令由类似于通用语言运行时 [22] 和 Java 虚拟机 [21] 的基于堆栈的解释器执行。 一条指令使用堆栈中的操作数并将结果压入堆栈。 指令还可以从当前过程的局部变量（包括形式参数）移动和复制值。

字节码解释器支持程序调用。 传递给被调用者的输入值和返回给调用者的输出值也通过堆栈进行通信。 首先，调用者将过程的参数压入堆栈。 接下来，调用者调用 Call 指令，该指令为被调用者创建一个新的调用堆栈帧，并将推送的值加载到被调用者的局部变量中。 最后，字节码解释器开始执行被调用程序的字节码指令。

字节码的执行按顺序执行操作，除非有一个分支操作导致跳转到当前过程中静态确定的偏移量。 当被调用者希望返回时，它将返回值压入堆栈并调用 Return 指令。 然后将控制权返回给调用者，调用者在堆栈上查找输出值。

Move程序的执行以类似于 EVM [9] 的方式计量。 每个字节码指令都有一个相关的gas单位成本，任何要执行的交易都必须包括一个gas单位预算。 解释器在执行期间跟踪剩余的gas单位，如果剩余量达到零，则停止并报错。

我们参考比较了基于寄存器和基于堆栈的字节码解释器，发现具有类型化局部变量的堆栈机器非常适合 Move 的Resource语义。 在局部变量、堆栈和调用者/被调用者对之间来回移动值的低级（指面向机器的是低级）机制密切反映了 Move 程序的高级（指面向人类可读的语言是高级）意图。 没有局部变量的堆栈机器会更加冗长，而寄存器机器会使跨过程边界移动resource变得更加复杂。

***指令集***：Move 支持六大类字节码指令
- 用于将数据从局部变量复制/移动到堆栈的 CopyLoc/MoveLoc 等操作，StoreLoc 用于将数据从堆栈移动到局部变量。
- 对类型化堆栈值的操作，例如将常量推入堆栈，以及对堆栈操作数的算术/逻辑操作。
- 模块内置函数，例如 Pack 和 Unpack 用于创建/销毁模块声明的类型，MoveToSender/MoveFrom 用于取得/取消 账户下的发布模块类型权利，以及 BorrowField 用于获取对模块类型之一的字段的引用。
- 与引用相关的指令，例如 ReadRef 用于读取引用，WriteRef 用于写入引用，ReleaseRef 用于销毁引用，以及 FreezeRef 用于将可变引用转换为不可变引用。
- 控制流操作，例如条件分支和从程序调用/返回。
- 特定于区块链的内置操作，例如获取交易脚本发送者的地址和创建新帐户。

***附录 A*** 给出了Move字节码指令的完整列表。 Move 还提供加密原语，例如 sha3，但这些原语是作为标准库中的模块而不是字节码指令实现的。 在这些标准库模块中，过程被声明为本机，过程体由 Move VM 提供。 只有 VM 可以定义新的本机过程，这意味着这些加密原语可以改为作为普通字节码指令来实现。 但是，本机过程很方便，因为 VM 可以依赖现有机制来调用过程，而不是为每个加密原语重新实现调用约定。

### 5.2. 字节码验证器
```
   𝐶 ∈ Code = TransactionScript ∪ Module
   𝑧 ∈ VerificationResult ::= ok | stack_err | type_err | reference_err | ...
   𝐶 ⇝ 𝑧 = bytecode verification
```
字节码验证器的目标是验证***提交发布的任何模块***和***提交执行的任何交易脚本静态强制执行**的安全。 不通过字节码验证器，任何 Move 程序都不能发布或执行。

字节码验证器强制执行任何格式良好的 Move 程序必须具备的一般安全属性。 我们的目标是在未来的工作中为特定于程序的属性开发一个单独的离线验证器（见第 7 节）。

Move 模块或交易脚本的二进制格式对实体表的集合进行编码，例如常量、类型签名、结构定义和程序定义。 验证者执行的检查分为三类：
- 结构检查以确保字节码表格式正确。 这些检查发现诸如非法表索引、重复表条目和非法类型签名（如对引用的引用）之类的错误。
- 程序主体的语义检查。 这些检查检测错误，例如不正确的过程参数、悬空引用和复制资源。
- 将结构类型和程序签名的使用与其声明模块联系起来。 这些检查检测错误，例如非法调用内部程序和使用与其声明不匹配的程序标识符。

在本节的其余部分，我们将描述语义验证和链接方面的内容。
   
***控制流图构造***。 验证者通过将指令序列分解为基本块的集合来构建控制流图（请注意，这些与区块链中的交易“块”无关）。 每个基本块都包含一个连续的指令序列； 所有指令的集合在块之间进行划分。 每个基本块都以分支或返回指令结束。 分解保证分支目标仅在某些基本块的开始处着陆。 分解还试图确保生成的块是最大的。 然而，字节码验证器的健全性并不依赖于最大值。

***堆栈余额检查***。堆栈平衡检查确保被调用者无法访问属于调用者的堆栈位置。基本块的执行发生在局部变量数组和堆栈的上下文中。程序的参数是局部变量数组的前缀。跨程序调用传递参数和返回值是通过堆栈完成的。当一个程序开始执行时，它的参数已经加载到它的参数中。假设程序开始执行时堆栈高度为 n。有效字节码必须满足以下不变量：当执行到达基本块的末尾时，堆栈高度为 n。验证者通过分别分析每个基本块并计算每条指令对堆栈高度的影响来确保这一点。它检查高度不低于 n，并且在基本块出口处为 n。一个例外是以 Return 指令结束的块，其中高度必须为 n+m（其中 m 是过程返回的值的数量）。
   
***类型检查***。验证器的第二阶段检查每个指令和程序（包括内置程序和用户自定义程序）是否使用适当类型的参数调用。指令的操作数是位于局部变量或堆栈中的值。字节码中已经提供了程序的局部变量的类型。但是，堆栈值的类型是推断出来的。这种推断和每个操作的类型检查是针对每个基本块单独完成的。由于每个基本块开头的堆栈高度为 n 并且在块执行期间不会低于 n，因此我们只需对从 n 开始的堆栈后缀进行建模，以便对块指令进行类型检查。我们使用类型堆栈对这个后缀进行建模，在处理基本块中的指令序列时，这些类型会被压入和弹出。类型堆栈和静态已知类型的局部变量足以对每个字节码指令进行类型检查。

***仔细的检查***。 验证者在类型检查阶段通过以下附加检查来执行resource安全：
- resource不能被复制：CopyLoc 不用于类型resource的局部变量，并且 ReadRef 不用于类型是对类型resource值的引用的堆栈值。
- resource不能被销毁：PopUnrestricted 不用于类型resource的堆栈位置，StoreLoc 不用于已保存resource的局部变量，并且 WriteRef 不用于对类型resource值的引用。
- 必须使用resource：当程序返回时，任何局部变量都不能保存resource值，并且评估堆栈的被调用者段必须只保存程序的返回值。
   
非resource结构类型不能具有resource类型的字段，因此这些检查不能通过（例如）复制具有resource字段的非resource结构来破坏。
   
非resource结构类型不能具有resource类型的字段，因此这些检查不能通过（例如）复制具有resource字段的非resource结构来破坏。resource不能被因错误而停止的程序执行破坏。 正如我们在 4.2 节中解释的那样，交易脚本的部分执行所产生的状态变化不会被提交到全局状态。 这意味着在运行时失败时位于堆栈或局部变量中的resource将（有效地）返回到它们在交易开始执行之前的任何位置。
   
原则上，非终止程序执行可以使resource无法访问。 但是，第 5.1 节中描述的gas计量方案可确保 Move 程序的执行始终终止。 耗尽gas的执行会因错误而停止，这不会导致resource丢失（如上所述）。
   
***参考检查***。 使用静态和动态分析的组合检查参考的安全性。 静态分析以类似于 Rust 类型系统的方式使用借用检查，但在字节码级别而不是在源代码级别执行。 这些参考检查确保了两个强大的属性：
- 所有引用都指向分配的存储（即，没有悬空引用）。
- 所有引用都具有安全的读写访问权限。 引用要么是共享的（没有写访问权和有限的读访问权），要么是独占的（有有限的读写访问权）。
   
为了确保这些属性对通过 BorrowGlobal 创建的全局存储的引用保持有效，字节码解释器执行轻量级动态引用计数。 解释器跟踪每个已发布资源的未完成引用的数量。 如果在对该资源的引用仍然存在的情况下借用或移动了全局资源，它会使用此信息停止并出现错误。

这种参考检查方案具有许多新颖的功能，将成为另一篇论文的主题。
   
***全局状态链接器***。
```
   𝐷 ∈ Dependencies = StructType∗ × ProcedureID∗
   deps ∈ Code → Dependencies            computing dependencies
   𝑙 ∈ LinkingResult := success | fail
   ⟨𝐷, Σ⟩ ↪ 𝑙.                           linking dependencies with global state
```
在字节码验证过程中，验证者假定当前代码单元使用的外部结构类型和过程 ID 存在并且如实表示。 链接步骤通过从全局状态 Σ 读取结构和程序声明并确保声明与它们的用法相匹配来检查此假设。 具体来说，链接器检查全局状态中的以下声明是否与它们在当前代码单元中的用法相匹配：
- 结构声明（名称和种类）。
- 程序签名（名称、可见性、形式参数类型和返回类型）。

## 6. Move虚拟机：所有模块组合在一起。
![](https://github.com/coming-chat/Move-white-paper/blob/main/figure3.png)
Move 虚拟机的作用是：引用全局状态 Σ 作为初始化状态，然后执行交易集合组成的区块 𝐵，并产生表示对全局状态的修改的交易效果 𝐸。 然后可以将执行结果 𝐸 应用于 Σ 以生成由执行𝐵 产生的状态 Σ′。将执行结果与实际状态更新分开，用来避免 VM 在交易执行失败的情况下保持 全局状态不更新，还是原来的 Σ。

直观地说，交易执行结果表示账户子集的全局状态更新。交易执行结果与全局状态具有相同的结构：它是从账户地址到账户的部分映射，其中包含Move模块和resource值的规范序列化表示。规范序列化从 Move 模块实现与语言无关的 1-1 函数或resource到字节数组。

为了从状态 Σ𝑖−1 执行区块 𝐵，VM 从区块𝐵获取交易𝑇𝑖，对其进行处理以产生执行结果𝐸𝑖，然后将𝐸𝑖应用于 Σ𝑖−1 以生成状态 Σ𝑖，用作初始区块中的下一笔交易执行状态。 整个区块的执行结果是区块中每笔交易的执行结果的有序组合。

每笔交易都根据一个工作流进行处理，该工作流包括验证交易中的字节码和检查交易发送者的签名等步骤。 [2] 中更详细地解释了执行单个交易的整个工作流程。

现有的区块链一个块中的交易由 VM 顺序执行，但 Move 语言被设计为支持并行执行。 原则上，执行一个交易可以产生一组读取以及一组写入结果的𝐸。 区块中的每个交易都可以推测性地并行执行，并且仅当其读/写集与块中的另一个交易冲突时才重新执行。检查冲突很简单，因为 Move 的树内存模型允许我们使用其访问路径唯一地标识全局内存单元。 如果虚拟机性能成为 Libra 区块链的瓶颈，我们将在未来探索推测执行方案。

## 7. Move 的未来计划

到目前为止，我们已经设计并实现了 Move 的以下组件：
- 适用于区块链执行的编程模型。
- 适合这种可编程模型的字节码语言。
- 用于实现具有强大数据抽象和访问控制的库的模块系统。
- 由序列化器/反序列化器、字节码验证器和字节码解释器组成的虚拟机。

尽管取得了这些进展，但前面还有很长的路要走。最后，我们讨论了一些近期的下一步措施和 Move 的长期计划。

***实现核心 Libra 区块链功能***。我们将使用 Move 来实现 Libra 区块链中的核心功能：账户、Libra 币、Libra 储备管理、验证节点添加和删除、收取和分配交易费用、冷钱包等。这项工作已经在进行中。

***新的语言功能***。我们将为 Move 语言添加参数多态性（泛型）、集合和事件。参数多态性不会破坏 Move 现有的安全性和可验证性保证。我们的设计以类似于 [25] 的方式将具有种类（即resouce或unrestricted）约束的类型参数添加到过程和结构中。

此外，我们将开发一个可信的机制，用于版本控制和更新 Move 模块、交易脚本和已发布的resouce。


***改善开发人员的体验***。 Move IR 是作为 Move 字节码验证器和虚拟机的测试工具而开发的。为了运行这些组件，IR 编译器必须故意生成将（例如）被字节码验证器拒绝的错误字节码。这意味着虽然 IR 适合于制作 Move 程序的原型，但它并不是特别用户友好。为了让 Move 对第三方开发更具吸引力，我们将改进 IR 并努力开发符合人体工程学的 Move 源语言。

***标准的规范和验证***。我们将创建一种逻辑规范语言和自动形式验证工具，以利用 Move 的验证友好设计（参见第 3.4 节）。验证工具链将检查程序特定的功能正确性属性，这些属性超出了 Move 字节码验证器（第 5.2 节）强制执行的安全保证。我们最初的重点是指定和验证实现 Libra 区块链核心功能的模块。

我们的长期目标是促进一种正确的文化，在这种文化中，用户将通过模块的正式规范来了解其功能。理想情况下，除非模块具有全面的标准规范并且已被验证符合该规范，否则没有 Move 程序员愿意与模块进行交互。然而，实现这一目标将带来一些技术和社会挑战。验证工具应该是精确和直观的。规范必须是模块化的和可重用的，但具有足够的可读性，可以作为模块行为的有用文档。

***支持第三方Move模块***。我们将开发第三方模块发布的路径。为 Libra 用户和第三方开发者创造良好的体验是一项重大挑战。首先，向通用应用敞开大门，不得影响系统对核心支付场景和相关金融应用的可用性。其次，我们希望避免诈骗、投机和有缺陷的软件带来的声誉风险。在鼓励高质量软件的同时构建开放系统是一个难题。诸如为高保证模块创建市场和提供验证 Move 代码的有效工具等步骤将有所帮助。

## 致谢
我们感谢 Tarun Chitra、Sophia Drossopoulou、Susan Eisenbach、Maurice Herlihy、John Mitchell、James Prestwich 和 Ilya Sergey 对本文提出的深思熟虑的反馈。
