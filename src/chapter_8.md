## 8.轻松入门Move: 对象（下）

上一章我们简单概括了四种不同类型的对象，这一章将详细介绍每种对象的使用方法。期间可能会有些关于ability的内容，如果对ability不太熟悉的朋友，建议先看：9.轻松入门Move: Ability

#### 独有对象

**独有对象属于某一个账户地址或者某个对象ID，只有该账户地址（或对象）能访问，修改，删除和转交它。**

##### 创建方法

**创建一个对象后，使用transfer或者public_transfer把所有权转交给一个账户地址（或对象ID），那么这个对象就是独有对象**。

```rust
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
    //company就是一个独有对象
    transfer::transfer(company, tx_context::sender(ctx))
}
```

那transfer和public_transfer有什么区别呢？我们来看一下实现这两个方法的源码：

```rust
public fun transfer<T: key>(obj: T, recipient: address) {
    transfer_impl(obj, recipient)
}
//public_transfer要求被转交的对象具有key, store ability
public fun public_transfer<T: key + store>(obj: T, recipient: address) {
    transfer_impl(obj, recipient)
}
```

我们可以看到函数内的实现完全一样，只是不同函数对类型T的限定有区别：public_transfer方法还要求对象必须有store ability。

根据这个源码，我们得出以下结论：

- 无论是transfer还是pubic_transfer,都只能用于转交对象，没有key ability的非对象结构体不能使用。
- 如果对象没有store ability 就只能使用transfer

还有一个区别就是transfer方法只能在定义对象的模块内使用,public_transfer则模块内外均可使用。

总结来说就是：**transfer适用于没有store ability的对象，只能在定义它的模块内转交。 public_transfer则只能用于有store ability的对象，可以在模块内外转交**。

<font size=4>笔者做了一个小实验，使用transfer在模块内转交有store ability的对象，会触发警告，不影响运行。但不建议这么做。</font>

##### 使用场景

只要是在任何时间点都只有一个拥有者的对象，都应该使用独有对象。相比共享对象，独有对象没有数据争用的问题，将会有更快的速度，更小的成本和更高的吞吐量，所以独有对象应用尽用。

#### 不可变对象

**不可变对象不能被编辑、删除和转交。它不属于任何人，所有人都可以访问它**。

##### 创建方法

**使用freeze_object或者public_freeze_object就可以freeze对象，把对象变成不可变对象**。注意！这个过程是不可逆的。

```rust
//创建一个不可变对象
public fun new_contract(text:String, ctx: &mut TxContext) {
    transfer::freeze_object(Contract{
        id: object::new(ctx),
        text: string::utf8(b"hello world"),
    })
}
```

freeze_object和public_freeze_object的区别，跟public_transfer和transfer的区别一样，这里不再详解。

##### 访问方法

因为不可变对象不属于任何人，也不允许编辑，所以不可变对象只允许不可变引用。把对象本身作为参数或者使用可变引用都会引起报错，

```rust
//正确
public fun access_contract(c: &Contract, _: &mut TxContext)
//报错
public fun access_contract(c: Contract, _: &mut TxContext)
//报错
public fun access_contract(c: &mut Contract, _: &mut TxContext)
```

##### 使用场景

只要对象有不可改变，不能转移，不能删除的特性都应该使用不可变对象。不仅是因为它的特性，而且它也没有数据争用问题具有较高性能。一个典型的不可变对象就是我们发布的包。

#### 共享对象

**共享对象是使用share_object或者public_share_object函数的对象，它属于所有人，所有人都可以访问、修改、删除和转交它**。这里值得注意的是，所有人都可以访问共享对象不代表模块内外都能直接访问对象内字段，模块外对共享对象字段的访问也只能通过调用定义对象的模块提供的函数实现。所有人都可以转交共享对象也不意味着模块外一定能转交共享对象，模块外要能转交共享对象，要求对象有store abiity 并且使用public_share_object方法。

share_object和public_share_object的区别请看transfer和public_transfer

##### 创建方法

```rust
public fun new_platform(ctx: &mut TxContext): Admin {
    let platform = RentalPlatform {
        id: object::new(ctx),
        deposit_pool: table::new<ID, u64>(ctx),
        balance: balance::zero<SUI>(),
        notices: table::new<ID, RentalNotice>(ctx),
        owner: tx_context::sender(ctx),
    };

    transfer::share_object(platform);
}
```

##### 访问方法

```rust
//这三种都可以
public fun access_platform(c: &Contract, _: &mut TxContext)
public fun access_platform(c: Contract, _: &mut TxContext)
public fun access_platform(c: &mut Contract, _: &mut TxContext)
```

##### 使用场景

只有必要的时候才使用共享对象，因为所有人都能编辑，转交，删除可能会存在数据争用问题，为了解决数据争用会耗费更多时间和资源。

#### 被嵌套的对象

**被嵌套的对象被一个对象嵌套在内，成为外层对象的一部分，在链上不能独立存在，也无法使用对象ID直接访问。只能通过嵌套他的对象访问，修改或转移。**如果它又被转交给了一个账户地址，这个对象就转变成了独有对象，该账户的用户就可以通过对象ID直接访问了。

嵌套对象的Owner是外层对象ID，那是不是Owner是对象ID的都是嵌套对象呢？？？别忘了独有对象的owner也可能是对象ID，这里需要注意区分。

注意，非对象结构体也能被嵌套，嵌套时也要求结构体有store ability.但是本文讨论的是对象，非对象结构体暂不详谈。

嵌套的方式有三种，分别是直接嵌套，通过Option嵌套和通过vector嵌套。我们接下来依次探究每种嵌套方式。

##### 直接嵌套

###### 创建方式

**直接把一个对象作为另外一个对象的字段，这种方式就是直接嵌套。**

```rust
public struct Company  has key,store {
    id: UID,     
    //嵌套Person对象
    person: Person,
    can_be_transfered: bool,
}
public struct Person has key,store {
    id:UID,
    name: String,
}
```

注意，不能循环嵌套，也就是说A嵌套B,B不能嵌套A。

###### 访问方式

被直接嵌套的对象，不能使用对象ID访问(sui client object命令也不行)，也不能在任何函数调用中将其作为参数，唯一的访问方式就是通过访问外层对象。

```rust
public entry fun access_person(company: &Company,_: &mut TxContext) {
    //使用.一层一层访问
    let _ = company.person.name;
}
```

###### 解除嵌套

被嵌套的对象，可以取出转交给账户地址，转变成一个独有对象。这个过程称之为解除嵌套。方法如下：

```rust
//这里的company对象必须按值传递才能获取到person的对象
public entry fun transfer_person(company: Company, ctx: &mut TxContext) {
    let Company{
        id:id,
        person:person,
        can_be_transfered:can_be_transfered,
    } = company;
    let _ = can_be_transfered;
    //需要使用person对象值来转交
    transfer::public_transfer(person, tx_context::sender(ctx));
    //必须删除外层对象
    object::delete(id);
}
```

本段代码执行完，输出的Transaction Effects 模块，会有一个 Unwrapped Objects，展示的就是解开嵌套的对象。如下：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/unwrap_object.png?raw=true)

###### 应用场景

被嵌套的对象无法直接访问只有通过调用模块内特定的函数才能访问，在这个函数内我们可以设置访问的条件，以实现对对象的访问限制。

被直接嵌套的对象在解除嵌套的时候，必须删除外层对象；并且实例化外层对象的时候也必须同时实例化被嵌套的对象。直接嵌套适用于外层对象必须有嵌套对象的场景。比如公司必须有员工，没员工就会倒闭。

那有没有一种更灵活的方式，外层对象可能嵌套对象，可能没有嵌套对象，可以动态嵌套；解除嵌套也无需删除外层对象？通过Option嵌套就能轻松实现。

##### 通过Option嵌套

Option是标准库实现的一个结构体，含有copy,drop和store ability，内部包含一个可指定类型的数组字段。源码如下：

```rust
//在使用Option类型的时候，需要使用<>指定类型。这是泛型相关知识后续将会讲到
struct Option<Element> has copy, drop, store {
    vec: vector<Element>
}
```

那如何使用Option来嵌套对象呢？我们通过一个人和笔记本的例子说明：

###### 创建方式

有的人拥有笔记本，有的人则没有。所以我们可以声明一个Person对象，通过Option嵌套Notebook对象。我们先实例化一个没有笔记本的Person对象：

```rust
public struct Person has key {
    id: UID,
    name: String,
    notebook: Option<Notebook>,
}
//被嵌套的对象必须要有store ability
public struct Notebook has key,store {
    id: UID,
    brand: String,
    model: String,
}
public fun new(ctx: &mut TxContext) {
    transfer::transfer(Person{
        id: object::new(ctx),
        name: string::utf8(b"hanmeimei"),
        //可以实例化一个没有Notebook对象的Peroson对象
        notebook: option::none<Notebook>(),
        }, tx_context::sender(ctx));
}

```

后面这位名为hanmeimei的人购买了一个笔记本，我们使用option::fill方法将Notebook对象嵌入Person对象：

```rust
//嵌套Notebook对象
public fun fill_notebook(person: &mut Person, ctx: &mut TxContext) {
    //在将Notebook嵌入Person之前，确定Person没有嵌套Notebook。否则会报错
    assert!(option::is_none(&person.notebook), EOptionNotEmpty);

    let notebook = Notebook {
        id: object::new(ctx),
        brand: string::utf8(b"HUAWEI"),
        model: string::utf8(b"v10"),
    };
    //嵌套Notebook
    option::fill<Notebook>(&mut person.notebook, notebook);    
}		
```

###### 访问方式

访问被Option嵌套的对象也只能通过外层对象访问，可以使用option::borrow方法不可变引用notebook对象，也可以使用option::borrow_mut方法可变引用notebook对象：

```rust
public entry fun access_notebook(person: &Person, _: &mut TxContext) {
    let notebook_ref = option::borrow<Notebook>(&person.notebook);
    _ = notebook_ref.brand;
}
```

###### 解除嵌套

如果要转卖笔记本，则使用option::extract取出，并转交给其他人。

```rust
//解除嵌套
public entry fun unwrap_notebook(person: &mut Person, ctx: &mut TxContext) {
    //确认有嵌套，否则会报错
    assert!(option::is_some(&person.notebook), EOptionEmpty);
    //解除嵌套并转交给当前用户
    let notebook = option::extract<Notebook>(&mut person.notebook);
    transfer::public_transfer(notebook, someone);
}
```

###### 应用场景

虽然Option类型的唯一字段是数组类型，但是Option\<element\>只能最多只能包含一个对象。所以Option适合可能有一个嵌套或者没有嵌套对象的场景。那如果我想嵌套多个同类型对象怎么办？我们可以使用vector嵌套对象。

##### 通过vector嵌套

很多人可能没有笔记本，开发人员则可能有一个或者多个笔记本，我们使用vector来创建Person对象：

###### 创建方式

我们可以声明一个Person对象,notebooks字段申明为Notebook类型的数组。并实例化一个没有笔记本的Person对象：

```rust
public struct Person has key {
    id: UID,
    name: String,
    notebooks: vector<Notebook>,
}
public struct Notebook has key,store {
    id: UID,
    brand: String,
    model: String,
}
public fun new(ctx: &mut TxContext) {
    transfer::transfer(Person{
        id: object::new(ctx),
        name: string::utf8(b"hanmeimei"),
        notebooks: vector::empty<Notebook>(),
        }, tx_context::sender(ctx));
}
```

###### 访问方式

跟option类型类似，可以使用borrow方法不可变引用。不过因为是数组，在访问的时候需要指定索引，确实访问多个笔记本中的哪一个。

```rust
public entry fun access_notebook(person: &Person, index: u64, _: &mut TxContext) {
    //也可以使用vector::borrow_mut方法可变引用
    let notebook_ref = vector::borrow<Notebook>(&person.notebooks, index);
    _ = notebook_ref.brand;
}
```

###### 解除嵌套

使用vector::remove方法解除嵌套

```rust
//解除嵌套
public entry fun unwrap_notebook(person: &mut Person, notebook: &Notebook, ctx: &mut TxContext) {
    //确认有嵌套，否则会报错
    let (contains,index) = vector::index_of<Notebook>(&person.notebooks, notebook);
    assert!(contains, EEmpty);
    //解除嵌套并转交给当前用户
    let notebook = vector::remove<Notebook>(&mut person.notebooks, index);
    transfer::public_transfer(notebook, tx_context::sender(ctx));
}
```

###### 应用场景

vector方法的嵌套，适用于外层对象有零个或者多个同类型嵌套对象的场景。



 了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587
