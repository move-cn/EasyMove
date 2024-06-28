## 10.轻松入门Sui Move: 动态字段

在第八章我们讲被嵌套的对象的时候，举了一个例子：人(Person对象)可能拥有0个或者多个笔记本电脑，但在实际生活中，我们不仅仅可以拥有笔记本电脑，我们还可以拥有平板电脑、手机、台式机、游戏机等电子设备。那Person对象的定义将会变成如下：

```rust
public struct Person has key {
    id: UID,
    name: String,
    //拥有的笔记本
    notebooks: vector<Notebook>,
    //拥有的手机
    mobile_phones: vector<MobilePhone>,
    //拥有的游戏机
    game_consoles: vector<GameConsole>,
    //拥有的平板电脑
    ipads: vector<Ipad>,
}
public struct Notebook has key,store {
    id: UID,
    brand: String,
    model: String,
}
public struct MobilePhone has key,store {
    id: UID,
    brand: String,
    model: String,
    number: u64,
}
public struct GameConsole has key,store {
    id: UID,
    brand: String,
    model: String,
    games: vector<u8>,
}
public struct Ipad has key,store {
    id: UID,
    brand: String,
    model: String,
    size: u64,
}
```

Person对象嵌套了一堆对象，但事实上有的老年人没有任何电子设备，那这个实例化这个老人的Person对象就得带着四个空的对象数组，不仅没有任何意义还会消耗GAS。再者如果每出现一个新型电子设备，就得给Person对象增加一个字段，不仅麻烦，后面Person对象的定义会变得又臭又长，难以维护。如果达到嵌套对象数量的上限，甚至会影响业务实现。

有没有一种方法，可以让对象只嵌套需要的对象，不限名称不限类型，还可以动态的嵌套，动态解除嵌套对象？

这时候就要用到动态字段了。

现在Person对象的类型定义就可以去掉所有电子设备相关的字段：

```rust
public struct Person has key {
    id: UID,
    name: String,
}
```

定义Notebook，MobilePhone等电子设备对象：

```rust
//注意：Notebook作为动态字段的值，必须具有store ability
public struct Notebook has key,store {
    id: UID,
    brand: String,
    model: String,
    weight: u64,
}
```

person对象购入一台笔记本电脑，只需调用add方法，给Person对象增加一个动态字段：

```rust
public entry fun add_notebook(person: &mut Person, ctx: &mut TxContext) {
    let notebook = Notebook{
        id: object::new(ctx),
        brand: string::utf8(b"brand"),
        model: string::utf8(b"model"),
        weight: 12,
    };
    //给Person对象增加一个动态字段
    ofield::add<String, Notebook>(&mut person.id, get_notebook_name(), notebook);
}
```

get_notebook_name()方法是用于获取String类型的动态字段名，而字段值是新实例化的Notebook对象。

add方法可以将不同类型的电子设备对象都加入到Person对象的动态字段中，而无需修改Person的类型定义。



如果要转卖笔记本电脑，就使用remove方法从Person对象中删除：

```rust
public fun remove_notebook(person: &mut Person, buyer: address, _:&mut TxContext) {
    //如果动态字段中不存在，就停止运行
    assert!(ofield::exists_<String>(&person.id, get_notebook_name()), ENotExsitsInOfiled);
	//remove方法从动态字段中删除
    let notebook:Notebook = ofield::remove<String, Notebook>(&mut person.id, get_notebook_name());
	//转交给买家
    transfer::public_transfer(notebook, buyer);
}
```

我们使用add和remove就可以灵活的给Person对象增加/删除字段，是不是特别简单。

但是，动态字段有两种，一种是dynamic_object_field，另一种则是dynamic_field。他们之间有什么区别呢，各自使用场景是什么？我们一起看看源码中add方法的定义：

```rust
//dynamic_object_field模块
public fun add<Name: copy + drop + store, Value: key + store>(
    object: &mut UID,
    name: Name,
    value: Value,
) 
```

```rust
//dynamic_field模块
public fun add<Name: copy + drop + store, Value: store>(
    object: &mut UID,
    name: Name,
    value: Value,
) 
```

两个模块对动态字段的字段名有相同的要求：必须是带有copy,drop,store 能力的数据类型。那就包括所有的基础类型和带有这三个能力的非对象结构体。在第9章中阐述了为什么对象不能有copy和drop能力，不懂的小伙伴可以翻阅一下这里不再赘述。

add函数的区别主要在于字段值。dynamic_object_field模块要求字段值必须是带有store能力的对象，而dynamic_field模块的字段值可以是带有store能力的任何数据类型。

所以如果字段值是非对象类型，就只能使用dynamic_field模块，但是如果值是对象，如何选择使用哪种动态字段呢？

假设现在Person对象有两个动态字段，一个值是笔记本另一个则是手机。分别使用两种动态字段添加到Person对象中。笔记本的代码已经在上面代码块add_notebook定义，就不再重复。

```rust
public struct MobilePhone has key,store {
    id: UID,
    brand: String,
    model: String,
    number: u64,
}
public entry fun add_mobilephone(person: &mut Person, ctx: &mut TxContext) {
    let mp = MobilePhone{
        id: object::new(ctx),
        brand: string::utf8(b"brand"),
        model: string::utf8(b"model"),
        number: 13512354569,
    };
    //使用dynamic_filed模块的add方法
    field::add<String, MobilePhone>(&mut person.id, get_mobilephone_name(), mp);
}
```

发布合约，再分别调用add_mobilephone方法和add_notebook方法，这两个方法都会创建一个归属于Person对象的Field对象，

这个Field对象是用于保存动态字段键值对的对象，它在源码中的定义如下：

```rust
public struct Field<Name: copy + drop + store, Value: store> has key {
    /// Determined by the hash of the object ID, the field name value and it's type,
    /// i.e. hash(parent.id || name || Name)
    id: UID,
    /// 键
    name: Name,
    /// 值
    value: Value,
}
```

调用add方法新创建的两个Filed对象类型分别是：Field\<String, MobilePhone\>和Filed\<dynamic_object_field::Wrapper\<0x1::string::String>, object::ID\>。

这两个Filed对象的name字段类型不同，是为了让dynamic_object_field和dynamic_filed添加的动态字段的字段名区分开，避免产生键冲突。

使用dynamic_filed::add方法生成的Field对象，通过value字段直接嵌套了MobilePhone对象，那这个MobilePhone对象就只能通过Field对象进行访问，修改，删除和转移了。执行sui client object 命令也会报错不存在这个对象。

与此不同的是，dynamic_object_field:add对象生成的Field对象值是Notebook对象的ID，并没有嵌套Notebook对象，那就意味着外界依然可以访问Notebook对象。

所以对象选择哪种方式添加进动态字段，取决于被添加的对象是否需要被外界访问。



动态字段模块还为我们提供了borrow和borrow_mut方法来不可变引用和可变引用；exists_方法判断是否存在动态字段。

虽然是为Person对象添加的动态字段，但是删除Person对象并不会默认删除对象的动态字段！所以删除对象的方法里，应该先删除动态字段，再删除对象：

```rust
public entry fun delete_person(mut person: Person, _: &mut TxContext) {
    assert!(ofield::exists_<String>(&person.id, get_notebook_name()), ENotExsitsInOfiled);
    assert!(field::exists_<String>(&person.id, get_mobilephone_name()), ENotExsitsInOfiled);
	//删除notebook动态字段
    let Notebook{id: notebook_id, brand:_, model:_,weight:_} = ofield::remove<String, Notebook>(&mut person.id, get_notebook_name());
    object::delete(notebook_id);
	//删除mobilephone动态字段
    let MobilePhone{id: mobilephone_id, brand:_, model:_,number:_} = field::remove<String, MobilePhone>(&mut person.id, get_notebook_name());
    object::delete(mobilephone_id);
	//删除person对象
    let Person{id:id, name: _} = person;
    object::delete(id);
}
```



了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587







