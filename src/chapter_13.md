## 13.轻松入门Sui Move:  事件和泛型

### 事件

学过设计模式的朋友们，应该都知道观察者模式，又叫做发布订阅模式（publish/subscribe）模式。

观察者模式定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。这个主题对象在状态发生变化时，会通知所有观察者对象，使它们能够自动更新自己。

使用Event模块将数据发送到链外就实现了观察者模式，今天我们要讲的就是如何使用Event发送主题来追踪链上活动。

Event模块源码非常简单，只包含一个方法：

```rust
module sui::event {
    /// Emit a custom Move event, sending the data offchain.
    ///
    /// Used for creating custom indexes and tracking onchain
    /// activity in a way that suits a specific application the most.
    ///
    /// The type `T` is the main way to index the event, and can contain
    /// phantom parameters, eg `emit(MyEvent<phantom T>)`.
    public native fun emit<T: copy + drop>(event: T);
}
```

从源码可以看到，发送的事件是包含copy和drop能力的任何类型。我们使用这个方法，就可以将数据发送到链外:

```rust
struct ObjectCreated has copy, drop {
   //...一些属性
}
event::emit(ObjectCreated {
    //一些属性赋值
});
```

我们自定义了一个包含copy和drop的结构体，在实例化这个结构体之后，通过emit方法将其发送到链外。

### 泛型

我们在前面章节，举了一个例子，一个人（Person对象）拥有多个电子设备，比如手机，笔记本等。我们可以使用Bag来保存人的电子设备。

```rust
public struct Person has key {
    id: UID,
    name: String,
    electronic_devices: Bag,
}
public entry fun add_notebook(person: &mut Person, notebook: Notebook, _: &mut TxContext) {
    bag::add<vector<u8>, Notebook>(&mut person.electronic_devices, b"notebook_1", notebook);
}
public entry fun add_mobilephone(person: &mut Person, mp: MobilePhone, _: &mut TxContext) {
    bag::add<u8, MobilePhone>(&mut person.electronic_devices, 1, mp);
}
```

add_notebook和add_mobilephone分别用于增加Person对象的笔记本和手机。除了笔记本和手机外还有Iwatch、Ipad、PC电脑等等电子设备，我们每增加一个电子设备的类型，都要增加一个add_方法来给Person对象新增电子设备。但实际上这两个方法除了参数类型不一样，实现都是无差别的。有没有一种方法可以使一个传入参数代表多种类型，从而让这个函数可以处理不同类型的传参？这就是我们今天要讲的泛型。泛型就是在函数或者结构体上新增一种特殊的参数——类型参数。在使用函数或者结构体的时候通过传入类型参数指定参数类型。

在函数中定义泛型的方法如下：

```rust
public entry fun add_electronic_device<T: store>(person: &mut Person, device: T, _: &mut TxContext) {
    bag::add<vector<u8>, T>(&mut person.electronic_devices, b"notebook_1", device);
}
```

在函数名称后<>内申明这个函数的参数device的类型，T只是一个占位符，可以是X也可以是Y,但是更多时候使用T代表一个类型参数。可以申明一个类型参数也可以申明多个类型参数，多个之间使用逗号隔开。也可以指定这个类型参数必须具有的能力，多个能力使用+号连接。

比如bag模块的add函数有K,V两个类型参数，并分别有不同能力限定：

```rust
public fun add<K: copy + drop + store, V: store>(bag: &mut Bag, k: K, v: V) {
    field::add(&mut bag.id, k, v);
    bag.size = bag.size + 1;
}
```

我们在调用这个函数需要使用尖括号指定参数device的类型，如下：

```rust
add_electronic_device<Notebook>(person, nt, ctx);
```

如果是使用命令行调用这个方法，则需要用到--type-args

```bash
sui client call --package $PACKAGE --module $MODULE --function add_electronic_device --args $PERSON $DEVICE --type-args "0xed4593bd4d24170af4eb6a52a13ca551d567297af55e003c52615cb467f41c74::person::Notebook" --gas-budget 10000000
```

--type-args选项后需要列举出这个函数的所有泛型，缺一个都会报错。值是由package_id::module::struct的结构组成。

除了在函数中使用泛型，结构体中也可以使用。比如我们前面讲到的动态字段中保存键值对的Field对象：

```rust 
/// Internal object used for storing the field and value
public struct Field<Name: copy + drop + store, Value: store> has key {
    /// Determined by the hash of the object ID, the field name value and it's type,
    /// i.e. hash(parent.id || name || Name)
    id: UID,
    /// The value for the name of this field
    name: Name,
    /// The value bound to this field
    value: Value,
}
```

方法跟函数中使用泛型类似，这里不再赘述。

泛型还有一种比较特殊的用法，申明一个类型参数但并不使用它，只是用于区分或者约束。但是如果申明了一个类型参数不使用，编译肯定会报错，如下：

```bash
warning[W09006]: unused struct type parameter
   ┌─ sources/test5.move:12:22
   │
12 │     public struct Yy<T> has copy, drop {
   │                      ^ Unused type parameter 'T'. Consider declaring it as phantom
   │
   = This warning can be suppressed with '#[allow(unused_type_parameter)]' applied to the 'module' or module member ('const', 'fun', or 'struct')
```

报错：存在一个未使用的类型参数T，考虑将其申明为phantom。像这种申明类型参数但不使用它的情况，应该将类型参数申明为“幻影”类型参数。

比如Sui Framework里的Coin对象，申明如下：

```rust
/// A coin of type `T` worth `value`. Transferable and storable
public struct Coin<phantom T> has key, store {
    id: UID,
    balance: Balance<T>
}
/// Storable balance - an inner struct of a Coin type.
/// Can be used to store coins which don't need the key ability.
public struct Balance<phantom T> has store {
    value: u64
}
```

 使用zero方法实例化一个Coin对象的时候也要指定幻影类型参数以表明Coin的货币类型：

```rust
//创建一个SUI币的Coin
let coin = coin::zero<SUI>();
```

coin对象包含Balance类型的字段，而Balance包含phantom类型参数，我们可以看到Balance结构体的定义中并没有使用类型T，这种时候就必须使用幻影类型参数了，这个幻影类型参数仅仅用于表明余额的货币类型。 Coin使用了Balance,那也必须申明幻影类型参数，否则会导致一个报错。

注意：使用幻影类型参数定义结构体或者函数，就不能再结构体或函数中使用它了。



了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587

