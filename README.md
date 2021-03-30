## GO语言实践分享

Go 是一个开源的编程语言，它能让构造简单、可靠且高效的软件变得容易。

Go是从2007年末由Robert Griesemer, Rob Pike, Ken Thompson主持开发，后来还加入了Ian Lance Taylor, Russ Cox等人，并最终于2009年11月开源，在2012年早些时候发布了Go 1稳定版本。现在Go的开发已经是完全开放的，并且拥有一个活跃的社区。

目前Go版本为Go1.16.2 @2021/3/30

![image](https://user-images.githubusercontent.com/10111580/112920625-ecfc7c80-913b-11eb-837d-a5c797fa82a6.png)

### Goroutine 原理
<img src="https://user-images.githubusercontent.com/10111580/112920889-6ac08800-913c-11eb-932c-f56a5eb8ade1.png" width="880" height="260">


### 内存分配原理
<img src="https://user-images.githubusercontent.com/10111580/112920933-7d3ac180-913c-11eb-8f57-da12daac67bf.png" width="880" height="260">


### GC 原理
<img src="https://user-images.githubusercontent.com/10111580/112920705-10bfc280-913c-11eb-93b1-eeea6eb0b726.png" width="880" height="260">


### Channel 原理
<img src="https://user-images.githubusercontent.com/10111580/112921205-f20dfb80-913c-11eb-8d48-5150a7ed6095.png" width="880" height="260">


#### 并行和并发

并行
***并发：***
并发问题一般有下面这几种：

* 数据竞争。简单来说就是两个或多个线程同时读写某个变量，造成了预料之外的结果。

* 原子性。在一个定义好的上下文里，原子性操作不可分割。上下文的定义非常重要。有些代码，你在程序里看起来是原子的，如最简单的 i++，但在机器层面看来，这条语句通常需要几条指令来完成（Load，Incr，Store），不是不可分割的，也就不是原子性的。原子性可以让我们放心地构造并发安全的程序。

* 内存访问同步。代码中需要控制同时只有一个线程访问的区域称为临界区。Go 语言中一般使用 sync 包里的 Mutex 来完成同步访问控制。锁一般会带来比较大的性能开销，因此一般要考虑加锁的区域是否会频繁进入、锁的粒度如何控制等问题。
 
* 死锁。在一个死锁的程序里，每个线程都在等待其他线程，形成了一个首尾相连的尴尬局面，程序无法继续运行下去。

* 活锁。想象一下，你走在一条小路上，一个人迎面走来。你往左边走，想避开他；他做了相反的事情，他往右边走，结果两个都过不了。之后，两个人又都想从原来自己相反的方向走，还是同样的结果。这就是活锁，看起来都像在工作，但工作进度就是无法前进。
 
* 饥饿。并发的线程不能获取它所需要的资源以进行下一步的工作。通常是有一个非常贪婪的线程，长时间占据资源不释放，导致其他线程无法获得资源。

#### CSP
CSP 全称是 “Communicating Sequential Processes”, 是 Go 在并发编程上成功的关键因素。

#### Channel
&nbsp;&nbsp;&nbsp;&nbsp;Goroutine 和 channel 是 Go 语言并发编程的 两大基石。Goroutine 用于执行并发任务，channel 用于 goroutine 之间的同步、通信。
&nbsp;&nbsp;&nbsp;&nbsp;Channel 在 gouroutine 间架起了一条管道，在管道里传输数据，实现 gouroutine 间的通信；由于它是线程安全的，所以用起来非常方便；channel 还提供“先进先出”的特性；它还能影响 goroutine 的阻塞和唤醒。

```golang
chan T // 声明一个双向通道
chan<- T // 声明一个只能用于发送的通道
<-chan T // 声明一个只能用于接收的通道
```

带缓冲区与不带缓冲区channl创建
```golang


补充
```
##### Channel本质
channel 的发送和接收操作本质上都是 “值的拷贝”，无论是从 sender goroutine 的栈到 chan buf，还是从 chan buf 到 receiver goroutine，或者是直接从 sender goroutine 到 receiver goroutine。

##### Channel优点
Go 通过 channel 实现 CSP 通信模型，主要用于 goroutine 之间的消息传递和事件通知。

有了 channel 和 goroutine 之后，Go 的并发编程变得异常容易和安全，得以让程序员把注意力留到业务上去，实现开发效率的提升。


lock or channel？
<img src="https://user-images.githubusercontent.com/10111580/112921524-824c4080-913d-11eb-87a0-6e9ce730417e.png" width="480">

##### 原理

***LV1***
<img src="https://user-images.githubusercontent.com/10111580/112921593-9f810f00-913d-11eb-83c4-239e1e2f3bdb.png" width="480">


***LV2***
<img src="https://user-images.githubusercontent.com/10111580/112922412-09e67f00-913f-11eb-9693-d678afea7c19.png" width="480">


***LV3***
```golang
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

type waitq struct {
	first *sudog
	last  *sudog
}
```


##### 资源泄漏

Channel 可能会引发 goroutine 泄漏。

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，不见天日。

##### 问题
关于 channel 的使用，有几点不方便的地方：
* 在不改变 channel 自身状态的情况下，无法获知一个 channel 是否关闭。
* 关闭一个 closed channel 会导致 panic。所以，如果关闭 channel 的一方在不知道 channel 是否处于关闭状态时就去贸然关闭 channel 是很危险的事情。
* 向一个 closed channel 发送数据会导致 panic。所以，如果向 channel 发送数据的一方不知道 channel 是否处于关闭状态时就去贸然向 channel 发送数据是很危险的事情。

原则

* don’t close a channel from the receiver side and don’t close a channel if the channel has multiple concurrent senders.
* don’t close (or send values to) closed channels.

FIX：

* 使用 defer-recover 机制，放心大胆地关闭 channel 或者向 channel 发送数据。即使发生了 panic，有 defer-recover 在兜底。
* 使用 sync.Once 来保证只关闭一次。


### 实践案例分享
