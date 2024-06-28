## 5.轻松入门Sui Move: Debug、单元测试和命令行工具

### 单元测试

单元测试函数是没有参数，也没有返回值，带有一个#[test]的标记的public函数。命令建议使用test_作为前缀加上被测试函数名称。单元测试函数跟被测试函数可以放置在同一个module内，也可以单独放置在跟sources文件夹同级别的tests文件夹内。

```rust
public fun a_greater_than_b(a: u64, b: u64): bool{ 
    a >= b
}
 #[test]
 public fun test_a_greater_than_b() {
    let a = 10;
    let b = 12;
    assert!(!a_greater_than_b(a, b), 0);
 }
```

assert即将被丢弃，建议使用assert!。assert!函数第一个参数是一个表达式，表达式值为false表示断言失败，单元测试报错不通过。

编写好单元测试后，在项目根目录执行如下命令即可运行单元测试

```move
sui move test
```

单元测试报告解读：

```bash
INCLUDING DEPENDENCY Sui
INCLUDING DEPENDENCY MoveStdlib
BUILDING hello_world
Running Move unit tests
[ PASS    ] 0x0::hello_world::test_a_greater_than_b    #通过的单元测试名
Test result: OK. Total tests: 1; passed: 1; failed: 0  #单元测试数，通过数，失败数
Total number of linter warnings suppressed: 1 (filtered categories: 1)
```

### Debug

Sui Move暂时没有本地的调试器，可以使用std::debug模块来调试代码，打印变量。

调用print函数打印变量值到命令行,注意print参数传递的不是变量本身，而是变量的引用。

```rust
std::debug::print(&v);
//在命令行输出中带有[debug]标记的就是打印结果
```

也调用print_stack_trace函数打印堆栈轨迹

```move
std::debug::print_stack_trace();
```

结合单元测试，就可以在命令行打印数据，调试代码：

```rust
public fun a_greater_than_b(a: u64, b: u64): bool{ 
    std::debug::print(&a);
    std::debug::print(&b);//调试打印
    a >= b
}
#[test]
public fun test_a_greater_than_b() {
    let a = 13;
    let b = 12;
      
    assert!(a_greater_than_b(a, b), 0);//单元测试调用函数
}
```

现在只需运行单元测试，就可以打印信息进行调试。

### 命令行工具

下面将讲解一些常用的命令：

#### sui move命令

##### 创建一个新包

```bash
# 指定路劲创建新包
sui move new <package_name> -p <path>
#当前目录创建名为hello_world的包
sui move new hello_world
```

##### 编译

```bash
sui move build
#想让编译更快，可以暂时不拉取最新依赖
sui move build --skip-fetch-latest-git-deps
#使用Move.toml申明的dev依赖和地址
sui move build --dev
```

##### 单元测试

单元测试是先编译后运行单元测试，所以上述编译的选项也可以用在单元测试中

```bash
sui move test
#列出所有单元测试
sui move test -l 
#指定运行单元测试的线程数，默认八个
sui move test -t 10
#使用dev模式编译并运行单元测试,--test同理
sui move test --dev
#在单元测试结尾生成统计信息并打印到终端
sui move test -s
#限制每个单元测试函数消耗的gas
sui move test -i 1
#收集覆盖率相关数据，以支持sui move coverage命令的使用。
sui move test --coverage
```

注意：单元测试的覆盖率只有debug模式的客户端支持，想使用此功能，可以使用源码构建sui move cli

##### 覆盖率统计

获取覆盖率数据之前，需要使用--coverage运行单元测试

```bash
#获取覆盖率的汇总信息
sui move coverage summary 
#根据源代码显示有关模块的覆盖率信息
sui move coverage source
#根据反汇编的字节码显示有关模块的覆盖率信息
sui move coverage bytecode
```

#### sui client命令

sui client提供与Sui网络交互的命令

##### 列出可用网络环境

```bash
sui client envs
```

##### 创建新的网络环境

```bash
sui client new-env --alias=mainnet --rpc https://fullnode.mainnet.sui.io:443
```

##### 切换当前网络环境

```bash
sui client switch --env mainnet
```

##### 新建地址

```bash
#选择ed25519密钥对方案，生成新的密钥对和地址，并设置地址别名为test
 sui client new-address  ed25519 test
```

##### 切换当前地址

```bash
sui client switch --address <address别名>
```

##### 获取当前活跃地址

当前活跃地址可以理解为是当前用户在Sui网络的标识符

```bash
sui client active-address
```

##### 使用对象ID获取对象信息

```bash
sui client object <object id>
#使用json格式返回对象数据
sui client object <object id> --json
```

##### 查看当前地址拥有的所有对象

```bash
sui client objects
#输出json
sui client objects --json
```

##### 获取动态字段信息

```bash
sui client dynamic-field <DYNAMIC-FIELD-ID>
```

##### 获取余额

```bash
sui client balance
sui client gas
```

##### 在devnet申请gas

```bash
# 执行前需要确认当前网络是否是devnet.执行后一分钟内就能到账
sui client faucet --address <你的地址>
```

##### 合并gas的余额

如果有多个gas对象，不想每次使用--gas选项指定，就可以合并gas对象的余额到一个gas对象中去

```bash
sui client merge-coin --primary-coin <gas coin id> --coin-to-merge <gas coin id> --gas-budget <GAS_BUDGET>
```

-  --primary-coin 余额都合并到这个gas对象中
- --coin-to-merge 被合并余额的gas对象

##### 发布包之前，检查字节码是否超过规定值

强烈建议在发包之前执行此操作，避免发布失败，消耗不必要的gas

```bash
sui client verify-bytecode-meter
```

##### 发布包

```bash
#发布当前目录的包
sui client publish 
#发布指定目录的包
sui client publish  /home/root/packages/hello_world
#发布时消耗指定的gas对象的gas
sui client publish--gas <gas coin id> 0 
```

注意：

- gas可以适当指定大一点，因为gas不够导致的发布失败，并不会退回gas
- gas coin id可以通过sui client gas获取

- 发布之前要进行编译，所以编译的选项在这个命令也是生效的

##### 调用已经发布包的方法

```bash
#调用一个没有参数的函数
sui client call [OPTIONS] --package <package id> --module <module名称> --function <函数名> --gas-budget <GAS_BUDGET>
#调用带参数的函数
sui client call [OPTIONS] --package <package id> --module <module名称> --function <函数名> --gas-budget <GAS_BUDGET>  --args <参数1> <参数2>
#调用泛型函数,必须指定所有的类型参数否则会报错
sui client call [OPTIONS] --package <package id> --module <module名称> --function <函数名> --gas-budget <GAS_BUDGET>  ----type-args <类型参数1> <类型参数2>

```

##### 查看交易消耗gas的详细信息

```bash
sui client profile-transaction --tx-digest <交易的digest>
```

##### 转移资产

```bash
#转移指定资产给指定地址
sui client pay --input-coins <被转移的coin id> --recipients <收款地址>--amounts <转账金额>--gas-budget <本次操作可消费最大gas>
#转移Sui资产给指定地址
sui client sui-pay --input-coins <被转移的coin id> --recipients <收款地址>--amounts <转账金额>--gas-budget <本次操作最大可消耗gas>
```

##### 转移对象所有权

```bashsui client transfer [OPTIONS] --to <TO> --object-id <OBJECT_ID> --gas-budget <GAS_BUDGET>
sui client transfer --to <收到对象所有权的地址> --object-id <对象id> --gas-budget <本次交易最大可消耗gas>
```

##### sui console

有没有觉得sui client相关的命令每次都要输入sui client 很麻烦？可以使用sui console进入sui的命令行，省略sui client字符直接输入命令即可。比如：

```sui console
#获取gas对象余额
gas
#切换当前地址
switch --address mystifying-sphene
```

##### 退出sui命令行模式

```bash
exit
```

##### 清屏

```bash
clear
```

##### 查看历史命令

```bash
history
```



了解更多Sui Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587







