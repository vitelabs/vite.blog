title: 浅析MPT在Ethereum中的应用
lang: zh-CN
date: 2018-08-21 17:59:08
tags:
---
# 序言
Ethereum的账本结构是一个单链结构(Block List)，是区块链最开始的数据结构，现在也衍生出一些其他数据结构，比如DAG（Directed acyclic graph）。在Ethereum中，需要通过“挖矿”产出一个block，“挖”一个block的权利是每个节点通过PoW机制公平竞争得到。每个block包含的数据可以参看Ethereum的黄皮书，这些数据里包含交易列表。交易列表在Ethereum中通过MPT（Merkle Patricia Tree）这种数据结构来表示的。MPT由Trie Tree和Merkle Tree演化过来，因此同时具备Trie Tree和Merkle Tree的特点。
<!-- more -->

# Trie Tree
为了更好地了解MPT，我们先来了解一下Trie Tree，Trie Tree是一种字典树，下图就是一个Trie Tree结构：
![upload successful](/images/pasted-lyd00-15.png)


Trie Tree非常简单且易于理解。上图中的Trie Tree表示["AOT"、”BR"、"DCA"、”DCB"]这个字符串集合，从根节点到叶子节点的路径表示字符串集合中的一个元素，如最左边那条路径就表示”AOT"。Trie Tree用来组织key-value数据也是很好的。key是字符串，用根节点到叶子节点的路径表示，value放在叶子节点里，或者在叶子节点里通过指针指向value。通过key搜寻value是常数级的时间复杂度。



# Patricia Tree
在Ethereum中，为了节省空间，实际上使用的是优化过的Trie Tree，即Patricia Trie（压缩前缀树），如下图所示:

![filename already exists, renamed](/images/pasted-lyd00-15.png)

Patricia Trie在Trie Tree基础上，对于只有单个子节点的父节点，将其与子节点进行了合并，优点是更加节省空间，缺点是在进行插入操作的时候比Trie Tree稍微复杂一些，在某种情况下需要做节点的分割，如在上面的树结构中插入一个字符串”AOB”，那么“AOT”这个节点就要被拆分成“AO”和“T”两个节点，树结构就要变成的下面的样子:
![filename already exists, renamed](/images/pasted-lyd00-16.png)


Patricia Trie会比Trie Tree更省空间一些。

# Merkle Tree

另一个数据结构是Merkle Tree（默克尔树），Merkle Tree也是区块链中一个十分重要的数据结构，如在SPV（Simplified Payment Verification）中就有所应用。下面也简单介绍下Merkle Tree，Merkle Tree的树结构大致如下：
![filename already exists, renamed](/images/pasted-lyd00-17.png)


Merkle Tree也叫哈希树，是一种二叉树。树中每个节点存储的是Hash值，叶子节点引用数据集合中真正的数据，如上图表示的数据集[A、B、C、D]。叶子节点的Hash值是由数据通过哈希函数计算得到，父节点的Hash值是由其两个子节点的Hash值做字符串拼接再通过哈希函数计算得到，最后递归可计算出树跟节点的Hash值。Merkle Tree的特点是，一旦某个叶子节点的数据发生了变更，根Hash就会发现变化。

在根Hash值可信的情况下，想要验证某个数据是否可信，只需要拿到这个数据所在叶子节点到根节点路径上的相邻Hash值即可，如下图：
![filename already exists, renamed](/images/pasted-lyd00-18.png)


假设从可信源拿到H7（根Hash），从不可信源拿到数据B、H1、H6之后，就可以验证B的正确性。先通过Hash(H1+Hash(B))得到H5，再通过H7 == Hash(H5 + H6)判断数据B是否可信。因为哈希函数特性是数据的一点变更，就会导致生成Hash值的不可预知的大幅度变化，所以想要同时伪造B、H1、H6来最终计算得出H7是十分困难的，所以在H7可信的情况下，可以验证从不可信数据源取得B、H1、H6的正确性。这种验证正确性的问题属于零知识证明的范畴。

# Merkle Patricia Tree
Patricia Trie具备结构简单、易搜索的优点，Merkle Tree具备零知识证明的能力，MPT由Patricia Trie和Merkle Tree优化而来，同时具备这两者的特点。下图是一个MPT结构:

![filename already exists, renamed](/images/pasted-lyd00-20.png)


MPT本质上是一颗Patricia Trie，比较便于存储Key-Value数据集。同时又具备Merkle Tree的能力，MPT也可以递归计算得到根hash，任意一个节点的内容变化都会导致根hash发生变化。
在MPT中，把节点分为3类，分别是叶子节点（Leaf Node），分支节点(Branch Node)，拓展节点(Extension Node)。这三类节点有各自的作用，叶子节点存储真正的value或指向value的指针。分支节点具备路由功能，上图中分支节点有[0…f]共16个字符，不同字符对应一个指针，分别引用不同的子节点，在进行key搜索时，可以快速找到正确的子树并搜索下去。拓展节点是一种省空间的路由类节点，将只有一个子节点的父节点和其子节点合并，就可得到拓展节点，例如，ADGHJ这个Key，需要4个分支节点 + 1个叶子节点，其中A、D、G、H为4个分支节点，但是有拓展节点之后，可以将ADGH合并成一个拓展节点，变成了1个拓展节点 + 1个叶子节点，从而节省了空间。
在MPT树中如何计算根Hash值呢，看了Ethereum的源码之后发现过程和Merkle Tree非常类似。需要对树进行深度遍历，当遇到叶子节点是对其数据进行哈希计算得到Hash值。哈希计算过程是，先将叶子节点的数据对象序列化成二进制数组，对二进制数组进行哈希。分支节点的哈希计算过程是将其指向子节点的指针替换成子节点的Hash值，然后序列化成二进制数组，在进行哈希。拓展节点和分支节点计算Hash值的过程是一样的，将指向子节点的指针替换成子节点的Hash值，再序列化成二进制数组，然后进行哈希。深度遍历完成之后，可得到每个节点的Hash值(包括根Hash值）。当树中某个节点内容发生变化之后，不需要重新深度遍历所有节点来计算根Hash值，只需要重新计算这个节点到根节点的路径中的每个节点及其相邻节点Hash值即可，大家可以思考一下这个过程。

那么MPT能够有哪些应用呢，在Ethereum中，交易列表、收据列表、世界状态、合约数据是通过MPT来组织的，简而言之，Key-Value数据基本都是通过MPT来组织。
拿世界状态来说，世界状态在数据结构层面是一个key-value集合，key是账号地址，value是一个四元组即[nonce(交易数量)、balance（余额）、storageRoot（存储内容根Hash）、codeHash(代码Hash)]，每个字段具体的含义和作用可以去看Ethereum黄皮书。Ethereum的世界状态保存着这一刻全网所有账号的状态消息，某些账号的状态随交易发生或合约程序的执行会发生变化，利用MPT每次状态变化只需要少量计算即可得到新的根Hash值，并且有可信根Hash值之后，可以从不可信源获取某个账号的状态信息，并快速验证其真实性。MPT本质上也是一颗PT，因而给定一个账号地址，可快速搜索到这个账号的信息。
正是因为有了MPT，Ethereum在区块头中可以只存储交易列表根Hash、收据列表根Hash、世界状态根Hash、合约数据根Hash，将数据的详细信息可以和区块头分开存储及传输，这个技术可以用来做轻节点。
MPT在存储Key-Value数据集有其独特的优势，易搜索、易增量计算根Hash、支持零知识证明等。