## 9.轻松入门Sui Move: Ability

在前面几章我们一直在说对象的ability，那什么是ability呢？ ability直译过来就是数据类型的能力。

Ability有四种，分别是key,store,copy,drop。基础数据类型和内建的数据类型的ability是默认的，不可修改的。他们默认有copy,drop,store这三种能力。结构体默认没有任何能力，但是我们可以自行设置结构体的能力。下面我主要讲解每种能力的含义和如何设置结构体的能力。

无论哪种ability，都是使用has关键字申明，具体如下：

```rust
//多个ability使用逗号隔开
public struct Person has key,store {
    id:UID,
    name: String,
}
```

#### Key Ability

有些资料说拥有key ability代表能在全局存储中作为key使用，这个并不适用于Sui Move。关于key ability的作用官网如下描述：

>On Sui, the `key` ability indicates that a struct is an object type and comes with an additional requirement that the first field of the struct has signature `id: UID`, to contain the object's unique address on-chain. 

翻译过来：**如果一个类型，带有key ability就代表他是一个对象，并且要求这个结构体的第一个字段必须是id:UID。**这个id字段包含了这个对象在区块链上的地址。

如果我们定义了一个结构体有key ability，但是没有id字段或者id字段没在第一位置，编译都会报错：有key ability第一字段就必须是类型为UID的id。如下：

```rust
 public struct Test3 has key {
     name: String     
}
```

```bash
public struct Test3 has key {
--- The 'key' ability is used to declare objects in Sui
name: String     
^^^^ Invalid object 'Test3'. Structs with the 'key' ability must have 'id: sui::object::UID' as their first field
```

所以**key ability就是用来标识结构体是否是对象的**。

#### Store Ability

key是对象必有的能力，而store则是对象可选的能力。**有以下两种情况需要指定对象的store abiity:**

- **1.当一个对象需要在定义他的模块之外被转交**
- **2.当 一个结构体需要被嵌套的时候**

如果你想限定某一个独有对象只能在定义它的模块内transfer,就无需予对象store ability。比如以下代码中的company对象，如果在定义他的模块外调用transfer方法，或者在命令行使用sui client transfer都会报错。

```rust
//没有store ability
public struct Company  has key {
    id: UID,     
    person: Person,
    can_be_transfered: bool,
}
public struct Person has key,store {
    id:UID,
    name: String,
}
```

如果你想限定某个对象只有满足特定条件的时候才能转交，就可以自定义transfer方法，并且限定只能在模块内transfer，这样。实现如下：

```rust
const ECanNotTransfer = 1;
//对象company没有store ability,只允许在定义对象的模块内transfer
public struct Company  has key {
    id: UID,     
    person: Person,
    can_be_transfered: bool,
}
public struct Person has key,store {
    id:UID,
    name: String,
}
//自定义transfer方法
public fun transfer_company(company: Company, someone: address) {
    //只有can_be_transfered字段为true才可以transfer，否则退出程序
    assert!(company.can_be_tra	nsfered, ECanNotTransfer);
    transfer::transfer(company, someone);
} 
```

如果是非对象结构体，想在对象中作为一个字段存储，也必须要有store能力。

#### Copy

与key ability相反，copy ability不能用于对象。copy ability 就是**用于标记这个结构体是否可以被复制**。

```rust
public struct Company  has key {
    id: UID,     
    person: Person,
    can_be_transfered: bool,
}
public struct Person has key,store {
    id:UID,
    name: String,
}
public entry fun new(ctx: &mut TxContext) {
    let person = Person {
        id: object::new(ctx),
        name: string::utf8(b"hanmeimei"),
    };
    let company = Company {
        id: object::new(ctx),
        person: person,
        can_be_transfered: false,
    };
    //使用关键词copy复制company对象
    let _company2 = copy company;
    transfer::transfer(company, tx_context::sender(ctx));
    transfer::transfer(_company2, tx_context::sender(ctx))
}
```

编译报错：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/copy.png?raw=true)

那我们是不是加上copy ability就可以顺利通过编译呢？？？我们加上之后继续编译，报错如下：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/coyp2.png?raw=true)

**如果要对一个结构体加上copy ability,那么这个结构体内所有字段都需要拥有该ability**然而对象Company的id字段不具有copy ability，而这个id字段是每个对象都有的字段，所以可以得出结论：**copy ability不能用于对象，只能用于非对象结构体**。

值得注意的是在对结构体设置copy 、store 和drop能力的时候，都需要先确保结构体内所有字段包含这些能力。

#### Drop

跟copy同理，drop ability也只能用于非对象结构体。drop表明**这个结构体是否能在作用域结束的时候自动删除**。如果不能自动删除则需要手动调用删除逻辑。删除结构体的方法详见：6.轻松入门Sui Move: 结构体



 了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587
