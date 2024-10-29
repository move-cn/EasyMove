# 18.轻松入门Move:  设计模式

今天要介绍两个比较常见的设计模式烫手山芋和一次性见证者，烫手山芋可以保证函数调用完必须调用某类函数，而一次性见证者则可以保证函数只会执行一次。

## 烫手山芋（hot phtato）

烫手山芋是指那些没人任何能力的结构体。没有key能力不是对象，就无法转移给账户地址或者另一个对象；没有store能力就无法保存在对象中；没有copy能力就不能复制；没有drop能力就无法自动析构。拿到“烫手山芋”的方法如果不在定义“烫手山芋”的模块内就没有权限手动析构它，那就只有调用有能力处理“烫手山芋”的方法。也就是说“烫手山芋”可以作为一定会调用某类方法的一个保证。

烫手山芋的应用非常广泛，常见的例子就是闪电贷，闪电贷需要在同一个事务中借款和还款。为了保证借款后一定会还款，贷款方法会返回借款和一个“烫手山芋”，在同一个事务中必须调用还款的方法来将“烫手山芋”销毁掉，否则会导致也无法成功借款。

还有一个例子就是上一章我们讲解包的升级流程的时候，包的升级流程分为三步：授权、升级、提交升级，这三步缺一不可且必须在同一个事务中完成。为了保证流程的完整性，授权这一步会返回UpgradeTicket这个“烫手山芋”，来保证同一个事务中一定会调用升级命令，在升级完又会返回UpgradeReciept（也是“烫手山芋”）用来保证同一个事务会调用提交升级。

下面我们通过一个示例来演示如何实现“烫手山芋”的设计模式。我们需要实现购买手机的功能，分为两个步骤：支付和取手机。为了保证支付成功后一定会在同一个事务调用取手机的方法，会在支付方法返回一个支付凭证（也就是“烫手山芋”），只有取手机的方法实现了消费支付凭证的逻辑，所以必须调用支付之后必须调用取手机的方法。

```rust
module test6::test6 {
    use sui::coin::Coin;
    use test6::store;
    use std::string::String;
    use sui::sui::SUI;

    public entry fun buy_phone(model: String, coin: Coin<SUI>, ctx: &mut TxContext) {
        //支付成功拿到票据，必须取货，否则事务失败
        let br =  store::pay(model, coin);
        //取货
        transfer::public_transfer(store::pick_up_phone(br, ctx), tx_context::sender(ctx));
    }
}
```

值得注意的是，只有在定义支付凭证的模块外调用支付方法，才能让“烫手山芋”的模式生效，否则在定义支付凭证的模块内程序可以手动析构支付凭证，以此绕开“烫手山芋”模式。

在另一个模块，定义支付凭证、实现支付方法和取手机方法

```rust
module test6::store {
    use sui::coin::Coin;
    use std::string::String;
    use sui::sui::SUI;

    public struct Phone has key,store{
        id: UID,
        model: String
    }
    //支付凭证
    public struct BoughtReciept {
        model: String,
    }
    //支付
    public fun pay(model: String, coin: Coin<SUI>): BoughtReciept {
        /*处理支付相关逻辑
            ...
            
        支付成功
        */
        //返回支付凭证
        BoughtReciept{
            model: model,
        }
    }
    //取手机
    public fun pick_up_phone(bought_reciept: BoughtReciept, ctx: &mut TxContext): Phone {
  		//消费支付凭证
        let BoughtReciept{model:model} = bought_reciept;
        Phone{
            id: object::new(ctx),
            model: model,
        }
    }
}
```

## 一次性见证者（One-Time Witness）

一次性见证者是一种特殊的类型，可以保证在包的整个生命周期中最多有一个实例。它的主要作用是用于保证函数只会执行一次。

如果这个结构体跟模块名称一样且全部大写，有且仅有一个drop能力，没有字段或者只有一个bool类型字段那么这个结构体就是一次性见证者。可以通过sui::types::is_one_time_witness函数来判断是否是一次性见证者。

这个结构体不允许手动实例化这个结构体，只会在发布包的时候自动调用init函数的过程中生成实例，并作为参数传递给init函数。那什么是init函数呢？

### init函数

init函数是模块的初始化函数，一个模块只允许有一个init函数。仅在包发布的时候自动执行一次，后续不再执行即便升级包也不再执行，也不能手动调用它。

init函数必须满足以下特征：

- 函数名为init
- 参数列表的最后一个参数一定是TxContent的可变引用或者不可变引用
- 参数列表的第一个参数，可能是一次性见证者
- 没有任何返回

### 为什么需要一次性见证者?

刚接触一次性见证者的朋友们可能会觉得疑惑，init函数已经保证只会在发布的时候执行一次了，为什么还需要一次性见证者？

以coin模块的create_currency为例，这个函数用于创建一个货币类型，并返回新货币类型的发币权限和元数据对象。源代码如下：

```rust
module sui::coin {    
	public fun create_currency<T: drop>(
        witness: T,
        decimals: u8,
        symbol: vector<u8>,
        name: vector<u8>,
        description: vector<u8>,
        icon_url: Option<Url>,
        ctx: &mut TxContext
    ): (TreasuryCap<T>, CoinMetadata<T>) {
        // 保证类型T一定是一个一次性见证者
        assert!(sui::types::is_one_time_witness(&witness), EBadWitness);

        (
            TreasuryCap {
                id: object::new(ctx),
                total_supply: balance::create_supply(witness)
            },
            CoinMetadata {
                id: object::new(ctx),
                decimals,
                name: string::utf8(name),
                symbol: ascii::string(symbol),
                description: string::utf8(description),
                icon_url
            }
        )
    }
}
```

这个函数的第一个参数就是一次性见证者类型，需要witness是因为调用balance::create_supply(witness)函数需要，那我们再看balance::create_supply函数为什么需要一次性见证者。

```rust
module sui::balance {
    public fun create_supply<T: drop>(_: T): Supply<T> {
        Supply { value: 0 }
    }
}
```

事实是它的实现根本不依赖这个一次性见证者，甚至在参数栏就把它丢弃了！

换个角度，如果我们需要调用create_currency方法就必须传入witness实例，前面讲到witness的实例只会在init函数中出现，那就意味着create_currency方法只能在init函数中调用，也就保证了同一个模块这个方法只会调用一次！定义方法的时候只需要要求传入witness就能控制方法的调用次数和调用时机，这个设计模式可以说十分精妙！

我们以eth模块为例来介绍一次性见证者的使用方法：

```rust
module bridged_eth::eth {
    use std::option;

    use sui::coin;
    use sui::transfer;
    use sui::tx_context;
    use sui::tx_context::TxContext;
	//定义一次性见证者
    struct ETH has drop {}

    const DECIMAL: u8 = 8;
	//一次性见证者一定是init函数的第一个参数
    fun init(otw: ETH, ctx: &mut TxContext) {
        //保证coin::create_currency函数只会在这个模块调用一次
        let (treasury_cap, metadata) = coin::create_currency(
            otw,
            DECIMAL,
            b"ETH",
            b"Ethereum",
            b"Bridged Ethereum token",
            option::none(),
            ctx
        );
        transfer::public_freeze_object(metadata);
        transfer::public_transfer(treasury_cap, tx_context::sender(ctx))
    }
}
```



参考资料：

https://docs.sui.io/concepts/sui-move-concepts/init

https://docs.sui.io/concepts/sui-move-concepts/one-time-witness



了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587

