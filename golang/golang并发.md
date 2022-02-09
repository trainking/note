# golang

- [golang](#golang)
  - [概念](#概念)
    - [临界区](#临界区)
    - [数据竞争](#数据竞争)
    - [Fork-join模型](#fork-join模型)
  - [并发原语](#并发原语)
    - [Mutex](#mutex)
      - [Mutex实现](#mutex实现)
    - [RWMutex](#rwmutex)
    - [WaitGroup](#waitgroup)
    - [Cond](#cond)
    - [Once](#once)
    - [Pool](#pool)
      - [示例](#示例)
      - [小心的坑](#小心的坑)
      - [第三方实现](#第三方实现)
    - [Context](#context)
      - [Context使用规则](#context使用规则)
  - [拓展原语](#拓展原语)
    - [Semaphore](#semaphore)
      - [Golang拓展库实现](#golang拓展库实现)
      - [常见错误](#常见错误)
    - [SingleFlight](#singleflight)
    - [CyclicBarrier](#cyclicbarrier)
    - [ErrorGroup](#errorgroup)
  - [分布式实现](#分布式实现)
    - [Leader选举](#leader选举)
    - [分布式锁](#分布式锁)
    - [队列](#队列)
    - [栅栏](#栅栏)
    - [STM](#stm)

## 概念

### 临界区

**临界区**是指在并发访问中，程序中某一部分会被并发访问或者修改，为了防止并发访问导致意想不到的结果，这这部分就是**临界区**。锁的存在就是为了保护临界区，**排他锁（互斥锁）**限定临界区同时只能有一个协程持有，**共享锁（读写锁）** 则是读可以多个携程共享，而写时只能一个协程加锁

### 数据竞争

多协程访问同一数据块，就会发生数据竞争 (data race)。`golang` 提供了`-race`选项检查代码中的数据竞争。

### Fork-join模型

`Golang`的协程，遵循`Fork-join`模型。一个程序必定有一个主携程，携程的创建都是从要给协程中`Fork`出来的，而根据闭包的原则，子携程可以通过共享内存知道父协程的变量，而父协程对字协程不可知。只能通过添加`join`点的方式，通信告知字协程的状态。`join`点的实现方式，就必须使用到`chan`。

这就是所谓**不要通过共享内存来通信，而是通过通信来共享内存**。

## 并发原语

### Mutex

`Mutex`是golang标准库`sync`提供的排他锁实现。实现了`sync.Locker`接口，接口定义如下:

```go
type Locker interface {
	Lock()    // 加锁
	Unlock()  // 释放
}
```

其具有以下特性：

* Mutex是排他锁，即意味着锁未释放时，试图加锁的协程都会被阻塞
* Mutex是**不可重入锁**，即持有此锁的携程，再次调用`Lock`会触发`panic`
* Mutex必须调用`Lock/Unlock`必须成对出现，且必须`Lock`之后才能调用`Unlock`，不然`panic`， **谁申请，谁释放**
* Mutex的零值即是一个`unlocked`状态的锁

#### Mutex实现

`Mutex`结构体实现:

```
type Mutex struct {
	state int32  // 标识锁状态
	sema  uint32  // 信号量，唤醒携程
}
```

1. `Lock/Unlock`是对`state`进行CAS操作标记
2. 锁有`正常状态`和`饥饿状态`, 饥饿状态下，锁优先给排队中的队头，正常状态下优先给新建申请者

### RWMutex

`RWMutex`结构体实现：

```go
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
```

`RWMutex`实现的`sync.Locker`接口之外，还增加了两个`RLock/RUnlock`方法。用于操作读锁(共享锁)，读锁机制是通过`atomic`包的原子操作，操作标记量来实现。

而写锁（排他锁）的实现，本质就是通过`Mutex`来实现。

因此`RWMutex`有以下特性:

* `RLock/RUnlock`与`Lock/Unlock`成对使用，不可交互，即：读锁只能读释放，写锁只能写释放
* RWMutex只被加读锁时，可以被多个协程获取到读锁
* RWMutex被加写锁时，只允许一个协程获取到锁，其他协程读锁和写锁都加不上
* RWMutex写锁加锁时，会先调用`w.Lock`，防止其他携程进入加写锁，然后将`readerCount`减最大值置为负数，让读锁加不上；最后等所有已获得读锁释放，完成加锁

### WaitGroup

`WaitGroup`是一个用于任务编排，解决**并发-等待**问题。在协程中添加一个`join`点，就会阻塞等待所有协程完成。

标准库中，提供了这三个方法：

```go

    func (wg *WaitGroup) Add(delta int)  // 添加一个暂停点
    func (wg *WaitGroup) Done()  // 结束标记
    func (wg *WaitGroup) Wait()  // 等待结束
```

而`WaitGroup`的结构，则是:

```go
type WaitGroup struct {
	noCopy noCopy

	// 64bit(8bytes)的值分成两段，高32bit是计数值，低32bit是waiter的计数
  // 另外32bit是用作信号量的
  // 因为64bit值的原子操作需要64bit对齐，但是32bit编译器不支持，所以数组中的元素在不同的架构中不一样，具体处理看下面的方法
  // 总之，会找到对齐的那64bit作为state，其余的32bit做信号量
	state1 [3]uint32
}
```

通过`state1`这个3个`uint32`元素来组成。对其方式是利用地址求于来计算:

```go
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```

注意事项：

* 需要注意，避免计数器的值变为负，会触发`panic`
* `Add()`增加计数，`Done()`减少计数，尽量保持这种原则
* 必须等所有`Add()`调用之后，再调用`Wait()`，不然会出现`panic`或不期望结果
* 在`Wait()`未结束之前，`WaitGroup`不可被重用

### Cond

`Cond`是一个条件变量，用于等待或这宣布事件发生时协程的交汇点。简而言之，是协程通信的一种手段。与`channel`不同的是，他可以通过`Signal`来唤醒一个协程，使用`Broadcast`唤醒所有的协程。

```go
var c = sync.NewCond(&sync.Mutex{})

	c.L.Lock()
	for conditionTrue() == false {
		c.Wait()
	}
	c.L.Unlock()

  ...
  c.Signal()

  c.Broadcast()
```

需要注意的是：

1. 对于触发`Wait`的条件或者事件的读取，需要使用`cond`初始化时的`Mutex`保证线程安全
2. 进去`Wait`后，会将`c.L`先释放，然后等到收到信号退出`wait`时，又把`c.L`加上锁
3. `Signal`是唤醒等待最久那一个的协程

`Conde`使用较少，因为他的功能都可以使用`Channel`来代替。唯一优势点，是使用`Broadcast`唤醒所有协程，但`channel`也可以通过`close`的特性来实现。

### Once

`Once`的语义是只此一次。可用作对变量的一次初始化，常用在实现单例模式中：

```go

var once sync.Once

once.Do(funcation(){

})
```

需要注意的是，`Once`的执行以来赋值的空间，所以意味着，我们常常使用声明成全局变量来使用它。

### Pool

`sync.Pool`保存一组可独立访问的**临时**对象。它池化的对象在未来的某个时候会后无征兆地移除掉，被移除的对象，没有引用时，会被回收。

`sync.Pool`有两个特性:

1. `sync.Pool`本身是线程安全的
2. `sync.Pool`不可在使用之后再复制使用

#### 示例

```go

var buffers = sync.Pool{
  New: func() interface{} { 
    return new(bytes.Buffer)
  },
}

func GetBuffer() *bytes.Buffer {
  return buffers.Get().(*bytes.Buffer)
}

func PutBuffer(buf *bytes.Buffer) {
  buf.Reset()
  buffers.Put(buf)
}
```

#### 小心的坑

1. 内存泄漏，使用`sync.Pool`回收buffer的时候，**一定要检查回收对象的大小**。如果buffer太大，就不用回收了，避免内存泄露
2. 内存浪费，定义的缓存池太大，而只是用部分时，则会造成浪费，定义多种级别内存池避免浪费。[bucketpool](https://github.com/vitessio/vitess/blob/main/go/bucketpool/bucketpool.go)
3. 因为`sync.Pool`会毫无征兆回收对象，所以不适合用来池化长连接操作

#### 第三方实现

- [TCP池](https://github.com/trainking/pool)
- [Work Pool](https://github.com/valyala/fasthttp/blob/9f11af296864153ee45341d3f2fe0f5178fd6210/workerpool.go#L16)

### Context

`context.Context`是`golang`标准库提供的信息穿透上下文。其接口定义了四个方法：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

- **Deadline** 返回这个Context被取消的截至时间，无设置ok为false，多次调用返回相同的值
- **Done** 放回一个Channel，当Context被取消时，此Channel会被close
- **Err** 返回被Close的原因
- **Value** 返回指定key关联的值

`context.Background()` 和 `context.TODO()` 会返回一个非nil，空的Context。二者本质上是相同的，只是不同别名。在使用上还是需要遵循一定规则：

- 所有要派生新的Context时，使用`context.Background()`
- 不知道用什么Context时，用`context.TODO()`，后续再改动

#### Context使用规则

1. 使用Context时，会把函数放在参数的第一个位置
2. 不用使用nil作为Context函数的值
3. Context只是作为函数间上下文传递，不要持久化和长久保存，避免内存泄露
4. 使用WithValue时，key的类型不应该是字符串类型或者其他内建类型，最好使用自定义类型

## 拓展原语

### Semaphore

**Semaphore**（信号量）是一种资源控制的并发模式。它定义了两种操作，P操作,V操作。

- P操作（descrease, wait, acquire）是减少信号量计数值
- v操作（increase, signal, release）是增加信号量的计数值

**两种操作都是原子的**，信号量的值除了初始化以外，只能由P/V操作改变。

#### Golang拓展库实现

Golang拓展库的实现叫[Weighted](https://pkg.go.dev/golang.org/x/sync/semaphore):

* `Acquire`：P操作，一次获取n个资源，如果没有足够的资源，调用者阻塞。可以通过第一个参数的context设置取消机制
* `Release`: V操作，将n个资源释放
* `TryAcquire`: 尝试获取n个资源，不会阻塞

#### 常见错误

1. 请求了资源未释放
2. 释放了未请求的资源
3. 长时间请求，即使未使用
4. 不请求信号量，直接使用资源
5. 释放的资源比请求的资源大，程序会panic
6. 如果请求的资源比最大容量大，程序会永远阻塞
7. 多个信号量控制资源时，应注意[哲学家就餐问题](https://zh.wikipedia.org/wiki/%E5%93%B2%E5%AD%A6%E5%AE%B6%E5%B0%B1%E9%A4%90%E9%97%AE%E9%A2%98)

### SingleFlight

`SingleFlight` 是golang拓展包提供的请求合并包，提供了控制多个请求同时发生的情况下，只请求一次，可以很好的控制类似缓存击穿的情况。

```go
// key 表示请求，返回函数执行结果和，shared是否将v 分享给了多个调用者
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool)

// 与Do类似, 返回的是返回的的是chan
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result

// 忘记对该键的请求合并
func (g *Group) Forget(key string)
```

### CyclicBarrier

`CyclicBarrier`是循环栅栏的实现，常常用于进行一组`goroutine`同时执行的场景中。它可以被重复使用，所以被称之为循环栅栏。具体机制就是所有携程都在栅栏前等候，全部到齐就抬起栅栏放行。

```go

type CyclicBarrier interface {
    // 等待所有的参与者到达，如果被ctx.Done()中断，会返回ErrBrokenBarrier
    Await(ctx context.Context) error

    // 重置循环栅栏到初始化状态。如果当前有等待者，那么它们会返回ErrBrokenBarrier
    Reset()

    // 返回当前等待者的数量
    GetNumberWaiting() int

    // 参与者的数量
    GetParties() int

    // 循环栅栏是否处于中断状态
    IsBroken() bool
}
```

`CyclicBarrier`看起来类似`WaitGroup`，但是使用比`WaitGroup`更加简单，且提供了观测函数，可以设定条件中断执行。而且其最大的用处**可以选择在携程内部中断**，等待其他携程的到达。

### ErrorGroup

`ErrGroup`是官方提供的，将大任务拆散多个小任务执行库。它和`WaitGroup`类似，但使用更加简单，且会返回一个`error`，将子任务的错误传递给调用者。

```go
// 创建一个errgroup，接收一个context,同时返回一个待cancel的context; 作为后续调用控制
func WithContext(ctx context.Context) (*Group, context.Context)

// 执行函数，传入一个函数作为执行体，该函数要返回error
func (g *Group) Go(f func() error)

// 等待所有任务完成，如果err有值则是执行错误
func (g *Group) Wait() error
```

有两个缺点:

1. 不能返回所有子任务的错误
2. 发生error，并不会中断其他子任务的执行

`1`可以通过一个`err chan`来收集，`2`则通过拓展go函数来实现：

-[b站/errgroup](https://github.com/go-kratos/kratos/blob/v1.0.x/pkg/sync/errgroup/errgroup.go)

## 分布式实现

### Leader选举

`Leader选举`常用于主从架构中，将服务节点分为主节点和从节点两种角色。在同一时刻，系统中不能有两个主节点，否则会有数据不一致的情况。一般情况我们使用`主`节点执行写操作，使用`从`节点执行读操作（多读少写系统结构）。如果读写都集中在主节点，则**主从结构退化为主备结构**。

常用的`Leader`选举算法`raft`，是etcd的核心，具体实现:

```
https://github.com/etcd-io/etcd/tree/main/raft
```

### 分布式锁

### 队列

### 栅栏

### STM