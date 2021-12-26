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
    - [Context](#context)
  - [拓展原语](#拓展原语)
    - [Semaphore](#semaphore)
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

### Once

### Pool

### Context

## 拓展原语

### Semaphore

### SingleFlight

### CyclicBarrier

### ErrorGroup

## 分布式实现

### Leader选举

### 分布式锁

### 队列

### 栅栏

### STM