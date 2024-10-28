# 2.轻松入门Move: 搭建开发环境
在编写Move程序之前，需要先安装开发环境，所以本章将介绍如何安装开发环境。

安装开发环境有三种方式：

- 1.使用二进制文件安装
- 2.使用源代码安装,比较复杂但是可以更多的控制安装过程
- 3.使用docker镜像安装

入门的朋友，我建议还是选择既能了解安装过程又不会太过复杂的二进制安装。

这篇文章将主要讲解如何在Windows上使用二进制文件安装本地开发环境。注意不是使用Windows子系统Ubuntu安装，而是直接安装到Windows系统。以下方法适用于win10、win11。

## 安装前准备

### curl

这个一般Windows是自带的，检测是否自带运行命令：

```bash
curl http://www.baidu.com
```

### git

详细的安装方法网络上已经有不少教程，详细参见：[git安装方法](https://zhuanlan.zhihu.com/p/443527549)

### cmake

[下载页面](https://cmake.org/download/)选择带有windows标志的.msi文件下载，具体下载x64、AMD64还是i386架构的，参见[git安装方法](https://zhuanlan.zhihu.com/p/443527549)。下载完成后，双击安装，一路点next完成安装

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/cmake_installer.jpg?raw=true)

### protocol buffer

[下载页面](https://github.com/protocolbuffers/protobuf/releases)下载最新版本，带有win字样的压缩包即可。下载完解压缩，把bin目录所在目录，添加到环境变量Path中，如下图：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/protocol.jpg?raw=true)

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/protocol_path.jpg?raw=true)

###  LLVM Compiler Infrastructure

[下载页面](https://releases.llvm.org/) 点击进入，选择最新版本download跳到github页面，选择带有win字样的exe文件下载，并安装。一路next即可。

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/llvm_download.jpg?raw=true)

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/llvm_github.jpg?raw=true)

### rustup

rustup是一个管理工具链，用于管理不同平台下的 Rust 构建版本并使其互相兼容。在Windows环境中，使用 [rustup-ini.exe](https://www.rust-lang.org/zh-CN/tools/install)下载后，双击运行，会有一个选项，如下图，输入1回车即可安装。

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/intsall_rustup.png?raw=true)

安装完成后，enter键关闭窗口，重新打开cmd窗口，输入下图命令，判断是否安装成功：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/rustc.jpg?raw=true)

## 安装Sui

[下载页面](https://github.com/MystenLabs/sui/releases)，点击带有windows字样的压缩包，下载并解压，打开target/release文件夹，把其中可执行文件的名称中的-windows-x86_64去掉，并复制到.cargo/bin文件夹中。如下图：

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/sui_exec.png?raw=true)

打开一个cmd命令行窗口，输入以下命令验证安装是否成功:

![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/sui_installed.png?raw=true)

## 安装编辑器及插件

vscode编辑器的安装教程，网上已经有很多，这里不再赘述。[详见](https://blog.csdn.net/msdcp/article/details/127033151)

这里要着重讲的是安装Move Analyzer

Move Analyzer是由MoveBit开发的适用于sui-move语言的Visual Studio代码插件，它有许多有用的功能，如高亮显示、自动完成、转到定义/引用等。

vscode安装好后，点击侧边栏EXTENSIONS,在搜索栏搜索Move Analyzer选中后，不要直接点击安装，先查看安装说明，这里有几点需要注意：

1.如果已经安装move-analyzer 或者 aptos-move-analyzer的需要先disable掉，避免产生冲突

2.要先安装sui-move-analyzer language server,然后再安装此插件。

## 申请开发环境gas

上文我们也讲到，无论是将代码部署到链上还是调用链上方法，都需要gas。那我们开发环境怎么办呢？难不成要付费开发？这倒是不用，我们可以免费申请devnet的gas。申请方法如下：

- 1.获取当前地址，第一次执行有一些交互，按照图示输入即可。生成完当前地址再执行sui client active-address就可以获取账户地址

  ![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/gas.png?raw=true)

- 2.在[Discord](https://blog.csdn.net/msdcp/article/details/127033151)中注册账号并通过验证

- 3.在#devnet-faucet频道输入框输入 !faucet <第一步获取到的地址> 。使用sui client gas几秒钟后就可以看到gas充值成功

  ![](https://github.com/Crazyjs123/crazyjs123.github.io/blob/main/pic/get_gas.png?raw=true)



现在既有编辑器、gas、运行环境都准备好了，那我们可以开始我们的Move之旅啦。



了解更多Move内容：

- telegram: t.me/move_cn
- QQ群: 79489587