## 12.轻松入门Move: 父子对象

我们在第八章讲对象的类别的时候，讲到独有对象，它只属于一个账户或者对象，只有这个账户或者对象可以对其进行访问、删除、修改和转交。这是因为Move并没有区分转交的目标是一个账户的地址还是一个对象ID。当一个对象被转交给另一个对象的时候，这两个对象就形成了父子关系，为了方便下面我统一将被转交的对象称之为子对象，作为owner的另一个对象称之为父对象。

如何创建父子对象，这个我们在上一章就讲过，只需要transfer给一个对象即可：

```rust
public struct Parent has key {
    id:UID,
    name: String,
}
public struct Child has key {
    id:UID,
    description: String,
} 
public entry fun new_parent(ctx: &mut TxContext) {
    transfer::transfer(Parent{
        id: object::new(ctx),
        name: string::utf8(b"hanmi"),
        }, tx_context::sender(ctx));
}
public entry fun new_child(parent_id: address, ctx: &mut TxContext) {
    transfer::transfer(Child{
        id: object::new(ctx),
        description: string::utf8(b"this is a description"),
        }, parent_id);
} 
```

但是要成功创建父子对象需要保证父对象的ID存在并且这个父对象不是不可变对象。因为后续需要对父对象进行可变引用，而不可变对象不允许这么做。

当对象属于一个账户地址的时候，只要使用对应账户的上下文就可以访问这个对象。但是如果对象属于另外一个对象，就只能通过这个对象访问了，如何通过这个对象访问呢？前面我们讲嵌套对象的时候讲到过如何通过一个对象访问另一个对象，是否可以沿用这个方法呢？

独有对象跟嵌套对象虽然都是一个对象属于另一个对象，但大相径庭，首先被嵌套的对象必须有store ability，而独有对象不用；其次被嵌套的对象是直接或者间接作为外层对象的一个字段，所以访问被嵌套对象只需要像访问外层对象的一个字段就可以实现被嵌套对象的访问。

但独有对象的值并不保存在对象中，自然无法像嵌套对象那样访问。Move提供了receive和public_receive方法。下面这个例子展示了如何接收子对象并修改它：

```rust
public entry fun modify_child(parent: &mut Parent, to_receive: Receiving<Child>, _: &mut TxContext) {
    let mut child = transfer::receive<Child>(&mut parent.id, to_receive);
    child.description = string::utf8(b"have changed");
    transfer::transfer(child, object::uid_to_address(&parent.id));
}
```

访问子对象的方法，必须对父对象使用可变引用，并且使用Receive结构体作为另一个参数。然后调用transfer::receive方法来返回子对象本身。修改完子对象后再次转交到父对象，就完成了子对象的修改。

那如何将Receive结构体值作为参数调用modify_child方法？我们直接使用子对象的ID作为参数传入即可，如下：

```bash
sui client call --package $PACKAGE --module $MODULE --function modify_child --args $PARENT_ID $CHILD_ID
```

上面的例子Parent、Child对象和所有方法都在同一个模块中，并且Child只有key ability没有store ability，所以我们使用receive方法。如果Child对象有store ability我们也可以在定义Child模块之外使用public_receive方法。现在我们尝试在定义子对象的模块外接收它，父子对象分别在不同模块定义：

父对象模块：

```rust
public struct Parent has key {
    id:UID,
    name: String,
}
public entry fun new_parent(ctx: &mut TxContext) {
    transfer::transfer(Parent{
        id: object::new(ctx),
        name: string::utf8(b"hanmi"),
        }, tx_context::sender(ctx));
}
public entry fun new_child(parent_id: address, ctx: &mut TxContext) {
    //创建父子对象关系，因为是在父对象模块，只能使用public_transfer
    transfer::public_transfer(child::new(ctx), parent_id);
}
public entry fun motify_child(parent: &mut Parent, to_receive: Receiving<Child>, _: &mut TxContext) {
    //接收子对象，因为是在父对象模块只能使用public_receive
    let mut child = transfer::public_receive<Child>(&mut parent.id, to_receive);
    //只能调用子对象的方法修改，不能在定义对象的模块外修改
    child::modify(&mut child);
    transfer::public_transfer(child, object::uid_to_address(&parent.id));
}
```

子对象模块：

```rust
//包含store ability才能在模块外转发，接收对象
public struct Child has key,store{
    id:UID,
    description: String,
}
//只能在定义对象的模块内，创建、修改对象。
public fun new(ctx: &mut TxContext):Child {
    Child{
        id: object::new(ctx),
        description: string::utf8(b"this is a description"),
    }
}
public fun modify(child: &mut Child) {
    child.description = string::utf8(b"have changed");
}
```

因为要在模块外接收它，子对象就必须有store ability。我们在前面章节讲过，对象的创建和修改，也都只能在定义它的模块中实现，所以在父对象模块中使用public_receive接收并修改子对象就只能调用子对象修改字段的方法，而不是直接修改。



如果我们想自定义接收规则，可以不给子对象store ability，在子对象模块内实现自定义接收方法，在方法内使用receive接收子对象。其他模块想接收子对象就只能调用这个接收方法以此保证遵循自定义接收规则。跟自定义transfer规则的机制相同，这里不再赘述。

我们前面说，父对象不能是不可变对象，那父对象可能是一个独有对象、共享对象或者被嵌套对象。因为子对象需要根据父对象先接收后访问，所以父对象的访问控制也会影响子对象的访问控制：

- 如果父对象是一个独有对象，那就只有在owner账户上下文中，可以通过对父对象可变引用来访问子对象
- 如果父对象是一个共享对象，那所有用户都可以访问子对象
- 如果父对象是一个被嵌套的对象，那就取决于外层对象的访问控制。



接下来我想介绍一种特殊的父子对象--灵魂绑定：可以从父对象中取出子对象访问，但是在这之后必须将子对象返还给父对象。代码如下：

```rust
struct Child has key {
    id: UID,
}
//归还子对象的清单,没有drop能力，不允许自动析构
struct ReturnReceipt { 
    object_id: ID, 
    return_to: address,
}
public fun get_object(parent: &mut UID, soul_bound_ticket: Receiving<Child>): (Child, ReturnReceipt) {
    let soul_bound = transfer::receive(parent, soul_bound_ticket);
    let return_receipt = ReturnReceipt { 
        return_to: object::uid_to_address(parent),
        object_id: object::id(&soul_bound),
    };
    (soul_bound, return_receipt)
}
public fun return_object(soul_bound: Child, receipt: ReturnReceipt) {
    let ReturnReceipt { return_to, object_id }  = receipt;
    assert!(object::id(&soul_bound) == object_id, EWrongObject);
    sui::transfer::transfer(soul_bound, return_to);
}
```

ReturnReceipt结构体是一个hot potato结构体,主要保存父子对象映射。get_object方法就是通过对父对象的可变引用获取子对象的方法。return_object方法则将取出的子对象又归还给父对象并销毁归还清单。

调用get_object方法不仅会返回子对象，还会返回一个归还子对象的清单。这个清单是ReturnReceipt结构体的实例，没有drop能力，不会自动析构，要想保证交易的成功，就必须再调用return_object去删除ReturnReceipt实例，以此保证在取出Child对象之后一定会返还，父子对象再次绑定，这就像绑定灵魂契约一次绑定永不改变！





了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587



