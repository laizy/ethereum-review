# trie的sync源码分析

这个模块主要是为了解决将一个在远端节点随区块增加不断变化的trie同步到本地的问题。核心数据结构是Sync，他负责请求远端的trie节点的调度。

## SyncBloom分析

整个同步的过程都遵循一个不变量：只有当某个trie节点的所有子节点都落账时，这个节点才满足提交条件才能落账，同时对于account trie的叶子节点，必须等对应的storage trie全部落账才能落账。因此当同步到一个trie节点的时候，需要向存储查询是否他们的子节点已经落账了，没有的话再继续构造请求向远端节点获取。SyncBloom用于加速查询的过程，内部使用了bloom filter，他的特性是如果查询说节点不存在，那就一定不存在，可以直接向远端请求；如果查询说可能存在，才向存储读取，看看是否真的存在。整个bloom filter的构建需要遍历整个账本，需要耗费点时间，在构建的过程中不能提供查询服务，因为bloom filter返回false必须是可靠的，账本里的某个key还没添加进bloom filter，这时候查询会返回false，那么调用方就会直接去远端请求数据，浪费了带宽。

### 结构定义

```go
type SyncBloom struct {
    bloom  *bloomfilter.Filter  // 这个结构可以并发修改
    inited uint32    // 整个bloom filter是否已经完全构建好了，原子读写
    closer sync.Once  // 防止并发和多次调用Close
    closed uint32     // 是否closed了，原子读写
    pend   sync.WaitGroup  // 用于同步背后线程的退出
}
```

其中bloom字段会并发修改：一个是初始构建的时候从db里迭代所有的key插入；另一个是当新的trie节点从网络上同步到写入db的时候也会顺带插入。

### 对外API函数

```go
func NewSyncBloom(memory uint64, database ethdb.Iteratee) *SyncBloom
func (b *SyncBloom) Close() error
func (b *SyncBloom) Add(hash []byte)
func (b *SyncBloom) Contains(hash []byte) bool 
```

NewSyncBloom函数：初始化新的SyncBloom，然后开一个新的init线程用于从database中遍历key，填充bloom。背后init线程不断迭代过程中会检查closed字段，检查退出信号。当所有的都初始化完全后原子设置inited字段并退出。

Close函数：原子设置closed，设置inited为false，bloom设为nil。

Add函数：检查closed，如果没关掉，那么添加到bloom字段中。（注：其实这里读取bloom字段和Close函数有并发问题）

Contains函数：先检查是否inited，没有直接返回true，否则返回bloom内部的查询值。



## Sync结构分析



### 数据结构定义

正在进行中的trie节点请求通过request结构管理，定义如下：

```go
type LeafCallback func(path []byte, leaf []byte, parent common.Hash) error
type request struct {
    path []byte      // 请求的trie节点的merkle路径，用于优先级
    hash common.Hash // 节点数据的hash
    data []byte      // 节点数据，缓存直到所有的依赖子树完成
    code bool        // 是否是code项

    parents []*request // 所有引用了这个节点的父节点
    deps    int        // 在允许提交这个节点前的依赖数

    callback LeafCallback // 遇到要处理叶子节点时的回调函数
}
```

其中deps表示要满足提交条件前还依赖的子trie节点数目，parents表示所有依赖了这个节点的父节点数目。之所以是数组是因为有多个trie根。

为了将满足提交条件的trie节点批量写入账本，定义了下面的结构将节点临时缓存在内存中：

```go
type syncMemBatch struct {
    nodes map[common.Hash][]byte 
    codes map[common.Hash][]byte 
}
```



请求trie节点的路径分成了两种：如果获取account trie中的路径是1-tuple，获取storage trie的路径是2-tuple。为了表示奇数个nibble的路径，第一种和第二种的第二个元素路径采用compact编码。 第二种的第一个元素因为总是32byte，所以采用简单的binary编码。之所以采用2-tuple，而不是一长串路径是因为方便服务端查询，account可以直接解析出来而不需要进行compact解码，而storage的路径则可以传递给trie直接处理。同时对于snap协议来说，可以很方便地将storage的请求打包一起发送。请求的路径定义如下：

```go
// Examples:
//   - Path 0x9  -> {0x19}
//   - Path 0x99 -> {0x0099}
//   - Path 0x01234567890123456789012345678901012345678901234567890123456789019  -> {0x0123456789012345678901234567890101234567890123456789012345678901, 0x19}
//   - Path 0x012345678901234567890123456789010123456789012345678901234567890199 -> {0x0123456789012345678901234567890101234567890123456789012345678901, 0x0099}
type SyncPath [][]byte

// newSyncPath converts an expanded trie path from nibble form into a compact
// version that can be sent over the network.
func newSyncPath(path []byte) SyncPath {
    if len(path) < 64 {
        return SyncPath{hexToCompact(path)}
    }
    return SyncPath{hexToKeybytes(path[:64]), hexToCompact(path[64:])}
}
```

Sync的数据结构定义如下：

```go
type Sync struct {
    database ethdb.KeyValueReader     // 用于查询db里是否存在trie
    membatch *syncMemBatch            // 提交过的node和code请求
    nodeReqs map[common.Hash]*request // 当前还不能提交的node请求，可能本身已经请求到了数据，但还在等子节点
    codeReqs map[common.Hash]*request // 当前还不能提交的code请求
    queue    *prque.Prque             // 当前等待对外发起的请求，包括node和code请求
    fetches  map[int]int              // 当前每个path深度对外发起了请求还没有提交的数量，做流量控制。
    bloom    *SyncBloom               
}
```

### commit函数

```go
func (s *Sync) commit(req *request) (err error) 
```

commit函数用于将所有依赖的子节点都已经下载好的节点放进membatch中。函数会递归调用尝试将没有依赖的父节点加入membatch，流程：

1. 如果req.code，那么将数据放进membatch.codes，并从codeReqs中删除；
2. 否则放进membatch.nodes，并从nodeReqs中删除；
3. 遍历req.parents，递减parent.deps后为0的话，也满足commit条件，则递归调用；

### Commit函数

```go
func (s *Sync) Commit(dbw ethdb.Batch) error 
```

这个是对外的api函数，用于将membatch中的数据写入参数提供的Batch，并重置membatch。

### schedule函数

```go
func (s *Sync) schedule(req *request)
```

这个函数用于将一个请求放进优先级队列和相应的nodeReqs或者codeReqs列表中：

1. 检查这个请求是否已经存在，如果存在的话将它的parents和存在的parents进行合并，并返回；
2. 否则将这个请求加入对应的请求列表；
3. 计算请求的优先级，并放入queue队列中，优先级顺序是路径长度越大，优先级越高；key越小，也即树左边的节点优先级越高；

```go
prio := int64(len(req.path)) << 56 // depth >= 128 will never happen, storage leaves will be included in their parents
for i := 0; i < 14 && i < len(req.path); i++ {
    prio |= int64(15-req.path[i]) << (52 - i*4) // 15-nibble => lexicographic order
}
```

### children函数

这个函数遍历给定节点的所有子节点，构造出子节点是hashNode，并且不存在membach和本地db的请求数据，以便接下来的调度。

```go
func (s *Sync) children(req *request, object node) ([]*request, error) {
    // Gather all the children of the node, irrelevant whether known or not
    type child struct {
        path []byte
        node node
    }
    var children []child

    switch node := (object).(type) {
    case *shortNode:
        key := node.Key
        // 有疑问：这里会把可以末尾的0x10移除，但是下面的fullNode的Value对应的path却保留了0x10.
        // 从目前的使用场景来看，fullNode里不可能有value，所以暂时没有出问题？
        // 问题解决：当hasTerm是真的话，node.Val一定是valueNode，因此在下面的循环中不会构造请求，
        // 而是会调用callback函数，传递出去的path参数是前缀，因此这里需要移除。下面的fullNode确实有问题，需要对value做处理。
        if hasTerm(key) {
            key = key[:len(key)-1]
        }
        children = []child{{
            node: node.Val,
            path: append(append([]byte(nil), req.path...), key...),
        }}
    case *fullNode:
        for i := 0; i < 17; i++ {
            if node.Children[i] != nil {
                children = append(children, child{
                    node: node.Children[i],
                    path: append(append([]byte(nil), req.path...), byte(i)),
                })
            }
        }
    default:
        panic(fmt.Sprintf("unknown node: %+v", node))
    }
    // Iterate over the children, and request all unknown ones
    requests := make([]*request, 0, len(children))
    for _, child := range children {
        // Notify any external watcher of a new key/value node
        if req.callback != nil {
            if node, ok := (child.node).(valueNode); ok {
                if err := req.callback(child.path, node, req.hash); err != nil {
                    return nil, err
                }
            }
        }
        // If the child references another node, resolve or schedule
        if node, ok := (child.node).(hashNode); ok {
            // Try to resolve the node from the local database
            hash := common.BytesToHash(node)
            if s.membatch.hasNode(hash) {
                continue
            }
            if s.bloom == nil || s.bloom.Contains(node) {
                // Bloom filter says this might be a duplicate, double check.
                // If database says yes, then at least the trie node is present
                // and we hold the assumption that it's NOT legacy contract code.
                if blob := rawdb.ReadTrieNode(s.database, hash); len(blob) > 0 {
                    continue
                }
                // False positive, bump fault meter
                bloomFaultMeter.Mark(1)
            }
            // Locally unknown node, schedule for retrieval
            requests = append(requests, &request{
                path:     child.path,
                hash:     hash,
                parents:  []*request{req},
                callback: req.callback, // 继承了父节点的callback
            })
        }
    }
    return requests, nil
}
```

注意这个函数有副作用：当在object这个node中发现了叶子节点时会调用req.callback函数。并且新生成的child节点的callback会继承这个节点的callback。也就是说只要trie的root设置了callback，那么所有的子节点都会用这个callback，这样最终遇到的所有叶子节点都会调用到。从core/state/sync.go处的使用来看，这个callback会设置在account trie上，当发现叶子节点时解析出Account结构，并拿到对应的Storage trie的root和codeHash，然后通过下面的AddSubTrie函数和AddCodeEntry函数添加进Sync里，Storage trie的同步不会设置callback。调用AddSubTrie的时候传递的parent就是这个account叶子节点，这个确保了在storage trie没有完全同步好之前，这个叶子节点不满足提交条件而先落账。

### Process函数

```go
type SyncResult struct {
    Hash common.Hash // trie节点的hash
    Data []byte      // 获取到的节点数据
}
func (s *Sync) Process(result SyncResult) error 
```

这个函数用于处理收到的trie节点响应，流程：

1. 检查nodeReqs和codeReqs，确保是我们之前请求的；
2. 如果是codeReq，则可以直接commit;
3. 如果是nodeReq，先把node解码出来，并调用children构造出获取子节点的请求，如果请求数为0，并且nodeReq.deps为0，则直接commit，否则将nodeReq.deps 加上请求数，并调用schedule函数调度每个请求。

### AddSubTrie函数

```go
func (s *Sync) AddSubTrie(root common.Hash, path []byte, parent common.Hash, callback LeafCallback)
```

添加一个根为root的子trie，路径前缀是path，对应的父节点是parent。执行流程：

1. 如果root是空，或者membatch里已经存在这个root，则直接退出；
2. 检查账本里是否存在，有则退出；
3. 构造req请求，设置req.path = path, req.hash=root, req.callback=callback;
4. 如果parent非空，要必须在nodeReqs中(否则panic)，然后递增parent的deps，并设置req.parents = {parentReq}
5. 调用schedule调度这个请求；

### AddCodeEntry函数

添加一个code hash请求，这个path对应的是account的路径，应该只用于计算优先级，parent是对应的account的hash。执行流程和AddSubTrie类似：

1. 如果hash是空，或者membatch里已经存在这个hash，则直接退出；
2. 检查账本里是否存在code，有则退出；
3. 构造req请求，设置req.path = path, req.hash=root, req.code=true;
4. 如果parent非空，要必须在nodeReqs中(否则panic)，然后递增parent的deps，并设置req.parents = {parentReq}
5. 调用schedule调度这个请求；

### NewSync函数

```go
func NewSync(root common.Hash, database ethdb.KeyValueReader, callback LeafCallback, bloom *SyncBloom) *Sync
```

创建新的Sync，并且通过AddSubTrie将root添加进sync里。

### Missing函数

```go
func (s *Sync) Missing(max int) (nodes []common.Hash, paths []SyncPath, codes []common.Hash) 
```

这个函数用于获取需要同步的trie节点和trie路径，以及code。流程：

1. 循环遍历queue队列，取出其中的元素item；
2. 根据item的优先级，反解出深度depth，检查s.fetches[depth]是否超过了maxFetchesPerDepth(16384)，超过了退出循环；
3. 递增s.fetches[depth]，并检查item是在s.nodeReqs还是s.codeReqs里，放进对应的返回值列表；
4. 当队列为空，或者总列表长度到达了参数max后退出；

