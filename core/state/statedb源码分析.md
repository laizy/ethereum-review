# StateDB源码分析

StateDB用于给智能合约执行提供状态读写，本身是一个内存缓存。尽管trie节点在内存中本身也是缓存，但每次合约状态的修改都涉及到对多个节点的修改，有不少开销。因此StateDB内部又增加了两层map，单个交易内所有修改都临时放进dirty map，等交易执行完从dirty map写入pending map。当区块执行完才将pending map写入trie，再将trie写入trie.Database缓存，因此数据要真正落盘要经过四五层的缓存！另外为了支持evm的call级别的状态回滚，内部维护了一个journal，每次状态变更都会往里面插入一条undo日志。另外为了支持evm的gas计费，又额外维护了access list，refund等结构。为了将trie节点提前从磁盘加载到内存进行预热，定义了trie prefetcher的结构。



## Database接口

trie缓存层已经在trie/database里实现好了，statedb这里的database做了一个简单的封装，增加了合约code和大小的缓存，默认是64M和100K。

```go
type Database interface {
	// 打开账户trie
	OpenTrie(root common.Hash) (Trie, error)
	// 打开某个用户的存储trie
	OpenStorageTrie(addrHash, root common.Hash) (Trie, error)
	// 拷贝一个独立的trie，因为底层trie节点是写时复制的，因此拷贝非常的轻量
	CopyTrie(Trie) Trie
	// 获取某个合约
	ContractCode(addrHash, codeHash common.Hash) ([]byte, error)
	// 获取每个合约的大小
	ContractCodeSize(addrHash, codeHash common.Hash) (int, error)
	// 获取底层的trie database缓存
	TrieDB() *trie.Database
}

type Trie interface {
	// 返回sha3的原像，这个函数只当外部查询用，可以不支持。
	GetKey([]byte) []byte

	// 获取key关联的value，value不能在外部修改。如果trie中有节点不存在db中，则报trie.MissingNodeError错误
	TryGet(key []byte) ([]byte, error)

	// 写入账户数据，这个只有账户trie有用
	TryUpdateAccount(key []byte, account *types.StateAccount) error

	// 将关联的key的值设置为value，如果val为nil，等效于删除。调用方调用之后不能对value进行修改。
    // 如果trie中有节点不存在db中，则报trie.MissingNodeError错误
	TryUpdate(key, value []byte) error

	// 将删除给定key，如果trie中有节点不存在db中，则报trie.MissingNodeError错误
	TryDelete(key []byte) error

	// 返回trie的根
	Hash() common.Hash

	// 将所有节点写入trie.database缓存
	Commit(onleaf trie.LeafCallback) (common.Hash, int, error)

	// 返回一个迭代器，以先序的方式遍历trie中的节点NodeIterator returns an iterator that returns nodes of the trie. Iteration
	// starts at the key after the given start key.
	NodeIterator(startKey []byte) trie.NodeIterator

    // 构造key的存在/不存在性证明，fromLevel表示只返回给定level开始的节点数据，为0表示从root开始,
    // key对应的value的值可以从证明的最后一个叶子节点中取出
	Prove(key []byte, fromLevel uint, proofDb ethdb.KeyValueWriter) error
}
```



## trie数据预取

数据获取由两个结构构成，triePrefetcher结构用于预先从db中加载trie节点的，将尽可能多的有用节点缓存在内存中。它可以加载多个trie，其中单个trie的获取由subfetcher结构完成。

### subfetcher分析

subfetcher采用event loop的结构，构造好之后会立马创建loop线程，subfetcher的定义如下：

```go
type subfetcher struct {
	db   Database    
	root common.Hash // 需要获取的trie根
	trie Trie        // 从db获取数据填充的内存版本trie

	tasks [][]byte   // 需要获取的key队列
	lock  sync.Mutex // 保护上面的队列

	wake chan struct{}  // 唤醒loop线程
	stop chan struct{}  // 暂停loop线程
	term chan struct{}  // Channel to signal iterruption
	copy chan chan Trie // 向loop线程发送请求，拷贝当前填充的trie

	seen map[string]struct{} // 跟踪已经加载过的key
	dups int                 // 重复加载的key的计数器
	used [][]byte            // 跟踪加载的key被使用的记录
}
```

#### 构造函数

```go
func newSubfetcher(db Database, root common.Hash) *subfetcher {
	sf := &subfetcher{
		db:   db,
		root: root,
		wake: make(chan struct{}, 1),
		stop: make(chan struct{}),
		term: make(chan struct{}),
		copy: make(chan chan Trie),
		seen: make(map[string]struct{}),
	}
	go sf.loop()
	return sf
}
```

注意到wake设置了长度为1的通道容量，确保可以采用不阻塞的方式唤醒loop线程。

#### schedule函数

```go
func (sf *subfetcher) schedule(keys [][]byte)
```

这个函数用于给loop线程设置预取任务，函数将keys放入sf.tasks中后，通过sf.wake唤醒loop线程

#### abort函数

```go
func (sf *subfetcher) abort() {
   select {
   case <-sf.stop:
   default:
      close(sf.stop)
   }
   <-sf.term
}
```

这个函数用于通知loop线程永久终止预取任务。先尝试关掉sf.stop来通知loop线程，然后等待loop线程关闭sf.term。这个函数可以多次调，但不能并发调用（会竞争sf.stop的关闭操作）。

#### peek函数

```go
func (sf *subfetcher) peek() Trie 
```

这个函数用于拿到当前预取好的trie的副本，如果预取之前已经终止了，那么直接拷贝一份sf.trie。否则通过sf.copy给loop线程发rpc消息，然后等结果。注：sf.copy等chan的容量设置为0是很关键的，确保了发过去的请求loop线程收到后能够立马处理并回复，否则发送进了chan，但loop线程又正在做关闭操作的话，可能永远拿不到回复。

#### loop函数

```go
func (sf *subfetcher) loop() 
```

这个函数当做单独routine执行，启动时先从db中打开要获取的trie，然后进入循环等消息：

1. 收到sf.wake：表示有新的任务，从sf.tasks中取出要获取的keys，然后遍历每一个不存在sf.seen中的key，从db中获取并放入sf.seen；
2. 如果收到sf.copy消息，则拷贝一份当前的trie发送回去；
3. 如果收到sf.stop消息，则将没有获取完的key放回sf.stasks并关闭sf.term返回；



### triePrefetcher分析

triePrefecher的定义如下：

```go
type triePrefetcher struct {
	db       Database                    
	fetches  map[common.Hash]Trie        
	fetchers map[common.Hash]*subfetcher 
    
	root     common.Hash                 // 账户trie的根，这个只用于metrics
}
```

其中fetchers和fetches只有一个字段会设置，由构造方式而定。fetchers是通过正常的方式构造设置的，管理了所有subfetcher，而采用拷贝方式构造时，会将当前所有subfetcher已经获取到trie进行拷贝，放进fetches中，这个拷贝出来的triePrefetcher不会再开routine进行预取。正常的构造方式如下：

```go
func newTriePrefetcher(db Database, root common.Hash, namespace string) *triePrefetcher {
	p := &triePrefetcher{
		db:       db,
		root:     root,
		fetchers: make(map[common.Hash]*subfetcher), // Active prefetchers use the fetchers map
	}
	return p
}
```

其中root和namespace参数不重要，都是配合metrics用的。

#### close函数

```go
func (p *triePrefetcher) close() {
	for _, fetcher := range p.fetchers {
		fetcher.abort() // safe to do multiple times
	}
	// Clear out all fetchers (will crash on a second call, deliberate)
	p.fetchers = nil
}
```

这个函数会关掉所有的subfetcher。只能调用一次。

#### copy函数

```go
func (p *triePrefetcher) copy() *triePrefetcher 
```

这个函数构造新的拷贝，将已经预取到的数据放进fetches，但是不会再开subfetcher的routine进行预取。



#### prefetch函数

```go
func (p *triePrefetcher) prefetch(root common.Hash, keys [][]byte)	
```

这个函数用于发布预取任务，函数流程：

1. 如果p.fetches为空，那么这是拷贝构造出来的，直接返回；
2. 从p.fetches中看是否有root的记录，没有就创建新的subfetcher；
3. 调用subfetcher.schedule方法发任务。



#### trie函数

```go
func (p *triePrefetcher) trie(root common.Hash) Trie 
```

这个函数用于拷贝一份已经预取的根为root的trie，如果给定的root不存在则返回nil。流程：

1. 如果p.fetches为nil，则从p.fetches中返回一份拷贝；
2. 否则从p.fetchers中找到对应的fetcher，调用fetcher.abort关掉预取，并返回fetcher.peek。



### Journal分析

EVM支持合约级别的状态回滚机制，允许在调用合约发生revert时，将调用时产生的状态修改回滚到调用前。journal是一个redo日志，用来处理这个逻辑。定义如下：

```go
type journalEntry interface {
	// 回滚这个操作
	revert(*StateDB)

	// 返回这个操作修改的地址
	dirtied() *common.Address
}

type journal struct {
	entries []journalEntry         // 当前的修改列表
	dirties map[common.Address]int // 修改过的账户和对应的修改次数
}
```



#### append函数

```go
func (j *journal) append(entry journalEntry) {
   j.entries = append(j.entries, entry)
   if addr := entry.dirtied(); addr != nil {
      j.dirties[*addr]++
   }
}
```

这个函数添加一个日志记录，如果有修改的地址，则递增相应的dirty数值。

#### revert函数

```go
func (j *journal) revert(statedb *StateDB, snapshot int) 
```

这个函数用于回滚将日志记录回滚到snapshot之前。其反向遍历所有的下标大于等于snapshot的日志，进行回滚，并相应递减dirty计数。

#### dirty函数

```go
func (j *journal) dirty(addr common.Address) {
	j.dirties[addr]++
}
```

这个函数用于强制设置某个地址为dirty，尽管实际上没有修改。这是一个hack，主要是兼容[实现EIP 161错误，导致RIPEMD合约误删的一个bug](https://github.com/ethereum/go-ethereum/pull/3341)。

#### 日志记录类型

所有对StateDB状态的修改都需要记录日志，定义如下：

```go
type (
	createObjectChange struct {
		account *common.Address
	}
	resetObjectChange struct {
		prev         *stateObject
		prevdestruct bool
	}
	suicideChange struct {
		account     *common.Address
		prev        bool // whether account had already suicided
		prevbalance *big.Int
	}

	balanceChange struct {
		account *common.Address
		prev    *big.Int
	}
	nonceChange struct {
		account *common.Address
		prev    uint64
	}
	storageChange struct {
		account       *common.Address
		key, prevalue common.Hash
	}
	codeChange struct {
		account            *common.Address
		prevcode, prevhash []byte
	}

	refundChange struct {
		prev uint64
	}
	addLogChange struct {
		txhash common.Hash
	}
	addPreimageChange struct {
		hash common.Hash
	}
	touchChange struct {
		account *common.Address
	}
	accessListAddAccountChange struct {
		address *common.Address
	}
	accessListAddSlotChange struct {
		address *common.Address
		slot    *common.Hash
	}
)
```



### Access List分析

访问列表用来支持evm的gas计费，evm在第一次加载账户和存储slot的时候因为要从底层db获取，会收取较高的gas。因此需要一个列表来保存当前已经访问过了哪些数据。accessList定义如下：

```go
type accessList struct {
	addresses map[common.Address]int       // value存储了这个地址的slot记录在slots变量中的位置，-1表示还没有slot记录
	slots     []map[common.Hash]struct{}
}
```

#### AddAddress函数

```go
func (al *accessList) AddAddress(address common.Address) bool {
	if _, present := al.addresses[address]; present {
		return false
	}
	al.addresses[address] = -1
	return true
}
```

这个函数用于添加address，如果已经添加过返回false，否则设置al.addresses[address] = -1。

#### AddSlot函数

```go
func (al *accessList) AddSlot(address common.Address, slot common.Hash) (addrChange bool, slotChange bool) 
```

这个函数会先添加地址记录，再添加slot记录，函数返回是否添加了，便于添加journal日志。

#### 删除操作

```go
func (al *accessList) DeleteSlot(address common.Address, slot common.Hash)
func (al *accessList) DeleteAddress(address common.Address)
```

这个操作是journal回滚的时候调用，因此必须是按照添加时相反的顺序依次调用，否则内部检测到可以直接panic。

#### 查询操作

```go
func (al *accessList) ContainsAddress(address common.Address) bool
func (al *accessList) Contains(address common.Address, slot common.Hash) (addressPresent bool, slotPresent bool) 
```

分别用于查询地址和slot的记录。

### State Object分析

stateObject用于管理单个正在被修改的账户，结构定义如下：

```go
type Storage map[common.Hash]common.Hash

type stateObject struct {
	address  common.Address
	addrHash common.Hash // 账户地址的hash，作为账户trie的key
	data     types.StateAccount
	db       *StateDB

	// db相关的操作报错存储在这里，当Commit的时候返回。所以当出错了evm还会继续瞎执行。
	dbErr error

	// Write caches.
	trie Trie // storage trie
	code Code // contract bytecode

	originStorage  Storage // 缓存合约原始的key-value，防止重复写，每个交易执行前重置
	pendingStorage Storage // 缓存了这个交易之前的执行数据，当整个区块执行结束时会flush到disk
	dirtyStorage   Storage // 当前交易执行的修改
	fakeStorage    Storage // 用于调试，由caller设置的假存储数据

	// Cache flags.
	dirtyCode bool // 账户的code是否更新过，为true的话commit时要保存code
	suicided  bool // 这个账户在当前交易执行时是否销毁了。
	deleted   bool // 销毁的账户在把交易修改提交到pending中的时候设置
}
```



#### empty函数

```go
func (s *stateObject) empty() bool
```

这个函数用于返回账户是否是空的，定义为：nonce和balance为0，codehash为keccak256(nil)

#### SetNonce函数

```go
func (s *stateObject) SetNonce(nonce uint64) 
```

这个函数设置s.data.Nonce，并往.journal中添加nonceChange日志。

#### SetCode函数

```go
func (s *stateObject) SetCode(codeHash common.Hash, code []byte) 
```

这个函数设置s.code，s.data.CodeHash和s.dirtyCode，并往journal中添加codeChange的日志。



#### touch函数

```go
func (s *stateObject) touch() {
	s.db.journal.append(touchChange{
		account: &s.address,
	})
	if s.address == ripemd {
		// Explicitly put it in the dirty-cache, which is otherwise generated from
		// flattened journals.
		s.db.journal.dirty(s.address)
	}
}
```

这个函数往journal中添加一个touchChange日志，用于递增journal.dirties中的计数器；如果s.address是ripemd合约地址时，那就再递增一次。

#### Balance修改

```go
func (s *stateObject) AddBalance(amount *big.Int) {
	// EIP161: We must check emptiness for the objects such that the account
	// clearing (0,0,0 objects) can take effect.
	if amount.Sign() == 0 {
		if s.empty() {
			s.touch()
		}
		return
	}
	s.SetBalance(new(big.Int).Add(s.Balance(), amount))
}

func (s *stateObject) SubBalance(amount *big.Int) 
func (s *stateObject) SetBalance(amount *big.Int)
func (s *stateObject) setBalance(amount *big.Int) {
	s.data.Balance = amount
}
```

这些函数用于修改账户的余额，如果修改的值和原来的不一样，那么会往journal中写入balanceChange日志，并且设置s.data.Balance。如果一样，并且这个账户是empty，那么调用touch函数往journal中添加touchChange，递增journal.dirties中的计数器。

> 注：在eip-161(最早是eip-158提出的，后被161取代)之前空的账户存在trie树中，在合约层面这和不存在trie树中的账户其实是一回事，所以其实是没必要存储的。因此eip-161规定这种空账户不存trie。为了把之前存的空账户删除掉，在转账额度为0的时候也会写一个journal日志，相当于渐进式的删除。

#### SetState函数

```go
func (s *stateObject) SetState(db Database, key, value common.Hash) 
```

这个函数更新账户存储，为了避免写入的值和旧值相同，先读取比较，相同就退出，否则设置进s.dirtyStorage，并往journal添加storageChange日志。



#### finalise函数

```go
func (s *stateObject) finalise(prefetch bool) 
```

这个函数在交易执行结束的时候调用，将所有s.dirtyStorage的数据移到s.pendingStorage中，并重置s.dirtyStorage；如果prefetch为true，则将dirtyStorage中修改的key的集合发送给s.db.prefetcher进行预取。

#### getTrie函数

```go
func (s *stateObject) getTrie(db Database) Trie 
```

这个函数用于获取这个账户的存储trie，注意函数接收一个其实不需要的db参数，它和s.db.db是同一个，这是程序多次重构遗留的下来的。函数流程：

1. 如果s.trie非空，则直接返回；
2. 如果s.data.Root非空，并且s.db.prefetcher非空，那么尝试通过prefetcher拿；
3. 如果s.trie还为空，那么尝试从调用db.OpenStorageTrie生成；
4. 如果出现db错误，那么设置s.dbErr，并且创建一个空的trie给s.trie，让程序继续盲跑下去；



#### updateTrie函数

```go
func (s *stateObject) updateTrie(db Database) Trie 
```

为了降低修改trie的成本，对账户存储的修改先是累积在pendingStorage和dirtyStorage中的，等到真正要落账的时候调用这个函数将所有的修改应用到账户的存储trie中。函数流程：

1. 先调用s.finalize将dirtyStorage刷入pendingStorage；
2. 如果pendingStorage是空的，直接返回；
3. 调用s.getTrie拿到之前的trie，遍历pendingStorage中所有和originStorage的值不同的key，应用到trie上；
4. 将所有的pendingStorage移到originStorage中(从这里可以看出originStorage和trie中的数据保持一致)；
5. 如果s.db.snap不为nil，遍历pendingStorage中所有和originStorage的值不同的key，设置进s.db.snapStorage；
6. 重置pendingStorage；



#### updateRoot函数

```go
func (s *stateObject) updateRoot(db Database) 
```

这个函数先调用s.updateTrie将更改应用到trie中，再调用trie.Hash拿到新的root，设置进s.data.Root中。

#### CommitTrie函数

```go
func (s *stateObject) CommitTrie(db Database) (int, error) 
```

这个函数先调用s.updateTrie将更改应用到trie中，然后调用trie.Commit将数据写入到内存database中，并更新s.data.Root。函数返回提交到database中的key的数量。

#### GetCommittedState函数

```go
func (s *stateObject) GetCommittedState(db Database, key common.Hash) common.Hash 
```

这个函数用于当前的交易执行前的状态，流程：

1. 如果s.fakeStorage非空的话，说明当前是在debug，直接返回这里的数据；
2. 检查s.pendingStorage是否有数据，有直接返回；
3. 检查s.originStorage是否有数据，有直接返回；
4. 如果s.db.snap非空的话，先通过s.db.snapDestructs检查当前账户是否在当前区块的之前交易中销毁过：
   * 如果存在，说明之后又重新创建出来了，这个重新建出来的账户的数据肯定全部存在了s.pendingStorage中，因此执行到这一步就可以直接返回空；
   * 不存在那么就可以安全地尝试通过s.db.snap.Storage查找；
5. 如果还没找到，那么就通过trie读取；
6. 将数据缓存进s.originStorage中；



#### GetState函数

```go
func (s *stateObject) GetState(db Database, key common.Hash) common.Hash
```

这个函数返回当前的状态数据，流程：

1. 先检查s.fakeStorage是否非空，是直接返回；
2. 检查s.dirtyStorage是否存，有直接返回；
3. 调用s.GetCommitedState返回；



### StateDB分析



```go
type StateDB struct {
   db           Database
   prefetcher   *triePrefetcher
   originalRoot common.Hash // The pre-state root, before any changes were made
   trie         Trie
   hasher       crypto.KeccakState

   snaps         *snapshot.Tree
   snap          snapshot.Snapshot
   // 当交易创建新账户时，写入老账户；当交易修改从dirty刷入pending的时候写入，
   // 不能像下面那样在区块层面写入，因为同一个区块中可以有多笔交易前者销毁，后者创建，
   // 尽管最后这个账户留下来了，但之前的存储得销毁掉。
   snapDestructs map[common.Hash]struct{}  
   snapAccounts  map[common.Hash][]byte    // 当账户数据写入账户trie的时候会一起写入这里
   snapStorage   map[common.Hash]map[common.Hash][]byte // 当账户存储从pending刷入trie的时候，会一起写入这里

   stateObjects        map[common.Address]*stateObject
   stateObjectsPending map[common.Address]struct{} // 跟踪所有修改并finalized但还没提交到trie的账户，
   stateObjectsDirty   map[common.Address]struct{} // 跟踪所有修改过并finalized但还没提交到底层DB的账户

   dbErr error

   refund uint64  // 记录了当前交易的累积refund

   thash   common.Hash
   txIndex int
   logs    map[common.Hash][]*types.Log   // 保存了每个交易生成的log
   logSize uint
   preimages map[common.Hash][]byte  // 保存了hash和对应的原像
   accessList *accessList

   journal        *journal
   validRevisions []revision
   nextRevisionId int
}
type revision struct {
	id           int
	journalIndex int
}
```



#### AccessList操作

```go
func (s *StateDB) AddAddressToAccessList(addr common.Address) 
func (s *StateDB) AddSlotToAccessList(addr common.Address, slot common.Hash) 
func (s *StateDB) AddressInAccessList(addr common.Address) bool 
func (s *StateDB) SlotInAccessList(addr common.Address, slot common.Hash) (addressPresent bool, slotPresent bool) 
```

PrepareAccessList函数：用于交易执行前的初始设置，它将sender，dst和precomiples合约地址，以及交易中的access list添加到s.accessList中；

AddAddressToAccessList函数将地址添加进s.accessList，成功添加则往journal中添加accessListAddAccountChange日志；

AddSlotToAccessList函数将slot添加进s.accessList，成功添加则往journal中添加accessListAddSlotChange日志； 



#### New函数

```go
func New(root common.Hash, db Database, snaps *snapshot.Tree) (*StateDB, error) 
```

这个函数创建StateDB，初始化相关字段，如果snaps非空，则调用snaps.Snapshot获取当前账户根所在的snapshot，赋值给StateDB.snap字段。

#### getDeletedStateObject函数

```go
func (s *StateDB) getDeletedStateObject(addr common.Address) *stateObject 
```

这个函数的名字容易让人误解，它不是获取删除的账户，而是获取账户，包括被删除的账户，被删除的账户的deleted字段会为true。返回删除的账户会在revert的时候用到。函数流程：

1. 尝试从s.stateObjects中查找，有就返回；
2. 如果s.snap非空，则尝试调用s.snap.Account获取，如果没出错且为空，直接返回nil；
3. 否则从trie中获取，为nil，直接返回nil；
4. 根据account数据构造出stateObject，然后保存到s.stateObjects中，并返回；

#### getStateObject函数

```go
func (s *StateDB) getStateObject(addr common.Address) *stateObject {
   if obj := s.getDeletedStateObject(addr); obj != nil && !obj.deleted {
      return obj
   }
   return nil
}
```

这个函数返回给定地址的账户，如果已经删除掉了则返回nil。

#### Exist和Empty函数

```go
func (s *StateDB) Exist(addr common.Address) bool {
   return s.getStateObject(addr) != nil
}

func (s *StateDB) Empty(addr common.Address) bool {
   so := s.getStateObject(addr)
   return so == nil || so.empty()
}
```

Exist函数返回某个地址的账户是否存储了；Empty函数返回某个地址存储了并且非空。



#### GetOrNewStateObject函数

```go
func (s *StateDB) GetOrNewStateObject(addr common.Address) *stateObject
```

这个函数先尝试调用s.getStateObject获取，如果没有就新建一个，并设置进s.stateObjects中，往journal添加createObjectChange日志并返回。

#### CreateAccount函数

```go
func (s *StateDB) CreateAccount(addr common.Address) 
```

这个函数用于新建账户，在evm create的时候调用，如果老的账户存在那么会替换掉，但balance会保留。同时相应地设置s.stateObjects并添加journal日志。



#### Suicide函数

```go
func (s *StateDB) Suicide(addr common.Address) bool 
```

这个函数用于合约销毁，其标记对应的stateObject.suicided。并且将账户balance设置为0，并添加对应journal。



#### Snapshot和RevertToSnapshot函数

```go
func (s *StateDB) Snapshot() int
func (s *StateDB) RevertToSnapshot(revid int) 
```

由于所有的修改操作都会记录journal，因此做快照和回滚比较简单，Snapshot函数记录当前journal日志的长度，并关联一个id返回，后面进行修改操作journal日志会跟着增长，当想撤销时，可以调用RevertToSnapshot函数，这样日志会按顺序回滚，直到到达snapshot时的日志长度。



#### Finalise函数

```go
func (s *StateDB) Finalise(deleteEmptyObjects bool) 
```

这个函数将所有修改过的账户存储进行提交叫pendingStorage里，清理掉销毁的和空的账户，journal日志和refund，因此在交易执行结束时调用。不过账户存储本身还不更新。函数流程：

1. 遍历s.journal.dirties，它中包含了当前所有修改过的账户地址addr，从s.stateObjects中取出obj；
2. 如果obj.suicided或者obj.empty()，那么标记obj.deleted；如果s.snap非空，那么添加进s.snapDestructs，并删除掉s.snapAcctouts和s.snapStorage的对应记录；
3. 否则调用obj.finalise，把修改的存储移到pendingStorage中；
4. 将账户地址添加到s.stateObjectsPending和s.stateObjectsDirty中，前者表示当前区块所有修改过的账户，后者表示当前交易修改的账户。
5. 清理掉journal和refund；
6. 如果prefetcher打开了的话，预取所有修改的账户；



#### IntermediateRoot函数

```go
func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash
```

以太早期版本每个交易执行完后都需要计算状态根，这个函数用于这个目的。流程：

1. 先调用s.Finalise函数，将账户存储进行提交到pendingStorage；
2. 遍历s.stateObjectsPending找到所有修改过的账户obj，调用obj.updateRoot，提交账户存储trie，并拿到最新的storageRoot；
3. 遍历所有修改的账户，提交到s.trie，重置s.stateObjectsPending；从这可以看出stateObjectsPending保存了所有还没将修改提交到账户trie中的账户地址；
4. 返回更新后的账户trie的根；



#### Commit函数

```go
func (s *StateDB) Commit(deleteEmptyObjects bool) (common.Hash, error) 
```

这个函数将所有的状态修改写入后端存储。其中账户的code会写入底层db，存储则会放进trie.Database缓存。主要流程：

1. 先调用s.IntermediateRoot将数据提交到trie；
2. 将所有修改code的账户写到s.db.TrieDB().DiskDB()；
3. 将所有非删除的修改账户obj，调用obj.CommitTrie，将trie的节点数据写入后端缓存；
4. 重置s.stateObjectsDirty；
5. 调用s.trie.Commit提交账户trie到后端缓存，并通过LeafCallback回调函数参数调用s.db.TrieDB().Reference(account.Root, parent)，将trie的账户叶子节点和账户存储根节点进行相互引用;
6. 如果s.snap非nil并且状态根不一样(跳过没有区块奖励的空块)，那么调用s.snaps.Update(root, parent, s.snapDestructs, s.snapAccounts, s.snapStorage)增加一层layer，更新snaps，然后调用s.snaps.Cap(root, 128)将多余的层写入后端；
7. 重置s.snap, s.snapDestructs, s.snapAccounts, s.snapStorage，从snap被重置来看，StateDB在Commit过后不适合再进行修改并Commit；
8. 返回最新的账户root；

