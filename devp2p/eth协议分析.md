# eth协议分析

eth是建立在RLPx传输层协议之上的协议，可促进peer之间的以太坊区块链信息交换。 当前的协议版本是 eth/66。 有关过去协议版本的更改列表，请参见文档末尾。

## Basic Operation
建立连接后，必须发送一个Status消息。 在收到peer的Status消息后，以太坊会话处于活动状态，并且可以发送任何其他消息。

在一个会话中，可以执行三个高级任务：链同步、块传播和交易交换。 这些任务使用不相交的协议消息集，客户端通常将它们作为所有对等连接上的并发活动来执行。

客户端实现应该强制限制协议消息的大小。 底层 RLPx 传输将单个消息的大小限制为 16.7 MiB。 eth 协议的实际限制较低，通常为 10 MiB。 如果接收到的消息大于限制，则应断开对等方的连接。

除了接收消息的硬限制，客户端还应该对他们发送的请求和响应施加“软”限制，建议的软限制因消息类型而异。 限制请求和响应的大小可确保其他活动可以并发进行，例如块同步和交易交换在同一个对等连接上顺利进行。

## Chain Synchronization（链同步）

参与 eth 协议的节点需要知道从创世块到当前最新块的所有块的完整链。 该链是通过从其他对等点下载获得的。连接建立后，双方都会发送他们的状态消息，其中包括总难度 (TD) 和他们知道的“best” known block哈希。

然后，具有最差 TD 的客户端使用 GetBlockHeaders 消息下载块头。 它验证接收到的区块头中的工作量证明值，并使用 GetBlockBodies 消息获取块体。 收到的区块使用以太坊虚拟机执行，重新创建状态树和收据。

请注意，区块头下载、区块体下载和区块执行可能同时发生。

## State Synchronization (状态同步，也即fast sync)

协议版本 eth/63 及更高版本还允许同步交易执行结果（即状态树和收据）。 这可能比重新执行所有历史交易更快，但会牺牲一些安全性。状态同步通常通过下载区块头链来进行，验证它们的有效性。 也会通过链同步请求区块体，但不执行块交易，仅验证其“数据有效性”。 客户端在链头附近选择一个块，并通过使用 GetNodeData 请求根节点、其子节点、孙子节点……逐步下载 merkle 树节点和合约代码，直到整个树同步完成。

## Block Propagation（区块广播）

新挖出的区块必须中继到所有节点。 这是通过块传播发生的，这是一个两步过程。 当收到来自 Peer 的 NewBlock 公告消息时，客户端首先验证该区块的区块头的有效性，检查工作量证明值是否有效。 然后，它将 NewBlock 消息发送给一小部分已连接的对等点（通常是对等点总数的平方根）。

在区块头有效性检查之后，客户端通过执行包含在块中的所有事务，将块导入其本地链，计算块的“post state”。 区块的状态根哈希必须与计算相匹配。一旦块被完全处理并被认为是有效的，客户端就会向它之前没有通知的所有对等方发送一条关于该块的 NewBlockHashes消息。 如果这些对等方未能通过NewBlock从其他任何人那里接收到完整块，它们可能会在稍后请求完整块。

节点永远不应将NewBlock消息发送回先前宣布相同块的对等方。 这通常是通过记住最近中继到每个对等点或从每个对等点中继的大量块哈希来实现的。

如果收到的NewBlock块不是客户端当前最新块的直接后继，则客户端可能触发链同步。

## Transaction Exchange

所有节点必须交换待处理的交易，以便将它们转发给矿工，矿工将选择它们以包含在区块链中。 客户端会跟踪“交易池”中的待处理交易的列表。交易池的容量由客户端自己设置，通常可以包含许多（即数千个）交易。

当建立新的对等连接时，需要同步双方的交易池。 启动交换前，两端应发送包含本地池中所有交易哈希的 NewPooledTransactionHashes 消息。收到 NewPooledTransactionHashes 通知后，客户端过滤接收到的集合，收集它自己的本地池中还没有的交易哈希。 然后可以使用 GetPooledTransactions 消息请求交易。

当客户端池中出现新事务时，它应该使用 Transactions 和 NewPooledTransactionHashes 消息将它们传播到网络。 Transactions 消息中继完整的交易对象，通常发送给一小部分随机连接的对等点。 所有其他对等方都会收到交易哈希的通知，如果他们不知道，可以请求完整的交易对象。 将完整的交易分发给一小部分对等点通常可以确保所有节点都接收到交易并且不需要请求它。

节点永远不应该将事务发送回它可以确定已经知道它的对等点（因为它以前被发送过，或者因为它最初是从这个对等点通知的）。 这通常是通过记住一组最近由对等点中继的交易哈希来实现的。

## Transaction Encoding and Validity

对等方交换的交易对象具有两种编码之一。 在本规范的定义中，我们使用标识符 txₙ 来指代任一编码的交易。

```
tx = {legacy-tx, typed-tx}
```
legacy transaction以 RLP 列表的形式给出。

```
legacy-tx = [
    nonce: P,
    gas-price: P,
    gas-limit: P,
    recipient: {B_0, B_20},
    value: P,
    data: B,
    V: P,
    R: P,
    S: P,
]
```

EIP-2718 定义的typed-transaction被编码为 RLP 字节数组，其中第一个字节是交易类型（tx-type），其余字节是不透明的特定于类型的数据。
```
typed-tx = tx-type || tx-data
```
交易必须在收到时进行验证。 有效性取决于以太坊链状态。 该规范关注的具体有效性不是交易是否能用EVM成功执行，而只是在本地池中临时存储和与其他对等方交换是否可以接受。

交易根据以下规则进行验证。 虽然typed-tx的编码是不透明的，但假设它们的 tx-data 提供了 nonce、gas-price、gas-limit 的值，并且可以从它们的签名中确定交易的发送者帐户。

* 如果transaction是类型化的，则实现必须知道 tx的具体类型。实现应该断开发送未知类型交易的对等方。
* 签名必须根据链支持的签名方案有效。 对于类型化交易，签名处理由引入类型的 EIP 定义。 对于 legacy transactions，两种正在使用的方案是基本的“Homestead”方案和 EIP-155 方案。
* gas限制必须涵盖交易的“intrinsic gas”。
* 交易的发送者账户必须有足够的以太币余额来支付交易的成本（gas-limit * gas-price + value）。
* 交易的 nonce 必须等于或大于发送者帐户的当前 nonce。
* 在考虑将交易包含在本地池中时，由实现决定比当前账户nonce大的“未来”交易的数量，以及“nonce gaps”在多大程度上是可以接受的。

实现可以对事务强制执行其他验证规则。 例如，通常的做法是拒绝大于 128 kB 编码的交易。

除非另有说明，否则实现不得因为对等方发送无效交易而断开连接，而应简单地丢弃它们。 这是因为对等方可能在稍微不同的验证规则下运行。

## Block Encoding and Validity

以太坊区块编码如下：

```
block = [header, transactions, ommers]
transactions = [tx₁, tx₂, ...]
ommers = [header₁, header₂, ...]
header = [
    parent-hash: B_32,
    ommers-hash: B_32,
    coinbase: B_20,
    state-root: B_32,
    txs-root: B_32,
    receipts-root: B_32,
    bloom: B_256,
    difficulty: P,
    number: P,
    gas-limit: P,
    gas-used: P,
    time: P,
    extradata: B,
    mix-digest: B_32,
    block-nonce: B_8
]
```
在某些协议消息中，交易和 ommer 列表作为称为“block body”的单个项目一起中继。

```
block-body = [transactions, ommers]
```
区块头的有效性取决于使用它们的上下文。 对于单个区块头，只能验证工作量证明seal（mix-digest、block-nonce）的有效性。 当使用一个 header 来扩展客户端的本地链时，或者在链同步过程中顺序处理多个 header 时，需要满足以下规则：
* Headers必须形成一个链，其中区块高度是连续的，并且每个Header的父哈希与前一个Header的哈希匹配。
* 在扩展本地存储的链时，实现还必须验证difficulty、gas-limit 和 time 的值是否在黄皮书中给出的协议规则的范围内。
* 区块头的gas-used字段必须小于或等于gas-limit。

对于完整的区块，我们区分区块的 EVM 状态转换的有效性和区块的（较弱的）“数据有效性”。 本规范不涉及状态转换规则的定义。 出于即时期间的目的，我们需要块的数据有效性是为了能够立即进行块传播和状态同步。

要确定块的数据有效性，请使用以下规则。 实现应该断开发送无效块的对等方：
* 区块header必须合法
* 包含在区块中的交易必须能够包含在指定的区块高度中。 这意味着，除了前面给出的交易验证规则之外，还需要验证区块高度是否允许该tx类型，并且交易gas的验证必须考虑区块高度。
* 所有交易的gas-limit总和不得超过区块的gas-limit。
* 该块的交易必须通过计算和比较交易列表的 merkle trie哈希值和区块头里的 txs-root 进行验证。
* ommers 列表最多可以包含两个headers。
* keccak256(ommers) 必须匹配区块头的 ommers-hash。
* ommers 列表中包含的Headers必须是有效的Headers。 它们的区块高度不得大于包含它的区块高度。ommer区块头的父区块哈希必须引用其包含块高度之前深度为 7 或更小的祖先，并且它不能包含在这个祖先集的区块中。


## Receipt Encoding and Validity

收据是区块的 EVM 状态转换的输出。 像交易一样，收据有两种不同的编码，我们将使用标识符receiptₙ来指代任何一种编码。

```
receipt = {legacy-receipt, typed-receipt}
```
无类型的 legacy receipts 编码如下：
```
legacy-receipt = [
    post-state-or-status: {B_32, {0, 1}},
    cumulative-gas: P,
    bloom: B_256,
    logs: [log₁, log₂, ...]
]
log = [
    contract-address: B_20,
    topics: [topic₁: B, topic₂: B, ...],
    data: B
]
```
EIP-2718定义的typed-receipts被编码为 RLP 字节数组，其中第一个字节给出了收据类型（匹配 tx-type），其余字节是特定于该类型的不透明数据。

```
typed-receipt = tx-type || receipt-data
```

在 Ethereum Wire 协议中，receipts总是作为包含在一个块中的所有收据的完整列表进行传输。 还假设包含收据的块是有效且已知的。 当节点收到一个区块receipts列表时，必须通过计算并将列表的 merkle trie 哈希值与区块的收据根进行比较来验证它。 由于收据的有效列表由 EVM 状态转换确定，因此在本规范中没有必要为receipts定义任何进一步的有效性规则。


## Protocol Messages

在大多数消息中，消息数据列表的第一个元素是 request-id。 对于请求，这是请求对等方选择的 64 位整数值。 响应对等体必须回应时带上这个值。

## Status (0x00)

```
[version: P, networkid: P, td: P, blockhash: B_32, genesis: B_32, forkid]
```

通知对等方其当前状态。 此消息应在连接建立后且在任何其他 eth 协议消息之前发送。

* `version`: 协议版本
* `networkid`: 标识区块链的整数，见下表
* `td`: 最佳链的总难度。 整数，在block header中找到。
* `blockhash`：best（即最高 TD）的区块哈希值。
* `genesis`: the hash of the genesis block
* `forkid`: 一个 EIP-2124 分叉标识符，编码为 [fork-hash, fork-next]。 

此表列出了常见的网络 ID 及其对应的网络。 存在未列出的其他 ID，即客户端不应要求使用任何特定的网络 ID。 请注意，网络 ID 可能与用于防止交易重放的 EIP-155 链 ID 对应，也可能不对应。

| ID | chain                         |
|----|-------------------------------|
| 0  | Olympic (disused)             |
| 1  | Frontier (now mainnet)        |
| 2  | Morden (disused)              |
| 3  | Ropsten (current PoW testnet) |
| 4  | [Rinkeby]                     |

有关社区策划的链 ID 列表，请参阅 https://chainid.network。

## NewBlockHashes (0x01)

```
[[blockhash₁: B_32, number₁: P], [blockhash₂: B_32, number₂: P], ...]
```

指定一个或多个出现在网络上的新块。 节点应该通知对等方他们可能不知道的所有块。如果发送方能够合理推断出对方知道这个hash而重复发送被认为是错误的，并且可能会降低发送节点的声誉。发送方发送出去的hash，但对方发送GetBlockHeaders消息来请求时又不提供数据也被认为是错误的形式，并且可能会降低发送节点的声誉。

## Transactions (0x02)

```
[tx₁, tx₂, ...]
```
交易消息必须至少包含一个（新）交易，不鼓励使用空交易消息并可能导致断开连接。

节点不得将同一交易重新发送到同一会话中的peer，并且不得将交易中继到它们从其接收该交易的peer。 在实践中，这通常是通过保留每个peer的布隆过滤器或保存已经发送或接收的交易哈希列表来实现的。

## GetBlockHeaders (0x03)

```
[request-id: P, [startblock: {P, B_32}, limit: P, skip: P, reverse: {0, 1}]]
```

请求对等方返回BlockHeaders 消息。 响应必须包含一些列块头，当 reverse 为 0 时从小到大，当为 1 时从大到小，跳过块，从主链的startblock（由数字或哈希表示）开始，跳过skip个块，并且最多不超过limit个。

## BlockHeaders (0x04)

```
[request-id: P, [header₁, header₂, ...]]
```
这是对 GetBlockHeaders 的响应，包含请求的Headers。 如果没有找到请求的区块头，则header列表可能为空。 单个消息中可以请求的header数量可能会受到实现定义的限制。BlockHeaders 响应的推荐软限制为 2 MiB。

## GetBlockBodies (0x05)

```
[request-id: P, [blockhash₁: B_32, blockhash₂: B_32, ...]]
```

该消息通过哈希请求块体数据。 可以在单个消息中请求的块数可能会受到实现定义的限制。

## BlockBodies (0x06)

```
[request-id: P, [block-body₁, block-body₂, ...]]
```
这是对 GetBlockBodies 的回应。 列表中的项目包含请求块的主体数据。 如果没有请求的块可用，则列表可能为空。BlockBody 响应的推荐软限制为 2 MiB。

## NewBlock (0x07)

```
[block, td: P]
```
指定对等方应该知道的单个完整块。 td 是区块的总难度，即直到并包括这个区块的所有区块难度的总和。

## NewPooledTransactionHashes (0x08)

```
[txhash₁: B_32, txhash₂: B_32, ...]
```
此消息宣布一个或多个已出现在网络中且尚未包含在块中的交易。 为了最大限度地提供帮助，节点应该通知对等方他们可能不知道的所有交易。此消息的推荐软限制为 4096 个哈希 (128 KiB)。节点应该只宣布远程对等点可以合理地被认为不知道的交易的哈希值，但最好返回更多的交易而不是在交易池中存在nonce间隙。

## GetPooledTransactions (0x09)

```
[request-id: P, [txhash₁: B_32, txhash₂: B_32, ...]]
```
此消息通过哈希从接收者的交易池中请求交易。GetPooledTransactions 请求的推荐软限制为 256 个哈希 (8 KiB)。 接收者可以对响应的大小强制实施任意限制，这不能被视为违反协议。

## PooledTransactions (0x0a)

```
[request-id: P, [tx₁, tx₂...]]
```
这是对 GetPooledTransactions 的响应，从本地池返回请求的交易。 列表中的项目是采用以太坊主规范中描述的格式的交易。

交易必须与请求中的顺序相同，但可以跳过不可用的交易。 这样如果达到响应大小限制，请求者能知道要再次请求哪些哈希（从最后一个返回的交易开始）以及哪些交易不可用（在最后一个返回的交易之前的所有间隙）。

允许首先通过 NewPooledTransactionHashes 宣布交易，然后拒绝通过 PooledTransactions 提供交易。当交易包含在区块（从交易池中删除了）可能会出现这种情况。如果没有哈希匹配其池中的交易，则对等点可能会以空列表进行响应。

## GetNodeData (0x0d)

```
[request-id: P, [hash₁: B_32, hash₂: B_32, ...]]
```
要求对等方返回包含与请求的哈希匹配的状态树节点或合约代码的 NodeData 消息。

## NodeData (0x0e)

```
[request-id: P, [value₁: B, value₂: B, ...]]
```

提供一组状态树节点或合约代码块，它们对应于先前从 GetNodeData 请求的哈希值。 不需要包含所有； 尽力而为就好。 如果对等方不知道任何先前请求的哈希，则此消息可能是一个空列表。 可以在单个消息中请求的项目数可能会受到实现定义的限制。NodeData 响应的推荐软限制为 2 MiB。

## GetReceipts (0x0f)

```
[request-id: P, [blockhash₁: B_32, blockhash₂: B_32, ...]]
```

要求对等方返回包含给定块哈希收据的收据消息。 可以在单个消息中请求的收据数量可能会受到实现定义的限制。


## Receipts (0x10)

```
[request-id: P, [[receipt₁, receipt₂], ...]]
```

这是对 GetReceipts 的响应，提供请求的块收据。 响应列表中的每个元素对应一个 GetReceipts 请求的区块哈希，并且必须包含该区块的完整收据列表。Receipts 响应的推荐软限制为 2 MiB。

## Change Log

* eth/66 (EIP-2481, April 2021)

版本 66 在消息 GetBlockHeaders、BlockHeaders、GetBlockBodies、BlockBodies、GetPooledTransactions、PooledTransactions、GetNodeData、NodeData、GetReceipts、Receipts 中添加了 request-id 元素。

* eth/65 with typed transactions (EIP-2976, April 2021)

当 EIP-2718 引入类型化交易时，客户端实现者决定在不增加协议版本的情况下接受有线协议中的新交易和收据格式。 此规范更新还添加了所有共识对象的编码定义，而不是参考黄皮书。

* eth/65 (EIP-2464, January 2020)

版本 65 改进了交易交换，引入了三个附加消息：NewPooledTransactionHashes、GetPooledTransactions 和 PooledTransactions。

在版本 65 之前，peer总是交换完整的交易对象。 随着以太坊主网上活动和交易规模的增加，用于交易交换的网络带宽成为节点运营商的重大负担。 该更新通过采用类似于块传播的两层交易广播系统减少了所需的带宽。

* eth/64 (EIP-2364, November 2019)

版本 64 更改了状态消息以包含 EIP-2124 ForkID。 这允许peer在不同步区块链的情况下确定链执行规则的相互兼容性。

* eth/63 (2016)

版本 63 添加了允许同步交易执行结果的 GetNodeData、NodeData、GetReceipts 和 Receipts 消息。

* eth/62 (2015)

在版本 62 中，NewBlockHashes 消息被扩展为包含哈希和块号。 状态中的块号被删除。 消息 GetBlockHashes (0x03)、BlockHashes (0x04)、GetBlocks (0x05) 和 Blocks (0x06) 被获取块头和块体的消息所取代。 BlockHashesFromNumber (0x08) 消息已被删除。

