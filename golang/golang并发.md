# golang

- [golang](#golang)
  - [概念](#概念)
    - [临界区](#临界区)
    - [数据竞争](#数据竞争)
  - [并发原语](#并发原语)
    - [Mutex](#mutex)
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


## 并发原语

### Mutex

`Mutex`是golang标准库`sync`提供的排他锁实现。实现了`sync.Locker`接口，接口定义如下:

```
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

### RWMutex

### WaitGroup

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