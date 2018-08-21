---
title: vite p2p 网络
lang: zh-CN
date: 2018-08-21 17:26:56
tags: vite,p2p
---

# 简介

P2P 是 peer-to-peer 的简写，一般称为对等网络。有别于传统的 C/S 架构，P2P有以下特点:
1. 没有同一的调度中心和数据库，既去中心化
2. 节点具有相同的逻辑和地位<sup>1</sup>，既是 client 又是 server
3. 节点之间的连接都是不可靠的

<div style="text-align: center;display:flex;">
<div style="flex: 1;">
C/S 架构<br>
<img style="display:block;max-width:100%;" src="https://upload.wikimedia.org/wikipedia/commons/thumb/f/fb/Server-based-network.svg/400px-Server-based-network.svg.png">
</div>
<div style="flex:1;">
P2P 架构<br>
<img style="display:block;max-width:100%;" src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/3f/P2P-network.svg/400px-P2P-network.svg.png">
</div>
</div>

去中心化的区块链底层网络模型正适合采用 P2P 架构。

[1] 节点具有相同的逻辑是指在网络层面，应用层面当然是可以区分的，比如全节点、轻节点等。

# 结构化模型与 DHT

因为 P2P 网络是去中心化的，并没有集中的数据库，节点可以随时加入和离开。如何快速有效的定位某个资源就成了核心问题，即：内容路由。

针对这个问题，P2P 的网络模型经过了一些演化：集中式、纯分布式、混合式、结构化<sup>2</sup>。

- 集中式使用集中索引、分布存储，强依赖索引服务器，可靠性不够高。
- 纯分布式节点与资源随机分布、无序，容易形成洪泛和消息风暴。
- 混合式采用超级节点和普通节点结合，较为依赖超级节点。

结构化 P2P 网络一般基于分布式哈希表(DHT, Distributed Hash Table)，对资源进行 hash 得到资源 ObjectID，每个网络节点分配一个 NodeID。理想情况下 ObjectID 与 NodeID 一一对应，资源 i 存储在节点 i 上<sup>3</sup>。这样的话，节点 n 查询资源 i 时可以直接请求节点 i 。

或者可以根据 ID 进行某种规则排序，节点 n 如果没有节点 i 的信息，可以先请求节点 i 的邻居节点，这样也会减少路由跳数，提高效率。

结构化模型中，节点和资源进行了映射和划分，不再是随机分配，所以负载更加均衡、扩展性也更好。

[2] 网络模型参考: https://en.wikipedia.org/wiki/Peer-to-peer

[3] 实际可用将资源存储在节点及其邻居节点上，提高资源冗余性


# KAD

DHT 只是一种思想，基于这种思想有很多种具体实现: Chord<sup>3</sup>  Pastry<sup>4</sup>  CAN<sup>5</sup>  KAD<sup>6</sup> 等。

Vite 采用的是 KAD 算法，这也是目前主流的 DHT 算法。使用可配置的 ED25519 加密公钥作为 NodeID。


[3] Chord 参考: https://en.wikipedia.org/wiki/Chord_(peer-to-peer)

[4] Pastry 参考: http://www.freepastry.org/PAST/overview.pdf

[5] CAN paper: http://www.icir.org/sylvia/thesis.ps

[6] Kademlia paper: https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf


KAD 算法要点如下：
1. 使用 XOR 计算两个 NodeID 之间的距离（逻辑距离，而不是物理距离）。
2. 每个节点保存 N 个 K-bucket，N 是 NodeID 的 bit 位数，K 可以自行调整。第 i 个 K-bucket 内的节点与当前节点的距离介于 2<sup>i</sup> 和 2<sup>i+1</sup>。


<img style="display:block;max-width:70%;margin:15px auto;" src="https://upload.wikimedia.org/wikipedia/commons/thumb/6/63/Dht_example_SVG.svg/840px-Dht_example_SVG.svg.png">


所有的节点分配到 N 个子树内，从每棵子树内挑选 K 个节点存放到对应的 K 桶内。

当节点 A 请求资源 i 时，假如 A XOR i 等于 d。那么 A 就从第 log<sub>2</sub>d 个 K 桶内查找，如果有节点 i，就直接向节点 i 请求资源 i。

如果没有节点 i，那么可以并发的向这 K 个节点查询节点 i。这 K 个节点将自己的 K-bucket 内距离节点 i 最近的 K 个节点返回给 A，从而一步步锁定节点 i 的位置。

可见 KAD 路由的算法复杂度是 O(log n)

## KAD 总结

KAD 相比 Chord 等算法有以下优点：
1. 实现简单
2. 足够灵活（N K 并发请求数都可调整）
3. 扩展性容错性好

实际应用中还是要适当调整的，比如 Vite 节点 ID 的长度是 256，但是 K-bucket 数少于 256。

## 节点通讯

节点之间的消息有以下 4 种:
1. PING    测试一个节点是否在线
2. STORE   要求节点存储数据
3. FIND_NODE    查询某个节点
4. FIND_VALUE   查询某个资源，同 find_node

> 目前并没有 STORE 的需求，所以 vite 内最常用的是 PING 和 FIND_NODE. 区块的同步及存储实现在应用层，而不在 P2P 网络层。


## 节点发现

一个 Vite 新节点启动时，会向配置的 bootnodes 查询自己的位置。再向 bootnodes 返回的节点查询自己的位置。如此循环往复，直到没有更近的节点返回。如此就构建了自己的 K-buckets。


## 节点维护

由于节点在频繁的变动中，KAD 采用如下策略维护 K-bucket。
<ol>
<li>每个 bucket 内的节点按照最近联系的时间倒序排列，最近通讯的节点在最后面。</li>
<li>当一个节点与自己通讯时，检查对方是否在 bucket 内。
    <ol>
        <li>如果存在，那么将其移动到 bucket 尾部。</li>
        <li>如果不存在，且 bucket 没有满，则放到 bucket 头部。</li>
        <li>如果不存在，bucket 已满，则 PING 当前 bucket 的头结点（最久没有联系）。PING 通则抛弃新节点，PING 不通则使用新节点替换该头结点。</li>
    </ol>
</li>
</ol>

这个策略保证了结点的及时更新，并且越常在线的节点越不容易被替换，网络也相对稳定。

# 总结

Vite 的底层 P2P 网络采用的是相对主流的方案，在此之上构建了 Vite 的通讯协议（比如交易、区块的广播，节点同步等）。

欢迎大家参观我们的[代码](https://github.com/vitelabs/go-vite)，也请关注我们的后续文章，谢谢支持。


---

Our official website: https://www.vite.org/

**Telegram:**

* English: https://t.me/vite_en
* Chinese: https://t.me/vite_cn
* Russian: https://t.me/vite_russia
* Korean: https://t.me/vite_korean
* Vietnamese: https://t.me/vite_vietnamese
* Thai: https://t.me/vite_thailand

**Twitter:** https://twitter.com/vitelabs

**Discord:** https://discordapp.com/invite/CsVY76q