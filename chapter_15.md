## 15.轻松入门Sui Move:  集合（下）

除了上一章提到的vector和PriorityQueue类型是集合外，还有我们讲动态字段的时候讲到的bag(object_bag)、table(object_table)、dynamic_filed(dynamic_object_field)等，这些已经讲过我们不再赘述。这一章我们将探究Move给我们提供了哪些额外实用的集合结构体。

### LinkedTable

Table用于保存同类型的数据，并且它动态字段的基础上增加了动态字段数量的管理，以此保证动态字段的删除不会遗漏。但是要访问或者删除动态字段也只能按照动态字段名逐个操作无法使用迭代。linked_table则完善了这个缺点。它不仅管理动态字段的数量，还把每个动态字段封装成Node结构体然后像链表一样把它们“串”起来。从表头或者表尾开始访问就能访问到所有的动态字段。

```rust
public struct LinkedTable<K: copy + drop + store, phantom V: store> has key, store {
    /// the ID of this table
    id: UID,
    /// the number of key-value pairs in the table
    size: u64,
    /// the front of the table, i.e. the key of the first entry
    head: Option<K>,
    /// the back of the table, i.e. the key of the last entry
    tail: Option<K>,
}
public struct Node<K: copy + drop + store, V: store> has store {
    /// the previous key
    prev: Option<K>,
    /// the next key
    next: Option<K>,
    /// the value being stored
    value: V
}
```

每个节点不仅保存了值还保存了访问前后节点所需的键名。值得注意的是这时候就不是V作为动态字段的值了，而是Node类型作为值，K作为键名添加到linked_table中。而linked_table包含head字段和tail字段用于指示链表头和链表尾，让我们可选择从head开始使用next指针逐个正序访问，也可以使用tail开始使用prev指针逐个倒序访问。

Move给我们提供了一系列的方法来操作这个链表,包括表头插入、表尾插入、表头弹出、表尾弹出、获取前后节点K值等。就不再一一讲解，我们用下面这个例子来演示如何对其进行迭代访问和迭代删除：

```rust
use sui::linked_table::{Self, LinkedTable};
//创建一个linked_table
public entry fun new(ctx: &mut TxContext) {
    transfer::public_transfer(linked_table::new<u8, u8>(ctx), tx_context::sender(ctx));
}
//添加字段
public entry fun add(table: &mut LinkedTable<u8, u8>, k:u8, v:u8, _: &mut TxContext) {
    linked_table::push_front<u8, u8>(table, k, v);
}
//迭代删除所有节点并删除linked_table
public entry fun remove_all(mut table: LinkedTable<u8, u8>, _: &mut TxContext) {
    let lt = &mut table;
    while (!lt.is_empty<u8, u8>()) {
        let (_,_) = lt.pop_front<u8, u8>();
    };
    linked_table::destroy_empty<u8, u8>(table);
}
```

如上例所示，删除动态字段值不需要根据键名删除，可以迭代删除全部字段，无任何遗漏。访问和修改也都可以从链表头（或者尾）迭代访问而无需使用键名。

值得注意的是同样的数据量下使用table存储会比使用linked_table存储更节省gas,所以建议大家只有在需要linked_table的链表功能的时候才使用，其他时间尽量使用table。

### TableVec

在Table基础上实现了vector的功能。我们可以像使用vector一样使用它，就连提供的方法都差不多。源码：

```rust
public struct TableVec<phantom Element: store> has store {
    /// The contents of the table vector.
    contents: Table<u64, Element>,
}
/// Add element `e` to the end of the TableVec `t`.
public fun push_back<Element: store>(t: &mut TableVec<Element>, e: Element) {
    let key = t.length();
    t.contents.add(key, e);
}
```

将Table的键名类型设置为u64，每次添加元素都将键名加一，以此将Table包装成vector。除此之外还提供了跟vector一样的borrow_mut、borrow、pop_back、length等。但是没有index_of、contains、insert等方法。

值得注意的是经过实验证明，同样的数据用TableVec将会耗费更多gas。可是为什么要创建一个跟vector差不多功能，还更耗费gas的类型呢？这是因为数据量非常大的时候vector可能会有达到上限的情况，在这时候就可以使用TableVec来保存数据。

### VecMap

这是一个基于vector实现的映射表。这个映射表保证不会有重复的键名，我们可以根据键名查询，删除，修改值。但是根据键名访问值并不像真正的映射表那样 时间复杂度为O(1)。VecMap根据键名访问值的时间复杂度是O(n)!

究其原因，其实是它底层实现并不是真的映射表。实现如下：

```rust
public struct VecMap<K: copy, V> has copy, drop, store {
    contents: vector<Entry<K, V>>,
}
/// An entry in the map
public struct Entry<K: copy, V> has copy, drop, store {
    key: K,
    value: V,
}
```

这是一个Entry类型组成的数组，每次根据键名取值，实际上是在遍历这个数组：

```rust
///根据键名查找在数组内的位置
public fun get_idx_opt<K: copy, V>(self: &VecMap<K,V>, key: &K): Option<u64> {
    let mut i = 0;
    let n = size(self);
    while (i < n) {
        if (&self.contents[i].key == key) {
            return option::some(i)
        };
        i = i + 1;
    };
    option::none()
}
```

在调用insert去添加键值对的时候，实际上是在调用vector::push_back。也就是说，VecMap并不按照键值排序，而是按照添加顺序存储的。

```rust
///添加键值对
public fun insert<K: copy, V>(self: &mut VecMap<K,V>, key: K, value: V) {
    assert!(!self.contains(&key), EKeyAlreadyExists);
    //调用vector的push_back
    self.contents.push_back(Entry { key, value })
}
```

由于根据键名访问值时间复杂度是O(n),所以在数据量比较大的时候不建议使用VecMap, 应该使用父子关系实现。

### VecSet

VecSet是基于vector实现的，是一个保证没有重复的数据集合。它是按照插入顺序保存的，跟VecMap一样，它访问每个元素的时间复杂度都是O(n)。只适用于数据量比较少的情况。



了解更多Sui Move内容：

- telegram: t.me/move_cn
- cQQ群: 79489587
