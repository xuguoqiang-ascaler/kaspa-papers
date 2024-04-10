# PHANTOM GHOSTDAG

## 这是PHANTOM GHOSTDAG论文的中文翻译，再次感谢作者。

## 摘要

在2008年，中本聪发明了基于区块链技术的分布式账本的基础理论。这一系统的核心部分是一个由节点（或者叫矿工）构成的公开的，匿名的网络来维护一个公共的交易账本。账本采用区块链的形式，每一个区块是用户的一批交易。中本聪的区块链最主要的问题是扩展性极度被限制。中本聪的最长链原则，也就比特币协议的安全性要求在其他节点产生新的区块后，所有的诚实的节点需要尽快知悉这个区块。为了这一目的，这个系统的吞吐量被人为的压低，从而使得新产生的区块在下一个区块产生之前被充分的广播，也可以保证极少的导致链上分叉的孤儿块被无意的创建。

在这篇论文中我们阐述了PHANTOM协议，另外一种创建无许可账本的协议。它基于工作量证明，将中本聪的区块链扩展到由区块构成的有向无环图（区块图）。PHANTOM协议包含一个参数k，用来控制对网络上同时产生的区块的数目的容忍程度。这个参数可以被调整以适应高吞吐量。它也避免了在安全性和扩展性之间的取舍，而这是中本聪协议不能避免的。

PHANTOM解决了一个区块图（blockDAG）上的一个优化问题，从而可以区分由诚实的节点铸造的区块和由不合作的节点铸造的区块，这些不合作的节点往往想偏离挖矿协议。通过这个区分，PHATOM协议提供了一种在区块图（blockDAG）上实现稳定的全排序的方法，这种方法可以使全排序的结果最终在所有的诚实节点上达成一致。实现PHANTOM协议需要解决一个NP-hard（NP难）的问题。因此，为了避免高昂的计算代价，我们想出了一种高效的贪心算法GHOSTDAG，这种算法的本质和PHANTOM是一致的。

GHOSTDAG协议是实现kaspa加密货币的底层技术。kaspa网络提供产生在现实场景下GHOSTDAG性能的统计数据。通过观察kaspa网络获取了网络确认时间的数据，我们分析了这些数据。我们也对GOHSTDAG协议的安全性进行了形式上的证明。也就是说，我们证明了在一个指数级别可以忽略的参数限制下，GHOTSDAG的区块的排序是不可逆的。我们讨论了GHOSTDAG的性质，并把它和其他基于有向无环图（DAG）的协议做了比较。

## 1 简介

<!--
The security of the Bitcoin protocol relies on blocks propagating
quickly to all miners in the network [9, 14, 19]. Block creation
itself is slowed down via the requirement that each block contain
a proof-of-work. For the Bitcoin protocol to be secure, block
propagation must be faster than the typical time it takes the network
together to create the next block. In order to guarantee this property,
the creation of blocks in Bitcoin is regulated by the protocol to occur
only once every 10 minutes, and the block size itself is limited to
allow for fast transmission. As a result, Bitcoin suffers from a highly
restrictive throughput on the order of 3-7 transactions per second
(tps).
-->

比特币协议的安全性依赖于将区块快速的传播给网络中的矿工。由于要求区块包含工作量证明，所以区块的创建速度变慢了。为了保证比特币协议的安全，区块传播要比整个网络通常创建一个区块的时间要快。为了保证这一网络的性质，在比特币协议中规定了区块的创建每隔十分钟发生一次，并且限制了区块的大小，使得区块可以快速传输。因此，比特币的吞吐量被极大的抑制，每秒钟大概只有3到7笔交易。

<!--
the PHANTOM protocol. In this paper we present PHANTOM,
a protocol that generalizes Nakamoto’s longest chain protocol. In
Bitcoin, blocks reference a single predecessor in the chain, hence
form a tree; in contrast, PHANTOM blocks reference multiple
predecessors, thus forming a Directed Acyclic Graph, a blockDAG.
Each block can thus include several hash references to predecessors.
PHANTOM then provides a total ordering over all blocks and
transactions, and outputs a consistent set of accepted transactions.
Unlike the Bitcoin protocol, where blocks that are not on the main
chain are discarded, PHANTOM incorporates all blocks in the
blockDAG into the ledger, but places blocks that were created by
attackers later in the order.
-->

*PHANTOM协议* 在这篇论文中我们描述了PHANTOM协议，这个协议扩展了中本聪的最长链协议。在比特币中，区块引用一个链上的父节点，因此，构成了一个树的结构。与此对比之下，PHANTOM协议引用了多个父节点，因此形成了一个有向无环图，也称作blockDAG。每一个区块因此可以包含多个父节点的哈希引用。PHANTOM然后提供了一种对所有区块和交易的全排序，输出一个一致的被接收的交易的集合。不像在比特币协议中，所有的不在主链上的区块都会被忽略，PHANTOM吸收所有在blockDAG中的区块到账本中，但同时将恶意攻击者创建的区块放在排序的后面。

<!--
In rough terms, PHANTOM consists of a three-step procedure:
(1) Using the structure of the blockDAG, we recognize a set
of well-connected blocks (we later refer to these as blue
blocks); this procedure is used to exclude blocks created by
misbehaving nodes and is the heart of the protocol: Blocks
that either reference only old blocks from the DAG, or are
withheld by their creator for some time, will be excluded
from the set of blue blocks with high probability.
(2) We complete the DAG’s naturally induced partial order
to a full topological one (i.e., an order which respects the
topology) in a way that favours blocks inside the selected
cluster and penalizes those outside it.
(3) The order over blocks induces an order over transactions;
transactions in the same block are ordered according to
the order of their appearance in the block. We iterate over
all transactions in this order, and accept each one that is
consistent (according to the underlying consistency notion)
with all transactions approved so far.
-->

粗略的来讲，PHANTOM协议包含三个步骤。
（1）通过使用blockDAG结构，我们识别一个由良好连接的区块构成的集合（这些良好连接的区块我后面通过蓝色来标记）。这个过程被用来提出恶意节点产生的区块。那些指引用老节点的区块和那些被创建者扣留一段时间的区块很大概率会被从蓝色的区块剔除。
（2）我们通过偏序顺序得到一个全拓扑的顺序（这个排序不改变拓扑结构）。在排序的过程中，奖励蓝色集合的区块，惩罚蓝色集合以外的区块。
（3）区块的顺序决定了交易的顺序。在同一区块中的交易，顺序由交易在区块中出现的时间顺序决定。我们按照这个顺序遍历所有的交易，并接受与历史上的交易相一致（依赖底层的一致性原则）的交易。

<!--
Propagation delay. The first step above involves assuming
an upper bound on the network’s delay diameter D, and
parameterizing the protocol accordingly; k denotes this parameter.
Such an assumption is made in Nakamoto Consensus as well. In fact,
if PHANTOM is set to process low throughput, we can set k =0,
in which case PHANTOM coincides with Nakamoto Consensus.
However, while Nakamoto Consensus suppresses the throughput
and sets the block creation rate λ such that D ·λ ≪1, PHANTOM
does not impose an a priori constraint over λ. Instead, the
throughput (in terms of λ and the block size) can be set to approach
the network’s capacity, and then k can be set after the fact to ensure
the safety of the protocol. This alleviates the security-scalability
tradeoff that Nakamoto Consensus suffers. Still, increasing k does
not come without cost, as we will discuss shortly.
-->

*传输延迟* 上面的第一步需要假设整个网络有一个延迟半径$D$的上限，并且根据这个上限确定协议中的参数。$k$就是这个参数。中本聪协议中也有这个假设。事实上，如果让PHANTOM处理很低的吞吐量，我们可以将$k$设置为0，在这种情况下，PHANTOM协议和中本聪协议是一致的。但是，中本聪协议压低了吞吐量，它将出块速度$\lambda$设置为$D \cdot \lambda \ll 1$。但是PHANTOM协议并不对$\lambda$进行预设。相反，吞吐量（由$\lambda$和区块大小决定）可以设置达到网络容量，在达到这一前提下可以设置$k$确保协议的安全性。这缓解了安全性和扩展性顾此失彼的权衡困境，而这是中本聪协议不得不面对的。然而，增加$k$并不是没有代价的，我们将简要的讨论这一问题。

<!--
GHOSTDAG. In its vanilla form, PHANTOM requires solving
an NP-hard problem, and is therefore unsuitable for practical
applications. Instead, we use the intuition behind PHANTOM to
devise a greedy algorithm, GHOSTDAG, which can be implemented
efficiently. We prove formally that GHOSTDAG is secure, in the
sense that its ordering of blocks becomes exponentially difficult to
reverse as time develops.
-->

*GHOSTDAG* 在基础的形式下，PHANTOM需要解决一个NP难的问题，因此并不适用于实际的应用场景。但是，我们通过把握PHANTOM协议的实质实现了一种贪心算法，这种算法可以被高效的实现。我们在逻辑上证明了GHOSTDAG是安全的。这个安全是指随着时间的增加，要回滚区块的排序的难度是指数级别增加的。

<!--
The main achievement of GHOSTDAG can be summarized as
follows:
Theorem (Informal). Given two transactions tx1,tx2 that were
published and embedded in the blockDAG at some point in time,
the probability that GHOSTDAG’s order between tx1 and tx2 changes
over time decreases exponentially as time grows, even under a high
block creation rate that is non-negligible relative to the network’s
propagation delay, assuming that a majority of the computational
power is held by honest nodes.
We will reformalize this theorem in Section 3, and provide a
formal proof in Appendix A.
We now proceed to describe the PHANTOM and GHOSTDAG
protocols more formall
-->

GHOSTDAG的成果可以被总结为以下内容：
在理论（形式）上，给定两个已经发布并写入blockDAG的交易，tx1和tx2，在GHOSTDAG协议中，随着时间的增加，tx1和tx2
两个交易的相对顺序的改变的概率是指数级别递减的，即使在一个很高的区块的创建速度下也是如此。这个很快是指在当大部分算力被诚实的节点掌握的时候，产生一个区块的时间与网络传输的延迟相比是不可忽略的。我们将在第三节再次阐述这个理论，并且在附录A中提供了一个形式化的证明。

## 2 PHANTOM协议
### 2.1 前置内容 

<!--
The following terminology is used extensively throughout this
paper. A DAG of blocks is denoted G =(C,E), where Crepresents
blocks and E represents the hash references to previous blocks. We
frequently write B ∈G instead of B ∈ C. past (B,G) ⊂ Cdenotes
the subset of blocks reachable from B (excluding B), and similarly
f uture (B,G) ⊂ C denotes the subset of blocks from which B is
reachable (excluding B); these are blocks that were provably created
before and after B, correspondingly. An edge in the DAG points
back in time, from the new block to previously created blocks which
it extends. We denote by anticone (B,G)the set of blocks outside
past (B,G)and f uture (B,G)(excluding B itself); this is the set of
blocks in the DAG which did not reference B (directly or indirectly
via their predecessors) and were not referenced by B (directly or
indirectly via B’s predecessors).2 Finally, tips(G)is the set of blocks
with in-degree 0 (usually, the most recent blocks). This terminology
is demonstrated in Figure 1.
-->

接下来的这些术语将被广泛的用在整篇论文中。一个由区块构成的有向无环图用$G=(C,E)$，$C$用来表示区块，$E$用来表示当前区块对父区块的引用。我们通常记作$B \in G$而不是$B \in C$。$past(B, G) \subseteq C$表示从$B可以遍历到的区块的集合（不好含$B$）。$future(B,G) \subseteq C$表示可以遍历到$B$的区块的集合（不好含$B$）。这些区块是已经被验证的在区块$B$之前和之后创建的。在有向无环图中，边从现在指向过去，从当前的新创建的区块指向过去的它引用的区块。$anticone(B,G)$表示在$past(B,G)$和$future(B,G)$之外的区块的集合（不包含$B$）。这个集合表示无法引用到$B$的集合（无论直接引用还是间接前驱引用）和被$B$引用的集合（无论是直接还是间接后继引用）。$tips(G)$表示入度为0的区块（通常，这些事最新被创建的区块）。这些术语在图1中被阐述。

