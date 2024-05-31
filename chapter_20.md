# 20.轻松入门Sui Move: 对象的展示

SUI提供了一个模板引擎，可以使用它将对象的数据转换为模板对象定义的键值对，来实现对链上对象的链下展示管理。

下面我们通过一个例子来展示如何使用模板引擎，在这个例子中我们试图创建一个Phone对象的模板对象。

#### 生成Publisher

```rust
module test6::test6 {
     use std::string::{Self, String};
    use sui::package::{Self, Publisher};
    use sui::display;
    //需要展示的对象
    public struct Phone has key{
        id: UID,
        model: String,
        image_url: String,
        create_time: u64
    }
    //一次性见证者
    public struct TEST6 has drop {}
    fun init(otw: TEST6, ctx: &mut TxContext) {
        //生成Publisher对象
        let publisher = package::claim(otw, ctx);
        transfer::public_transfer(publisher, tx_context::sender(ctx));
    }
}
```

因为Display对象的创建需要Publisher权限,所以要先创建一个Publisher对象。创建Publisher的函数在源代码中定义如下：

```rust
module sui::package {
    public fun claim<OTW: drop>(otw: OTW, ctx: &mut TxContext): Publisher {
        assert!(types::is_one_time_witness(&otw), ENotOneTimeWitness);

        let tyname = type_name::get_with_original_ids<OTW>();

        Publisher {
            id: object::new(ctx),
            package: tyname.get_address(),
            module_name: tyname.get_module(),
        }
    }
}
```

claim函数主要作用是消费一次性见证者、生成Pulisher对象并返回。它的参数中包含一次性见证者，那就意味着这个函数只能在init函数中调用。init函数仅在包发布的执行一次，所以可以保证一个模块只会有一个Publisher对象，但是一个包可能包含多个Publisher对象。

除了claim函数外，还可以使用claim_and_keep函数创建并转交Publisher对象给当前上下文环境的账户(即包的拥有者)。

#### 创建Display对象

```rust
 	//创建模板
    public entry fun new_display(p: &Publisher, ctx: &mut TxContext) {
        	//定义模板的字段名
            let keys = vector[
            string::utf8(b"name"),
            string::utf8(b"link"),
            string::utf8(b"image_url"),
            string::utf8(b"model"),
            string::utf8(b"crete_time"),
            string::utf8(b"creator"),
        ];
		//定义模板的字段值
        let values = vector[
            string::utf8(b"{name}"),
            string::utf8(b"https://www.phone.com/phone/{id}"),
            string::utf8(b"ipfs://{image_url}"),
            string::utf8(b"{model}"),
            string::utf8(b"{create_time}"),
            string::utf8(b"some studio")
        ];
        //创建模板对象
        let mut display = display::new_with_fields<Phone>(
            p, keys, values, ctx
        );

        transfer::public_transfer(display, tx_context::sender(ctx));
    }
```

在生成Display对象之前，我们要定义展示的字段名和字段值。字段名数组和字段值数组都是字符串数组，两个数组个数必须一致否则会导致报错。字段值与其他模板语言类似，可以通过大括号“{}”来嵌入对象的字段值。我们使用display::new_with_fields函数来创建Display对象并指定它的键值对。也可以使用display::new函数创建空的Display对象。或者使用display::create_and_keep来创建Display空对象并将其转交给调用这个函数的账户。

一个包可能有多个Publisher, 其他模块的Publisher肯定没有权限创建Phone对象的模板对象,所以在创建Display对象的时候还需要验证创建Publisher的模块是否是Phone对象的模块，源码如下：

```rust
module sui::display { 
    //判断传入的Publisher对象是否有权限创建类型T的Display对象
    public fun is_authorized<T: key>(pub: &Publisher): bool {
        pub.from_package<T>()
    }
}
module sui::package {
	/// 判断泛型所属的包，是否与Publisher对象的package字段吻合
    public fun from_package<T>(self: &Publisher): bool {
        type_name::get_with_original_ids<T>().get_address() == self.package
    }
}
```

#### 修改和删除模板内容

display模块还提供了新增键/值对，修改键/值对和删除键/值对的方法，需要注意的是，修改和删除键值对之前要确保键值对已经存在于Display对象中，否则会产生报错。

在必要时候可以调用display::update_version升级Display的版本，并且它会发布一个版本升级的事件，监听这个事件的代码可以接到通知做相应处理。

#### 使用Display对象展示Phone对象

只需要在查询Phone对象的时候，使用 `{ showDisplay: true }` 选项就可以返回模板变量定义的内容:

```typescript
const rpcObject = await toolbox.client.getObject({
    id,
    options: {
        showBcs: true,
        showContent: true,
        showDisplay: true,//指定返回模板对象定义的内容
        showOwner: true,
        showPreviousTransaction: true,
        showStorageRebate: true,
        showType: true,
    },
});
```



参考资料：

https://docs.sui.io/standards/display

了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587

