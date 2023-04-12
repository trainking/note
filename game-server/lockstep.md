# 帧同步服务器

## 0x01 概述

### 同步是什么？

所谓**同步** 是指多人游戏中，多个客户端表现应该一致，因此产生了，必须要同步这些客户端的数据。

### 帧是什么？

所谓**帧** 是指将数据打包，以一定的步长发送的一种通信手段。

### 帧同步是什么？

帧同步直白的可以理解为，用帧通信技术，做游戏中的同步，可以看作是帧同步。而发送的数据可以是：**指令**和**状态。**

- 指令：游戏中网络对象（可以是玩家，也可以是预设的怪物等等）的操作

- 状态：游戏中网络对象当前操作结果，例如具体位置，技能作用等等

因此，广义来说，帧同步可以有三种实现方式：

- 指令-指令：将指令发送到服务端，服务端再同步到各个客户端

- 指令-状态：将指令发送到服务端，服务端同步指令计算出来的状态

- 状态-状态：客户端直接跑出状态，发送到服务端同步其他客户端

狭义的表述中，帧同步是指第一种**指令-指令**的同步，其他两种会说是利用了帧通信的状态同步。

## 0x02 锁步模式

帧同步服务器，一般被称之为LockStepServer。即，该服务器是一个应用锁步模式的服务器。

所谓**所部模式**，是指将所有玩家的在一帧中的操作，封装到一个同步帧（确认帧）中，然后广播给所有玩家。古典帧同步中，是必须要等待所有玩家的数据到达，才能广播。这在广域网中的网络游戏，显然是不合适的。因此衍生出了，乐观模式帧同步。

### 乐观模式

帧同步的乐观模式，可以概括为**定时不等待。**服务器以定时步长，向客户端发送确认帧。帧中的数据，以收到的玩家操作为准，如果玩家的网络慢了，导致帧发送慢了，则丢弃。

实现表现为，服务端维持一个当前帧号，客户端发送的数据，带上帧号，大于等于帧号则保留，小于帧号则丢弃

## 0x03 技术实现

一个帧同步服务器设计，需要实现如下内容：

- 帧号

- 定时发送

- 同步帧结构

- 帧缓存

### 帧号

帧号是标识每帧数据的唯一标识，它是一个连续的Int值。保证帧号的有序，是实现帧同步服务器基础。

因为帧同步的开始有一定的条件，例如多个玩家都连上了服务器，才开始同步。因此，以第一帧的校准为游戏的开始，第一帧需要传输游戏开局的初始化内容。例如：**随机种子**。

### 定时发送

服务器以固定帧率（步长）向所有客户端广播同步帧，设计上一般设计为66毫秒一帧。即一秒种，同步30帧左右。

这个可以根据不同游戏设置，因为网络传输存在延迟，所以对同步帧的发送，需要考虑RTT的动态修改发送时间优化。

### 同步帧结构

同步帧的结构，应该包含：

- 帧号：标识这是那一帧的数据

- 指令数组：所有网络对象（玩家）的操作指令数组

- 发送时间：标识服务端发送的时间点，毫秒

### 帧缓存

服务器需要维护帧的缓存，帧缓存应该包含两个部分：

- 未确认帧缓存：缓存未发送帧的数据，这里面的帧数据，需要大于最后同步帧号

- 确认帧缓存：缓存已发送数据

#### 未确认帧缓存

未确认帧缓存，应该设计成一个**用id做键的map**。因为，此中数据，随着玩家的指令的发送，会动态的改变。当到达帧的同步节点，此帧从未确认帧中移动到确认帧缓存中，并修改最后同步帧号。

#### 确认帧缓存

确认帧缓存，应该设计成一个**双向的队列**形式，可以从头尾顺序读取帧数据。

## 0x04 难点问题

### 断线重连

因为服务器并不保存玩家的状态，因此当玩家断线重连时。服务器需要将玩家第一帧到最后同步帧的所有帧一起发给客户端，客户端加速跑出最新状态。

### 丢帧严重

因为客户端网络问题，客户端出现落后于服务器的情况。需要直接向服务端请求丢失帧，服务端将客户端发送的帧号之后的所有帧一起打包发送给客户端。

客户端加快追帧，恢复最新状态。

### 浮点数难题

帧同步因为服务器只做转发，因此可能出现各个客户端进行浮点数运算，得出不一样的结果，然后导致偏差越来越大。

解决方案一般是客户端采用定点数的确定性算法计算，或者将所有浮点数计算交给服务端计算，再同步相应的结果。

### 校验问题

帧同步不跑客户端逻辑，所以存在可能被串改的可能性。难以避免外挂存在，改进方式可以通过游戏客户端，定时上报状态，然后通过服务器校验状态值是否满足预期。

或者保存整局游戏帧，然后延后校验帧数据，判断是否有作弊情况。

### 数据存储问题

帧同步一局游戏累计下来的帧数据，是非常庞大的。完全保存再内存中是不合适的，因此需要将帧异步落地到数据库中。

选择什么样的数据库，也是一个很大的考验。

## 0x05 总结

游戏开发中，往往没有放之四海而皆准的方案。因为帧同步场景常用于多人战斗场景，所以对于帧的设计尤其重要，是使用状态帧，还是指令帧，还是状态-指令混用。都要看具体的业务来考量。

例如：

- 当游戏玩法并不太复杂，一局人数有限，那么可以使用指令-状态帧的同步方式，这样服务器的计算负载是很需要关注

- 当游戏参与人数多，玩法复杂（如吃鸡），使用指令-指令，或者状态-状态方式同步，计算压力在客户端。这时候，服务端需要有一定数据校验机制

## 0x05 参考实现

- [https://github.com/JiepengTan/Lockstep-Tutorial](https://github.com/JiepengTan/Lockstep-Tutorial)

- [https://github.com/byebyebruce/lockstepserver](https://github.com/byebyebruce/lockstepserver)

- [https://gitee.com/NKG_admin/NKGMobaBasedOnET](https://gitee.com/NKG_admin/NKGMobaBasedOnET)




