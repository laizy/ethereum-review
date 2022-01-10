# miner挖矿源码分析

本文分析miner目录下的源码：miner.go，worker.go和unconfirmed.go



## worker分析

出块的主要功能是在worker.go中的worker结构体实现的。由四个routine和一堆chan一起配合工作。

```go
	go worker.mainLoop()
	go worker.newWorkLoop(recommit)
	go worker.resultLoop()
	go worker.taskLoop()
```



### taskLoop分析

这个routine负责最终的出块，和共识模块的Seal方法打交道。

```go
	// Seal generates a new sealing request for the given input block and pushes
	// the result into the given channel.
	//
	// Note, the method returns immediately and will send the result async. More
	// than one result may also be returned depending on the consensus algorithm.
	Seal(chain ChainHeaderReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error
```

共识的Seal方法是异步的，函数接收要打包的block,当seal工作做完后通过参数results将结果传回，stop参数用于中断共识当前的打包任务。

taskLoop维护两个变量，prev记录上一个发给共识打包任务的区块hash；stopCh用于中断共识的打包。这个routine循环从taskCh里读取任务，检查这个任务是否和prev的一样，一样就直接跳过。否则先中断共识的打包，然后调用Seal方法重新打包。并将任务注册在worker.pendingTasks中。共识完成会将区块放进worker.resultCh中。

pendingTasks中存放的task的定义：

```go
type task struct {
	receipts  []*types.Receipt
	state     *state.StateDB
	block     *types.Block
	createdAt time.Time
}
```

以太是采用先执行后出块的方式，在共识前就已经有了执行结果。因此task包含了当前要打包的区块信息和执行的结果。这样等共识出块好后，可以从pendingTask取出执行好的结果直接落账，而不用重新执行。

```go
func (w *worker) taskLoop() {
	var (
		stopCh chan struct{}
		prev   common.Hash
	)

	// interrupt aborts the in-flight sealing task.
	interrupt := func() {
		if stopCh != nil {
			close(stopCh)
			stopCh = nil
		}
	}
	for {
		select {
		case task := <-w.taskCh:
			if w.newTaskHook != nil {
				w.newTaskHook(task)
			}
			// Reject duplicate sealing work due to resubmitting.
			sealHash := w.engine.SealHash(task.block.Header())
			if sealHash == prev {
				continue
			}
			// Interrupt previous sealing operation
			interrupt()
			stopCh, prev = make(chan struct{}), sealHash

			if w.skipSealHook != nil && w.skipSealHook(task) {
				continue
			}
			w.pendingMu.Lock()
			w.pendingTasks[sealHash] = task
			w.pendingMu.Unlock()

			if err := w.engine.Seal(w.chain, task.block, w.resultCh, stopCh); err != nil {
				log.Warn("Block sealing failed", "err", err)
			}
		case <-w.exitCh:
			interrupt()
			return
		}
	}
}
```



### resultLoop

这个routine循环从worker.resultCh中读取共识出的区块。检查这个区块不在账本中，因为有resubmit(后面会讲到)，共识有可能出重复的区块)，然后从pengdingTask中取出这个区块并调用worker.chain.WriteBlockWithState写入账本。然后调用worker.mux.Post(core.NewMinedBlockEvent{Block: block})广播区块，并调用worker.unconfirmed.Insert(block.NumberU64(), block.Hash())将区块注册到unconfirmed列表中。

值得注意的是任务并没有从pendingTask中移除，移除的操作放在单独的函数批量处理。从代码上看pendingTask也只有这个routine会读取应该是可以直接删除的。



### mainLoop

这个routine负责执行交易，构造task发送给taskLoop处理。执行的上下文环境environment保存在worker.current中，定义如下：

```go
type environment struct {
	signer types.Signer // 当前区块的签名类型

	state     *state.StateDB // apply state changes here
	ancestors mapset.Set     // ancestor set (used for checking uncle parent validity)
	family    mapset.Set     // family set (used for checking uncle invalidity)
	uncles    mapset.Set     // uncle set
	tcount    int            // 总打包的交易数
	gasPool   *core.GasPool  // 打包交易剩余的区块gas量

	header   *types.Header       
	txs      []*types.Transaction  
	receipts []*types.Receipt
}
```



#### task任务的构造

task的构造工作主要由commitNewWork完成：

##### commitNewWork函数

1. 调用worker.chain.CurrentBlock拿到当前区块作为parent;
2. 构造出Header,填充部分信息：如ParentHash，Number,GasLimit，Time，Coinbase等；
3. 调用共识worker.engine.Prepare给Header填充共识相关字段；
4. 调用worker.makeCurrent设置worker.current，创建执行的上下文；
5. 计算当前区块的uncle(uncle的相关操作比较杂，也只有POW用得到，后面相关的代码不分析直接忽略)；
6. 调用worker.eth.TxPool().Pending()从交易池拉取所有的pending交易列表；
7. 判断pending交易列表是否为空，并判断是否能出空快；
8. 调用worker.commitTransactions执行交易；
9. 执行完后调用worker.commit提交任务；

##### makeCurrent函数

1. 调用worker.chain.StateAt(parent.Root())拿到上一个区块执行后的StateDB
2. 开启TriePrefetcher预先加载TrieNode；
3. 构造environment，并替换worker.current；
4. 关掉老的TriePrefetcher预加载。



##### commitTransactions函数

1. 依次遍历所有交易执行；
2. 如果超过了block的gasLimit，那么退出循环；
3. 调用worker.commitTransaction尝试执行，并检查系统错误；
4. 循环同时会检查是否触发了中断。中断信号有两个：commitInterruptNewHead有新区块头或者挖矿开启的时候触发(挖矿可以随时开启和关闭)；commitInterruptResubmit定期触发，要求重新构造区块，这样可以将交易池中新的交易包含进来。当收到前者时当前执行好的交易状态会被丢弃，当收到后者时，可以在当前执行好状态基础上继续执行。
5. 为避免频繁resubmit，当收到commitInterruptResubmit信号，会计算当前执行的gasLimit和区块总limit的比值，并构造提高resubmit间隔的消息，发送给worker.resubmitAdjustCh。
6. 循环执行结束后，如果当前的挖矿是关闭的，那么将执行交易Log通过worker.pendingLogsFeed推送出去。之所以挖矿开启时不推送是因为the worker will regenerate a mining block every 3 seconds，导致推送重复的pending log。
7. 给worker.resubmitAdjustCh发送一个减少resubmit间隔的消息，对方收到消息后，如果当前的间隔大于用户指定的，会进行递减。



##### commitTransaction函数

这个函数只有commitTransactions函数调用。他的功能是尝试执行交易，失败时进行回滚（这里的失败时系统层面的比如nonce过低等，不包括执行合约时的revert等业务层面错误）。成功的话将交易和receipt添加进worker.current。

```go
func (w *worker) commitTransaction(tx *types.Transaction, coinbase common.Address) ([]*types.Log, error) {
	snap := w.current.state.Snapshot()

	receipt, err := core.ApplyTransaction(w.chainConfig, w.chain, &coinbase, w.current.gasPool, w.current.state, w.current.header, tx, &w.current.header.GasUsed, *w.chain.GetVMConfig())
	if err != nil {
		w.current.state.RevertToSnapshot(snap)
		return nil, err
	}
	w.current.txs = append(w.current.txs, tx)
	w.current.receipts = append(w.current.receipts, receipt)

	return receipt.Logs, nil
}
```



##### commit函数

1. 调用worker.engine.FinalizeAndAssemble进行最后的状态修改并构造最终的区块；
2. 如果挖矿开启了，那么给worker.taskCh发送任务消息；
3. 更新worker.snapshotBlock和snapshotState，这两个字段主要是给外部接口查询当前的pending交易等信息用的。



#### mainLoop的实现

这个routine循环从若干chan中读取消息：

1. worker.newWorkCh：收到这个消息立马调用commitNewWork构造任务；
2. worker.chainSideCh: 收到侧链区块消息，则检查当前挖矿的区块uncles是否太少，太少就调用worker.commit重新构造任务；
3. worker.txsCh: 更新worker.newTxs计数器(记录seal之后交易池新增加的交易数，resubmit的时候会检查这个字段，如果没有交易的话就不触发)，在非挖矿状态下，执行收到的交易，并更新snapshot供api查询；



### workLoop

这个routine主要用于给worker.newWorkCh发送挖矿请求，负责安排resubmit和清理过期的pendingTask（小于当前区块前七个高度的）。routine循环从若干chan读取消息：

1. worker.startCh: 挖矿开启的时候会发送这个消息，收到后清理过期pendingTask，给之前的挖矿任务发送commitInterruptNewHead信号，构造新的挖矿请求发送给worker.neWorkCh，重置worker.newTxs计数器；
2. worker.chainHeadCh: 收到了账本有新的区块头消息，处理同上；
3. timer.C:这是routine维护的局部变量，用于resubmit的定时器。当定时器触发时，如果worker.newTxs不为零且挖矿开启了，则给之前的挖矿任务发送commitInterruptResubmit信号，并构造新的挖矿请求发送给worker.neWorkCh，重置worker.newTxs计数器；
4. worker.resubmitIntervalCh：这个是用户发送的，按照收到的值设置最小的resubmit间隔；
5. worker.resubmitAdjustCh：这个是mainLoop发送的，用于调节resubmit的间隔；
6. worker.exitCh：监听退出信号，收到结束routine;



### worker中的消息传递总结

```go
type worker struct {
   // ...
   // Feeds
   pendingLogsFeed event.Feed     // 发送方： mainLoop

   // Subscriptions
   mux          *event.TypeMux         // 发送方： resultLoop，消息是NewMinedBlockEvent
   txsCh        chan core.NewTxsEvent  // 发送方：txpool, 处理方：mainLoop
   txsSub       event.Subscription  
   chainHeadCh  chan core.ChainHeadEvent // 发送方：账本， 处理方：workLoop
   chainHeadSub event.Subscription
   chainSideCh  chan core.ChainSideEvent // 发送方：账本， 处理方：mainLoop
   chainSideSub event.Subscription

   // Channels
   newWorkCh          chan *newWorkReq  // 发送方：workLoop，处理方：mainLoop
   taskCh             chan *task  // 发送方：mainLoop， 处理方：taskLoop
   resultCh           chan *types.Block // 发送方：共识，处理方：resultLoop
   startCh            chan struct{}  //  发送方：启动时和外部api， 处理方：workLoop
   exitCh             chan struct{}  // 发送方：外部， 处理方：所有routine
   resubmitIntervalCh chan time.Duration // 发送方：外部api，处理方：workLoop
   resubmitAdjustCh   chan *intervalAdjust// 发送方：mainLoop， 处理方：workLoop
}
```



## Miner分析

miner本身就是worker的包装类，包装了一些方法共外部使用。另外它监听了downloader的同步消息，当在同步中时，关闭挖矿，等同步结束后打开挖矿。



## unconfirmed分析

这个结构用于跟踪挖出的区块是否Confirmed，看下来本身只用于辅助打印日志的作用。





