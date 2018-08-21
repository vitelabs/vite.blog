title: 500行代码了解vite基本设计
lang: zh-CN
date: 2018-08-21 13:18:15
tags:
---
# 简介
从白皮书了解到，vite的核心账本结构是DAG，DAG中存储了每个用户的每笔交易信息。
为了提高整个DAG账本的数据安全性，vite首创提出了快照链，使用快照链进一步保障DAG的不可篡改性。
白皮书描述的账本结构比较抽象，本文通过简单的500行代码来实现vite中DAG的账本结构以及快照链。
<!-- more -->

# 核心概念
1. vite中DAG账本结构；
2. vite中快照链；

## vite中DAG账本结构

![Alt text](/images/pasted-viteshan-15.png)
vite在设计账本结构的时候，从降低伪分叉率和防篡改(vite白皮书中有详细解释定义)这两个角度做了一次权衡，上图是vite白皮书中描述的几种的区块链账本结构在伪分叉和防篡改两个方面的表现。

vite选择了block-lattice的DAG，下面来主要描述一下block-lattice的DAG结构。

![upload successful](/images/pasted-viteshan-16.png)

上图中，世界状态用S表示，S1中有三个初始的账户A、B、C，三个账户状态分别是A0、B0、C0，实线箭头方向表示这个相同账户的上一个账户状态，虚线箭头表示不同账户状态间的间接影响(如转账)，假设初始状态账户余额都是100元，

1. 初始世界状态：S1={A0=100, B0=100, C0=100}
2. 账户B向账户A转账20元之后，世界状态：S2={A1=120, B1=80, C0=100}
3. 账户A向账户C转账30元之后，世界状态：S3={A2=90, B1=80, C1=130}
在vite中，每一个账户状态就是一个区块(如图中A0、A1等)，区块之间的引用关系(即图中箭头)即账户间转账过程。

所有账户转账的历史流水，构成了一张有向无环图(DAG)，这就是vite的DAG账本结构。

## vite中快照链结构
由账户区块组成的DAG链，能够清晰的描述世界状态的变化。但是，由于每个账户链条长度与交易频率相关，这就导致了交易频率不够高的账户链条比较容易被篡改。

为了解决这个问题，vite中引入了快照链结构，通过快照链的不断增长来加大单纯账户链的防篡改度。

![upload successful](/images/pasted-viteshan-17.png)

上图中，描述了账户S1、S2、S3的世界状态，箭头的方向标识的是该状态的上一个状态。

其实这就是快照链，快照链中的每一个区块都能表示一个世界状态(所有账户状态的集合)，同时每一个区块都引用上一个区块。

当然，为了简化存储，vite的实际规范中，每个区块描述的是从上一个区块到这个区块变动的账户状态集合，如下图所示。而且区块的生成是固定频率，而不是每笔交易都去生成一个snapshot区块。

![upload successful](/images/pasted-viteshan-18.png)



# 代码实现
上面简单介绍了vite中DAG账本和snapshot链结构，下面会通过代码简单实现这两个数据结构，并简单实现转账过程和snapshot链生成过程。

## 数据结构
账本间的转账是通过交易进行的，下面定义了交易的数据结构：

![upload successful](/images/pasted-viteshan-19.png)

TxType共有两个类型，分别代表着发送交易和接收交易，与vite白皮书中一次转账会生成两个交易相对应。

账本账户状态的数据结构如下，可以看到，账户状态引用了交易数据tx，这个表示了每次账户状态变化都是由一个交易引起的。

发送账户会引用TxType为send的交易，接收账户会引用TxType为received的交易。

![upload successful](/images/pasted-viteshan-20.png)

账户状态会定期被快照链快照，snapshot block定义如下，Height代表snapshot block的高度，每次增加1，AccountsHash代表所有被缓存账户状态的hash。

![upload successful](/images/pasted-viteshan-21.png)

以上介绍的是组成vite的基本数据结构，下文的转账交易和snapshotblock生成都是在这几个数据结构上进行操作。

![upload successful](/images/pasted-viteshan-22.png)

这两个数据结构分别代表着vite账户链和快照链。

accoutStateBlockChain是一个map数据结构，key为每个账户的地址，value是一个AccountStateBlock链表，通过PreHash指向上一个block；

snapshotBlock是一个SnapshotBlock链表，也是通过PreHash指向上一个block；

## 转账过程
### 发送交易

![upload successful](/images/pasted-viteshan-23.png)
发送交易的核心代码如上图：

1. 构造发送交易，并对发送交易进行签名；
2. 打包账户状态块，并对打包的账户状态块进行签名；
3. 将账户状态块插入到账户链上，并将该区块结果进行广播；

### 接收交易

![upload successful](/images/pasted-viteshan-24.png)
会有一个协程等待广播事件产生，并验证block合法性，同时如果需要生成对应的received交易，将会将tx转发到对应的节点。

![upload successful](/images/pasted-viteshan-25.png)

对应节点收到send交易之后，验证ok之后：

1. 生成received的交易，并对交易进行签名；
2. 打包账户状态块，并对打包的账户状态块进行签名；
3. 将账户状态块插入到账户链中，并将该区块结构进行广播；

## 快照链快照过程

![upload successful](/images/pasted-viteshan-26.png)
挖矿账户会定期进行快照块的生成，并将快照块插入到snapshot链中。

# 总结

本文的主要目的是简单介绍naive-vite的主要流程，同时通过这个简单的demo来让大家了解vite的基本原理。

如果大家想了解更多细节，欢迎关注[vite官网](http://vite.org/)和[vite博客](https://vite.blog/)，同时欢迎watch [vite的第一版官方实现](https://github.com/vitelabs/go-vite)。

# 参考资料
- [vite白皮书](https://www.vite.org/whitepaper/vite_cn.pdf)
- [vite-naive源码](https://github.com/viteshan/naive-vite)
- [Block-tutorial](https://github.com/mycoralhealth/blockchain-tutorial)
