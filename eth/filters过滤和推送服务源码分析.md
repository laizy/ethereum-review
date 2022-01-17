# filters过滤和推送服务

以太坊的智能合约事件都包含推出这个事件的合约address，以及一个topic列表(长度不超过4)，RPC提供了接口供外部用户查询某个区块区间的事件，并可以通过指定地址和topic来过滤出感兴趣的事件，这些事件按照区块递增的顺序推送给用户。

> 注：阅读这个模块前需要理解core/bloombits。

## 支持的过滤条件

支持的过滤条件由下面的结构体指定：

```go
type FilterQuery struct {
   BlockHash *common.Hash     // 设置了表示单区块查询
   FromBlock *big.Int         // 区间的起始高度(包含)，nil表示创世块
   ToBlock   *big.Int         // 区间的终止高度(包含)，nil表示最新区块
   Addresses []common.Address // 只匹配指定合约吐出的事件
   // Examples:
   // {} or nil          matches any topic list
   // {{A}}              matches topic A in first position
   // {{}, {B}}          matches any topic in first position AND B in second position
   // {{A}, {B}}         matches topic A in first position AND B in second position
   // {{A, B}, {C, D}}   matches topic (A OR B) in first position AND (C OR D) in second position
   Topics [][]common.Hash
}                       
```

规则如下：

1. 支持单个区块的匹配，由BlockHash指定；或者指定FromBlock，ToBlock指定过滤的高度闭区间，前者为nil表示从创世块开始，后者为nil表示到最新区块；
2. Addresses指定过滤出指定的合约地址的消息；
3. Topics是一个二维数组，第一维表示事件topic的位置，第二维是对应这个位置的可选匹配项，过滤条件是：每个位置都要有一个匹配。当某个位置上的可选项是空的，表示这个位置匹配。

从描述上看可以把Addresses列表当做一个特殊位置的topic匹配，因此可以将两者统一。

 

## Filter

这个模块依赖的后端接口：

```go
type Backend interface {
	ChainDb() ethdb.Database
	HeaderByNumber(ctx context.Context, blockNr rpc.BlockNumber) (*types.Header, error)
	HeaderByHash(ctx context.Context, blockHash common.Hash) (*types.Header, error)
	GetReceipts(ctx context.Context, blockHash common.Hash) (types.Receipts, error)
	GetLogs(ctx context.Context, blockHash common.Hash) ([][]*types.Log, error)

	SubscribeNewTxsEvent(chan<- core.NewTxsEvent) event.Subscription
	SubscribeChainEvent(ch chan<- core.ChainEvent) event.Subscription
	SubscribeRemovedLogsEvent(ch chan<- core.RemovedLogsEvent) event.Subscription
	SubscribeLogsEvent(ch chan<- []*types.Log) event.Subscription
	SubscribePendingLogsEvent(ch chan<- []*types.Log) event.Subscription

    BloomStatus() (uint64, uint64) // sectionSize(4096), totalSection
	ServiceFilter(ctx context.Context, session *bloombits.MatcherSession)
}
```

由于bloombits索引是将section个区块放一起创建的，区块不够就做不了，BloomStatus函数返回两个数，一个是section的大小，一个是当前建立的section数量。ServiceFilter通过MatcherSession提供bloombits数据。

Filter定义如下：

```go
type Filter struct {
	backend Backend
	addresses []common.Address
	topics    [][]common.Hash

	block      common.Hash // 按单个block查询
	begin, end int64       // 按block区间查询
	matcher *bloombits.Matcher
}
// 构造函数，分别构造区间查询和单区块查询
func NewRangeFilter(backend Backend, begin, end int64, addresses []common.Address, topics [][]common.Hash) *Filter 
func NewBlockFilter(backend Backend, block common.Hash, addresses []common.Address, topics [][]common.Hash) *Filter 
```



### Logs函数

```go
func (f *Filter) Logs(ctx context.Context) ([]*types.Log, error) 
```

这个函数从源代码的注释来看是搜索区块，找到第一个满足过滤条件的区块，并更新过滤的起始位置。但实际实现不是这样，是遍历整个查询区块区间！而且这个函数是暴露给了web3接口，没有任何限制，因此如果用户传进来的区间起止位置是创世块和最新区块的话，会遍历整条链，能把节点搞崩。函数流程：

1. 如果是但区块查询，加载这个区块的区块头，如果匹配header.Bloom，那就加载区块logs，遍历进行精确匹配，并返回；
2. 否则就是区间查询，如果区间范围没指定，就用查当前最新区块高度，进行赋值；
3. 调用backend.BloomStatus，查询当前可用的section数；
4. 先对可以使用bloombits的区间前面部分进行查询：通过backend.ServiceFilter提供bloombits获取服务；
5. 接着对区间后半部分挨个遍历区块的方式获取剩下的部分；
6. 将两部分查询到的logs返回；



## 订阅和推送

除了查询过滤历史区块的数据外，也可以通过订阅的方式，实时拿到最新生成的数据的推送。支持订阅的方式有如下几种：

```go
type Type byte  // 可以订阅的类型
const (
	LogsSubscription Type = iota  // 最新生成和因链reorg导致移除的log
	PendingLogsSubscription  // pending区块里的log
	MinedAndPendingLogsSubscription // pending区块和新挖出来区块的log
	PendingTransactionsSubscription // pending区块中的交易hash
	BlocksSubscription // 存入账本的区块hash
)
```



### 推送的对外API

```go
// 创建订阅系统，内部通过backend监听所有系统发出的事件  
func NewEventSystem(backend Backend, lightMode bool) *EventSystem
// 订阅交易log，满足条件的log推送到参数logs中，并返回Subscription对象。可用来取消订阅。    
func (es *EventSystem) SubscribeLogs(crit ethereum.FilterQuery, logs chan []*types.Log) (*Subscription, error)
// 订阅最新的区块头信息    
func (es *EventSystem) SubscribeNewHeads(headers chan *types.Header) *Subscription
// 订阅pending交易hash    
func (es *EventSystem) SubscribePendingTxs(hashes chan []common.Hash) *Subscription
// 返回推送过程中的错误
func (sub *Subscription) Err() <-chan error
// 取消订阅
func (sub *Subscription) Unsubscribe()
```



### 具体实现

整个订阅系统采用event loop的结构实现，loop线程循环监听新订阅事件和backend发出的链消息，并分发给各个订阅者。

单个订阅信息由subscription进行维护，定义如下：

```go
type subscription struct {
	id        rpc.ID  //   订阅的id
	typ       Type   // 订阅类型
	created   time.Time // 订阅创建时间
	logsCrit  ethereum.FilterQuery // 过滤条件
	logs      chan []*types.Log    // 当订阅类型是log的时候，保存订阅时用户传进来的chan
	hashes    chan []common.Hash   // 当订阅类型是pending交易时，保存用户传入的chan
	headers   chan *types.Header   // 当订阅类型是新区块hash时，保存用户传入的chan
	installed chan struct{} // 当这个订阅由loop线程设置好后关闭
	err       chan error    // 当这个订阅由loop线程取消后关闭
}
```



EventSystem的定义如下：

```go
type EventSystem struct {
	backend   Backend
	lightMode bool             // 是否为light client
	lastHead  *types.Header    // 记录当前的区块头，只用于light client
    
	// Channels
	install       chan *subscription         // install filter for event notification
	uninstall     chan *subscription         // remove filter for event notification
    
	// 下面这些都是从backend订阅的chan
	txsSub         event.Subscription // Subscription for new transaction event
	logsSub        event.Subscription // Subscription for new log event
	rmLogsSub      event.Subscription // Subscription for removed log event
	pendingLogsSub event.Subscription // Subscription for pending log event
	chainSub       event.Subscription // Subscription for new chain event
	txsCh         chan core.NewTxsEvent      // Channel to receive new transactions event
	logsCh        chan []*types.Log          // Channel to receive new log event
	pendingLogsCh chan []*types.Log          // Channel to receive new log event
	rmLogsCh      chan core.RemovedLogsEvent // Channel to receive removed log event
	chainCh       chan core.ChainEvent       // Channel to receive new chain event
}
```

这个结构看起来复杂，但实际非常简单，里面定义的一堆chan是用于从backend接收相应的信息，install和uninstall用于给loop线程发送订阅和取消消息。注意订阅信息subscription的记录并没有保存在这个结构中，因为它是loop线程私有的，作为了线程局部变量。

这里有个lightMode变量，表示当前节点是否是轻客户端，从后面的实现来看，应该是这类节点backend不会在链reorg的时候推送remove log。因此如果这个结构体里记录的lastHead和最新拿到的出现reorg的时候，会从账本里取出历史log进行通知。

#### NewEventSystem函数

```go
func NewEventSystem(backend Backend, lightMode bool) *EventSystem
```

这个函数构造EventSystem，然后通过backend.SubscribeNewTxsEvent等注册订阅系统消息，之后在背后启动loop线程后返回。

#### eventLoop函数

```go
index map[Type]map[rpc.ID]*subscription // routine局部变量
func (es *EventSystem) eventLoop()
```

这个函数当做loop线程启动，包含一个index线程局部变量，用于保存订阅信息。启动之后，循环监听事件：

1. 监听到es.install事件后subscription，根据类型放置进对应index中，当类型为MinedAndPendingLogsSubscription时，同时放置到LogsSubscription和PendingLogsSubscription中，同时关闭subscription.installed；
2. 监听到es.uninstall事件后，从index中移除对应订阅信息，并且关闭subscription.err；
3. 监听到backend推上来的各类事件后，从index中遍历对应类型的订阅，将过滤后的数据通过的subscription的对应chan发送给用户，阻塞直到发送成功；
4. 如果lightMode为真，当收到backend的新区块头信息，并且出现了reorg的时候会遍历账本手动发送remove log。

注意：subscription中有一个created字段，订阅的时候设置，但是并没有使用到，本以为会当做超时自动取消订阅使用；另外将数据发送给用户提供的chan时，是阻塞直到发送为止，因此只要有一个chan阻塞，整个推送系统都会卡死。



#### 订阅函数

```go
func (es *EventSystem) SubscribeNewHeads(headers chan *types.Header) *Subscription 
func (es *EventSystem) SubscribePendingTxs(hashes chan []common.Hash) *Subscription 
func (es *EventSystem) SubscribeLogs(crit ethereum.FilterQuery, logs chan []*types.Log) (*Subscription, error) 
```

订阅过程都是构造对应的subscription，然后通过es.install发送给loop线程，并等待subscription.installed。其中SubscribeLogs有个过滤参数的FromBlock和ToBlock需要重点讲一下：

1. 如果fromBlock > toBlock，那么直接报错返回；
2. 如果两个数字都大于0，则能收到在这个区间里所有新落账的区块log；
3. 因为只推送新生成的实时log，不返回历史log，因此如果toBlock比当前的区块高度还小，那么全部会被过滤掉，啥也不会推送；
4. 如果两个都没设置，那么默认都为LatestBlockNumber，能收到所有新落账的区块log；
5. 如果两个都为PendingBlockNumber，则能收到pending log的推送；
6. 如果toBlock为PendingBlockNumber，fromBlock不是，则同时推送落账的和pending的log；
7. 如果toBlock为LatestBlockNumber，from大于等于0，则推送所有新落账的区块log；



#### Subscription实现

Subscription的实现如下，注意到前面提到的，只要一个订阅者没有及时将推送到数据取出来就会造成卡死整个系统，因此当这个订阅者取消订阅时，需要使用循环select一起尝试发送取消订阅和读取数据。

```go
type Subscription struct {
	ID        rpc.ID
	f         *subscription
	es        *EventSystem
	unsubOnce sync.Once
}
func (sub *Subscription) Err() <-chan error {
	return sub.f.err
}
func (sub *Subscription) Unsubscribe() {
	sub.unsubOnce.Do(func() {
	uninstallLoop:
		for {
			select {
			case sub.es.uninstall <- sub.f:
				break uninstallLoop
			case <-sub.f.logs:
			case <-sub.f.hashes:
			case <-sub.f.headers:
			} 
		}

		// wait for filter to be uninstalled in work loop before returning
		// this ensures that the manager won't use the event channel which
		// will probably be closed by the client asap after this method returns.
        // 这个地方的注释没看明白。
		<-sub.Err()
	})
}
```



## web3接口

内部管理结构定义：

```go
type filter struct {
	typ      Type
	deadline *time.Timer  // 超时时间，必须隔三差五通过GetFilterChanges调用来刷新。
	hashes   []common.Hash
	crit     FilterCriteria
	logs     []*types.Log
	s        *Subscription // EventSystem中关联的订阅
}

type PublicFilterAPI struct {
   backend   Backend
   mux       *event.TypeMux
   quit      chan struct{}
   events    *EventSystem
   filtersMu sync.Mutex
   filters   map[rpc.ID]*filter
   timeout   time.Duration
}
```



#### NewPublicFilterAPI函数

```go
func NewPublicFilterAPI(backend Backend, lightMode bool, timeout time.Duration) *PublicFilterAPI
```

函数构造PublicFilterAPI实例后，后台启动timeoutLoop线程，从filters中移除掉超时timeout(以太设置为5min)的。

#### 过滤器创建和数据获取

```go
// 创建过滤器，用户获取所有的pending交易hash
func (api *PublicFilterAPI) NewPendingTransactionFilter() rpc.ID
// 创建过滤器，用户获取导入链的区块hash
func (api *PublicFilterAPI) NewBlockFilter() rpc.ID 
// 创建过滤器，用于获取交易执行相关log。过滤条件由参数具体设置。
func (api *PublicFilterAPI) NewFilter(crit FilterCriteria) (rpc.ID, error) 
// 查询之前创建的过滤器累积的数据，并刷新超时时间。
func (api *PublicFilterAPI) GetFilterChanges(id rpc.ID) (interface{}, error) 
// 注销过滤器
func (api *PublicFilterAPI) UninstallFilter(id rpc.ID) bool 
```

这系列函数用于创建过滤器，通过poll的形式从节点拉取数据。内部实现都是调用api.events的对应订阅方法进行订阅，数据由函数内部定义的临时chan变量接收。并将过滤信息记录在api.filters中，同时开启一个routine，循环从临时chan中读取数据写入filter.hashes或者filter.logs中。



#### 实时推送

```go
func (api *PublicFilterAPI) NewHeads(ctx context.Context) (*rpc.Subscription, error) 
func (api *PublicFilterAPI) Logs(ctx context.Context, crit FilterCriteria) (*rpc.Subscription, error) 
func (api *PublicFilterAPI) NewPendingTransactions(ctx context.Context) (*rpc.Subscription, error) 
```

这个函数可以用于创建订阅，和上面的过滤器不同的是实时推送，因此可用于websocket。内部实现都是调用api.events的对应订阅方法进行订阅，数据由函数内部定义的临时chan变量接收。订阅**不会记录**在api的结构体中，同时开启一个routine，循环从临时chan中读取数据直接写入rpc框架提供的Notifier中推送给用户，因此这系列函数的超时等错误不在api内部管理，而是依赖rpc框架去维护。



#### 历史Log获取

```go
func (api *PublicFilterAPI) GetLogs(ctx context.Context, crit FilterCriteria) ([]*types.Log, error) 
```

这个函数用于根据给定的条件通过Filter结构获取过滤账本历史log。注意到前面提到FIlter本身没有施加任何限制，因此查询区块区间比较大的情况下，可能会搞挂节点。

```go
func (api *PublicFilterAPI) GetFilterLogs(ctx context.Context, id rpc.ID) ([]*types.Log, error)
```

这个函数和上面函数功能一样，只是过滤条件是通过上面的NewFilter创建的。整体上感觉这些api设计得比较混乱。
