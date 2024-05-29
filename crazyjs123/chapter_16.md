## 16.轻松入门Sui Move:  升级（上）

在编写合约部署上链后，如果发现有Bug怎么办？在web2中我们可以修改代码，重新部署即可。但是在Move中包是一个不可变对象，也就是说一旦发布就无法修改和删除，以此保证不会因为修改线上包对使用者造成不可预见的问题。不过虽然无法修改链上合约，但Move提供了一个升级包的方法来重新生成一个包。下面我先演示一下升级的方法。

#### 升级的方法

###### 1.发布包

在我们写好合约之后，在项目根目录执行发布包的命令：

```bash
sui client publish
```

注意：在Sui v1.24.1版本之后，--gas-budget选项不再是必填项。

包发布成功后，我们查看本次交易的对象变更：

```bash
╭──────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Object Changes                                                                                   │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Created Objects:                                                                                 │
│  ┌──                                                                                             │
│  │ ObjectID: 0xba952a24d7855908cf825f789e8318219c410068aab1448b349edc0ad97019df                  │
│  │ Sender: 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb                    │
│  │ Owner: Account Address ( 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb ) │
│  │ ObjectType: 0x2::package::UpgradeCap                                                          │
│  │ Version: 6                                                                                    │
│  │ Digest: 4mHhhDaL4sfSN6QQ86FhxbZhNUPtR6vaSoNJeQFVbSYo                                          │
│  └──                                                                                             │
│ Mutated Objects:                                                                                 │
│  ┌──                                                                                             │
│  │ ObjectID: 0x28cdaee082d3a58b5b0f31dd396655920f0f7c2109f46a61c8eb79d7c46ce5dd                  │
│  │ Sender: 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb                    │
│  │ Owner: Account Address ( 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb ) │
│  │ ObjectType: 0x2::coin::Coin<0x2::sui::SUI>                                                    │
│  │ Version: 6                                                                                    │
│  │ Digest: B3xgxJ9YStCzkqW2wyXNYeSBu2sjyMaHZa8sH743MzqB                                          │
│  └──                                                                                             │
│ Published Objects:                                                                               │
│  ┌──                                                                                             │
│  │ PackageID: 0x272713c478b3f04670f65056b36f03c0602925227e743344e80bd161e037da69                 │
│  │ Version: 1                                                                                    │
│  │ Digest: E63ByxG6cj8BQ8DRUgkfJuXWWs1c7wtBD6hWsP74BLZ8                                          │
│  │ Modules: test6                                                                                │
│  └──                                                                                             │
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯
```

可以看到创建了一个UpgradeCap对象，这个对象的ID需要保存好，在升级的时候需要使用它生成升级需要的“票”。这个对象也保存了一些基本信息,在源代码中的定义如下：

```rust
///可用于控制是否可以升级
public struct UpgradeCap has key, store {
    id: UID,
    /// 可以升级的包ID,值是最新版本的包ID
    package: ID,
   	///成功升级的次数，原始值是0
    version: u64,
    ///使用的升级的策略，有哪些可选策略将在下一章节详解
    policy: u8,
}
```

###### 2.编辑Move.toml

在发布之后我们需要编辑Move.toml以保证其他包能正确的引用这个包。这里只展示片段：

```toml
[package]
name = "test6"
version = "0.0.0"
edition = "2024.beta" # edition = "legacy" to use legacy (pre-2024) Move
published-at = "0x272713c478b3f04670f65056b36f03c0602925227e743344e80bd161e037da69"
[addresses]
test6 = "0x272713c478b3f04670f65056b36f03c0602925227e743344e80bd161e037da69"
```

设置published-at和test6包的地址为发布后包的地址。

###### 3.升级操作

在升级之前，需要再次修改Move.toml,将包的地址设置为0x0,以便验证器给升级后的包分配一个新的包地址。也不要忘记修改version字段:

```toml
[package]
name = "test6"
version = "0.0.1"
edition = "2024.beta" # edition = "legacy" to use legacy (pre-2024) Move
published-at = "0x272713c478b3f04670f65056b36f03c0602925227e743344e80bd161e037da69"
[addresses]
test6 = "0x0"
```

现在我们来执行升级的命令

```bash
sui client upgrade --upgrade-capability <UPGRADE-CAP-ID>
```

这个命令会生成升级的摘要，使用UpgradeCap授权升级以获取UpgradeTicket，生成新的包，并在升级成功后使用UpgradeReceipt确保成功升级后会更新UpgradeCap。

命令执行完后，返回如下：

```bash
╭──────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Object Changes                                                                                   │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Mutated Objects:                                                                                 │
│  ┌──                                                                                             │
│  │ ObjectID: 0x28cdaee082d3a58b5b0f31dd396655920f0f7c2109f46a61c8eb79d7c46ce5dd                  │
│  │ Sender: 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb                    │
│  │ Owner: Account Address ( 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb ) │
│  │ ObjectType: 0x2::coin::Coin<0x2::sui::SUI>                                                    │
│  │ Version: 7                                                                                    │
│  │ Digest: A7cV91fjF4MLEYLXwbmQnfrza5WbmxKh5B2rVFvUpGh3                                          │
│  └──                                                                                             │
│  ┌──                                                                                             │
│  │ ObjectID: 0xba952a24d7855908cf825f789e8318219c410068aab1448b349edc0ad97019df                  │
│  │ Sender: 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb                    │
│  │ Owner: Account Address ( 0xc571b07c805118eb0177af2e4e69913af6e9de1bf3fb3fc4df52a8b9d31343cb ) │
│  │ ObjectType: 0x2::package::UpgradeCap                                                          │
│  │ Version: 7                                                                                    │
│  │ Digest: 8eKeChSfT5nv1SLfiE893c83fU1H5C2L4J3wYXSUTaVD                                          │
│  └──                                                                                             │
│ Published Objects:                                                                               │
│  ┌──                                                                                             │
│  │ PackageID: 0x68f4a731627e2fb1b212e685ce041e8d2cb834d3d4429c7a74eeb622e3aa9536                 │
│  │ Version: 2                                                                                    │
│  │ Digest: 9utSnMUMTHdDd4aCeKKpmv7Go4TRgynTtEngUqhvfUfG                                          │
│  │ Modules: test6                                                                                │
│  └──                                                                                             │
╰──────────────────────────────────────────────────────────────────────────────────────────────────╯
```

我们可以看到，除了创建了新的包对象以外，UpgradeCap对象也被修改了，version字段加一，package变为了最新发布的包地址。

###### 4.设置Move.toml

注意：每次升级完，都要把published-at设置为最新版本包地址。而包的地址则一直是原始地址。

```toml
[package]
name = "test6"
version = "0.0.1"
edition = "2024.beta" # edition = "legacy" to use legacy (pre-2024) Move
#设置为最新版本包的地址
published-at = "0x68f4a731627e2fb1b212e685ce041e8d2cb834d3d4429c7a74eeb622e3aa9536"
[addresses]
#设置为最初版本包的地址
test6 = "0x272713c478b3f04670f65056b36f03c0602925227e743344e80bd161e037da69"
```

#### 升级可能会不成功！

要想顺利的升级包，默认情况下必须要满足以下要求：

- 必须要有发布包需要的票，也就是前面提到的UpgradeTicket。在升级的示例中我们使用的是UpgradeCap来自动生成的UpgradeTicket

- public函数的签名，必须与上个版本保持一致

  - 非public函数，包括friend和entry函数的签名可以在升级中修改

  - 不可以改public函数签名，但是可以修改public函数（及其他函数）的实现

  - 可以删除函数中泛型的约束

  - 可以添加新的函数

- 结构体的定义，包括ability都必须与上个版本保持一致

  - 不可以在结构体中新增字段
  - 可以添加新的结构体

那在满足以上条件之后，是不是就可以保证升级后没有兼容性问题了呢？然而事情并没有那么简单。

public函数允许修改代码的实现，那我在一个public函数中新增一个事件的发布。可以顺利升级。升级后前一个版本的包和当前版本的包都在链上且都可以调用。如果调用方依然调用旧版本，监听事件的程序就会错过这个事件，导致程序运行不正确。那这种情况怎么处理呢？我们可以使用一些方法强制调用方调用最新版本的包。

#### 强制调用方使用最新版本的包

###### 方法一：在新包中使用新类型

在新包中定义新的类型，并新增一个方法用于将旧类型的数据转换为新类型。新包只支持新类型的访问，而旧类型数据已经被转换，以此强制接入方使用最新版本的包。

值得注意的是，数据迁移的方法需要做好权限管理，确保是包的拥有者才能调用此方法。可以通过在init的时候生成一个AdminCap对象，并在迁移数据的方法中使用这个对象验证实现。

###### 方法二：使用版本标记共享对象

在共享对象中记录包的最新版本，并在访问中验证版本，如果不是最新版本的包访问共享对象就直接报错。



参考资料：

https://docs.sui.io/concepts/sui-move-concepts/packages/upgrade



了解更多Sui Move内容：

- telegram: t.me/move_cn
- cQQ群: 79489587
