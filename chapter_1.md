# 1.轻松入门Sui Move: 快速了解基本概念

2009年，中本聪发布比特币的开源软件，实现了去中心化的数字货币系统，开启了区块链1.0的时代。

比特币区块链虽然为去中心化的数字货币提供了一个安全的基础，但其功能受到了限制，只能用于简单的价值交换。有没有一种机制可以在链上编写复杂的代码实现区块链智能化，让区块链能应用于更多场景呢？

2015年以太坊主网正式上线，为这个问题交出了答卷。也开启了区块链2.0的时代。以太坊为去中心化应用提供一个通用的智能合约平台，使得开发者可以开始编写和部署智能合约，通过智能合约为区块链技术的应用提供了更广泛的可能性。

下图为区块链2.0的逻辑架构图，展示了智能合约在区块链中的位置：

![](https://raw.githubusercontent.com/Crazyjs123/crazyjs123.github.io/main/pic/%E5%8C%BA%E5%9D%97%E9%93%BE%E9%80%BB%E8%BE%91%E6%9E%B6%E6%9E%84%E5%9B%BE.png)



区块链为智能合约提供执行环境，而智能合约为区块链扩展功能。智能合约赋予了区块链智能的特性，在区块链中智能合约的作用如同一个智能助理，对区块链中的数据和事件按照预先设定的逻辑进行处理。它作为一个在区块链上可以自动执行的计算机程序，可以处理信息，接收、发送和存储资产。

智能合约部署到区块链中，作为区块链的一部分，自然具有区块链不可篡改的特点；部署后也会作为区块在区块链网络中广播，每个节点都会保存一份，所以分布式保证了它的高可靠性；智能合约在满足一定条件后会自动执行，无需人工触发，更不需要三方担保。以上这些特点使得基于智能合约的交易更加安全，高效和低成本。

智能合约具有这么多优秀的特质，那我们如何编写它呢？就不得不说到Move语言了。Mysten Labs联合创始人兼首席技术官Sam Blackshear为Diem区块链开发了Move，不过Move旨在成为一个跨平台的嵌入式语言，可不是只能在Diem区块链平台运行，当然也可以在Sui区块链网络中运行, 这就是Sui Move。

Sui Move准确来说应该是Move On Sui。是在Sui区块链平台上运行的原生语言。开发人员使用它可以创建、管理和操作数字资产，并编写智能合约。Sui Move 引入了以对象为中心的数据存储模型，这使得Sui可以并行处理事务，比串行事务的区块链具有更高性能。从开发的角度，Sui Move 也无需在交易前后做大量关于资产所有权的处理；针对对象的处理也非常简单灵活。

接下来让我们使用一个简单的例子来演示一下，使用Sui Move如何操作数据对象，如何在区块链上部署以及如何运行。需要注意的是，初学者先不要陷入细节，只需跟着我的例子，一探Sui Move的宏观即可。

```Move
module test::test {
    use sui::object::{Self, UID};
    use sui::tx_context::{Self, TxContext};
    use sui::transfer;
    use std::string;
	//定义一个博客结构体
    struct Blog has key{
        id: UID,
        content: string::String,
        like_cnt: u64,
    }
    //此函数用于发布博客，每次发布完成会返回博客对象信息
    public entry fun publish_blog(content: string::String, ctx: &mut TxContext){
         transfer::transfer(Blog {
            id: object::new(ctx),
            content: content,
            like_cnt: 0
        }, tx_context::sender(ctx));
    }
    //调用此函数，增加博客对象的点赞数
    public entry fun like_blog(blog: &mut Blog){
        blog.like_cnt = blog.like_cnt + 1;
    }
}
```

此合约包含两个函数，一个是publish_blog根据传入的content参数，新建一个Blog对象，并把这个Blog对象所有权赋给调用函数的用户地址。另一个则是like_blog修改博客对象的属性，将指定的Blog对象的点赞数加1。

我们如何将这段代码发布到Sui区块链网络上呢？只需要使用Sui Move命令行工具的publish命令即可。需要注意的是，这个操作会消耗gas(gas这里暂不多做介绍，我们简单理解为付费的一种形式)

```bash
sui client publish --gas-budget 100000000
```

执行完这个命令后，会返回一个所有者为Immutable(不可改变的)的对象，这个对象的ID就是这个代码在区块链的地址。拿着这个地址，指定模块名和函数名，就可以在区块链上调用publish_blog函数：

```bash
sui client call --package <合约地址> --module <合约模块名> --function publish_blog --args "this is a blog" --gas-budget 100000000
```

建好的对象会保存在区块链中，并返回一个对象ID。我们使用对象ID就可以查询到刚新建的Blog对象

```bash
sui client object <对象ID>
```

Web2.0的开发者可能对此感到惊奇，数据存储的过程没有连接数据库的操作，更没有繁琐的SQL语句执行。我们只需要新建一个对象指定所有权就保存了一个对象。是不是非常简单？

接下来我们继续调用like_blog函数

```bash
sui client call --package <合约地址> --module <合约模块名> --function publish_blog --args <Blog对象ID> --gas-budget 100000000
```

通过sui client object 命令查看对象可以发现：虽然我们没有显式的对存储做任何操作，但是Blog对象的like_cnt属性值已经加1。

现在我们对Sui Move有了大致的了解，得出了Sui Move 不仅安全、性能高、开发也简单灵活的结论。那我们可以使用它做些什么呢？

1.创建智能合约来管理和转移各种类型的数字资产，包括加密货币、代币化资产、非同质化代币（NFT）等

2.Sui Move可以用于编写智能合约，实现去中心化交易、借贷等金融服务；也可以利用智能合约实现去中心化的投票和治理机制；还可以用于NFT和数字艺术领域，确保数字内容的版权和所有权。

3.Sui Move可以用于创建数字身份认证系统，使用户能够安全地管理和共享他们的身份信息，同时保护个人隐私。



了解更多Sui Move内容：
- telegram: t.me/move_cn
-  QQ群: 79489587
