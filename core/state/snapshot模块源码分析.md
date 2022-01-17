# snapshot模块源码分析

snapshot模块主要通过将Account和Storage数据压平直接存储在底层DB中，这样可以避免读取数据时需要先读取大量的中间Trie节点而影响性能，同时也允许按顺序迭代account和storage中的数据，方便同步。这个模块的核心难点是snapshot需要能够正常出块的同时在线生成。因为本身的生成时间久，生成的过程中需要一边将新更改的数据应用到已经生成的snapshot之中。否则等生成完了，区块已经涨了很高了，这个snapshot还是很早以前的状态的话就没什么用了。



## snapshot的持久层

### 数据存储和格式

snapshot的持久层存储的数据和格式定义在core/rawdb/accessors_snapshot.go文件中:

1. SnapshotRoot：保存该snapshot对应的trie根，key:"SnapshotRoot"， value: Hash；
2. AccountSnapshot: 保存账户信息，key:"a" + AccountHash，value: account trie value；
3. StorageSnapshot: 保存账户的存储信息，key: "o"+AccountHash+StorageHash， value: storage trie value；
4. SnapshotJournal: 保存内存中的Diff层，主要用于进程关闭重启；key:"SnapshotJournal", value：所有diff层系列化的结果；
5. SnapshotGenerator: 保存snapshot在生成时的进度，用于进程关闭重启；
6. SnapshotRecoveryNumber: 保存最后持久化的snapshot的区块号，用于进程关闭重启的恢复；
7. SnapshotSyncStatus: 保存snapshot的同步状态，用于进程关闭重启；



### diskLayer持久层管理

snapshot的持久层由diskLayer管理，定义如下：

```go
type diskLayer struct {
	diskdb ethdb.KeyValueStore // 持久层db
	triedb *trie.Database      // Trie node cache for reconstruction purposes
	cache  *fastcache.Cache    // Cache to avoid hitting the disk for direct access

	root  common.Hash // Root hash of the base snapshot
	stale bool        // 设置当前状态不可用，当在生成的过程中，同时要应用当前的最新区块的状态修改时设置为true

	genMarker  []byte                    // 标记持久层生成进度的位置
	genPending chan struct{}             // 接收持久层生成的消息 
    // 当外部需要中止构建的过程时，给这chan发送chan *generatorStats，构建routine会发送当前的进度状态并退出
    // 如果当前没有构建，那么genAbort为nil
    genAbort   chan chan *generatorStats

	lock sync.RWMutex
}
```

diskLayer实现了Snapshot接口，实现本身也比较直接，从cache中尝试读，没有到db中读出并进行相应的反序列化。注意因为持久层可能还在生成的过程中，因此需要确保key小于genMarker，否则读取报错。

```go
type Snapshot interface {
	// Root returns the root hash for which this snapshot was made.
	Root() common.Hash

	// Account directly retrieves the account associated with a particular hash in
	// the snapshot slim data format.
	Account(hash common.Hash) (*Account, error)

	// AccountRLP directly retrieves the account RLP associated with a particular
	// hash in the snapshot slim data format.
	AccountRLP(hash common.Hash) ([]byte, error)

	// Storage directly retrieves the storage data associated with a particular hash,
	// within a particular account.
	Storage(accountHash, storageHash common.Hash) ([]byte, error)
}
```



### 持久层的删除

持久层的删除由wipeSnapshot函数完成，它创建一个routine,遍历DB的键值对，删除所有和snapshot相关的数据。删除完成后对snapshot区间的key进行compact操作，以便清理掉没有的数据块。函数本身会返回一个chan，当routine完成后会关闭，以便通知等待方。迭代删除的操作有两点需要注意：1. 由于数据量比较大，如果迭代一次会导致Batch很大，造成内存压力，或者使迭代器保存太长时间影响compact操作，因此需要分批迭代，代码中是每间隔1w个进行一次；2. 由于trie的数据是不带前缀的，因此snapshot存储的数据并不是连续的，需要加上key长度的判断来确定是否是snapshot的数据。



### 持久层的创建

持久层的创建包括先删除老的持久层，再创建新的，由于两个操作都会耗时比较久，因此需要将进度信息记录到DB，以便进程关闭重启时能够继续之前的操作。同时创建的过程中新区块更新的数据也要重新覆盖已经生成的老数据，因此创建的routine会随时被打断。

进度信息保存在SnapshotGenerator下，内容是下面结构体的rlp编码：

```go
type journalGenerator struct {
	Wiping   bool // 是否现在处于清理老持久层阶段
	Done     bool // 是否已经完成了新持久层的创建
	Marker   []byte   // 生成的key
	Accounts uint64   // 统计数据只用于展示进度用
	Slots    uint64
	Storage  uint64
}
```

创建由generateSnapshot函数完成：

1. 先异步构造清理老snapshot的routine；
2. 向diskdb保存新的SnapshotRoot,并记录SnapshotGenerator为正在进行清理操作；
3. 构造出diskLayer结构体，并创建routine，执行diskLayer.generate操作；
4. 返回构造出的diskLayer;



diskLayer.generate函数的流程：

1. 如果当前处于清理老snapshot阶段，则等待清理操作完成；
2. 根据diskLayer.root和diskLayer.triedb构造account trie；
3. 从diskLayer.genMarker中，解析出当前处理的account hash，并依次迭代账户和账户的存储信息，批量写入账本，并更新diskLayer.genMarker。
3. 迭代过程中，检查是否被打断，如果打断的话结束构建过程。



## snapshot的内存diff层

内存层是一个区块执行之后的修改集合，包含排序好的account列表和对应account下的storage列表。由diffLayer数据结构管理，定义如下：

```go
type diffLayer struct {
	origin *diskLayer // Base disk layer to directly use on bloom misses
	parent snapshot   // Parent snapshot modified by this one, never nil
	memory uint64     // Approximate guess as to how much memory we use
	root  common.Hash // Root hash to which this snapshot diff belongs to
	stale uint32      // Signals that the layer became stale (state progressed)

	destructSet map[common.Hash]struct{}               // 标记这个区块中删除的账户
	accountList []common.Hash                          // 修改的账户列表，排序好了
	accountData map[common.Hash][]byte                 // 修改后的账户数据
	storageList map[common.Hash][]common.Hash          // 包含每个账户排序好的storage slot
	storageData map[common.Hash]map[common.Hash][]byte // 每个账户的每个slot的值，nil表示删除
	diffed *bloomfilter.Filter 
	lock sync.RWMutex
}
```

字段中的destructSet 比较特殊，因为有的账户删除之后还能重新创建，所以在这个列表中账户也可能存在于其他的字段列表中。当和上一层parent合并的时候先将destructSet的所有账户对应的parent的accountData和storageData等清除掉，然后将destructSet进行合并。其他字段的列表按照正常合并，相同key的值覆盖parent的。

另一个字段是diffed，用于跟踪当前layer一直到disk之间的所有layer的所有key的变更，这样可以快速判断是否可以直接跳过所有内存layer,直接从db读取数据。

diffLayer也实现了Snapshot接口，比较简单。



## snapshot的Journal

journal的作用是为了使进程重启后还能够恢复snapshot的内存diffLayer，当进程退出前会以当前区块高度的root为顶层layer直到diskLayer之间的所有layer做journal操作，将数据写入DB的key为SnapshotJournal中，value的格式：

journalVersion+diskRoot + diffLayer0+... + diffLayerCurrentBlock

每个diffLayer的格式：dl.root + dustructs+ accountData+ storageData

注意：当进程意外退出的时候journal是不会落账的，因此需要通过其他的手段恢复。



## Snapshot Tree

snapshot由Tree结构进行管理，因为存在分叉的情况，内存中的这些层逻辑上构成了一棵树。所以当区块发生reorg的时候可以非常方便切换，不过分叉区块比持久化的版本还小的时候，那么就会导致所有数据都失效。

结构定义：

```go
type Tree struct {
	diskdb ethdb.KeyValueStore      // 持久层
	triedb *trie.Database           // trie的内存缓存
	cache  int                      // 读缓存的大小，单位是M
	layers map[common.Hash]snapshot // 所有layer的集合
	lock   sync.RWMutex
}
```

对外的API:

```go
    func New(diskdb ethdb.KeyValueStore, triedb *trie.Database, cache int, root common.Hash, async bool, rebuild bool, recovery bool) (*Tree, error)
    func (t *Tree) AccountIterator(root common.Hash, seek common.Hash) (AccountIterator, error)
    func (t *Tree) Disable()
    func (t *Tree) Cap(root common.Hash, layers int) error
    func (t *Tree) DiskRoot() common.Hash
    func (t *Tree) Journal(root common.Hash) (common.Hash, error)
    func (t *Tree) Rebuild(root common.Hash)
    func (t *Tree) Snapshot(blockRoot common.Hash) Snapshot
    func (t *Tree) Snapshots(root common.Hash, limits int, nodisk bool) []Snapshot
    func (t *Tree) StorageIterator(root common.Hash, account common.Hash, seek common.Hash) (StorageIterator, error)
    func (t *Tree) Update(blockRoot common.Hash, parentRoot common.Hash, destructs map[common.Hash]struct{}, accounts map[common.Hash][]byte, storage map[common.Hash]map[common.Hash][]byte) error
    func (t *Tree) Verify(root common.Hash) error
```

下面重点分析几个重要的函数，其他的实现比较简单。

### New函数流程

1. 加载出snapshot root；
2. 加载出Journal和当前的generator状态；
3. 如果还处于构建，则启动routine继续处理；
4. 返回生成的tree;

### diffToDisk函数

这个函数用于将最底层的diffLayer写入diskLayer。执行流程：

1. 如果diskLayer处于构建中，先中断它；
2. 删除SnapshotRoot；
3. 将diskLayer.stale设置为true;
4. 遍历diffLayer的所有destructSet，清理掉删除的账户和存储，当然如果还在构建过程中且diskLayer.genMarker比较小的时候，则可以跳过；
5. 遍历diffLayer.accountData，覆盖DB中的账户，如果还在构建中且genMarker比较小时，可以跳过；
6. 遍历diffLayer.storageData，覆盖DB中账户的存储，如果还在构建中且genMarker更小时，可以跳过；
7. 上述遍历过程中，一旦batch到一定大小就刷入DB；
8. 更新snapshotRoot和当前的构建进度信息，如果还在构建中，则开routine继续构建；



### Cap函数

这个函数将root所在的layer向上回溯layers层后将剩下的diffLayer进行压缩，执行流程：

```go
    func (t *Tree) Cap(root common.Hash, layers int) error
```

1. 找到这个root对应的diffLayer;
2. 如果diskLayer还在构建过程中的话，设置layers不超过8，如果太大的话，正在构建的trie和当前区块所在的trie差距过大，可能有些trie节点会被gc掉；
3. 压缩给定的diffLayers，如果压缩后的内存占用过大就调用diffToDisk函数写入DB，同时如果diskLayer正在构建中,也进行写入，原因同上面的；
4. 把压缩之后很多分叉节点的diffLayer也会跟着失效，因为他们的parent已经失效了，也需要从tree.layers中清理掉；
5. 如果写入了diskLayer，则各个diffLayer的bloom已经失效了，需要重新生成。



## 使用方式



```go
// 从diskdb中加载根为root的snapshot，如果不存在则会开routine重建，async表示是否等重建完成，rebuild表示是否重新生成snapshot
// 如果async是false的话，返回的tree是部分构建的；
// 如果存在snapshot的话，会加载对应的journal，重建内存diffLayer；
tree = New(diskdb, triedb, cache, root, async, rebuild, recovery)
// 往tree中基于parentRoot添加一层diffLayer，blockRoot是该层的根，后面是该层的数据
tree.Update(blockRoot, parentRoot, destructs, accounts, storage)
// 当添加了足够的diffLayer之后，调用这个方法将多余的层刷入diskLayer
tree.Cap(root, layers)
// 调用这个方法可以返回trie根为root，起始位置为seekHash的account的迭代器，后面通过这个迭代器可以访问连续的账户数据
accIter = tree.AccountIterator(root, seekHash)
// 返回trie根为root的account的下的起始为值为seekHash的storage迭代器；
storeIter = tree.StorageIterator(root, account, seekHash) 
// 进程需要关闭时，调用这个方法将diffLayer进行持久化
tree.Journal(root common.Hash)
```

