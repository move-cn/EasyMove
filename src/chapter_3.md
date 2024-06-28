## 3.轻松入门Sui Move: 清单文件和模块

按照国际惯例，学习一门语言,编写的第一个程序一定是输出一个Hello World。今天我们也来一起写一个Hello World并以此引出一些Sui Move项目结构，并作详细介绍

首先我们新建一个名为hello_world的项目，使用命令：

```bash
sui move new hello_world
```

这个命令会自动新建一个名为hello_world的文件夹，文件夹结构如图：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/tree.png?raw=true)

这个文件夹包含一个sources文件夹和一个Move.toml清单文件，其中sources目录是存放我们编写的代码，里面的一个文件对应一个模块。Move.toml文件则是一个清单文件，用于申明包的元数据信息、依赖、地址等。详情请看下面代码块：

```toml
[package]
#在这个部分申明包的元数据信息,比如名称、版本信息、证书信息等
name = "hello_world"

# edition = "2024.alpha" # 使用Move 2024 alpha 版本 
# license = ""           # 申明证书，比如, "MIT", "GPL", "Apache 2.0"
# authors = ["..."]      # 申明作者，比如 ["Joe Smith (joesmith@noemail.com)", "John Snow (johnsnow@noemail.com)"]

[dependencies]
#在这个部分列出这个包依赖的其他包
Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/testnet" }

# 对于远程包的引用，请使用 `{ git = "...", subdir = "...", rev = "..." }`.
# 其中rev可以是一个分支，一个tag或者一个提交哈希，如下例：
# MyRemotePackage = { git = "https://some.remote/host.git", subdir = "remote/path", rev = "main" }

# 如果是本地包的引用，使用 `local = path`即可. Path 是包的根目录下的相对路径
# Local = { local = "../path/to" }

# 如果有版本冲突，则指定版本号，并且使用`override = true`来解决冲突
# Override = { local = "../conflicting/version", override = true }

[addresses]
# 申明这个包的地址，在后续可以使用hello_world代指这个包。默认情况这个地址是"0x0"，但在发布到链上会替换成区块链上的地址。这个名称甚至不局限于在包内使用，比如std标准包，我们直接在自己的包中使用std引用。
hello_world = "0x0"

[dev-dependencies]
# 这个部分用于声明开发或者测试模式下才需要的依赖。
# 额外提一句，开发或测试模式是编译时通过指定--test(--dev)来指定模式的。
# Local = { local = "../path/to/dev-build" }

[dev-addresses]
# 这个部分用于申明开发或者测试模式下的包地址。
```

上面代码块中提到的package就是所谓的包，**包是一组模块的集合，是发布代码到链上的单元。**

那什么又是模块呢？**模块是一组函数和结构体的集合。模块是一种组织代码的方式，可以将相关的功能组织在一起，并且通过模块可以控制代码的可见性和访问权限，提高代码的可维护性和可扩展性。**

现在我们在sources文件夹内新建一个文件，命名为helloworld.move,然后编写一个名为hello_world的模块：

```rust
module hello_world::hello_world {
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

    struct HelloWorldObject has key, store {
        id: UID,
        text: std::string::String 
    }

    #[lint_allow(self_transfer)]
    public fun mint(ctx: &mut TxContext) {
        let object = HelloWorldObject {
            id: object::new(ctx), 
            text: std::string::utf8(b"Hello World!") 
        };
        transfer::public_transfer(object, tx_context::sender(ctx));
    }
}
```

上面这段代码申明了一个名称为HelloWorldObject的结构体，在mint方法中创建一个HelloWorldObject对象并将所有权转交给当前上下文的用户。此段代码包含以下几个知识点：

#### 1.模块的申明方法

```bash
module <包的地址>::<模块名称> {
	模块内容
}
```

包的地址和模块名称可以标识一个模块，包的地址可以是包名也可以是清单文件中申明的包地址。

在一个包内，模块名必须唯一。模块文件名与模块名不一致可以通过编译也不影响其他模块的调用，但不建议这么做。模块名建议使用小写字母和下划线组成。

#### 2.模块之间的关系：引用

模块之间可以互相引用，引用方式分为以下几种：

- 直接引用:

  ```rust
  struct HelloWorldObject has key, store {
  	id: UID,
      text: std::string::String //直接引用std::string模块的utf8方法
  }
  ```

- 使用use引用结构体或者函数

  ```rust
  use sui::object::UID //申明引用UID结构体（或函数）
  struct HelloWorldObject has key, store {
  	id: UID, //直接使用结构体名（或函数）
      text: std::string::String 
  }
  ```

- 使用use 引用模块

  ```rust
  use sui::transfer;
  transfer::public_transfer(object, tx_context::sender(ctx));
  ```

- 使用Self关键字引用模块自身

  ```rust
   use sui::tx_context::{Self, TxContext};
   tx_context::sender(ctx)//使用模块名调用函数
   //直接使用TxContext引用TxContext结构体
  ```

- 同一个模块多个引用

  ```rust
  #使用花括号括起来，并逗号隔开
  use sui::tx_context::{Self, TxContext};
  ```

不同的申明方式之间也可以转换使用，效果是一样。比如：

```rust
use sui::tx_context::{Self, TxContext};
tx_context::sender(ctx)//使用模块名调用函数
//直接使用TxContext引用TxContext结构体
```

等价于：

```rust
use sui::tx_context;
tx_context::sender(ctx)//使用模块名调用函数
//TxContext结构体则使用tx_context::TxContext引用
```

还等价于：

```rust
use sui::tx_context::{sender, TxContext};
sender(ctx)//直接调用函数
//直接使用TxContext引用TxContext结构体
```

#### 3.模块如何控制访问权限?

我们上面讲了如何引用模块，那如果模块不愿意被引用怎么办呢？这就涉及到访问权限的问题。访问模块内容分为访问结构体和函数。而结构体内部的字段不能跨模块使用，只能通过调用与结构体同模块的函数实现，如下图：

```rust
 struct HelloWorldObject has key, store {
        id: UID,
        text: std::string::String 
  }
  //通过调用此方法访问结构体内部值
  public fun getText(obj: &HelloWorldObject) :std::string::String {
        obj.text
  }
```

所以，访问权限其实主要就是对函数的访问权限的控制。对函数的访问权限控制粒度从粗到细分为：

- 所有模块都可以调用

  所有模块都可调用，使用关键字public申明函数即可

- 部分模块可以调用

  使用关键词public(friend)申明函数，那就只有在模块内申明了是“朋友”的模块才可以调用。如下：

  ```rust
  //申明朋友模块
  friend hello_world::test;
  //只有朋友模块才可以引用的函数
  public(friend) fun getName(obj: &Game) :std::string::String {
      obj.name
  }
  ```

- 只有同模块可调用

​      只有同模块都可调用，使用关键字private申明函数即可。private是默认权限，也可以省略。



了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587













