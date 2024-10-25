---

layout: post
title:  Scroll深度剖析：架构、技术与zkEVM电路审计
subtitle: 10月10日，以太坊二层解决方案 Scroll 主网产生了第一个区块
date:   2023-11-05 19:45:00 +0800
author: "Yewbs"
header-img:  'images/gallery/four-immortals-paying-homage-to-longevity-shang-xi-15th-c.jpg'
tags:   [区块链, ZKP, Scroll]
---

> <cite>Title: Four Immortals Paying Homage to Longevity  
Creator: Shang Xi (15th c.)  
Date Created: AD 1368-AD 1644  
Type: Painting  
Rights: National Palace Museum  
Dynasty: Ming dynasty  
</cite>  

10月10日14:00，以太坊二层解决方案 Scroll 主网产生了第一个区块，标志着 Scroll 的主网成功上线。截止到10月25日，已有超过7600个ETH通过跨链桥进入 Scroll 网络，24个去中心化交易平台上线 Scroll 主网，总TVL约1千万美元。

10月17日，Scroll 官方在宣布主网启动的同时，继续坚持开源和去中⼼化的承诺。下⼀阶段 Scroll 将致力于构建去中⼼化的证明⽹络和排序器。**本文将详细分析 Scroll 的架构、技术和 Scroll zkEVM 电路与审计知识**

## 整体架构

Scroll 的架构分为以下三个主要组件：

**Scroll Node**：它根据用户交易产生 Scroll 网络的区块，将这些交易提交到以太坊基础层，并处理以太坊和 Scroll 之间的消息传递。

**Roller**：Roller负责将智能合约转化成 zkEVM 电路，随后生成证明，用于证明交易的正确性。在 Scroll 网络中存在多个Roller，并行处理并通过硬件加速以减少生成证明的时间。Scroll 由于直接证明EVM的字节码处理正确与否，因此对于EVM做到了字节码层面的兼容。

**Rollup 和 Bridge Contract**：这些合约为 Scroll 的交易提供数据可用性，验证 zkEVM 生成的有效性证明。可以说，Scroll 是通过 Rollup 合约和 Bridge 合约和以太坊基础层连接的。通过这些合约，用户可以在以太坊和 Scroll 之间传递任意消息，并且借助网关合约向任一方向转移ERC-20资产。

![scroll_1](/images/posts/blockchain/scroll_1.png)

source: [An overview of Scroll's architecture](https://scroll.mirror.xyz/nDAbJbSIJdQIWqp9kn8J0MVS4s6pYBwHmK7keidQs-k){:target="_blank"}

Scroll 在以太坊部署的主要合约：

**网关路由代理合约**（保证代币在跨链操作中的正确映射）：0xF8B1378579659D8F7EE5f3C929c2f3E332E41Fd6

**消息代理合约**（传递L1与L2之间的消息）：0x6774Bcbd5ceCeF1336b5300fb5186a12DDD8b367

值得注意的是以上合约可由代理管理员（Proxy Admin）和持有者（Owner）进行修改。此外，Scroll 还设置了白名单功能，可以改变特定地址在 Scroll 的gas费。Scroll的排序器目前是中心化的，可以审查 Scroll 网络的消息和交易，并且有可能跳过消息队列中的任何消息直接最终确认某一消息。

## Scroll zkEVM工作流程

Scroll 生成区块后，会经过协调器（Coordinator）和多个证明器（Roller）后生成聚合证明，然后提交给以太坊上的 Rollup 合约进行验证。详细过程如下：

![scroll_2](/images/posts/blockchain/scroll_2.png)

1、排序器获得新的交易后，虚拟机会读取该笔交易相关的字节码，生成对应的执行踪迹(Execution Trace)并将其发送给协调器。同时，排序器还将交易数据提交给Rollup合约。

2、Roller先将从协调器接收到的执行踪迹转化为 zkEVM 电路。每一步执行踪迹都有对应的 zkEVM 电路，此处对于某些zk不友好的函数（hash，Keccak），Scroll 通过构建查找表，将执行踪迹中此类函数的输入和输出映射到查找表，利用额外的电路来验证查找表的正确性。随后 Roller 为这些 zkEVM 电路生成对应的证明。

3、在生成证明后，Roller 将其发送回协调器。每经过若干个区块，协调器将聚合任务随机分配给某一 Roller，然后该 Roller 将若干个区块的证明聚合成单个证明。

4、最后，协调器将聚合证明提交给 Rollup 合约，Rollup合约再根据聚合证明来验证之前提交给 Rollup 合约的状态和交易数据，确定区块的正确性。

## Scroll zkEVM电路与审计

### 主要电路

zkEVM 由许多电路组成，每个电路负责检查EVM的某个方面。这些电路最后以某种方式进行聚合或组合，共同完成交易执行的证明。下图显示了这些电路和表格之间的关系：

![scroll_3](/images/posts/blockchain/scroll_3.png)

其中有一些较小的子电路，例如 ECDSA 电路和操作码相关的子电路，不会以影响电路的组合方式与其他表和电路交互，因此为了清楚起见，它们没有在图中显示。

#### EVM Circuit

以太坊虚拟机（Ethereum Virtual Machine，简称EVM）是一个状态机，它定义了以太坊协议中有效状态转换的规则。这意味着它规定了一个确定性函数，根据当前的EVM状态计算下一个有效的EVM状态。

EVM的执行部分使用操作码（opcodes）来实现这些状态转换，从而产生执行轨迹（Execution trace）。EVM电路的目标是构建与执行轨迹相对应的约束系统，可以通过后端的零知识证明系统进行证明。

EVM电路的高级设计思想在某种程度上类似于EVM本身的设计（例如go-ethereum）。在go-ethereum中，解释器循环遍历执行轨迹上的所有指令操作码。在每个指令中，解释器帮助检查相关的上下文信息，如gas、堆栈、内存等，然后将操作码发送到JumpTable，从中获取该操作码应执行的详细操作。

类似地，在EVM电路中，Scroll 根据执行轨迹中的步骤构建执行步骤，并为操作码和执行上下文提供证明。对于每个执行步骤，会施加一组约束来检查上下文信息。对于每个操作码，会施加一组约束来检查操作码的行为。

在执行轨迹中，相同的操作码应具有相同的约束。Scroll使用选择器来“打开”执行轨迹中相同操作码的所有步骤，并使用后端的证明系统证明它们的行为。

#### State Circuit

在执行过程中，EVM 的所有读写操作都记录在 rw_table中，并按计数器变量 rw_counter排序。而state Circuit的目的就是证明正确的生成了rw_table。

#### MPT Circuit

Merkle Patricia Tree 是以太坊存储层使用的关键数据结构之一。在 Scroll 的 zkevm- Circuits 中，将原始 MPT 修改为zkTrie，它本质上是一个稀疏二进制 Merkle Patricia Trie。在 zkevm- Circuits 中，Scroll 使用 MPT 表来逐步跟踪 MPT 操作的状态转换。MPT 表具有以下表布局：

![scroll_4](/images/posts/blockchain/scroll_4.png)

MPT 电路的目标是验证上述 MPT 表的正确性，即确保MPT表中记录的每次更新都会导致正确的更改。为了实现这个目标，MPT 电路使用约束系统来强制执行 MPT 的唯一更改。

这意味着对于 MPT 表中的每个更新，MPT 电路会确保只有一种可能的更改方式。这样可以防止意外或非法的更改，并确保 MPT 的完整性和正确性。

特别的，当 MPT 由于帐户或存储的更新而发生更改时，MPT 电路必须证明这次更新会导致正确的根更改。这意味着 MPT 电路需要验证更新操作是否按照规定的规则进行，并且确保根哈希正确地反映了所有更改的结果。

#### Keccak Circuit

Scroll 在遵循[NIST Keccak 规范](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf){:target="_blank"}、[Keccak 团队 Keccak 规范](https://keccak.team/files/Keccak-implementation-3.2.pdf){:target="_blank"}的条件下实现了他们自己的[Keccak256](https://github.com/scroll-tech/zkevm-circuits/tree/scroll-stable/keccak256){:target="_blank"}。

而 Keccak 电路则用于证明 Keccak256 运算结果的正确性。这部分电路的实现复杂，主要因为 keccak256 算法本身就是zk-unfriendly的。

#### Tx Circuit

Tx电路提供了验证交易正确性的约束条件。它主要检查交易的以下几个方面：

1. CallDataLength和累积CallDataGasCost的正确性：通过自定义门和查找tx表中tx的最后一行call data字节；

2. TxSign和TxHash相关数据的正确性：通过查找RLP表和Keccak表；

3. 证明 “若tx_type为L1Msg，则msg_hash” 的正确性：通过查找RLP表进行验证；

4. 通过ECDSA正确执行tx签名，并且能够正确的从ECDSA签名中恢复调用者地址：通过查找sig表进行验证；

5. tx id、cum_num_txs和call_data_length等的正确过渡行为。

6. 一些基本约束，如一些指示变量的布尔值等。

#### Bytecode Circuit

EVM 电路需要查找存储正确字节码信息的字节码表。这确保了合约中存储的字节与表中加载的字节相同。而字节码电路的目的是约束上述字节码表的正确性。这包括：

1. 与标签（tag）的边界行为相关的约束：首行和末行的约束条件，从tag==byte转换到header以及反之的转换，从header转换到header的转换；

2. 约束代码大小：包括通过约束字节码的最后一个字节的索引来计算字节码的长度；

3. 约束代码哈希：对代码哈希中字节的RLC行为进行正确约束，并通过查找Keccak表来验证代码哈希；

4. 确保PUSH行为的正确性：is_code = push_data_left == 0（必须是布尔值），并通过查找push_table来确保PUSH1-PUSH32的推送数据大小；

5. 确保在一个字节码中每行的正确传播。

### 安全审计

不同的链拥有各自的自定义业务模块功能，这些模块通常会修改EVM中的预编译合约以及操作码，其中 Scroll zkEVM 作为一种基于零知识证明的二层扩容方案，**该方案使用电路重构了相关操作码并根据执行跟踪生成证明，这个复杂的实现极大增加了审计难度。Beosin安全专家评估后认为，目前 zkEVM 安全审计主要分为以下几个方面：**

1. GAS：zkEVM 电路在生成执行跟踪对应的证明时，会同时校验交易耗费gas的正确性。如果在操作码的实现电路中高频次地使用没有约束的自由变量，可能导致证明生成失败或其他未知错误。

2. 内存安全：部分 zkEVM 电路实现的数学基础是多项式承诺，如 Scroll 使用的KZG承诺。而多项式计算不会自动对齐，因此如果电路缺乏约束会导致取值域与计算机程序中的字节范围不一致，在部分合约开发者开启了gas优化的情况下，数据的紧凑排列可能导致内存安全问题，如 Polygon zkEVM 中的BYTE_C4096常数多项式。多项式允许参数的取值范围超过字节的最大取值范围255，这在一些采取AMM模式的交易所中，可能导致恶意的Sequencer伪造参数获利。本质上，这一类的漏洞都是由于电路表示的数值有效范围与程序的变量取值范围不一致导致的，如Beosin安全研究员在Snarkjs库中发现的漏洞CVE-2023-33252。

3. 操作码安全：zkEVM 操作码实现时，存在普遍的欠约束等安全问题，尤其是精度问题。例如底层电路在实现两个数的比较时，如果程序中比较运算的精度为1个字节，那么电路约束需要规定取值范围，否则电路中运算的精度将远超程序精度，导致结果错误。

4. 安全EIP支持：EIP-2、EIP-155等安全类eip的支持。

5. Sequencer中心化问题：目前 Scroll 生成的证明全部依赖Sequencer生成的执行跟踪，如果Sequencer作恶，zkEVM 无法保护用户资产安全。

6. 兼容性问题：zkEVM 根据执行踪迹生成电路证明并在合约验证，即使Sequencer进行微小的升级，也可能导致底层语言级别生成的执行踪迹存在较大差异。

## 附图

笔者后续在研究 scroll 的过程中，绘制了 scroll 的架构流程图，已分享至博客的 [diagram](/diagram) 区：

[![Scroll](/images/diagram/Scroll-t.png)](/images/diagram/Scroll.png)

---

> 参考链接：
>
> [An overview of Scroll's architecture](https://scroll.mirror.xyz/nDAbJbSIJdQIWqp9kn8J0MVS4s6pYBwHmK7keidQs-k){:target="_blank"}

> 备注：文章微改后发表于「Beosin」公众号
