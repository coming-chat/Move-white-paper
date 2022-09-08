
# 代码审计师眼中的Move 语言

![](https://github.com/coming-chat/Move-white-paper/blob/main/auditor1.jpeg)
### 本文 讨论Move 语言的 类型系统 和形式化验证

作为我们工作的一部分，我们寻求了解如何消除漏洞类别。设计更安全的语言使开发人员能够自信地编写代码。 Move 究竟如何适用于更安全的编程实践？我们可以从 Move 中学到什么来概括其他执行环境的安全设计原则？

最近，似乎有很多流行语在流传。形式验证、基于类型的安全性、“rust或许适用于区块链”。

在这篇文章中，我将试图确切地讨论 move 是如何使自己更安全的编程实践、潜在的缺点以及为希望构建结构上更安全的程序的协议开发人员提供的实用设计技巧。

## 类型

Move 的主要卖点之一是使用类型化的资源。 但Aptos 和 Sui 在具体化这种模式的方式上略有不同，以 coin.move 为例。

``` rust
  /// Main structure representing a coin/token in an account's custody.
  struct Coin<phantom CoinType> has store {
      /// Amount of coin this address has.
      value: u64,
  }
```
***Aptos***
``` rust
  /// A coin of type `T` worth `value`. Transferable and storable
  struct Coin<phantom T> has key, store {
      id: UID,
      balance: Balance<T>
  }
```
***Sui***
``` rust
    /// Liquidity pool with reserves.
    struct LiquidityPool<phantom X, phantom Y, phantom LP> has key {
        coin_x_reserve: Coin<X>,
        coin_y_reserve: Coin<Y>,
        // ...
    }
```
这具有在编译时对齐类型信息的优点。不小心将错误类型的Coin传递给函数是很困难的。
``` rust
      public fun mint<X, Y, LP>(
          pool_addr: address,
          coin_x: Coin<X>,
          coin_y: Coin<Y>
      ): Coin<LP> acquires LiquidityPool, EventsStore {
          // ...

          let (x_reserve_size, y_reserve_size) = get_reserves_size<X, Y, LP>(pool_addr);
```

顺便说一句，这个通用类型信息是在运行时在 vm 级别的 ty_args 中实现的。这种 VM 级别的实现选择使得迭代任意泛型类型变得相当困难，例如对池中的Coin求和。我们将很快发布对 move 的 VM 内部结构的深入介绍。

在伪代码中，这会检查 coin_x.type 是否等于 pool.x_type，并且 coin_y.type 是否等于 pool.y_type。

这种类型的系统有两个优点
- 这是必需的。必须指定类型参数，这样就不可能忘记这样的约束
- 很简洁。约束是通过类型参数对齐而不是冗长的等价检查来完成的

然而，这个系统并不完美。

事实上，我甚至会争辩说使用类型来创建这样的关联是一种反模式。

仅使用类型来强制关系有效，因为类型与实例唯一关联。例如，在 Aptos 的Coin初始化函数中，他们明确断言之前没有初始化过的 ***CoinInfo<CoinType>***

``` rust
    fun initialize_internal<CoinType>(
      // ...
  ): (BurnCapability<CoinType>, FreezeCapability<CoinType>, MintCapability<CoinType>) {
      // ...

      assert!(
          !exists<CoinInfo<CoinType>>(account_addr),
          error::already_exists(ECOIN_INFO_ALREADY_PUBLISHED),
      );
```
虽然这个 CoinInfo 不是直接返回的，但它仍然确保了能力对象的唯一性。
  
Aries Markets的 ReserveCoinContainer 结构存储用于管理借贷市场的所有相关数据和资源。
  
``` rust
    /// The struct to hold all the underlying `Coin`s.
  /// Stored as a resources.
  struct ReserveCoinContainer<phantom Coin0> has key {
      /// Stores the available `Coin`.
      underlying_coin: Coin<Coin0>,
      /// Stores the LP `Coin` that act as collateral.
      collateralised_lp_coin: Coin<LP<Coin0>>,
      /// Mint capability for LP Coin.
      mint_capability: MintCapability<LP<Coin0>>,
      /// Burn capability for LP Coin.
      burn_capability: BurnCapability<LP<Coin0>>,

      // ...
  }
```
  创建 ***ReserveCoinContainer*** 时，通过将其移动到硬编码地址来隐式强制执行唯一性。
  ``` rust
    public(friend) fun create<Coin0>(
      lp_store: &signer,
      // ...
  ) acquires Reserves {
      lp::assert_is_lp_store(signer::address_of(lp_store));

      // ...

      move_to(lp_store, ReserveCoinContainer<Coin0> {
        // ...
      });
  ```
在这两种情况下，类型关联仅起作用，因为我们为每种类型创建了一个实例。
另一方面，考虑您是否有一个 Position<T> 和一个 Market<T>，其中 T 是Coin类型。
  
``` rust
      struct Market<phantom T> {
        reserves: Coin<T>,
        // ...
    }

    struct Position<phantom T> {
        amount: u64,
        // ...
    }
  ```
如果 Market<T> 不是唯一类型——或者换句话说，如果你能够为每个类型 T 创建多个市场实例——你可能会为给定头寸传入不正确的市场。这是 Solana 上常见的漏洞模式。
  
类型的动态迭代也是不可能的（至少目前由 Move VM 设计）导致开发人员非常头疼。在这些场景中，我们凭经验观察到开发人员默认返回类型反射 API，使代码不必要地复杂化。以牺牲可用性为代价的安全是以牺牲安全为代价的。
  
``` rust
      /// Get the price of the token per lamport.
    public fun get_price(type_info: TypeInfo): Decimal acquires Oracle {
        let oracle = borrow_global_mut<Oracle>(@oracle);
        let price = table::borrow_mut_with_default<TypeInfo, Decimal>(
            &mut oracle.prices,
            type_info,
            decimal::one()
        );
        *price
    }
```
类型关联感觉就像是预期模式的代理——将资源与实例相关联。能够存储对另一个资源实例的引用非常有用（这在 Diem 风格的Move中是可能的）。

总之，当使用类型系统将资源相互绑定时，重要的是
- 为您的资源提供独特的初始化程序
- 直接将资源与实例关联

## 形式验证

形式验证是另一个令人兴奋的功能。
  
作为我们协议工作的一部分，我们积极使用形式验证来证明安全性的各个方面。
  
然而，这不是灵丹妙药。关键是弄清楚要证明什么。
  
一个明显的想法可能是跨特定函数的属性。例如，我们可能希望确保交换不会降低池子的价值——类似于我们报告的 [Solana AMM](https://osec.io/blog/reports/2022-04-26-spl-swap-rounding/) 舍入问题。
  
然而，这也可以通过一个简单的运行时断言来检查。例如，我们建议 Pontem 断言流动性池代币价值正在严格增加。

``` rust
  let cmp = u256::compare(&lp_value_after_swap_and_fee, &lp_value_before_swap_u256);
  assert!(cmp == 2, ERR_INCORRECT_SWAP);
```
当我们证明函数之间的关系时，Move验证器真的很出色。
无法通过断言轻松证明的更复杂关系的一个示例是Move存储库中的 no_free_money_theorem 函数。
``` rust
  // #[test] // TODO: cannot specify the test-only functions
  fun no_free_money_theorem(coin1_in: u64, coin2_in: u64): (u64, u64) acquires Pool {
      let share = add_liquidity(coin1_in, coin2_in);
      remove_liquidity(share)
  }
  spec no_free_money_theorem {
      pragma verify=false;
      ensures result_1 <= coin1_in;
      ensures result_2 <= coin2_in;
  }
```
没有明确的方法可以用断言来表达这一点，因为这会在时间上分离的两个函数之间进行观察。
  
不变量也非常有用。例如，对费用参数（费用永远不会超过 100%）或池子的供应量强制执行不变量可以更容易地对协议进行推理。
  
例如，Ian 使用不变量来清楚地定义他的 AMM 状态的核心属性。
  
``` rust
spec PoolState {
    invariant supply >= MINIMUM_LIQUIDITY;
}
```
Move 验证器的另一个有用模式是 ***aborts_if***。更具体地说，用 ***aborts_if*** 为 ***false*** 断言函数永不中止会很有帮助。

尽管循环不变量有点笨拙，但 Ian 也能够证明一个相对不平凡的函数不会中止。
  
``` rust
    fun multiply_vec_by_n_coins(input: vector<u64>): vector<u128> {
      let amounts_times_coins = vector::empty<u128>();
      let i = 0;
      let n_coins = vector::length(&input);
      while ({
          spec {
              invariant len(amounts_times_coins) == i;
              invariant i <= n_coins;
              invariant forall j in 0..i: amounts_times_coins[j] == input[j] * n_coins;
          };
          (i < n_coins)
      }) {
          vector::push_back(
              &mut amounts_times_coins,
              (*vector::borrow(&input, (i as u64)) as u128) * (n_coins as u128)
          );
          i = i + 1;
      };
      spec {
          assert i == n_coins;
          assert len(input) == n_coins;
      };
      amounts_times_coins
  }
  spec multiply_vec_by_n_coins {
      pragma opaque;
      aborts_if false;
      ensures len(result) == len(input);
      ensures forall j in 0..len(input): result[j] == input[j] * len(input);
  }
```

## 总结
在这篇文章中，我们探讨了 Move 的类型系统和形式化验证的含义，这是 Move 语言的两个强大功能，可以实现更安全的编程语言。
  
虽然 Move 作为一种语言仍然是一种积极开发的语言，但它显示了一些令人兴奋的特性，似乎允许开发人员创建结构上更安全的程序。
  
如果您有任何想法，我很乐意讨论更多。请在 Twitter 上给我发消息@notdeghost。
