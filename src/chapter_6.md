# 6.轻松入门Move: 结构体

我们前面讲解基本数据类型的时候，讲到基本数据类型只有三种：整型，布尔型和地址。细心的朋友可能会疑惑，为什么连字符串类型都没有？我想使用Move程序保存一段文本如何实现？ 这时候就要用到自定义类型，也就是结构体。

## 创建String结构体类型

我们可以使用结构体定义一个String类型：使用基本数据类型u8组成的数组来存储字符串。那我们自己实现一个String类型? 大可不必！Sui框架已经为我们实现了String类型，源码如下：

```rust
//struct <type name> <has abilities>
public struct String has copy, drop, store {
    bytes: vector<u8>,
}
```

**结构体名称要使用大写字母开头，首字母后可以包含大小写字母、下划线和数字（建议使用大驼峰命名法）。结构体内的字段名则由小写字母、数字和下划线组成。**

那我们如何使用这个自定义的类型呢？

```rust
//先引用String类型
use std::string::String;
//申明一个HelloWorld类型，包含String类型
public struct HelloWorld has drop {
    no: u64,
    text: string::String //申明text字段类型是String类型
}
public fun new_hello_world():HelloWorld{
    //实例化一个结构体
    HelloWorld{
        no: 1,
        //调用string模块提供的utf8函数实例化String类型
        text: string::utf8(b"hello world")
    } 
}
```

上述例子中，结构体HelloWorld带有两个字段，字段no是u64类型,另一个字段text则是标准库定义的结构体。 我们可以得出一个结论：**结构体是一种自定义类型，可以包含基础数据类型和自定义类型的字段。**但是值得注意的是，**结构体不能包含自身类型**。比如说：

```rust
public struct HelloWorld has drop {
    no: u64,
    text: string::String,
    hello_world: HelloWorld //包含自身类型
}
```

以上代码在编译的时候会报错：Invalid field containing 'HelloWorld' in struct 'HelloWorld'.

## 访问结构体字段值

### 模块内访问

我们使用一个单元测试用例来演示如何访问HelloWorld类型的text字段：

```rust
#[test]
public fun test_create_hello_world() {
    let hw = create_hello_world();//先实例化一个类型
    let text = hw.text;//使用符号.即可访问结构体内字段
    assert!(string::utf8(b"hello world") == text, 0);
}
```

注意：有的文档说只有基本数据类型的访问能使用.符号，本人在sui-move 1.20.0上测试发现自定义类型也能使用！也就是说不管字段是什么类型都可以使用.访问。

上面的代码通过了单元测试。那我们用同样的方法访问text字段的bytes字段是不是也能通过？

```rust
#[test]
public fun test_create_hello_world() {
    let hw = create_hello_world();
    let bytes = hw.text.bytes;
    assert!(b"hello world" == bytes, 0);
}
```

运行sui move test直接报错：Invalid access of field 'bytes' on 'std::string::String'. Fields can only be accessed inside the struct's module（**只有在模块内才有访问结构体字段的权限**）

### 模块外访问

也就是说我们在自己编写的模块里不能直接访问标准库string模块的结构体字段值，那我们就”绕一个弯“，通过调用函数来访问其他模块的结构体字段值。

比如访问String类型的bytes字段，我们可以使用string模块提供的bytes方法，这个方法是公共方法所以任何模块都有权限调用。如下：

```rust
#[test]
public fun test_create_hello_world() {
    let hw = create_hello_world();//实例化一个HelloWorld类型
    let bytes = string::bytes(&hw.text);//bytes函数返回的不是bytes字段而是一个指针
    assert!(b"hello world" ==  *bytes, 0);//*号+指针表示指针指向的值，并断言值是"hello world"
}
```

同样的我们在自己模块定义结构体的时候，可以定义函数来开放结构体字段的访问。函数的名称建议直接延用字段的名称。比如：

```rust
public fun no(hw: &HelloWorld):u64 {
    return hw.no   
}
```

注意：实例化结构体、访问结构体字段值、修改结构体实例和销毁结构体都只能在定义这个结构体的模块内。

## 修改结构体字段值

方法如下：

```rust
public fun changeText(hw: &mut HelloWorld) {
    hw.text = string::utf8(b"change it");
}
```

这里值得注意的是函数参数引用结构体的方式，**引用结构体分为可变引用（&mut）和不可变引用(&)。** 因为上述代码块要修改结构体实例的值就必须使用&mut。如果只是访问结构体实例则使用&即可。

不仅仅是函数参数可以这么引用，变量赋值也是如此：

```rust
let s = S {f : 10};
let s_ref = &s;//不可变引用
let s_mutref = &mut s;//可变引用，可以使用s_mutref变量修改s实例的值
```

除了引用外，还可以直接使用按值传输对象也就是直接传输对象本身。按值传输对象用于销毁对象、嵌套在对象里或者移交。	

## 销毁结构体实例

如果结构体含有drop的能力，会在使用后自动销毁。但是如果结构体没有drop能力且有销毁的需求，就需要编写函数并调用函数销毁。销毁的方法如下：

```rust
public fun drop(hello_world: HelloWorld) {
    let HelloWorld{text:_,test_vector:_} = hello_world;
}
```

注意：

- 官方建议统一使用drop命名销毁函数。

- 上述代码块是把结构体所有值都丢弃了，如果其中字段没有drop能力就不允许这么做。
- 参数就不允许传递实例的地址，必须是传递实例本身。



 了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587
