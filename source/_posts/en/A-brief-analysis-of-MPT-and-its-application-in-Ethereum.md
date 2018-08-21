title: A brief analysis of MPT and its application in Ethereum
lang: en
date: 2018-08-21 18:23:02
tags:
---
# Introduction
The data structure of Ethereum ledger is a blockchain list, which is the original data structure of  blockchain. Nowadays, some other data structures are derived such as DAG (Directed acyclic graph). In Ethereum, mining is a way to generate a block, the right to "mine" a block is that each node gets a fair competition through the PoW mechanism. The data contained in each block can refer to the Ethereum yellow paper, transaction list is included in these data. Transaction list is represented by a data structure called MPT (Merkle Patricia Tree) in Ethereum. MPT is evolved by Trie Tree and Merkle Tree, hence it has the characteristics of both of them.
<!-- more -->

# Trie Tree
To get a better understanding of MPT, we should get to know about Trie Tree first. Trie Tree is a kind of dictionary tree, the figure below is a Trie Tree structure:
![upload successful](/images/pasted-lyd00-14.png)


Trie Tree is extremely simple and easy to understand. The Trie Tree above represents the string collection of ["AOT"、”BR"、"DCA"、”DCB"], the path from root node to leaf nodes represents an element of string collection. For example, the leftmost path means "AOT". It is also good for Trie Tree to organize key-value pairs. Key is a string that represented by the path from root node to leaf nodes, value is put or points to value by pointer in leaf node. Searching for a value by key has constant-level time complexity.

# Patricia Tree
In order to save space in Ethereum, they actually use the optimized Trie Tree, which means Patricia Trie (Compressed prefix tree), as shown below
![filename already exists, renamed](/images/pasted-lyd00-15.png)


On the basis of Trie Tree, Patricia Trie merged child node with parent node who has only a single child node. The advantage is that it is more space-saving. However, the disadvantage is that it is slightly more complicated than Trie Tree when inserting. In some cases, it is necessary to do node segmentation, For instance, if we insert a string "AOB" in the above tree structure, then the node "AOT" will be split into two nodes "AO" and "T", the tree structure will be like the following figure:
![filename already exists, renamed](/images/pasted-lyd00-16.png)


Patricia Trie is a bit more space-saving than Trie Tree.

# Merkle Tree
Another data structure is Merkle Tree, an important data structure of blockchain which has been applied in SPV (Simplified Payment Verification). The following part is a brief introduction to the Merkle Tree. The structure of the Merkle Tree is as follows:
![filename already exists, renamed](/images/pasted-lyd00-17.png)


Merkle Tree is also called Hash Tree, a kind of binary tree. Each node of this tree stores hash value, and only leaf nodes have data contained just like [A、B、C、D] shown above. The value of leaf node comes from the hash calculation on its associated data while the value of parent node comes from the hash of the concatenated hash strings of two child nodes. The hash process repeats recursively on every node until the hash of root node is computed. As a result, in a Merkle Tree whenever a change occurs in any leaf node the root node hash will change accordingly, which is known as its most important feature.

Given that root node hash is trustworthy, it is easy to verify whether some data is reliable or not, by acquiring a few necessary hashes of sibling nodes who are in the path from the certain leaf node to root. As shown below:
![filename already exists, renamed](/images/pasted-lyd00-18.png)


For example, we could validate B by given H7 (root hash) from trusted source and B, H1, H6 from untrusted source. Firstly we compute H5 by Hash(H1+Hash(B)), then we could confirm data B's correctness by judging H7 == Hash(H5 + H6). One characteristic of hash function is that it leads to unpredicted and significant variation in the hash result even if source data has only a slight change, therefore it is extremely difficult to forge B, H1, H6 simultaneously to figure out H7. As a result, we can verify the correctness of B, H1, H6 as long as H7 is reliable. This kind of verification process can be categorized into the realm of Zero-knowledge proof.

# Merkle Patricia Tree
Since Patricia Trie has a simple structure and is easy to search and Merkle Tree supports zero-knowledge proof, MPT is evolved by optimizing Patricia Trie and Merkle Tree with the merits of both. The below figure shows a MPT structure:
![filename already exists, renamed](/images/pasted-lyd00-20.png)


MPT is actually a Patricia Trie which is easy to store key-value data set. Meanwhile it has the feature of Merkle Tree, figuring out the root hash by recursion, and the change of any node content will lead to the change of root hash.

In MPT, there are 3 kinds of nodes, Leaf Node, Branch Node and Extension Node. Each of them has its own functions. Leaf node stores the value itself or the pointer that points to the value. Branch node has function of router. For example, the figure above has 16 characters [0…f] in total and each character is mapped to one pointer referencing to an independent child node. It's rapid to locate the correct subtree to continue when searching for a key. The extension node is a space-saving router node which merges the parent node that has only one child node with its child node. For another example, key ADGHJ is composed of 4 branch nodes and 1 leaf node, where A, D, G, H are branch nodes. However, by involving the concept of extension node we can merge ADGH into 1 extension node, plus 1 leaf node J to save space.

Ethereum source code shows the root hash calculation in MPT is very similar to that in a Merkle tree. During a depth first search(DFS) traversal, data hash is calculated on the data associated with leaf node encountered, where the data object is serialized into a binary array and then the binary array is hashed. The hashing process on a branch node is to replace the pointers with the hash values of the child nodes, and then serialize them into a binary array for hashing. Extension node has the same hash process as branch node. After DFS is completed, the hash of each node is generated(including root node). Once any node in the tree changes data content, it is not necessary to traverse all the nodes for root hash value, but just only to recalculate the hash value of each node in the path from the node to the root node and its siblings.

MPT is widely used in Ethereum. Transaction lists, receipt lists, state of the world and contract data are all organized by MPT. To be brief, all key-value data are basically organized by MPT.

Taking the state of the world as an instance, it is represented as a key-value set structure. The key is account address and the value is a 4-tuple value: [nonce, balance, storageRoot, codeHash](the specific meaning of each field could refer to Ethereum yellow paper). In Ethereum, the state of the world stores status message of all accounts among the whole network for a certain moment. The status of accounts may change when transactions occur or smart contracts are running. MPT is employed to help generate new hash value with less calculation during status changing. Furthermore, account status from untrusted source can be swiftly verified as long as reliable root hash is present. In addition, MPT is essentially a Patricia Trie, so account status can be quickly searched by address. Thanks to MPT, Ethereum node is able to only store transaction list root hash, receipt list root hash, world state root hash and contract data root hash in block header, and store or transfer the detail data information and block header separately, making a lightweight node possible.

In summary, MPT has unique advantages in storage of key-value data, such as search-in-ease, incremental root hash calculation as well as zero-knowledge proof support.