# trie的database源码分析

Database是处于disk和内存trie数据结构的中间层，用于累积内存trie的写入，然后定期写入disk，并对剩余的进行垃圾回收。这个Database本身不支持线程安全的并发修改，但是可以提供线程安全的独立的node访问。这样设计的目的是即便内部在执行昂贵的垃圾回收操作也能够同时对外和rpc接口提供数据读取服务。

主要的结构是Database和cachedNode。

## cachedNode分析

cachedNode是Database内部管理的缓存节点，结构定义如下：

```go
type cachedNode struct {
   node node   // 实际缓存坍缩过的节点（坍缩的节点是存入db的最小单位），或者已经rlp系列化的节点数据
   size uint16 // 这个节点实际占用的内存byte数

   parents  uint32                 // 引用了这个节点的父节点数
   children map[common.Hash]uint16 // 以这个节点为父节点的所有子节点列表

   flushPrev common.Hash // 这个节点的在flush列表中的前一个节点
   flushNext common.Hash // 这个节点在flush列表中的后一个节点
}
```

如果管理的这些节点在内存中占用太高，那么需要把一部分节点刷入disk，因此逻辑上把这些要刷入disk的节点构成一个双向链表，通过每个节点的fluashPrev和flushNext来维护。节点的引用计数通过parents和children来跟踪。

cachedNode中的node有一下几种类型，都是坍缩过的：

1. rawNode：已经通过rlp系列化好的bytes;
2. rawFullNode：将FullNode中的cache和flags清理掉后，只保留叶子和value信息；
3. rawShortNode: 把ShortNode中cache和flags清理掉后保留下value和key信息，key是compact格式；

```go
type rawNode []byte
type rawFullNode [17]node
type rawShortNode struct {
	Key []byte
	Val node
}
```

cachedNode定义了一个关键的forChilds函数，用于更新引用计数等。注意这个函数不但遍历这个节点内部所有子节点包含的hashNode，还遍历外部引用了这个节点的子节点：

```go
func (n *cachedNode) forChilds(onChild func(hash common.Hash)) {
   for child := range n.children {
      onChild(child)
   }
   if _, ok := n.node.(rawNode); !ok {
      forGatherChildren(n.node, onChild)
   }
}

// forGatherChildren traverses the node hierarchy of a collapsed storage node and
// invokes the callback for all the hashnode children.
func forGatherChildren(n node, onChild func(hash common.Hash)) {
   switch n := n.(type) {
   case *rawShortNode:
      forGatherChildren(n.Val, onChild)
   case rawFullNode:
      for i := 0; i < 16; i++ {
         forGatherChildren(n[i], onChild)
      }
   case hashNode:
      onChild(common.BytesToHash(n))
   case valueNode, nil, rawNode:
   default:
      panic(fmt.Sprintf("unknown node type: %T", n))
   }
}
```

注意遍历所有子节点包含的hashNode的原因是这个节点已经是坍缩过的，对于存储来说是作为一个整体系列化存储的，存储后节点之间的关联关系就是通过hashNode联系起来的。



## Database分析

Database的结构定义如下：

```go
type Database struct {
   diskdb ethdb.KeyValueStore // 持久层

   cleans  *fastcache.Cache            // 可以为nil，缓存了从disk中加载的干净节点的rlp编码bytes，
   dirties map[common.Hash]*cachedNode // 脏的trie节点列表和相互引用关系
   oldest  common.Hash                 // flush列表的头节点hash
   newest  common.Hash                 // flush列表的末尾节点hash

   preimages map[common.Hash][]byte // 保存了secure trie的数据原像，可选的

   // 下面的都是统计信息，
   gctime  time.Duration      // Time spent on garbage collection since last commit
   gcnodes uint64             // Nodes garbage collected since last commit
   gcsize  common.StorageSize // Data storage garbage collected since last commit

   flushtime  time.Duration      // Time spent on data flushing since last commit
   flushnodes uint64             // Nodes flushed since last commit
   flushsize  common.StorageSize // Data storage flushed since last commit

   dirtiesSize   common.StorageSize // Storage size of the dirty node cache (exc. metadata)
   childrenSize  common.StorageSize // Storage size of the external children tracking
   preimagesSize common.StorageSize // Storage size of the preimages cache

   lock sync.RWMutex
}

// 创建时候的配置信息
type Config struct {
   Cache     int    // 缓存trie节点的内存上限，单位MB
   Journal   string // journal文件名称，用于将cleans中的数据存入文件，方便重启时恢复，可选的
   Preimages bool   // 是否记录preimage
}
```

cleans字段缓存了从disk中加载的干净节点的rlp编码bytes，因此Database本身也可以当做读缓存使用。



### New函数分析

```go
func NewDatabaseWithConfig(diskdb ethdb.KeyValueStore, config *Config) *Database 
```

执行流程：

1. 构造空的db实例；
2. 如果config不为空，并且config.Cache大于0，那么初始化cleans，如果config.Journal不为空，那么从文件系统中加载；
3. 如果config.Preimages为true的话，初始化db.preimages。



### preimage插入

```go
func (db *Database) insertPreimage(hash common.Hash, preimage []byte)
```

更新db.preimages和preimagesSize字段。

### 节点的插入

```go
func (db *Database) insert(hash common.Hash, size int, node node) 
```

Database的插入函数不是公开的，只能通过Trie调用。插入的node必须是已经坍缩过的，插入的流程：

1. 检查这个节点是否存在db.dirties中，存在则直接退出；
2. 构造cachedNode，将它添加到flush列表的末端，并更新db.newest，如果是第一次插入的话也初始化db.oldest；
3. 遍历这个节点的所有子hashNode，如果这些hashNode存在dirties中，则累加他们的parents计数器；
4. 将这个节点保存到db.dirties中，并更新dirtiesSize;



### 节点的查找

```go
func (db *Database) node(hash common.Hash) node
func (db *Database) Node(hash common.Hash) ([]byte, error) 
```

节点查找有内部访问的node和外部访问的Node版本，区别是一个返回rlp编码后的bytes，一个返回解码过的。主要流程：

1. 如果db.cleans启用了，则尝试读取返回；
2. 尝试从dirties中查找返回；
3. 尝试从diskdb查找，并设置到db.cleans缓存中；



### 添加引用关系

```go
func (db *Database) Reference(child common.Hash, parent common.Hash) {
	db.lock.Lock()
	defer db.lock.Unlock()

	db.reference(child, parent)
}
func (db *Database) reference(child common.Hash, parent common.Hash) 
```

因为root节点是没有parent引用的，为避免被gc掉，认为他的parent是空hash。

主要流程：

1. 如果child不在dirties中，那么这个节点是从disk中加载出来的，直接返回；
2. 如果parent节点的children列表中已经包含了child节点，并且child节点不是root节点，直接返回；也就是root节点会重复计数，所以cachedNode的map字段children的值类型用了uint16；
3. 将child节点的parent字段和parent节点的对应children中的计数器递增；

> 注：root节点允许重复计数的原因是公开的Reference函数支持并发调用(持有了锁)，同一个trie根可能同时很多个routine感兴趣，如果不重复计数的话，当某个routine不感兴趣了进行解引用，会导致这个trie被gc清理掉，其他routine就没得访问了。


### 解除引用

```go
func (db *Database) Dereference(root common.Hash) {
   db.lock.Lock()
   defer db.lock.Unlock()
   db.dereference(root, common.Hash{})
}

func (db *Database) dereference(child common.Hash, parent common.Hash)
```

应用上来说，当某个trie已经不再需要的时候，那么可以从这个trie的root节点开始进行解除引用的操作，以便将它从内存中清理掉。由于需要递归处理，主要工作是委托到了dereference方法中：

1. 先递减parentNode.children的计数器；
2. 再从db.dirties中获取childNode，如果不存在，那么这个节点在之前已经commit过了，直接返回；
3. 递减childNode.parent计数器，如果计数器不为0就直接返回，否则这个节点可以清除了：
4. 将这个childNode从flush列表中移除；
5. 调用`childNode.forChilds(func(hash common.Hash) {  db.dereference(hash, child) })`，递归解除childNode和他自己的子节点的关系；
6. 将child从db.dirties中删除；



### 减小内存占用

```go
func (db *Database) Cap(limit common.StorageSize) error 
```

为了减少内存占用，可以调用这个函数强制将内存中的部分节点写入disk。

执行流程：

1. 如果preimages的大小超过4M，那么写入disk;
2. 依次遍历flush列表，将dirties中的节点写入diskdb，直到内存占用满足要求；
3. 确保所有数据写入diskdb之后，才将flush列表中遍历过的节点从dirties中删除，并且重置preimages，这样确保外部能够访问到一致的状态；

注意：这种方式会将一些后面没用的过期trie节点写入存储，但不会妨碍正常使用，只会增加点存储开销。



### 提交到diskdb

在分析commit的实现前需要分析一个cleaner的辅助结构。为了数据的一致性，Database.Commit的时候需要先将数据完整写入diskdb后，才清理内存里的数据，清理操作由cleaner的来完成，它是作为batch replayer，他接收一个写操作的batch，并将所有写入diskdb的数据从trie database中清理掉，定义如下：

```go
type cleaner struct {
	db *Database
}
func (c *cleaner) Put(key []byte, rlp []byte) error 
```

Put操作:

1. 从db.dirties中读取对应key的节点，如果不存在的话就退出；
2. 将节点从flush列表中移除；
3. 将节点充db.dirties中删除；
4. 如果db.cleans启用了话，将节点数据放入cleans中，避免下次使用时从diskdb中加载；



下面分析Commit操作：

这个函数将迭代给定的node下的所有children，将他们写入diskdb，并将两个方向的相关引用计数清理掉。同时所有累积的preimages也顺便会写入diskdb。

```go
func (db *Database) Commit(node common.Hash, report bool, callback func(common.Hash)) error
```

执行流程：

1. 如果db.preimages非空，则写入diskdb的batch;
2. 调用node.forChilds方法，先递归将子节点commit，再将自己写入batch;
3. batch.Write之后，调用batch.Replay(cleaner)，清理掉内存中的数据；



### cache保存

db.cleans中的cache数据也可以保存在独立的文件中，因此提供了两个方法，分别用于手动和定期触发备份操作。

```go
func (db *Database) SaveCache(dir string) error 
func (db *Database) SaveCachePeriodically(dir string, interval time.Duration, stopCh <-chan struct{}) 
```

