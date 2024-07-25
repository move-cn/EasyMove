## 11.轻松入门Move: Bag和Table

上一章我们讲到使用动态字段可以给Person对象动态添加电子设备的例子，因为无法直接获取Person对象的动态字段个数，在删除Person对象之前，具体应该删除多少个动态字段也是不确定的，所以其实特别容易漏删，造成资源浪费。

Sui框架基于dynamic_field实现Bag和Table对象，解决了这个问题。Bag是一个异构的映射集合，也就是说值是键值对形式，而且每对键值对的类型可以不同。Table也是一个映射集合，但是所有键值对的类型必须一致。这一点从名字也有体现，包（bag）里可以塞任何东西，表格（Table）则只能按条目填写。

基于dynamic_object_field则实现了ObjectBag和ObjectTable。他们与Bag、Table的区别跟dynamic_field和dynamic_object_field的区别一样，这里不再赘述。本章我们以Bag和Table为例，讲解如何使用“升级版”动态字段。

我们先看Bag和Table的定义：

```rust
public struct Bag has key, store {
    /// the ID of this bag
    id: UID,
    /// the number of key-value pairs in the bag
    size: u64,
}
public struct Table<phantom K: copy + drop + store, phantom V: store> has key, store {
    /// the ID of this table
    id: UID,
    /// the number of key-value pairs in the table
    size: u64,
}
```

Bag和Table对象只有两个字段，其中size字段用于记录键值对的个数。

有的朋友可能会疑惑，这两个对象不都是映射集合嘛，怎么没有保存键值对的字段？别忘了我们讲到这俩对象是通过dynamic_field实现的。他们的add方法实现如下：

```rust
public fun add<K: copy + drop + store, V: store>(bag: &mut Bag, k: K, v: V) {
    field::add(&mut bag.id, k, v);
    bag.size = bag.size + 1;
}

public fun add<K: copy + drop + store, V: store>(table: &mut Table<K, V>, k: K, v: V) {
    field::add(&mut table.id, k, v);
    table.size = table.size + 1;
}
```

往Bag对象中添加键值对，本质就是往Bag对象添加动态字段，键作为动态字段的名称，值作为动态字段的值。我们上一节讲到每次调用dynamic_filed::add方法，都会创建一个Field对象，现在这个Field对象跟Bag对象关联，所以调用dynamic_filed::add的时候，传入的是bag的id，并且Bag对象对动态字段的数量进行了管理，每次新增+1，每次删除动态字段减一。Table对象也是如此，不再赘述。

所以官网说，Bag和Table不像传统的映射集合在其中保存键值对，它们的键值对作为对象保存在Sui的对象系统中，而Bag和Table只提供处理键值对的方法。那提供了哪些方法呢？

我们还是延用上一章的例子讲解Bag用法，人（Person）可以拥有多个不同种类的电子设备，比如笔记本、手机等。

```rust
//人对象定义
public struct Person has key {
    id: UID,
    name: String,
    electronic_devices: Bag,
}
//笔记本对象定义
public struct Notebook has key,store {
    id: UID,
    brand: String,
    model: String,
    weight: u64,
}
//手机对象定义
public struct MobilePhone has key,store {
    id: UID,
    brand: String,
    model: String,
    number: u64,
}
```

Person对象新增一个名为电子设备的字段，字段类型是Bag对象。我们来实例化一个Person对象：

```rust
public entry fun new(ctx: &mut TxContext) {
    transfer::transfer(
        Person{
            id: object::new(ctx),
            name: string::utf8(b"hanmeimei"),
            electronic_devices: bag::new(ctx),
        }, tx_context::sender(ctx)
    );
}
```

这里调用了bag::new方法，实例化了一个没有任何内容的电子设备。

###### 添加键值对

假如现在购买了一个笔记本和一个手机，我们使用如下方法，给electronic_devices字段添加键值对：

```rust
//添加一个笔记本
public entry fun add_notebook(person: &mut Person, ctx: &mut TxContext) {
    let notebook = Notebook{
        id: object::new(ctx),
        brand: string::utf8(b"brand"),
        model: string::utf8(b"model"),
        weight: 12,
    };
    //键是vector<u8>类型，值是Notebook对象
    bag::add<vector<u8>, Notebook>(&mut person.electronic_devices, b"notebook_1", notebook);
}
public entry fun add_mobilephone(person: &mut Person, ctx: &mut TxContext) {
    let mp = MobilePhone{
        id: object::new(ctx),
        brand: string::utf8(b"brand"),
        model: string::utf8(b"model"),
        number: 13512354569,
    };
    //u8类型，值是MobilePhone对象
    bag::add<u8, MobilePhone>(&mut person.electronic_devices, 1, mp);
}
```

两个方法都调用bag::add方法，往electronic_devices字段添加了不同类型的键值对。添加完两个电子设备，我们可以直接通查看Person对象，就可以获取这个对象拥有的电子设备数量:

![](bag.png)

###### 访问键值对

跟其他类型一样，访问分为不可变访问和可变访问，使用哪个取决于是否需要改变键值对的值（注意这里只能改变值，不能改变键）。

```rust
public entry fun access_notebook(person: &Person, _: &mut TxContext) {
        assert!(bag::contains<vector<u8>>(&person.electronic_devices, b"notebook_1"), 1);
    	//不可变访问
        let notebook_ref = bag::borrow<vector<u8>, Notebook>(&person.electronic_devices, b"notebook_1");
        let _ = notebook_ref.brand;
    }
    public entry fun modify_notebook(person: &mut Person, _: &mut TxContext) {
        assert!(bag::contains<vector<u8>>(&person.electronic_devices, b"notebook_1"), 1);
        //可变访问
        let notebook_ref = bag::borrow_mut<vector<u8>, Notebook>(&mut person.electronic_devices, b"notebook_1");
        notebook_ref.brand = string::utf8(b"pingguo");
    }
```

在访问之前必须使用bag::contains判断是否存在该键值，否则会报错。使用bag::borrow方法会返回对Notebook对象的不可变引用，而bag::borrow_mut方法则是可变引用。

###### 删除键值对

```rust
public entry fun remove_notebook(person: &mut Person, _: &mut TxContext) {
    assert!(bag::contains<vector<u8>>(&person.electronic_devices, b"notebook_1"), 1);
    let Notebook{id:id,brand:_,model:_,weight:_} = bag::remove<vector<u8>, Notebook>(&mut person.electronic_devices, b"notebook_1");
    id.delete();
}
```

为避免报错，删除之前也需要使用bag::contains确定是否包含该键。调用bag::remove方法会返回Notebook对象本身，我们可以选择删除这个对象或者转交。

注意，这里我使用的是id.delete()来删除对象这是Move 2024新增用法，是不是比原来的object::delete(id)顺眼？

###### 删除Person对象

因为Bag是基于dynamic_field实现的，所以删除Person对象，也不会自动删除Bag内的键值对。所以删除Person对象之前也需要先删除键值对：

```rust
public entry fun delete_person(person: Person, _: &mut TxContext) {
    let Person{id:id, name: _, electronic_devices: mut electronic_devices} = person;

    assert!(bag::contains<vector<u8>>(&electronic_devices,  b"notebook_1"), 1);
    let Notebook{id:notebook_id,brand:_,model:_,weight:_} = bag::remove<vector<u8>, Notebook>(&mut electronic_devices, b"notebook_1");
    notebook_id.delete();

    assert!(bag::is_empty(&electronic_devices), 1);
    bag::destroy_empty(electronic_devices);
    id.delete();       
}
```

与上一章的删除Person对象不同，这里可以使用Bag提供的bag::is_empty方法判断是否已经删除所有的键值对，以此避免漏删。从这个角度来说我们应该尽量使用Bag而不是dynamic_field。



Table、ObjectTable、ObjectBag的用法都跟Bag一样，这里就不再赘述。总结来说Bag和Table其实只是一种在dynamic_field的基础上又封装了一层带有数量管理功能的对象。本文不仅仅介绍Bag和Table的用法，更是希望给读者展示Move如何通过装饰器模式扩展功能，希望读者能举一反三，开发出高效优美的Move代码。



了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587



