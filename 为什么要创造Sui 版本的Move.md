# 为什么要创造 Sui 版本的Move

***本文重点介绍：***
- Sui版Move 功能齐全，为您编写安全高效的智能合约做好准备
- Sui 是第一个在集成 Move 方面改进原始 Diem 设计的区块链，我们分享了这些改进的具体示例。

## 介绍

Move 诞生于 2018 年 Libra 项目初期——两位 Mysten 创始人（Evan 和我自己）也在 Libra 的创始团队中。
在我们决定创建一种新语言之前，早期的 Libra 团队深入研究了现有的智能合约用例和语言，以了解开发人员想要做什么以及现有语言无法提供的地方。
我们发现的关键问题是智能合约都是关于资产和访问控制的，但早期的智能合约语言缺乏两者的类型/值表示。 
Move 假设是，如果我们为这些关键概念提供一流的抽象，我们可以显著的提高智能合约的安全性和智能合约程序员的生产力——为手头的任务拥有正确的词汇会改变一切。
多年来，许多人为 Move 的设计和实现做出了贡献，因为该语言从一个关键思想演变为一种与平台无关的智能合约语言，其大胆的目标是成为“web3 的 JavaScript”。

今天，我们很高兴地宣布将 Move 集成到 Sui 中的一个里程碑：Sui 版 Move 功能齐全，由高级工具支持，并具有大量文档和示例，包括：
- Sui 版Move 对象编程教程[系列](https://docs.sui.io/build/programming-with-objects)
- Sui 版Move 基础知识、设计模式和示例的[cookbook](https://examples.sui.io/)
- [增强的 VSCode 插件](https://sui.io/resources-move/announcing-enhanced-move-vs-code-plugin/)，支持由 Mysten Move 团队构建的代码理解和错误诊断！
- 将 Move 构建、测试、包管理、文档生成和 Move Prover 与 sui CLI 集成
- 一系列[示例](https://github.com/MystenLabs/sui/tree/main/sui_programmability/examples)，包括可替代代币、NFT、DeFi 和游戏。

当我们在 2021 年底开始研究 Sui 时，我们重新审视了 Move，并反思了哪些早期的设计决策没有很好地老化，以及如何改进 Move 以利用 Sui 的独特功能。
我们之前已经写过 Sui Move 在语言层面的新功能，但我们还没有深入探讨引入这些差异的动机。 本文的其余部分以示例驱动的方式详细介绍了此查询。

## 等等，有不同的Move吗？

Move 是一种跨平台的嵌入式语言。 核心语言本身很简单：它有通用的概念，比如结构体、整数和地址，但它没有区块链特定的概念，比如账户、交易、时间、密码学等。
这些特性必须由区块链提供 集成 Move 的平台。 
重要的是，这些区块链不需要自己的 Move 分支——每个平台都使用相同的 Move VM、字节码验证器、编译器、证明器、包管理器和 CLI，但通过构建在这些核心组件之上的代码添加区块链特定功能。

Diem 是第一个嵌入 Move 的区块链，后续基于 Move 的链（包括 0L、StarCoin 和 Aptos）在很大程度上使用了 Diem 风格的方法。
尽管 Diem 风格的 Move 具有一些不错的品质，但 Diem 的许可性质和 Diem 区块链的某些实施细节（特别是存储模型）都使得一些基本智能合约用例的实施变得困难。
特别是，Move 和 Diem 的原始设计早于 NFT 的流行爆发，并且有一些怪癖使得与 NFT 相关的用例特别难以实施。

在这篇文章中，我们将通过三个这样的示例来展示原始 Diem 风格的 Move 嵌入的问题，并描述我们如何在 Sui Move 中解决这个问题。 
我们假设对 Move 有一些基本的了解，但希望任何具有编程背景的人都能理解其中的关键点。

## 无摩擦的大规模资产创建

批量创建和分发资产的能力对于吸引 web3 用户都至关重要。 也许 Twitch 主播想要分发纪念 NFT，创作者想要发送特殊活动的门票，或者游戏开发者想要向所有玩家空投新物品。

这是一个（失败的）尝试编写代码以使用 Diem 风格的 Move 大规模铸造资产。 此代码将收件人地址向量作为输入，为每个地址生成一个资产，并尝试转移该资产。

``` rust
struct CoolAsset { id: GUID, creation_date: u64 } has key, store
public entry fun mass_mint(creator: &signer, recipients: vector<address>) {
  assert!(signer::address_of(creator) == CREATOR, EAuthFail);
  let i = 0;
  while (!vector::is_empty(recipients)) {
    let recipient = vector::pop_back(&mut recipients);
    assert!(exists<Account>(recipient), ENoAccountAtAddress);
    let id = guid::create(creator);
    let creation_date = timestamp::today();
    // error! recipient must be `&signer`, not `address`
    move_to(recipient, CoolAsset { id, creation_date })
}
```
在 Diem 风格的 Move 中，全局存储由（地址，类型名称）对进行键控——也就是说，每个地址最多可以存储一个给定类型的资产。
因此， ***move_to(recipient, CoolAsset { ... }*** 行试图通过将 ***CoolAsset*** 存储在***recipient***地址下来传输它。

但是，此代码将无法在 ***move_to(recipient, ...)*** 行编译。 关键问题是，在 Diem 风格的 Move 中，您不能将 ***CoolAsset*** 类型的值发送到地址 A，除非：
- 非 A 地址发送交易以用 A 创建帐户
- A 的所有者发送交易以明确选择接收 ***CoolAsset*** 类型的对象

这是为了获得资产而进行的两次交易！ 以这种方式做事的决定对 Diem 来说是有意义的，这是一个需要谨慎限制帐户创建并防止帐户由于存储系统限制而持有过多资产的许可系统。 
但这对于想要使用资产分配作为入职机制的开放系统，或者只是允许资产像在以太坊和类似区块链上一样在用户之间自由流动 [1] 时，具有极大的限制性。

现在，让我们看一下 Sui Move 中的相同代码：
``` rust
struct CoolAsset { id: VersionedID, creation_date: u64 } has key
public entry fun mass_mint(recipients: vector<address>, ctx: &mut TxContext) {
  assert!(tx_context::sender(ctx) == CREATOR, EAuthFail);
  let i = 0;
  while (!vector::is_empty(recipients)) {
    let recipient = vector::pop_back(&mut recipients);
    let id = tx_context::new_id(ctx);
    let creation_date = tx_context::epoch(); // Sui epochs are 24 hours
    transfer(CoolAsset { id, creation_date }, recipient)
  }
}
```

Sui 版Move 的全局存储，以对象 ID 为键。 每个具有关键能力的结构都是一个“Sui 对象”，它必须具有全局唯一的 id 字段。 
Sui 版Move 没有使用受限的 move_to 构造，而是引入了可以在任何 Sui 对象上使用的传输原语。 
在底层，这个原语将 id 映射到全局存储中的 CoolAsset 并添加元数据以指示该值由recipient拥有。

sui 版本的 mass_mint 的一个有趣特性是它可以与所有其他交易（包括其他调用 mass_mint！）交互。 
Sui 运行时会注意到这一点，并通过不需要共识的拜占庭一致广播“快速路径”发送调用该函数的交易。
这样的交易可以并行提交和执行！ 这不需要程序员付出任何努力——他们只需编写上面的代码，运行时负责其余的工作。

也许微妙的是，此代码的 Diem 变体并非如此——即使上面的代码有效，***exists<Account>*** 和 ***guid::create*** 调用都会与其他生成 ***GUID*** 或
接触 ***Account*** 资源的交易产生争用点。 在某些情况下，可以重写 Diem 风格的 Move 代码以避免争用点，但是编写 Diem 风格的 Move 的许多惯用方式引入了阻碍并行执行的微妙瓶颈。
  
## 原生资产的所有权 转让
  
让我们用一个可以实际编译和运行的变通方法来扩展 Diem 风格的 Move 代码。
执行此操作的惯用方式是“包装器模式”：因为 Bob 不能直接将 ***CoolAsset*** 移动到 Alice 的地址，我们要求 Alice 通过首先发布具有集合类型的包装器类型 ***CoolAssetStore***（表 ） 里面。 
Alice 可以通过调用 ***opt_in*** 函数来做到这一点。 然后我们添加允许 Bob 将 ***CoolAsset*** 从他的 ***CoolAssetStore*** 移动到 Alice 的 ***CoolAssetStore*** 的代码。
  
在这段代码中，让我们添加另一个问题：我们只允许 ***CoolAsset*** 的创建时间至少为 30 天。 这种政策对于（例如）想要阻止投机者购买/翻转活动门票的创作者很重要，因此真正的粉丝更容易以合理的价格获得它们。

``` rust
struct CoolAssetStore has key {
  assets: Table<TokenId, CoolAsset>
}
public fun opt_in(addr: &signer) {
  move_to(addr, CoolAssetHolder { assets: table::new() }
}
public entry fun cool_transfer(
  addr: &signer, recipient: address, id: TokenId
) acquires CoolAssetStore {
  // withdraw
  let sender = signer::address_of(addr);
  assert!(exists<CoolAssetStore>(sender), ETokenStoreNotPublished);
  let sender_assets = &mut borrow_global_mut<CoolAssetStore>(sender).assets; 
  assert!(table::contains(sender_assets, id), ETokenNotFound);
	let asset = table::remove(&sender_assets, id);
  // check that 30 days have elapsed
  assert!(time::today() > asset.creation_date + 30, ECantTransferYet)
	// deposit
	assert!(exists<CoolAssetStore>(recipient), ETokenStoreNotPublished);
  let recipient_assets = &mut borrow_global_mut<CoolAssetStore>(recipient).assets; 
  assert!(table::contains(recipient_assets, id), ETokenIdAlreadyUsed);
  table::add(recipient_assets, asset)
}
```
  
此代码有效。 但这是实现将资产从 Alice 转移到 Bob 的简单目标的一种非常复杂的方法！ 再次，让我们看一下 Sui 版本Move：
``` rust
public entry fun cool_transfer(
  asset: CoolAsset, recipient: address, ctx: &mut TxContext
) {
  assert!(tx_context::epoch(ctx) > asset.creation_date + 30, ECantTransferYet);
  transfer(asset, recipient)
}
```
  
这段代码要短得多。 这里要注意的关键是，***cool_transfer*** 是一个入口函数（意味着它可以由 Sui 运行时通过交易直接调用），但它有一个 ***CoolAsset*** 类型的参数作为输入。
这又是 Sui 运行时的魔法！ 一个交易包括它想要操作的一组对象 ID，以及 Sui 运行时：
- 将 ID 解析为对象值（消除上面 Diem 样式代码中对 ***borrow_global_mut*** 和 ***table_remove*** 部分的需要)
- 检查该对象是否由交易的发送者拥有（消除对上述signer::address_of部分 + 相关代码的需要）。 这部分特别有趣，我们稍后会解释：在 Sui 中，安全的对象所有权检查是运行时的一部分！
- 根据调用的函数***cool_transfer***的参数类型检查对象值的类型
- 将对象值和其他参数绑定到***cool_transfer***的参数并调用函数
  
这允许 Sui Move 程序员跳过逻辑中“withdraw”部分的样板，直接跳到有趣的部分：检查 30 天到期政策。
类似地，“deposit”部分通过上面解释的 Sui Move 转移结构得到了极大的简化。
最后，没有必要引入像 ***CoolAssetStore*** 这样带有内部集合的包装类型——id 索引的 Sui 全局存储允许地址存储任意数量的给定类型的值。
  
另一个需要指出的区别是 Diem 风格的 ***cool_transfer*** 有 5 种中止方式（即，失败并在未完成传输的情况下向用户收费），而 Sui Move ***cool_transfer*** 只能以 1 种方式中止：
当 30 -day 政策被违反。
  
将对象所有权检查卸载到运行时不仅在人为体验方面，而且在安全方面都是一个巨大的胜利。 在运行时级别的安全实现可以防止通过构造实现这些检查（或完全忘记它们！）的错误。
  
最后，请注意 Sui Move 入口点函数签名 ***cool_transfer(asset: CoolAsset, ...)*** 如何为我们提供有关函数将要做什么的大量信息（与 Diem 样式的函数签名相反，后者更不透明 ）。
我们可以将此函数视为请求转移 ***CoolAsset*** 的权限，而另一个函数 ***f(asset: &mut CoolAsset, ...)*** 请求写入（但不转移）***CoolAsset*** 的权限，
而 ***g(asset: &CoolAsset, .. .)*** 只要求读取权限。
  
由于此信息可直接在函数签名中获得（无需执行或静态分析！），因此可以直接由钱包和其他客户端工具使用。
在 Sui 钱包中，我们正在处理人类可读的签名请求，利用这些结构化函数签名向用户提供 iOS/Android 样式的权限提示。
钱包可以说类似“此交易请求允许读取您的 ***CoolAsset***、写入您的 ***AssetCollection*** 并转移您的 ***ConcertTicket***。 继续？”。
  
人类可读的签名请求解决了许多现有平台（包括使用 Diem 风格的 Move！）上存在的大规模攻击向量，在这些平台上，钱包用户必须盲目地签署交易而不了解其影响可能是什么。
我们认为降低钱包体验的危险性是促进加密钱包主流采用的关键一步，并设计了 Sui Move 通过启用人类可读的签名请求等功能来支持这一目标。
  
## 捆绑异构资产
  
最后，让我们考虑一个关于捆绑不同类型资产的示例。 
这是一个相当常见的用例：程序员可能希望将不同类型的 NFT 打包到一个集合中，将要在市场上一起出售的商品捆绑在一起，或者将配件添加到现有商品中。
具体来说，假设我们必须遵循以下场景：
- Alice 定义了一个要在游戏中使用的 ***角色*** 对象
- Alice 希望支持使用后来创建的不同类型的第三方配件来装饰她的角色
- 任何人都应该能够创建配饰，但角色的所有者应该决定是否添加配饰
- 转移角色应该会自动转移其所有配件
  
这一次，让我们从 Sui 版Move 代码开始。 我们将利用 Sui 运行时内置对象所有权特性的另一个方面：一个对象可以为另一个对象所拥有。 每个对象都有唯一的所有者，但父对象可以有任意数量的子对象。 父/子对象关系使用 ***transfer_to_object*** 函数创建的，它是上面介绍的传递函数的兄弟。
``` rust
// in the Character module, created by Alice
struct Character has key {
  id: VersionedID,
  favorite_color: u8,
  strength: u64,
  ...
}
/// The owner of `c` can choose to add `accessory`
public entry fun accessorize<T: key>(c: &mut Character, accessory: T) {
  transfer_to_object(c, accessory)
}
// ... in a module added later by Bob
struct SpecialShirt has key {
  id: VersionedID,
  color: u8
}
public entry fun dress(c: &mut Character, s: Shirt) {
  // a special shirt has to be the character's favorite color
  assert!(character::favorite_color(c) == s.color, EBadColor);
  character::accessorize(c, shirt)
}
// ... in a  module added later by Clarissa
struct Sword has key {
  id: VersionedID,
  power: u64
}
public entry fun equip(c: &mut Character, s: Sword) {
  // a character must be very strong to use a powerful sword
  assert!(character::strength(c) > sword.power * 2, ENotStrongEnough);
  character::accessorize(c, s)
}
```
在此代码中，角色模块包含一个 ***accessorize*** 函数，该函数允许角色所有者添加具有任意类型的附件对象作为子对象。 这使得 Bob 和 Clarissa 可以创建自己的配件类型，这些配件类型具有 Alice 没有预料到的不同属性和功能，但仍以 Alice 已经完成的工作为基础。 例如，Bob的衬衫只有在角色最喜欢的颜色时才能装备，而Clarissa的剑只有在角色强大到可以使用它的情况下才能使用。
  
在 Diem 风格的移动中，不可能实现这种场景。 以下是一些失败的实施策略尝试：
``` rust
// attempt 1
struct Character {
  // won't work because every Accessory would need to be the same type + have
  // the same fields. There is no subtyping in Move.
  // Bob's shirt needs a color, and Clarissa's sword needs power--no standard
  // representation of Accessory can anticipate everything devs will want to
  // create
  accessories: vector<Accessory>
}
// attempt 2
struct Character {
  // perhaps Alice anticipates the need for a Sword and a Shirt up front...
  sword: Option<Sword>,
  shirt: Option<Shirt>
  // ...but what happens when Daniel comes along later and wants to add Pants?
}
// attempt 3
// Does not support accessory compositions. For example: how do we represent a 
// Character with Pants and a Shirt, but no Sword?
struct Shirt { c: Character }
struct Sword { s: Shirt }
struct Pants { s: Sword }
```
关键问题是在 Diem 风格的Move中：
- 仅支持同质集合（如第一次尝试所示），但配件基本上是异质的
- 对象之间的关联只能通过“包装”（即，将一个对象存储在另一个对象中）来创建； 但是可以包装的对象集必须预先定义（如第二次尝试）或以不支持附件组合的临时方式添加（如第三次尝试）。

## 总结

Sui 是第一个在使用 Move 方面与原始 Diem 设计大相径庭的平台。 设计充分利用 Move 和平台独特功能的嵌入既是一门艺术，也是一门科学，需要深入了解 Move 语言和底层区块链的功能。 我们对 Sui版本Move 取得的进步以及它将实现的新用例感到非常兴奋！
 
[1] 另一个支持 Diem 风格的 Move 政策“必须选择加入以接收给定类型的资产”的论点是，它是一种很好的垃圾邮件预防机制。
但是，我们认为垃圾邮件预防属于应用层。
与其要求用户发送花费真金白银的交易来选择接收资产，垃圾邮件可以通过丰富的用户定义策略和自动垃圾邮件过滤器在（例如）钱包级别轻松解决。
