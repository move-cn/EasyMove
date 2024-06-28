## 7.轻松入门Sui Move: 对象（上）

有过后端编程经验的小伙伴会发现：无论什么语言核心其实都是对数据的增删改查，Sui Move也不例外，但是Sui Move并不会跟其他语言一样连接数据库、使用特定的数据库语言存储数据，而是使用**对象作为最小的存储单元**。也就是说如果你想持久化一些数据，先申明一个对象模型，使用要存储的数据实例化对象即可。如下：

```rust
//申明对象模型
struct Article has key {
    id: object::UID,
    title: string::String,
    content: string::String,
    word_cnt: u64,
}
public fun new(title: vector<u8>, content: vector<u8>, ctx: &mut tx_context::TxContext) {
 	let content = string::utf8(content);
    //实例化一个对象
    let article = Article{
        id: object::new(ctx),
        title: string::utf8(title),
        content: content,
        word_cnt: string::length(&content),
    };
	//对象设置为共享对象，所有用户皆可访问修改
    transfer::share_object<Article>(article);
}
```

在发布这段代码后，调用new方法就可以生成一个对象并返回对象的元数据，对象也保存在了链上。

**对象模型是一种特殊的结构体**，所以上一章讲到的结构体相关的用法对象也一样适用。那如何区分这是一个普通的结构体还是对象呢？**对象一定具有key能力且对象的第一个字段是全局唯一ID**  ，这个全局唯一id可以确定对象在链上的位置，所以也可以理解为就是对象的地址。对象的其他字段则可以是基础数据类型、对象或者非对象结构体。

第一章我们也讲过，在发布代码的时候会返回一个所有者为Immutable(不可改变的)的对象,这个对象的ID就是对应的包地址。这句话隐含了一个信息，就是**我们发布的包也是一个对象**。这个对象永远不可改变或删除。所以发布代码的过程也可以理解为是把代码存储到区块链的过程。

#### 对象的结构

使用sui client object <object id>就可以看到对象的完整信息，无论是包还是结构体对象都可以执行这个命令。结果如下：

结构体对象：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/object.jpg?raw=true)

包：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/package.jpg?raw=true)



我们可以把对象分为两部分：元数据和内容。

##### 公共元数据

以下是无论是对象模型还是包都有的公共元数据：

- 32个字节的全球唯一标识符（objectId），也就是对象ID，也可以说是对象的地址。
- 8个字节的版本号(version)，每次修改对象都会加一。
- 32个字节的交易的摘要（prevTx），记录最后一次输出这个对象的交易。
- 33个字节的owner字段，对象的所有者，也可以根据这个字段推测如何访问这个对象。
- objectType则指明对象类型，值可能是package可能是自定义的结构体类型。
- digest字段是这个对象元数据和内容的哈希，也就是对象的摘要。
- storageRebate字段表示，如果这个对象后期在链上删除，将会返还的gas值。

##### 对象内容

对象内容就一个content字段，它里面的dataType字段用于区分是结构体对象（moveObject）还是包(package)。content字段的其他内容则因类型的不同而各有区别了。

###### 结构体对象

- type字段：表明结构体类型
- hasPublicTransfer字段：是否能使用publish_transfer转移所有权，上图这个对象因为是共享对象所以不能被转移所有权，值为false。
- fields字段就是对象的键值对，使用BCS(Binary Canonical Serialization)编码。我们可以在sui client object命令中指定--bcs选项来查看编码后的值。

###### 包

Move Packages包含包中的代码。查看上图可以发现引用包的名称已经自动变成了包的地址。

#### 删除对象

我们上面说对象就是一种特殊的结构体，按理说上一章的销毁实例应该也适用于对象吧？我们按照上一章讲解的方法定义drop函数：

```rust
public fun drop(a: Article) {
    let Article{id:_,title:_,content:_,word_cnt:_}=a;
}
```

编译的时候直接报错：id字段Cannot ignore values without the 'drop' ability. The value must be used。id字段的类型是sui::object::UID，这个结构体类型没有drop的能力，所以不能丢弃id字段值。好在sui::object模块提供了一个删除UID的方法，也是删除对象id的唯一方法，如下：

```rust
//注意：要删除对象，必须按值传入
public fun drop(a: Article) {
    let Article{id:id,title:_,content:_,word_cnt:_}=a;
    object::delete(id);
}
```

我们发布合约后调用这个方法删除对象。在浏览器查看本次交易的费用明细可以发现：本次交易给我们返还了0.000351964 SUI！删除对象释放存储空间就会返还一部分gas，那我们在编码过程中应该积极删除无用对象，以减少gas消耗！

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/delete_object.png?raw=true)

#### 对象的分类

根据对象的所有者和访问权限的不同，可以将对象分为以下几类：

- 独有对象

  这种对象的owner字段值是账户的地址或者是对象的ID。它只能属于某一个地址，可以切换对象的所有者。也只有所有者能访问，修改，转交它。

- 共享对象

  这种对象的owner字段值带有Shared标记。这种对象属于所有人，对所有人开放访问，修改的权限。

- 不可变对象

​	 这种对象的owner字段值是Immutable,一旦创建不能修改和转交，但是对所有人开放访问权限。 我们每次发布包就会返回一个不可变对象，所有人都可以访问这个包但包一经发布不可修改。	

- 被嵌套的对象

​		这种对象的owner字段值是嵌套对象的地址,。它被一个对象嵌套在内，在链上不能独立存在，也无法使用对象ID直接访问。只能通过嵌套他的对象访问，修改或转移。如果转移给了一个账户地址，该账户的用户就可以通过对象ID直接访问了。

本节我们只简单介绍一下每种对象的特征，下一节我们将详解讲解如何创建，访问和转交这些对象。



 了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587

