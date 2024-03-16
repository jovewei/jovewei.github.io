---

layout: post
title:  链平台审计-Clique POA共识链环境搭建
subtitle: 最近在审计某个POA共识的链，记录一下Clique POA 链平台环境搭建的避坑经验
date:   2024-01-24 22:39:00 +0800
author: "Bryce"
header-img:  'images/gallery/Remarkable-Views-of-Bridges-in-Various-Provinces.jpg'
tags:   [区块链, 审计, POA]
---

> <cite> Title: 諸國名橋奇覧　飛越の堺つりはし|The Suspension Bridge on the Border of Hida and Etchū Provinces (Hietsu no sakai tsuribashi), from the series Remarkable Views of Bridges in Various Provinces (Shokoku meikyō kiran)  
Creator: Katsushika Hokusai  
Date Created: ca. 1830  
Physical Dimensions: H. 10 1/8 in. (25.7 cm); W. 15 1/8 in. (38.4 cm)  
Type: Woodblock print  
Medium: Polychrome woodblock print; ink and color on paper  
Repository: Metropolitan Museum of Art, New York, NY  
Period: Edo period (1615–1868)  
Culture: Japan </cite>

链平台审计时，首先先把平台环境搭建起来，在本地把私链成功跑起来，包括挖矿节点、数据同步节点等重要节点，以及挖矿节点的新增和退出等基本操作

本文主要记录采用 Clique POA 共识的，Fork Ethereum 的链平台的环境搭建过程和避坑经验

## 编译 geth

该链是fork Ethereum的，所以前期准备过程也eth搭建私有链的过程基本一致

首先，git clone 代码，这里就以[以太坊](https://github.com/ethereum/go-ethereum)为例

然后，按照文档说明，进行编译：

```bash
make all
```

或者

```bash
make geth
```

编译成功后，可以在 `./build/bin/` 目录下找到可执行程序 `geth`，运行：

```bash
./build/bin/geth version
```

出现版本信息表示编译运行成功

![Geth Version](/images/posts/blockchain/chainaudit_poa_1.png)

接着将 `geth` copy到本地执行环境中（Ubuntu）:  

```bash
sudo cp ./build/bin/geth /usr/bin/
```

## 运行私有链

### 创建账户

规划创建1个miner，3个数据同步node

先创建4个账户目录

![node](/images/posts/blockchain/chainaudit_poa_2.png)

依次运行新建账户的命令，为每个节点新建账户，密码要记住，后面需要密码授权：

```bash
geth --datadir ./miner1/data account new

geth --datadir ./node1/data account new

geth --datadir ./node2/data account new

geth --datadir ./node3/data account new
```

![new_account](/images/posts/blockchain/chainaudit_poa_3.png)

博主这里新增的账号分别为：  

miner1:  
0x461665a4Ce5eAfEB20107E14F066b8F6A67aB34D

node1:  
0x1A84945C8543efFCE763E8AF29DaBfC3EB315E18

node2:  
0xa5baaF6C4C7AfcA68404409D9b146C338C5F12E8

node3:  
0xF4D658c0700F77362A80e0708e28f4f039320e14

### 设置创世文件

接下来需要为链设置创世文件才能启动，这里就不在重新创建，直接使用下面的模板，需要注意修改以下几个变量的值：

- `chainId`  
  链ID，根据自己的需求修改

- `extraData`  
  这里只要是添加初始出块的节点账号信息，博主这里添加了miner1节点的地址

- `alloc`  
  这里主要是初始token的分配，博主这里给miner1和node1分配初始分配了一定的token

genesis.json

```json
{
  "config": {
    "chainId": 1947,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "clique": {
      "period": 15,
      "epoch": 30000
    }
  },
  "nonce": "0x0",
  "feePerTx": 100000000000,
  "proposedFee": 200000000000,
  "votes": 11,
  "timestamp": "0x6597c0db",
  "extraData": "0x0000000000000000000000000000000000000000000000000000000000000000461665a4Ce5eAfEB20107E14F066b8F6A67aB34D0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "gasLimit": "0x47b760",
  "difficulty": "0x1",
  "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "alloc": {
    "461665a4Ce5eAfEB20107E14F066b8F6A67aB34D": {
      "balance": "0x200000000000000000000000000000000000000000000000000000000000000"
    },
    "1A84945C8543efFCE763E8AF29DaBfC3EB315E18": {
      "balance": "0x200000000000000000000000000000000000000000000000000000000000000"
    }
  },
  "number": "0x0",
  "gasUsed": "0x0",
  "baseFeePerGas": null
}
```

### 根据创世文件初始化节点

```bash
geth --datadir miner1/data init genesis.json

geth --datadir node1/data init genesis.json

geth --datadir node2/data init genesis.json

geth --datadir node3/data init genesis.json
```

如下则成功：  

![init](/images/posts/blockchain/chainaudit_poa_4.png)

### 启动节点

#### 启动挖矿节点

miner1 :  
先进入到miner1目录下，然后运行，输入密码，成功启动

```bash
geth --datadir ./data --port 31000 --unlock 461665a4Ce5eAfEB20107E14F066b8F6A67aB34D --http  --http.corsdomain  "*" --http.port 9545 --http.api "db,eth,net,web3" --networkid 1947 --identity "VarnarChain"  --http.addr "0.0.0.0" --allow-insecure-unlock --miner.etherbase=0x461665a4Ce5eAfEB20107E14F066b8F6A67aB34D --nodiscover  --authrpc.port "8880" console
```

解释几个重要的参数：  

- `--datadir`：数据库和keystore密钥的数据目录
- `--port`：本地服务端口
- `--unlock`：需解锁账户，用逗号分隔。这里我们主要解锁启动节点自身
- `--http.corsdomain  "*"`：允许跨域请求的域名列表(逗号分隔)，`*`表示允许所有的跨域请求
- `--http.port`：HTTP-RPC服务器监听端口(默认值:8545)
- `--http.addr "0.0.0.0"`：HTTP-RPC服务器接口地址，`0.0.0.0`表示任何地址都能访问
- `--networkid`：指定链ID，创世文件中已经指定，这里也可以不要
- `--miner.etherbase=0x382884530E675b7ebE81B6c35621b83176010449`：挖矿奖励地址，这里指定miner1
- `--nodiscover`：禁用节点发现机制(手动添加节点)
- `console`：启动交互命令行

启动成功结果如下：

![miner](/images/posts/blockchain/chainaudit_poa_5.png)

然后，运行挖矿命令：

```bash
miner.start()
```

![miner_start](/images/posts/blockchain/chainaudit_poa_6.png)

#### 启动数据同步节点

新开终端页签，进入到node1\2\3目录下，分别执行（解锁地址都改为执行命令的节点地址）  

node1:  

```bash
geth --datadir ./data --port 31001 --unlock 1A84945C8543efFCE763E8AF29DaBfC3EB315E18 --http  --http.corsdomain  "*" --http.port 9546 --http.api "db,eth,net,web3" --networkid 1947 --identity "VarnarChain"  --http.addr "0.0.0.0" --allow-insecure-unlock  --nodiscover  --authrpc.port "8881" console
```

node2:  

```bash
geth --datadir ./data --port 31002 --unlock a5baaF6C4C7AfcA68404409D9b146C338C5F12E8 --http  --http.corsdomain  "*" --http.port 9547 --http.api "db,eth,net,web3" --networkid 1947 --identity "VarnarChain"  --http.addr "0.0.0.0" --allow-insecure-unlock  --nodiscover  --authrpc.port "8882" console
```

node3:  

```bash
geth --datadir ./data --port 31003 --unlock F4D658c0700F77362A80e0708e28f4f039320e14 --http  --http.corsdomain  "*" --http.port 9548 --http.api "db,eth,net,web3" --networkid 1947 --identity "VarnarChain"  --http.addr "0.0.0.0" --allow-insecure-unlock  --nodiscover  --authrpc.port "8883" console
```

启动成功如图：

![node_start](/images/posts/blockchain/chainaudit_poa_7.png)

### 数据节点添加挖矿节点，进行p2p通信同步更新区块信息

数据节点虽然启动成功，但是并没有同步挖矿节点的数据，所以这里需要添加p2p通信，获取miner1的数据

先在miner1的运行成功日志中找到这一行：

```bash
INFO [01-25|16:30:16.661] Started P2P networking                   self="enode://307e8621ac5bb1f2635c876770e9dff35113e8a59773b27c0222f1e2a36c9483b273a85654997e76fb2be2a0a77cac15c26702e79934f277736d1afa476c9750@127.0.0.1:31000?discport=0"
```

`self`的值即为miner1的p2p地址，copy一下，放在下面命令中，

```bash
admin.addPeer("enode://307e8621ac5bb1f2635c876770e9dff35113e8a59773b27c0222f1e2a36c9483b273a85654997e76fb2be2a0a77cac15c26702e79934f277736d1afa476c9750@127.0.0.1:31000?discport=0")
```

分别在 node1、node2、node3 中运行

![node_start](/images/posts/blockchain/chainaudit_poa_8.png)

可以看到已经同步了miner1产出的区块信息

至此，私有链成功跑起来了！

## 在稳定运行的私有链上新增和删除挖矿节点

### 新增一个miner2账户

新增一个miner2文件夹，并添加账户：  

```bash
geth --datadir ./miner2/data account new
```

miner2:  
0xF2173115475BA224b630bEDcce0bdbDDAc606202

![miner2](/images/posts/blockchain/chainaudit_poa_9.png)

### 根据创世文件初始化新挖矿节点

同上一样，运行：  

```bash
geth --datadir miner2/data init genesis.json
```

### 启动新挖矿节点

在miner2文件夹下运行：  

```bash
geth --datadir ./data --port 31010 --unlock F2173115475BA224b630bEDcce0bdbDDAc606202 --http  --http.corsdomain  "*" --http.port 9555 --http.api "db,eth,net,web3" --networkid 1947 --identity "VarnarChain"  --http.addr "0.0.0.0" --allow-insecure-unlock --miner.etherbase=0xF2173115475BA224b630bEDcce0bdbDDAc606202 --nodiscover  --authrpc.port "8890" console
```

启动成功，记录下miner2的p2p地址，如红框所示

![miner2](/images/posts/blockchain/chainaudit_poa_10.png)

### 可信节点进行新节点的授权投票

在miner1节点上对新增的miner2进行授权投票（ _至少要一半以上的节点投票通过_）：  

```bash
clique.propose("0xF2173115475BA224b630bEDcce0bdbDDAc606202", true)
```

这里只有miner1一个节点，所以直接可以通过

使用`clique.getSigners()`可以查询当前的签名节点，如图可以看到，已经添加成功

![miner2](/images/posts/blockchain/chainaudit_poa_11.png)

这时候miner2启动`miner.start()`也不会挖矿，原因是挖矿节点之间没有建立p2p通信

#### 在新旧挖矿节点之间互增p2p通信

根据上面记录的p2p地址，在miner1和miner2中添加彼此为通信节点

miner1的console中执行：  

```bash
admin.addPeer("enode://9c50add41757b104d611bdaf2ac9718ede93e502f2783d38c8c4076330f54aa49dc9da5a83e94dc90df84ecca8f9f8ae0ce8ea12093394e754e9ce1a200e4901@127.0.0.1:31010?discport=0")
```

miner2的console中执行：  

```bash
admin.addPeer("enode://307e8621ac5bb1f2635c876770e9dff35113e8a59773b27c0222f1e2a36c9483b273a85654997e76fb2be2a0a77cac15c26702e79934f277736d1afa476c9750@127.0.0.1:31000?discport=0")
```

此后，miner2启动挖矿：

```bash
miner.start()
```

![miner2_start](/images/posts/blockchain/chainaudit_poa_12.png)

可以看到，两个节点交替出块，至此，添加新挖矿节点成功

### 删除挖矿节点

_注意：网上很多资料都是错误的，删除节点的方法不是 clique.discard ！而是使用 clique.propose(address, false) 投票删除！_

在miner1和miner2中都要执行，因为必须要投票大于50%，这里自己也可以给自己投票删除

```bash
clique.propose("0xF2173115475BA224b630bEDcce0bdbDDAc606202", false)
```

![delete_miner2](/images/posts/blockchain/chainaudit_poa_13.png)

可以看到，经过一个区块后，验证者只剩下miner1，删除成功

miner2挖矿变成了未授权的验证者，出块失败

这里顺便看看`discard`的代码：

```go
func (api *API) Discard(address common.Address) {
 api.clique.lock.Lock()
 defer api.clique.lock.Unlock()

 delete(api.clique.proposals, address)
}
```

可以看出，`discard`方法主要是用来删除提案，而不是删除节点

## remix 连接本地RPC

### 通过 Metamask 钱包连接

#### 在 Metamask 钱包中配置本地网络

进入 Metamask 插件钱包的添加网络页面，填好信息后，如果出现下图 _无法添加_ 的情况，那就是RPC和Chain ID不匹配

![metamask1](/images/posts/blockchain/chainaudit_poa_14.png)

博主这里就是RPC地址错了，在虚拟机中启动的RPC，在宿主机的浏览器添加时，需要改成虚拟机地址

![metamask2](/images/posts/blockchain/chainaudit_poa_15.png)

#### 在 remix 中连接 Metamask

"Deploy & Run Transactions" 页签的`Environment`选项中选择`Injected Provider`

![delete_miner2](/images/posts/blockchain/chainaudit_poa_16.png)

授权连接后就能看到`Account`中已经导入了Metamask中的账户了

![delete_miner2](/images/posts/blockchain/chainaudit_poa_17.png)

**注意，如果使用浏览器插件钱包，就不要选择`WalletConnect`**，`WalletConnect`主要是连接移动端和桌面端钱包的，博主在这里遇到了坑

### Remxi 直接连接本地网络

"Deploy & Run Transactions" 页签的`Environment`选项中选择`Custom-External Http Provider`

![delete_miner2](/images/posts/blockchain/chainaudit_poa_18.png)

然后输入本地（虚拟机）RPC地址和端口

![delete_miner2](/images/posts/blockchain/chainaudit_poa_19.png)

但是，这里有几个坑需要注意下：

- 如果本地使用虚拟机启动的链，则需要在启动链时添加参数`--http.addr "0.0.0.0"` 和 `--http.corsdomain  "*"`

- 如果本地使用虚拟机启动的链，这里在输入地址时，就不是`localhost`或`127.0.0.1`，而是你虚拟机相对于主机的地址，这个自行查看

- 如果仍然无法连接，则可以尝试将 remix网站 和 RPC网站的 http 协议保持一致：将 `https://remix.ethereum.org/` 改成 `http://remix.ethereum.org/`

## Tips

**p2p通信信息不会保存**

私有链环境中，如果节点（miner节点和数据节点都是）exit后，再重新start，需要重新添加p2p通信，否则无法同步数据

## 总结

至此，Clique POA共识链环境搭建已经搭建完成，接下来就可以通过remix进行合约运行测试，也可以后台查看链平台的运行日志

总体来说，过程并不复杂，但是有些过程需要自己实践才能出真知，网上的资料也不一定都对
