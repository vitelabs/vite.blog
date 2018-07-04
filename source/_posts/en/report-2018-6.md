title: Vite Update — June 2018
tags:
  - Report
categories:
  - Monthly Report
date: 2018-07-03 15:24:00
---
![upload successful](/images/pasted-0.png)
In the last month, we made substantive progress in a few fronts: We continued system development, which was bolstered by additional key engineering hires. We also completed the release of the first batch of Vite tokens and reached a number of milestones in community development and business development.

<!-- more -->

## Software Development

* We began developing Vite Core and made it open-source: https://github.com/vitelabs/go-vite
* We completed the design for the signature algorithm and hash algorithms: https://doc.vite.org/zh/technology/design-features.html
* We completed the designs for local storage structure and indexing for Vite Core. Local storage uses LevelDb. Since there is no official golang library for LevelDb, we have temporarily settled on a third-party library, also used by Ethereum. We plan to review the code in said library and assess its functionality and security.
* We completed the design of the network protocol and the definition of the message structure. Vite will implement inter-node communication with the structured P2P network. Routing between nodes will be using the KAD algorithm, with the heartbeat using the quic protocol for communications (http://www.chromium.org/quic). The quic protocol is a UDP-based communications protocol proposed by Google; it can effectively reduce latency in data transfers, and is used in multiple Tencent applications.
* We completed the product design, demand audit, architectural design, definition of API interfaces and planning of development for Vite’s official blockchain browser.
* In performing detailed design of the protocol layer, in order to implement the feature of forging rewards, we have made the following decision: responsive reactions in the ledger will not only reference request transactions, but also reference the creation of snapshot chain blocks. This design connects the DAG ledger and the snapshot chain in a clean manner, and makes implementation easier.


## Recruiting

We added four engineers (including system architects, backend engineers and front end engineers) and one product manager. All new hires came from first-rate internet technology firms. The current engineering team headcount is 11.

## Operations and Community Development

Due to high ratings from various overseas review websites, the number of our English Telegram Group has reached 42000 and our Chinese Telegram Group has reached 13000. We also added Telegram Groups for Vietnam, Russia, Korea and Thailand. A sample of websites that rated us: icodrops.com, monoico.com and top7ico.

## The Vite Incubator

Vite is proactively planning for the ecosystem development of our public blockchain. We are pleased to announce that our newly created incubator will be led by Frank Deng:


Frank is a an expert in digital and mobile marketing. He graduated BA from Tsinghua University, served in Google Ads Operations Group, was COO of Suizong Technology and co-founder of Yunke Technology.

The first set of incubated projects will be in the financial technology sector, with two important items in the pipeline. Details will be announced at a later date.

## External Communications

Sample of events from this month:

* Roadshow in Beijing on June 9
* Meeting with key advisors in Beijing on June 14
* Project information exchange in Shanghai held by our investor UniValue Associates, June 26
* Vite’s COO Richard was interviewed by the blockchain media firm Tuoniao for his experience of doing a Chinese startup as an ex-pat, June 26
* Vite’s COO Richard presented at Genesis Capital’s roadshow in Silicon Valley and networked with fellow blockchain investors and entrepreneurs at the Silicon Valley Blockchain Connect conference, June 26–28

![upload successful](/images/pasted-1.png)
> Charles Liu (CEO) presenting at Beijing roadshow


![upload successful](/images/pasted-2.png)
> Richard Yan (COO) presenting at Silicon Valley Blockchain Week


![upload successful](/images/pasted-3.png)
> Richard Yan (COO) with Blockchain Brad and CoinCrunch’s Sia at Silicon Valley Blockchain Connect


![upload successful](/images/pasted-4.png)
> Meeting with key advisors Daniel Wang (Loopring) and Leah Zhang (LinkVC, ex-Huobi). From left to right: Charles (CEO), Daniel, Leah, Richard (COO)

## Partnership with OK Blockchain Capital

Previously announced: https://medium.com/vitelabs/vite-and-ok-blockchain-capital-enter-into-strategic-partnership-38a1ad9c5b37

## Token release

We released 20% of tokens on June 15. The remainder will be released over the next four months, with 20% released on each of the 15th.


## Vite Tidbits

* The DAG (Directed Acyclic Graph) is a graph data structure, with directed edges and no loops. Our ledger makes use of this structure to preserve partial order between transactions. The ordering of transactions affects the state of the system. To prevent historical transactions from being tampered with, nodes in the graph are referenced by hashes. The higher amount of partial orders, the “lean and long” the DAG looks, and the more secure is the ledger. On the other hand, the fewer partial orders there are, the “flat and horizontal” the DAG looks, and the higher the performance. The one DAG with the highest amount of partial orders preserves the “total order” of transactions and has the highest level of security, and this DAG is called a blockchain. That’s right, a blockchain is also a special case of DAG.
* The Snapshot Chain is an important innovation of Vite. This technology addresses the innate deficiency of security in DAG, and provides the mechanism for asynchronous verification of transactions. Also, the Snapshot Chain is the global clock for the system, as it helps with the accurate calculation of the TPS for each account.
* Vite’s token issuance mechanism is similar to ERC20 of Ethereum, but there are some fundamental differences. ERC20 is a standard, but not a part of the Ethereum protocol. To issue new tokens on the Ethereum, the user needs to develop and deploy a smart contract, and the balance of tokens is preserved in the state of the said contract. Any error in the development of that contract will create security risks. As an example, if the creator of the smart contract forgets to reference the SaftMath library, they may have introduced an overflow bug. On the other hand, the issuance of Vite tokens takes place within Vite’s own protocol, and the balance of tokens is preserved in the state of the user’s account. The new tokens share the same base protocol for transfers, and therefore the same level of security as Vite’s native token. In order to issue a new token in Vite, the user only needs to initiate a transaction, and place the new token’s parameters inside the data field of the transaction. There is no need for writing code for the smart contract, thereby lessening security risks.
