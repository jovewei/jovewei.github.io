---

layout: post
title:  Move语言深度解析：安全特性、与Solidity的对比及潜在漏洞
subtitle: 从安全设计到实战对比，深入探讨Move语言的独特魅力与隐藏风险
date:   2022-12-10 21:40:00 +0800
author: "Bryce"
header-img:  'images/gallery/the-conspiracy-of-the-batavians-under-claudius-civilis-rembrandt-harmensz-van-rijn.jpg'
tags:   [区块链, Move]
---

> <cite>Title: The Conspiracy of the Batavians under Claudius Civilis  
Creator: Rembrandt Harmensz. van Rijn  
Date Created: 1661/1662  
Physical Dimensions: h 196cm Ń w 309cm  
External Link: See more details about this work on the Rijksmuseum Website  
Medium: oil on canvas  
</cite>  

## 前言

随着新公链 Aptos、Sui 的崛起，Move 作为面向资产编程的语言也吸引了越来越多的关注，相关公链上使用 Move 语言编写的项目也越来越多。本文就将从安全角度去介绍这一智能合约语言新贵——Move。

在本文中，首先将深入探讨Move语言的特性，从安全角度去介绍Move的设计，然后将通过 Move 与 solidity 的比较来进一步探讨 Move 的特性，最后再回到安全——介绍 Move 项目可能出现的安全漏洞点。

## 基础概念

Move 语言最初是由 Facebook 团队为 Diem（原名Libra）区块链而设计开发的一门新语言。而 Libra 的使命是打造一个简单的全球货币和金融基础设施，为数十亿人提供支持。

Move 语言旨在提供一个安全、可编程的基础，可以在此基础上构建这一愿景。Move 必须能够以精确、可理解和可验证的方式表达 Diem 货币和治理规则。从长远来看，Move 必须能够对构成金融基础设施的各种资产和相应的业务逻辑进行编码。

那么，如何实现这一愿景呢？在 Move 的白皮书中提出了设计时考虑的**四个关键目标：first-class 资产、灵活性、安全性和可验证性**。

其中first-class 资产是实现这一愿景的基础。要理解first-class 资产，我们先来介绍Move语言中的几个比较重要的基础概念。

### 结构体(Structs)

和其他很多语言一样，Move语言中的结构体也是使用`struct`定义。它是自定义类型，也是Move中创建自定义类型的唯一方法。

结构体可以包含复杂数据，也可以不包含任何数据，但不允许定义递归结构体。结构体由字段组成，可以简单地理解成"key-value"存储，其中`key`是字段的名称，而`value`是存储的内容。

### 能力(Abilities)

Move 的类型系统非常灵活，每种类型都可以被四种限制符所修饰。这四种限制符我们称之为 abilities，它们的功能分别是：

- **Copy** - 被修饰的值可以被*复制*。
- **Drop** - 被修饰的值在作用域结束时可以被*丢弃*。
- **Key** - 被修饰的值可以作为*键值*对全局状态进行访问。
- **Store** - 被修饰的值可以被*存储*到全局状态。

而Move的基本类型缺省具有 store, copy 和 drop 限制，自定义类型缺省情况下结构体不带任何限制符。

### 资源(Resource)

介绍完了结构体和能力，接下来就是Move语言中的核心概念：**资源（resource）**。

如果**定义的结构体（struct）不能复制（copy）也不能丢弃（drop），我们就将它们称为资源**。默认情况下，结构体是线性和短暂的。它们不能复制，不能丢弃，不能存储在全局存储中。这意味着所有值都必须拥有被转移的所有权，并且值必须在程序执行结束时处理。

我们可以通过赋予结构体能力（abilities）来简化这种行为，允许值被复制或删除，以及存储在全局存储中或定义全局存储的模式。

Move语言中的这个属性对于现实世界中资源（例如货币）的建模非常有用，因为货币在理论上是不希望在流通过程中被复制或丢失的。

### 模块(Modules)

Move语言中，**代码都是以模块（module）的形式进行组织**的。

如何理解模块呢？抽象的理解，Move 语言中的模块（module）/ 资源（resource）/ 方法（function）之间的交互与传统的面向对象语言中的类（class）/ 对象（object）/ 方法（function）之间的关系非常相似。

而与其他区块链语言相比，Move中的模块类似于其他区块链中的智能合约。开发者在模块中声明资源类型和方法，这些类型和方法定义了创建、销毁和更新已声明资源的规则。

### 泛型(Generics)

Move 因为是静态语言，所以为了保证扩展性，Move 选择了**泛型编程**的范式。

以 Aptos 为例，只要查看你持有的资产是 `0x1::coin::CoinStore<?>` 这样的类型，就可以知道该资产是由标准模组 `0x1::coin` 所定义。

比如说 Aptos 的原生币 AptosCoin 的类型是 `0x1::aptos_coin::AptosCoin` ，意思是 0x1 帐号之下的 aptos_coin 模块所定义的 AptosCoin 类型，不过这个类型并没有赋予 key 能力，所以不能成为资源，他只提供一个种类，而这个类型可以作为泛型来使用，像是被 `0x1::coin::CoinStore` 使用。

那么， `0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>` 就是 0x1 帐号下 coin 模块所定义的 CoinStore 类型，记录地址下某种 coin 的资产状态。所以你的地址下任何 `0x1::coin::CoinStore<?>` 的资产都通用相同逻辑、有一样的行为，都是定义在 `0x1::coin` 这个模块里。

## 安全性

### 设计安全

Move 在白皮书中明确表示，Move 必须拒绝不满足关键属性的程序，例如**Resource安全、类型安全和内存安全**。 我们如何选择一个可执行的表示，以确保在区块链上执行的每个程序都满足这些属性？ 两种可能的方法是：

(a) 使用带有检查这些属性的编译器的高级编程语言

(b) 使用低级无类型汇编并在运行时执行这些安全检查。

Move 采取了介于这两个极端之间的方法。 Move 的可执行格式是一种类型化的字节码，它比汇编高级，但比源语言低。 字节码由字节码验证器在链上检查 Resource、类型和内存安全性，然后由字节码解释器直接执行。 这种选择允许 Move 提供通常与源语言相关的安全保证，但无需将源编译器添加到受信任的计算库或将编译成本添加到交易执行的关键路径中。

除此之外，Move在设计时**内置了很多安全特性，例如算数溢出、默认可见性导致的权限泄露**等等。

### 底层安全

#### 资源

在现实社会中，资产有两个属性难以用数字表示：

1）稀缺性。必须控制系统中资产发行的数量。必须禁止复制现有资产，而创建新资产是一项特权操作。

2）访问控制。系统参与者必须能够使用访问控制策略保护资产。

因此，Move引入了资源来表示资产。而在区块链应用中，代币就是一种资源，资源必须要存储在账户下面，且在交易过程中，资产必须要流向一个地方，要么转移到另一个地址，要么被销毁，代币不可被复制或被使用多次或被悬挂（dangling）。此处可以把资源的操作类比成c++中的唯一指针。

#### 先字节验证，后合约执行

Move 在设计时就设计为一种可执行的字节码语言，其具有内置的安全算法和字节码验证器（Bytecode verifier），可以防止许多常见错误。即 **Move 中的合约代码要能被执行，必须先被验证**，这使得合约可以免受编译器的潜在故障和可能遭遇到的攻击。

Krešimir Klas在其《Smart Contract Development — Move vs. Rust》中表示：“Move 的显著特征是可执行的字节码表示，为所有程序提供了资源安全保证。考虑到合约的开放部署模型，这一点至关重要——回想一下，任何合约都必须容忍与不可信代码进行任意交互。如果源码级线性可以被可执行级别上不受信任的代码违反，那么其价值就很有限。”

#### 静态调用

在区块链中，合约的调用方式可以分为静态调用和动态调用。若程序调用必须在运行时才能确定被调用的目标，则称该调用为动态调用；反之，在运行前即可确定被调用目标，且在运行时无法变更该目标，则称该调用为静态调用。

Move 采用静态调用方式。即**Move实现的合约在部署时，其执行逻辑已经被确定。那么我们可以通过静态分析字节码，得到合约所有可能路径上操作的状态，在区块浏览器或钱包里提示给用户。**

因此，钱包提供商可以在钱包设计中，在预执行合约时把合约执行后的状态变更提示给用户，让用户可以知道这个交易操作了自己的哪些重要资产，以及执行后的结果。如下图，在StarMask中的实现效果：

![Untitled](/images/posts/blockchain/move_1.png)

图源自jolestar.eth

## Move 与 Solidity 的比较

### 账户模型

**Solidity：**

在大多数以太坊 ERC-20 合约中，每个地址的余额都存储在一个类型的状态变量`mapping(address => uint256)`中，该状态变量存储在特定智能合约的全局存储中。其结构如下图所示：

![Untitled](/images/posts/blockchain/move_2.png)

图源自[https://github.com/move-language/move/tree/main/language/documentation/tutorial](https://github.com/move-language/move/tree/main/language/documentation/tutorial){:target="_blank"}

**Move：**

Move 中，类比 solidity 智能合约的 **模块Module** 没有自己存储空间的。相反，Move 的“全局存储”是由地址索引的，每个地址下存储了 Move 模块和 Move 资源，而资源存储是类型到值的映射。其结构如下图所示：

![Untitled](/images/posts/blockchain/move_3.png)

图源自[https://github.com/move-language/move/tree/main/language/documentation/tutorial](https://github.com/move-language/move/tree/main/language/documentation/tutorial){:target="_blank"}

### 代码存储

**Solidity：**

在基于EVM的链中，所有智能合约都有一个独特的地址，称为 "合约拥有地址"。合约账户与部署者账户（EOA）存在于在同一级别，代码hash存储在合约账户地址中，合约部署后不与部署者地址绑定。

![Untitled](/images/posts/blockchain/move_4.png)

**MOVE：**

在基于 MoveVM 的链中，代码存储在 Account resource 的 code module 里面。

![Untitled](/images/posts/blockchain/move_5.png)

### 安全隔离

**Solidity：**

智能合约的运行环境是链的节点给构造出的沙箱环境，多个合约程序是运行在同一个进程内的不同的虚拟机沙箱。智能合约之间的调用是同一个进程内不同的智能合约虚拟机之间的调用，安全完全依赖于智能合约虚拟机之间的隔离。

![Untitled](/images/posts/blockchain/move_6.png)

**Move：**

Move 的做法则是通过 MoveVM 让采用 Move 语言的区块链具备确定性，将合约调用放在同一个虚拟机沙盒中，通过编程语言内部的安全性对智能合约的状态进行隔离，而非依赖虚拟机进行隔离。

![Untitled](/images/posts/blockchain/move_7.png)

### 合约升级

**Solidity：**

EVM 中合约升级的方法是将合约数据和逻辑分析：代理合约（Proxy) 负责转发交易到逻辑合约，并保存合约数据；逻辑合约（Logic）负责实现功能逻辑。升级时，只需要重新部署新版本的逻辑合约，并将代理合约中的逻辑合约实例指向新版本逻辑合约实例即可。此时，逻辑合约升级并不会影响合约原来已有的数据。

如下图，代码存储字段指定的是代理能合约被调用时做 `delegate call` 的合约代码，合约升级本质上是部署一个新的逻辑合约，并改变 code 字段以重定向 delegate call。

![Untitled](/images/posts/blockchain/move_8.png)

**Move：**

Move 语言中对于合约升级，其实现是在系统模块`code.move`中执行升级逻辑，在代码部署前检查升级策略和兼容性。在兼容性检查后，写在 resource 中的代码通过一个**原生函数**调用被替换，并将执行新的逻辑。

![Untitled](/images/posts/blockchain/move_9.png)

图截取自[https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/code.move#L132](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/code.move#L132){:target="_blank"}

## Move 程序可能出现的漏洞点

1） 开发者在使用 Aptos、Sui，或者其他基于 Move 的 blockchain 中独特的 Framework 进行开发时，应保持一定程度的安全意识，确保**供应链安全**。

2） 函数**权限问题**。对于一些函数调用的权限要仔细划分，因为一些关键函数会涉及到治理，严重的会影响到资金安全，针对这种函数调用需要对调用者进行鉴权。

3） 业务逻辑在设计和代码实现时，均需注意其中的**逻辑问题**。

4） 关于Move系项目，在**模块升级**时需要注意的点：

- 根据合约升级政策，代码的所有者对升级权限有完全的控制权。
- 代码的 owner 在初始部署后是不可改变的。
- 部署者的地址在部署后永远拥有升级权限。

---

> 参考文献
>
> [Move Paper](https://developers.diem.com/docs/technical-papers/move-paper/){:target="_blank"}
>
> [加密后浪强势来袭：Move 将如何书写新篇章](https://mirror.xyz/asmp.eth/xUzqdBXewRPhLKLAYekiwU5S9uFiTnJ3f56KbkcOz-M){:target="_blank"}
>
> [为什么是 Move 之编程语言的生态构建](https://mirror.xyz/jolestar.eth/sQ0nMCO3eNig6gCzqQO7xew1mn8oUi1-rKtfZKmGlNI){:target="_blank"}
>
> [Smart Contract Development — Move vs. Rust](https://medium.com/@kklas/smart-contract-development-move-vs-rust-4d8f84754a8f){:target="_blank"}
