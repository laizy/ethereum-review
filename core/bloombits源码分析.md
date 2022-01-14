# bloombits源码分析



## 基本原理

以太坊的交易包含了很多的日志记录，每个日志记录包含了合约的地址和多个Topic，交易的收据中存在一个布隆过滤器，记录了所有的日志记录的信息。区块头中也包含了一个logsBloom，是所有交易收据的布隆过滤器的并集。

布隆过滤器共2048bit，定义在core/types/bloom9.go中。计算bloom的算法：

* 对输入进行keccak256得到hash；
* 截取hash值的前6个字节，当做三个大端编码的uint16,得到三个数 a, b, c；
* 对三个数取模截取到[0,2048)区间，得到的三个数字就是对应的布隆过滤器被设置的bit；

```
func bloom9(b []byte) *big.Int {
  b = crypto.Keccak256(b)

  r := new(big.Int)

  for i := 0; i < 6; i += 2 {
    t := big.NewInt(1)
    b := (uint(b[i+1]) + (uint(b[i]) << 8)) & 2047
    r.Or(r, t.Lsh(t, b))
  }

  return r
}
```

从上面的算法可以看出，每个元素只有三个bit会设置，因此本身是非常稀疏的。如果想要查找某个合约的历史日志记录，挨个区块去比对bloom看看匹不匹配那效率就比较低了。bloombits通过将多个Bloom(称为section，以太中设置为4096)构成的bit矩阵进行转置，所以Bloom的个数必须是8的倍数。比如：

```
[A0, A1, ..., A2047]
[B0, B1, ..., B2047]
...
[H0, H1, ..., H2047]
转置后：
[A0, B0, ..., H0]
[A1, B1, ..., H1]
...
[A2047, B2047, ..., H2047]
```

eth下的BloomIndexer负责区块数据后处理，以每4096个区块一个section，生成转置的数据，存储在rawdb中：

key: `bloomBitsPrefix + bit (uint16大端，0到2047) + section号(uint64大端) + hash（section的最后一个区块hash)`.

value: 4096个bit构成的bytes，经过压缩后的bytes。

从key的构造来看，bit放置在section的前面，因此非常方便批量读取某个bit下多个section的数据。

因此如果要检索哪些区块中包含每个topic，只需要读取这个topic对应的三个bit位置的bloomBits数据，就可以快速地进行匹配。甚至如果byte为0的话，可以直接跳过8个区块。



## 转置数据的生成

bloombits的转置数据由Generator生成：

```go
type Generator struct {
   blooms   [types.BloomBitLength][]byte // 转置后的bloom数据
   sections uint                         // 总bloom数
   nextSec  uint                         // 下一个要添加的bloom编号
}
// 初始化，sections设置总bloom数，必须是8的倍数，以太使用4096
func NewGenerator(sections uint) (*Generator, error) 
// 按编号添加bloom，只能顺序调用，每次index递增，否则报错
func (b *Generator) AddBloom(index uint, bloom types.Bloom) error 
// 当所有bloom添加完成后，这个函数可以返回第idx处的转置数据，没添加完会返回error
func (b *Generator) Bitset(idx uint) ([]byte, error) 
```



## bloom获取调度和逻辑匹配的流水线系统

以太坊的智能合约事件都包含推出这个事件的合约address，以及一个topic列表(长度不超过4)，RPC提供了接口供外部用户查询某个区块区间的事件，并可以通过指定地址和topic来过滤出感兴趣的事件，这些事件必须按照区块递增的顺序推送给用户，因此这个模块采用了流水线的结构(说实在的，这代码有点过度工程化)，主要功能是调度bloom bit的读取和进行逻辑匹配过滤操作。另外抽离出了读取bloom bit的方式，以便能够支持网络或者本地db上获取。

## bloom bits的获取调度

scheduler用于对某个bit位的多个section的的bloom bit的获取请求进行调度，解决两个问题：过滤掉重复的获取请求；缓存已经获取到的响应数据。结构定义如下：

```go
type scheduler struct {
   bit       uint                 // 负责section第bit位的bloom获取
   responses map[uint64]*response // 当前正在获取的请求，或者缓存的响应，key是section编号
   lock      sync.Mutex           // 保护responses的并发访问
}
type response struct {
	cached []byte        // 缓存的bloom bit
	done   chan struct{} // 当外部成功获取到数据写入了cached后close
}
type request struct {
	section uint64 // 想要获取的section编号
	bit     uint   // 想要获取这个section的bit位的bloom
}
```

### scheduler的创建

这个比较简单。

```go
func newScheduler(idx uint) *scheduler {
   return &scheduler{
      bit:       idx,
      responses: make(map[uint64]*response),
   }
}
```



### 请求的调度

```go
func (s *scheduler) scheduleRequests(reqs chan uint64, dist chan *request, pend chan uint64, quit chan struct{}, wg *sync.WaitGroup)
```

这个函数当做一个单独的routine执行，循环从reqs中读取要获取的section，判断s.response中是否有记录(有记录说明之前发出过请求，或者已经有缓存结果)，没有记录则构造request发送到dist，通知外部去获取数据。同时把section放进pend中。当这期间quit关闭了，那么调用wg.Done，并关闭掉pend。

### 响应的返回

```go
func (s *scheduler) deliver(sections []uint64, data [][]byte) 
```

当外部线程从dist中拿到请求，查询存储或者网络拿到数据后，通过deliver函数将数据放入scheduler，遍历所有的sections，将数据放入s.responses[section].cached，然后关闭s.responses[section].done。

### 响应的推送

```go
func (s *scheduler) scheduleDeliveries(pend chan uint64, done chan []byte, quit chan struct{}, wg *sync.WaitGroup) 
```

这个函数当做一个独立的routine执行，依次从pend中取出section，侦听s.responses[section].done，然后将s.responses[section].cached发送到done，这个过程中只要quit关闭，就调用wg.Done退出。

> 注：这里是挨个按顺序从pend里取出请求等待，因此有线头阻塞(Head-of-line blocking)的问题：只要前面的一个请求卡住了，后面的请求即便老早就回复了也得跟着等。

### 调度的启动

比较简单，直接看代码。由于pend是函数内的局部变量，这个函数支持并发调用，可以共享响应的缓存和重复请求过滤，非常完美！

```go
func (s *scheduler) run(sections chan uint64, dist chan *request, done chan []byte, quit chan struct{}, wg *sync.WaitGroup) {
   pend := make(chan uint64, cap(dist))

   wg.Add(2)
   go s.scheduleRequests(sections, dist, pend, quit, wg)
   go s.scheduleDeliveries(pend, done, quit, wg)
}
```



整体的流程图：

```go
                   +-----------+         +-----------+                  
                   | scheduler |  pend   | scheduler | 
   reqs chan   --> | routine1  | ------> |  routine2 | --> done chan 
                   +-----------+         +-----------+                  
                          |                 ^
                          | dist chan       | response
                          |                 | 
                       +--v-------------------+  
                       |   outside handler    | 
                       +----------------------+ 
```



## Matcher

整个流水线系统的由Matcher实现。



### 对外API

```go
// 构建一个Matcher实例，指定bloom bits使用的section大小(以太为4096)，过滤条件下面详述
func NewMatcher(sectionSize uint64, filters [][][]byte) *Matcher
// 启动流水线，匹配高度区间[begin，end]，匹配到的高度通过results返回，
// 函数返回MatcherSession，外部通过他拿到bloom bit获取请求并将结果写回。
func (m *Matcher) Start(ctx context.Context, begin, end uint64, results chan<- uint64) (*MatcherSession, error)
// batch是一个hint值，设置希望Retrieval中的section数(以太设置为16)，如果没有那么多的话等wait的时间再通过mux发
func (s *MatcherSession) Multiplex(batch int, wait time.Duration, mux chan <- chan *Retrieval)
// 关闭整个匹配系统
func (s *MatcherSession) Close()
func (s *MatcherSession) Error() error
type Retrieval struct {
	Bit      uint      // 请求的bit位
	Sections []uint64  // 请求的section列表
	Bitsets  [][]byte  // 对应每个section的数据

	Context context.Context // 是Matcher.Start传入的参数
	Error   error           // 外部出错信息可以设置在这里
}
```

其中过滤条件filters是这样约定的：

1. item是[]byte，生成的3个bit位全部匹配这个item才算匹配；
2. filter有多个item，只要有一个item匹配，则这个filter匹配；
3. filters有多个filter，全部filter匹配，filters才算匹配；

API的使用方式：

```go
matcher := NewMatcher(4096, filters)
results := make(chan uint64) // 用于接收匹配到的区块高度
session := matcher.Start(ctx, begin, end, results)

for i:=0; i< 3; i++ {
    mux := make(chan chan *Retrieval, 10)
	go session.Multiplex(10, time.Second, mux); // 注意这里需要开多个routine，否则性能大打折扣，详细分析见后文介绍。
    go func() {
    	for recv := range mux {
        	retrieval <- recv
       		// 去存储或者网络获取bloom bit数据
        	recv <- retrieval // 将结果返回
    	}
	}
}

for matched := range results {
    fmt.Println("height matched", matched)
}
```



### 具体实现

Macher的类型定义如下：

```go
type bloomIndexes [3]uint

type Matcher struct {
	sectionSize uint64 // 每个section的大小

	filters    [][]bloomIndexes    // 这个流水线处理的filters
	schedulers map[uint]*scheduler // 负责每个bit位的schedulers

	retrievers chan chan uint       // 用于将bit位任务传递给取任务的routine
	counters   chan chan uint       // 用于传递某个bit位有多少section的任务给取任务的routine
	retrievals chan chan *Retrieval // 用于传递实际的任务给去任务的routine
	deliveries chan *Retrieval      // 用于任务的routine完成bloombits的获取后返回给matcher

	running uint32 // 指定是否运行的原子变量
}
```

具体的字段的用途下面会分别阐述。注意这里的通道有chan chan的形式，用于routine之间的rpc调用。本来通常的做法是：

```go
type request {
    param Param
    result chan Result
}
rpc chan request
// 请求端：
result := make(chan Result)
rpc <- request{ param, result}
res <- result
// 服务端
req := <- rpc
req.result <- response
```

而他的做法是把请求和响应数据合在一个结构体里，整个过程多了一次chan的发送接收开销，我没看出来有啥优点：

```go
type ReqRes {
    param Param
    result Result
}
rpc chan chan ReqRes
// 请求端
reqresC := make(chan ReqRes)
rpc <- reqresC
reqresC <- ReqRes { param }
result := <- reqresC
// 服务端：
reqresC := <- rpc
reqres <- reqresC
reqresC <- ReqRes {result}
```



#### subMatch函数

```go
func (m *Matcher) subMatch(source chan *partialMatches, dist chan *request, bloom []bloomIndexes, session *MatcherSession) chan *partialMatches
```

subMatch用于过滤一个filter，代码看起来很难受，但看到了流程图就很简单，如下：

```go
/*
 *            section request    section reply
 *               ^   ^              |    |
 *               |   |              |    |
 *            +--+---+---+       +--v----v--+
 *            |          |       |          |
 * source --->| send req +------->  filter  +---> output
 *            |          |       |          |
 *            +----------+       +----------+
*/
```

他内部开两个routine，第一个从source中取出前一个filter的处理结果partialMatches，然后把相关的section通过dist发给m.scheduler，然后把partialMatches传递给第二个routine；第二个从第一个routine读取partialMatches，并等待m.scheduler返回的section数据，拿到数据后完成这个filter的匹配，并和读取的partialMatches进行合并，然后通过chan返回出去。

#### distributor函数

distributor用于从所有的scheduler那里收集request，并按照bit归类，对每个bit中要获取的section进行排序。外部线程通过MatcherSession向distributor拿任务，这个过程又分成好几个阶段(过度工程化的典型)：

1. 外部线程先通过retrievers这个chan拿到有任务要处理的某个bit位，占坑；
2. 外部线程再通过counters这个chan过来问这个bit下有多少section的量要拿；
3. 外部线程拿到数量后，如果数量太少就等一段时间；(这就是MatcherSession中的Multiplex中的wait参数)；
4. 外部线程这下通过retrievals这个chan真正过来拿section任务了，因为第一步占坑位了，因此其他外部线程不会和它抢这个bit的任务了，所以肯定能拿到足够的任务量；
5. 外部线程拿到任务完成后通过deliveries通道发送给Matcher。

```go
func (m *Matcher) distributor(dist chan *request, session *MatcherSession) {
// 源代码里有个类型为chan chan uint的局部变量retrievers在没有bit位的任务分配的时候会设置为nil,
// 这是为了确保当从retrievers里拿到fetcher的时候，一定可以立马分配任务给他，避免卡死
}
```

#### run函数

```go
func (m *Matcher) run(begin, end uint64, buffer int, session *MatcherSession) chan *partialMatches 
```

这个函数流程：

1. 通过调用subMatch构建每一个filter处理流，然后将他们首尾串起来；
2. 开一个routine用于填充全为1的section大小的bits(相当于告诉第一个filter需要过滤所有的section)，作为第一个filter的source输入；
3. 开一个routine调用distributor函数，分发m.scheduler发出来的section请求；
4. 返回最后一个filter的输出chan；



### MatcherSession

结构定义：

```go
type MatcherSession struct {
	matcher *Matcher

// ... 
}
```

主要逻辑就是这个Multiplex函数：

```go
func (s *MatcherSession) Multiplex(batch int, wait time.Duration, mux chan chan *Retrieval)
```

这个函数由独立routine执行，主要流程：

1. 通过s.matcher.retrievers发送rpc从distributor获取任务bit位，阻塞等待直到获取为止；
2. 通过s.matcher.counters发送rpc从distributor获取这个bit位的section数目，阻塞等待直到获取为止；
3. 如果数目比较小就等待wait时间；
4. 通过s.matcher.retrievals发rpc想distributor获取具体的section，阻塞等待直到获取为止；
5. 通过mux给外部发送这个请求，阻塞等结果；
6. 把外部收到的结果通过s.matcher.deliveries发送给distributor；

**注意到这个Multiplex函数内部全阻塞的，每次还只能服务一个bit位的请求**，因此必须单独开多个(至少三个，因为一个item生成三个bit位)routine。

