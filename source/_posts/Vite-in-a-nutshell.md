title: Vite in a nutshell
author: Vite Labs
tags: []
categories: []
date: 2018-08-16 09:46:00
---
# Introduction


As mentioned in our white paper, Vite uses DAG (directed acyclic graph) technology for its core ledger structure, which stores transaction information for every account.

In order to improve the data security of the whole DAG ledger and make it tamper-resistant, Vite has created the concept of Snapshot Chain.

Since the white paper is abstract, this article hopes to describe the core DAG ledger structure and snapshot chain with the help of 500 lines of code.

## Core Concept
1. The DAG ledger structure of Vite
2. The snapshot chain of Vite


### The DAG Ledger Structure of Vite

![Alt text](/images/pasted-viteshan-15.png)
Vite's design made a trade-off between two aspects, reduction of false fork ratio and tamper resistance.

The above figure shows the performances of false fork and tamper resistance in several blockchain ledger structures, also described in the white paper.

Vite decided on the block-lattice of DAG.  The following paragraphs describe the block-lattice of DAG structure in more detail.

![upload successful](/images/pasted-viteshan-16.png)

In the figure above, S stands for the state of the world, and there are 3 initial accounts A, B, C. The status of these accounts are A0, B0, C0. Solid arrows point from the current status of a given account to the previous status of said account, and dotted arrows indicate any indirect impact (e.g. caused by transfers) between the statuses of different accounts. Assume that in the initial status, the balance of accounts is $100 each,

1. Initial state of the world: S1={A0=100, B0=100, C0=100}
2. After account B transferred $20 to account A, state of the world : S2={A1=120, B1=80, C0=100}
3. After account A transferred $30 to account C, state of the world : S3={A2=90, B1=80, C1=130}


In Vite, every account status is a block (such as A0, A1, etc in figure); reference relationship among blocks (arrows in figure) is the transfer process among accounts.

The entire history of transactions among accounts is represented by a directed acyclic graph, or DAG.



### The Snapshot Chain Structure of Vite
The DAG chain, which is composed of account blocks, can explicitly describe the change of world status. However, the frequency of transactions for a given account affects the length of the relevant chain, and an account chain with infrequent transactions is easy to be tampered with.

To solve this problem,  Vite introduces snapshot chain that grows continuously, thereby increasing the tamper-resistant level of the block-lattice structure.

![upload successful](/images/pasted-viteshan-17.png)

The above describes the world status of 3 accounts: S1, S2, S3. The arrow direction shows the previous status of the current status. The above structure is basically the snapshot chain, where every block represents the state of the world (A set of all account statuses), and each block references the previous one. Of course, in order to reduce storage, each block only contains the change between successive states of the world, as shown below. Also, the blocks are generated on a fixed schedule; so not every transaction generates a snapshot block. 

![upload successful](/images/pasted-viteshan-18.png)




# Implementation of Code
The context above gave a brief introduction of DAG ledger and snapshot chain structure of Vite.  The following sections implement these two data structures as well as the process of transfer and snapshot generation via code.



## Data Structure
Transfers among ledgers are made through transactions, the code below defines the data structure of a transaction:

![upload successful](/images/pasted-viteshan-19.png)

There are two types in TxType, representing send and receive transactions separately. These are corresponding to the fact that in Vite, a transfer generates two transactions.

Below is the data structure of account status, we can see that the account status references the transaction data Tx. This means that every change in account status is caused by a transaction.



Send accounts reference TxType as send transaction and receive accounts reference TxType as receive transaction.

![upload successful](/images/pasted-viteshan-20.png)


Account status will be snapshotted by snapshot chain periodically, the definition of snapshot block is as below, Height param means the height of snapshot block, and it increments by 1 each time. AccountsHash represents all the hashes of cached account status.


![upload successful](/images/pasted-viteshan-21.png)



The context above describes the fundamental data structure that constitutes Vite. The following sections of transfer transactions and snapshot block generation are based on these data structures.



![upload successful](/images/pasted-viteshan-22.png)


These two data structures represent the Vite account chain and snapshot chain respectively.

AccountStateBlockChain is a map data structure, key is the address of each account, value is a linked list of AccountStateBlock, pointing to the previous block via PreHash.



## Transaction Process
### Send Transaction


The core code of the send transaction is as shown above:

1. Construct a transaction and sign the transaction
2. Pack the account status block and sign the packaged account status block
3. Insert the account status block into the account chain and broadcast the block results

![upload successful](/images/pasted-viteshan-23.png)


### Receive Transaction


![upload successful](/images/pasted-viteshan-24.png)
There will be a coroutine waiting for the broadcast event to be generated and verify the validity of the block.  Meanwhile, if it needs to generate the corresponding received transaction, it will forward Tx to the corresponding node.



![upload successful](/images/pasted-viteshan-25.png)

After the corresponding node receives the send transaction and passes the verification:



1. Generate a received transaction and sign it
2. Pack the account status block and sign the packaged account status block
3. Insert the account status block into the account chain and broadcast the block structure


### The Snapshotting Process of Snapshot Chain



![upload successful](/images/pasted-viteshan-26.png)

The mining account will periodically generate the snapshot block and insert it into the snapshot chain.



# Summary
The key point of this article is to briefly introduce the main process of [naive-vite](https://github.com/viteshan/naive-vite), and at the same time let everyone know the fundamental principles of Vite through this simple demo.

For more details, [visit official Vite website](http://vite.org/), and watch [the official implementation of the first version on Github](https://github.com/vitelabs/go-vite).



# References
- [The white paper of Vite](https://www.vite.org/whitepaper/vite_en.pdf)
- [Source Code of naive-vite](https://github.com/viteshan/naive-vite)
- [Block-tutorial](https://github.com/mycoralhealth/blockchain-tutorial)