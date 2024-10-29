# 17.轻松入门Move:  升级（下）

上一章我们讲到UpgradeCap的定义，讲到它有一个policy字段表示这个包使用的升级策略。那Move内置的升级策略有哪些呢？

## 内置的升级策略

Move内置了几种升级策略供我们选择，下面我们分别介绍各种策略以及设置方法。不过在这之前，设置升级策略的规则需要了解：

- 默认的升级策略是最宽松的策略
- 如果想设置升级策略只能设置比当前策略更严苛的策略，设置更宽松的策略将会导致报错。

### 升级策略

我们从严苛到宽松列举内置的升级策略：

- 不可升级

  这是最严格的升级策略，发布之后不允许任何升级。可以通过调用sui::package::make_immutable方法设置:

  ```bash
  sui client call --package 0x2 --module package --function make_immutable --args <UpgradeCap ID>
  ```

- 只允许修改依赖项

  只能修改包的依赖项，其他一律不允修改

  ```bash
  sui client call --package 0x2 --module package --function only_dep_upgrades --args <UpgradeCap ID>
  ```

- 只允许添加

  只能在包内添加新的函数和结构体，不允许修改原来的代码（即便是私有函数的内部实现也不行）

  ```bash
  sui client call --package 0x2 --module package --function only_additive_upgrades --args <UpgradeCap ID>
  ```

- 兼容

​	最宽松的升级策略，也是默认的升级策略。允许修改函数的实现代码，但是不允许修改public函数签名（非public函数的签名允许修改）；允许祛除函数的泛型约束（只允许放松约束); 只允许新增结构体不允许修改。

​	兼容策略是默认的升级策略，并且不允许从更严苛的策略设置为此策略，所以兼容策略没有设置方法。

## 自定义升级策略

除了内置的升级策略外，我们也可以自定义升级策略，但是自定义之前我们需要了解包的升级过程，才能理解如何编写策略。

### 升级包的流程

我们上一章讲解包升级方法，了解到包的升级是在一个事务中完成的。不过在这个事务中，实际上是分别调用了三个命令，下面我们通过介绍这三个命令来了解升级的底层原理。

#### 授权（authorize_upgrade）

这个命令的作用是使用UpgradeCap验证权限、验证通过则返回UpgradeTicket。这个命令在sui包package模块中的签名是：

```rust
public fun authorize_upgrade(
        cap: &mut UpgradeCap,
        policy: u8,
        digest: vector<u8>
): UpgradeTicket 
```

第一个传参是UpgradeCap对象，在发布包的时候返回它的ID, 包的发布者即是它的拥有者。它的拥有者可以使用它升级包或者修改升级策略。它的定义再上一章已经讲过，这章不再赘述。

第二个参数是正在使用的升级策略

第三个参数则是摘要，摘要如何计算的可以参考

[摘要计算方法]: https://docs.sui.io/concepts/sui-move-concepts/packages/custom-policies#package-digest	"摘要计算方法"

我们可以通过在build包的时候通过--dump-bytecode-as-base64选项来指示程序打印base64编码的字节码，在打印的内容中可以获取digest值

```bash
sui move build --dump-bytecode-as-base64
```

打印内容：

```json
{"modules":["oRzrCwYAAAALAQAKAgoUAx4pBEcEBUslB3CKAQj6AWAG2gIkCv4CEgyQA1AN4AMCAA8BDQILAhACEQAACAAAAQgAAQIHAAIEBAAEAwIAAAoAAQAACQABAAAFAgEAAA4CAQABEgQFAAIIAAMAAxAJAQEIBAwGBwAGCAYKAQcIBAACBwgABwgEAQgDAQoCAQgCAQYIBAEFAQgBAgkABQEIAAZQZXJzb24FUGhvbmUGU3RyaW5nCVR4Q29udGV4dANVSUQNY2hhbmdlX3BlcnNvbgJpZARuYW1lA25ldwpuZXdfcGVyc29uCW5ld19waG9uZQZvYmplY3QGc2VuZGVyBnN0cmluZwR0ZXN0BXRlc3Q2CHRyYW5zZmVyCnR4X2NvbnRleHQEdXRmOAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgoCBwYxMjM0NTYKAgkIdmVyc2lvbjEKAgsKdmVyc2lvbjE5MAACAgYIAwcIAgECAgYIAwcIAgABBAABCgoAEQUHABEEEgELAC4RBzgAAgEBBAABCgoAEQUHABEEEgALAC4RBzgBAgIBBAABBgcBEQQLAA8AFQIDAAAAAQYHAhEECwAPABUCAAEA"],"dependencies":["0x0000000000000000000000000000000000000000000000000000000000000001","0x0000000000000000000000000000000000000000000000000000000000000002"],"digest":[79,58,190,101,62,35,128,195,167,4,23,212,223,242,100,90,123,173,107,231,106,142,168,236,51,207,220,158,121,59,154,142]}
```

其中digest字段就是第三个参数的值。

这个函数的返回是UpgradeTicket，旨在作为授权成功的凭证。它的定义为：

```rust
public struct UpgradeTicket {
    /// 生成ticket的UpgradeCap对象ID
    cap: ID,
    /// 可以升级的包的ID，一定是最新版本包的ID,以此保证只有最新包能升级
    package: ID,
    /// 在本次升级中使用的升级策略
    policy: u8,
    /// 在本次升级中使用的摘要,用于标识升级后的包的内容
    digest: vector<u8>,
}
```

每个UpgradeCap一次只能生成一个UpgradeTicket，以此保证不会并发升级或者修改升级策略等操作。

UpgradeTicket是一个“烫手山芋”，所以在拿到这个UpgradeTicket之后，必须要调用后续的升级命令来将UpgradeTicket传递出去，否则事务将失败。

#### 执行升级

执行升级这一步是使用的内建命令，作用是消费UpgradeTicket、验证包的更新是否符合升级策略、生成升级包的对象并返回一个代表升级成功的UpgradeReceipt对象。

在这个命令中，拿到UpgradeTicket后验证器便会使用这个类型的所有字段值来验证，只要有一个不符合要求都会升级失败。比如，要升级的包的字节码必须要跟digest字段匹配，policy字段至少要跟UpgradeCap的升级策略一样严格等。

值得注意的是，UpgradeTicket是一次性“票据”，会在这一步被析构，不能将其保存多次使用。

这一步骤返回的UpgradeReceipt是升级成功的证明，它也是一个“烫手山芋”用来保证一定会调用提交升级的方法。它的定义如下：

```rust
public struct UpgradeReceipt {
    ///UpgraceCap的ID
    cap: ID,
    ///升级后的包的ID
    package: ID,
}
```

#### 提交升级(commit_upgrade)

这个命令的作用是消费UpgradeReceipt，并更新UpgradeCap的version字段和package字段。它的定义：

```rust
public fun commit_upgrade(
    cap: &mut UpgradeCap,
    receipt: UpgradeReceipt,
) {
    //析构UpgradeReceipt
    let UpgradeReceipt { cap: cap_id, package } = receipt;

    assert!(object::id(cap) == cap_id, EWrongUpgradeCap);
    assert!(cap.package.to_address() == @0x0, ENotAuthorized);
	//修改UpgradeCap对象的值
    cap.package = package;
    cap.version = cap.version + 1;
}
```

### 自定义升级策略

我们可以通过重写授权和提交升级这两个命令来自定义升级策略。但是要自定义升级策略之前我们最好先了解官方建议的最佳实践：

#### 最佳实践

- 自定义升级策略单独一个包，不要与使用这个策略的代码放同一个包
- 自定义的升级策略的包升级策略设置为不可升级
- 锁定UpgradeCap的policy字段，不允许违反只能收紧升级策略的原则。

#### 自定义升级策略

下面我们使用官网的例子来演示如何升级。我们自定义一个升级策略：在兼容的升级策略基础上，只允许在一周内的指定一天进行升级操作。

我们新创建一个包，自定义一个UpgradeCap对象，在这个对象中保存验证规则所需字段:

```rust
public struct UpgradeCap has key, store {
    id: UID,
    //包含package的UpgradeCap对象，用于调用基础的授权和提交升级等函数
    cap: package::UpgradeCap,
    //指定可发布的时间，值在1-7
    day: u8,
}
```

定义新建升级策略的方法,这个方法返回UpgraceCap对象：

```rust
public fun new_policy(
    cap: package::UpgradeCap,
    day: u8,
    ctx: &mut TxContext,
): UpgradeCap {
    assert!(day < 7, ENotWeekDay);
    UpgradeCap { id: object::new(ctx), cap, day }
}
```

重写授权函数,在这个函数中验证是否满足升级策略：

```rust
public fun authorize_upgrade(
    cap: &mut UpgradeCap,
    policy: u8,
    digest: vector<u8>,
    ctx: &TxContext,
): package::UpgradeTicket {
    //如果不是可升级的日期，就报错
    assert!(week_day(ctx) == cap.day, ENotAllowedDay);
    //调用基础的授权方法，也就是说在遵循自定义升级策略的基础上还要遵循内建的升级策略！
    package::authorize_upgrade(&mut cap.cap, policy, digest)
}
```

重写提交升级函数：

```rust
public fun commit_upgrade(
    cap: &mut UpgradeCap,
    receipt: package::UpgradeReceipt,
) {
    package::commit_upgrade(&mut cap.cap, receipt)
}
```

#### 发布升级策略

编写完成后，将包发布到链上

```bash
sui client publish
```

遵循最佳实践，将这个包设置为不可升级

```bash
sui client call --gas-budget 10000000 \
    --package 0x2 \
    --module 'package' \
    --function 'make_immutable' \
    --args '<POLICY-UPGRADE-CAP>'
```

#### 使用升级策略

我们在发布完一个包之后，拿着这个发布包返回的UpgradeCap对象的ID,调用new_policy函数，并保存返回的自定义UpgradeCap对象的ID。

在后续的升级中，按照上面讲得升级的流程，依次调用自定义的authorize_upgrade函数、内置的升级函数和commit_upgrade函数即可。



参考资料：

https://docs.sui.io/concepts/sui-move-concepts/packages/custom-policies

了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587

