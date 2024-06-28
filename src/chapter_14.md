## 14.轻松入门Sui Move:  集合（上）

这一章我们将讲解如何保存数据的集合。说到数据的集合首先想到的肯定是数组，Move标准库给我们提供了vector模块以支持数组类型。

### 数组

vector是一个可变长度的，任意类型的容器。跟其他语言的数组一样，使用索引访问，索引从0开始。

如何使用数组，在结构体章节讲解String类型讲过：

```rust
//struct <type name> <has abilities>
struct String has copy, drop, store {
    bytes: vector<u8>,
}
```

String类型本质就是一个字节数组。vector使用了泛型来支持任何类型，所以在使用它的时候需要指定类型参数。

###### 创建数组

我们可以使用empty方法，创建一个空数组

```rust
let arr = vector::empty<u64>();
```

或者创建一个长度为1的数组

```rust
let arr = vector::singleton<u64>(12);
```

也可以使用[]创建数组

```rust
let arr = vector<u64>[1,2,3,4];
```

###### 添加元素

- 在数组尾部插入一个元素

  ```rust
  vector::push_back<u64>(&mut arr, 67);
  ```

- 在数组指定位置插入一个元素

  ```rust
  //在数组arr的第三个元素位置插入100
  vector::insert<u64>(&mut arr, 100, 2);
  ```

  insert函数第三个参数，用于指定插入位置。如果插入位置已经超过数组长度将会报错；等于数组长度则在数组末尾插入元素；小于数组长度该位置的元素及后续元素均往后移一位。注意这里不是替换原位置元素，而是插入。

- 合并两个数组

  ```rust
  public fun append<Element>(lhs: &mut vector<Element>, mut other: vector<Element>) 
  ```

  将other数组合并到lhs数组。被合并数组必须传值，合并数组则传可变引用。

  ```rust
  let mut arr1 = vector::singleton<u64>(1);
  vector::push_back(&mut arr1, 2);
  vector::push_back(&mut arr1, 3);
  
  let mut arr2 = vector::singleton<u64>(3);
  vector::push_back(&mut arr2, 4);
  
  vector::append(&mut arr2, arr1);
  print(&arr2);
  //打印结果：[ 3, 4, 1, 2, 3 ]
  ```

  合并完的数据，顺序是：lhs数组、other数组。如果有重复的值也会保留。

###### 获取元素

- 弹出数组尾部元素

  ```rust
  assert!(!vector::is_empty<u64>(&arr), 1);
  let item = vector::pop_back<u64>(&mut arr);
  ```

  注意，弹出后返回的是元素值，数组中不再包含该值，长度-1。在调用pop_back之前请确保数组不为空否则将会报错。

- 获取指定位置元素

  与弹出不同，borrow和borrow_mut只是获取元素引用，并没有从数组中取出元素。

  borrow是不可变引用，只用于读；如果要修改元素则使用borrow_mut

  ```rust
  let mut arr1 = vector::singleton<u64>(1);
  vector::push_back(&mut arr1, 2);
  vector::push_back(&mut arr1, 3);
  //可变引用第一个元素
  let item = vector::borrow_mut(&mut arr1, 0);
  *item = 2;
  
  print(&arr1);
  //输出结果：[ 2, 2, 3 ]
  ```

###### 交换位置

- 反转数组

  ```rust
  let mut arr1 = vector::singleton<u64>(1);
  vector::push_back(&mut arr1, 2);
  vector::push_back(&mut arr1, 3);
  
  vector::reverse(&mut arr1);
  
  print(&arr1);
  //输出结果：[ 3, 2, 1 ]
  ```

- 两个位置互换

  ```rust
  let mut arr1 = vector::singleton<u64>(1);
  vector::push_back(&mut arr1, 2);
  vector::push_back(&mut arr1, 3);
  //位置2和3交换值
  vector::swap(&mut arr1, 1, 2);
  
  print(&arr1);
  //输入结果：[1, 3, 2]
  ```

###### 删除元素

- 弹出末尾元素

  - 这个在上面已经讲过，不再赘述

- 删除指定位置元素

  ```rust
  let mut arr1 = vector::singleton<u64>(1);
  vector::push_back(&mut arr1, 2);
  vector::push_back(&mut arr1, 3);
  
  vector::remove(&mut arr1, 1);
  
  print(&arr1);
  //输出结果：[1,3]
  ```

  注意：在删除指定位置的元素后，后续元素会自动前移一位。

- 删除并使用末尾元素填充

  ```rust
  let mut arr1 = vector::singleton<u64>(1);
  vector::push_back(&mut arr1, 2);
  vector::push_back(&mut arr1, 3);
  
  vector::swap_remove(&mut arr1, 0);
  
  print(&arr1);
  //输出结果：[3, 2]
  ```

  被删除的位置使用末尾元素填充，省去了前移元素的时间，如果元素顺序不重要，优选此方法。

- 删除空数组

  ```rust
  assert!(is_empty(&arr1), 1);
  vector::destroy_empty(arr1);
  ```

  注意，请在调用destroy_empty之前确保数组是空的，否则会导致报错

###### 其他

- 获取数组长度

  ```rust
  let len = vector::length(&arr1);
  ```

- 判断数组是否为空

  ```rust
  assert!(vector::is_empty(&arr1), 1);
  ```

- 根据值获取元素位置

  ```rust
  public fun index_of<Element>(v: &vector<Element>, e: &Element): (bool, u64) 
  ```

  ```rust
  let (exsits, index) = vector::index_of<u64>(&arr1, &item);
  ```

  注意第二个参数是传入元素的引用不是本身.返回两个字段,第一个代表是否存在,第二个代表元素位置.

- 是否包含某元素

```rust
let item: u64 = 2;
let exists = vector::contains<u64>(&arr1, &item);
```

vector模块具有丰富的函数,利用pop_back和push_back可以轻松实现栈, 利用insert和remove也能轻松实现队列.

### 优先级队列

优先级队列使用最大堆排序方法来对优先级进行排序.优先级高的先出列.如果我们需要有序的数据集合,就可以使用优先级队列.

最大堆是二叉树并且每个节点的值都大于或等于其子节点.也就是说根节点就是堆中最大值.最大堆排序的平均时间复杂度是O(nlogn)是一种比较优秀的排序方法.

我们以学生成绩排名为例来介绍优先级队列的功能,创建一个学生结构体和一个学校期末报告结构体.期末报告结构体负责保存学生信息和成绩,并按照成绩从高到低输入分数和学生的信息.

```rust
public struct Student has drop,store {
    name: String,
    score: u64,
}
public struct SchoolReport has drop{
    report: PriorityQueue<Student>,
}
```

期末报告report字段,类型就是优先级队列.这里需要注意的是,Student结构体作为优先级队列中实体的值,必须要有drop和store能力.

期末出成绩后,往优先级队列中塞入成绩

```rust
public fun new(): SchoolReport{
    let student1 = Student{
        name: string::utf8(b"hanmeimei"),
        score: 89,
    }; 
    let student2 = Student{
        name: string::utf8(b"lilei"),
        score: 97,
    }; 
    let p = vector<u64>[89,97]; 
    let students = vector<Student>[student1, student2]; 

    SchoolReport{
        report: priority_queue::new<Student>(priority_queue::create_entries<Student>(p, students)),
    }
}
```

在创建优先级队列之前,我们需要调用create_entries方法创建实体数组.

塞入一个成绩为70分的学生:

```rust
public fun add_student(sr: &mut SchoolReport) {
    let student1 = Student{
        name: string::utf8(b"libai"),
        score: 70,
    }; 
    priority_queue::insert<Student>(&mut sr.report, 70, student1);       
}
```

按照成绩从高到低排序并输出

```rust
public fun rank(sr: &mut SchoolReport) {
    while(true) {
        let (p, student) = priority_queue::pop_max<Student>(&mut sr.report);

        print(&p);
        print(&student.name);
    }
}
```

pop_max每次一定会输出堆内最大值,多次调用直到堆内无数据,即可实现对分数的排序.



了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587
