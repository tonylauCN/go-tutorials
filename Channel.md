# Channel
Channel 是 Go 语言中被用来实现并行计算方程之间通信的类型。其功能是允许线程内/间通过发送和接收来传输指定类型的数据。

> 线程内：runtime.GOMAXPROCS(1) 
> 线程间：runtime.GOMAXPROCS(2+) 发送/接收端在同一个M，其他M任务偷取

### 前言：并发问题的挑战

***并发问题一般有下面这几种： ***

* 数据竞争。简单来说就是两个或多个线程同时读写某个变量，造成了预料之外的结果。

* 原子性。在一个定义好的上下文里，原子性操作不可分割。上下文的定义非常重要。有些代码，你在程序里看起来是原子的，如最简单的 i++，但在机器层面看来，这条语句通常需要几条指令来完成（Load，Incr，Store），不是不可分割的，也就不是原子性的。原子性可以让我们放心地构造并发安全的程序。

* 内存访问同步。代码中需要控制同时只有一个线程访问的区域称为临界区。Go 语言中一般使用 sync 包里的 Mutex 来完成同步访问控制。锁一般会带来比较大的性能开销，因此一般要考虑加锁的区域是否会频繁进入、锁的粒度如何控制等问题。
 
* 死锁。在一个死锁的程序里，每个线程都在等待其他线程，形成了一个首尾相连的尴尬局面，程序无法继续运行下去。

* 活锁。想象一下，你走在一条小路上，一个人迎面走来。你往左边走，想避开他；他做了相反的事情，他往右边走，结果两个都过不了。之后，两个人又都想从原来自己相反的方向走，还是同样的结果。这就是活锁，看起来都像在工作，但工作进度就是无法前进。
 
* 饥饿。并发的线程不能获取它所需要的资源以进行下一步的工作。通常是有一个非常贪婪的线程，长时间占据资源不释放，导致其他线程无法获得资源。

大多数的编程语言的并发编程模型是基于线程和内存同步访问控制，Go 的并发编程的模型则用 goroutine 和 channel 来替代。Goroutine 和线程类似，channel 和 mutex (用于内存同步访问控制)类似。
Channel解决 数据竞争和内存访问同步。

### Channel介绍

#### CSP
CSP 全称是 “Communicating Sequential Processes”, 中文可以叫做通信顺序进程，是 Go 在并发编程上成功的关键因素。
它是一种并发编程模型，由 Tony Hoare 于 1977 年提出。简单来说，CSP 模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。CSP 模型的关键是关注 channel，而不关注发送消息的实体。
Go 语言实现了 CSP 部分理论，goroutine 对应 CSP 中并发执行的实体，channel 也就对应着 CSP 中的 channel。

> Do not communicate by sharing memory; instead, share memory by communicating.<br>
> 不要通过共享内存来通信，而要通过通信来实现内存共享。

#### Channel
Goroutine 和 channel 是 Go 语言并发编程的两大基石。Goroutine用于执行并发任务，channel用于goroutine 之间的同步、通信。
Channel 在gouroutine间架起了一条管道，在管道里传输数据，实现 gouroutine 间的通信；由于它是线程安全的，所以用起来非常方便；channel还提供FIFO“先进先出”的特性；它还能影响 goroutine 的阻塞和唤醒。


#### Channel优点
Go 通过 channel 实现 CSP 通信模型，主要用于 goroutine 之间的消息传递和事件通知。

有了 channel 和 goroutine 之后，Go 的并发编程变得异常容易和安全，得以让程序员把注意力留到业务上去，实现开发效率的提升。
(用同步代码解决异步问题)

#### Channel声明
Channel 分为两种：带缓冲、不带缓冲。对不带缓冲的 channel 进行的操作实际上可以看作“同步模式”，带缓冲的则称为“异步模式”。
同步模式下，发送方和接收方要同步就绪，只有在两者都 ready 的情况下，数据才能在两者间传输（后面会看到，实际上就是内存拷贝）。否则，任意一方先行进行发送或接收操作，都会被挂起，等待另一方的出现才能被唤醒。
异步模式下，在缓冲槽可用的情况下（有剩余容量），发送和接收操作都可以顺利进行。否则，操作的一方（如写入）同样会被挂起，直到出现相反操作（如接收）才会被唤醒。
单向通道的声明，用 <- 来表示，它指明通道的方向。
因为channel 是一个引用类型，所以在它被初始化之前，它的值是 nil，channel 使用 make 函数进行初始化。
可以向它传递一个 int 值，代表 channel 缓冲区的大小（容量），构造出来的是一个缓冲型的 channel；不传或传 0 的，构造的就是一个非缓冲型的 channel。

```golang
chan T 		// 声明一个双向通道
chan<- T 	// 声明一个只能用于发送的通道
<-chan T 	// 声明一个只能用于接收的通道
```

带缓冲区与不带缓冲区channl创建
```golang
# 带缓冲channel创建
make(chan T, 6)

# 不带缓冲channel创建
make(chan T)
```

Channel 分为两种：带缓冲、不带缓冲。对不带缓冲的 channel 进行的操作实际上可以看作“同步模式”，带缓冲的则称为“异步模式”。

同步模式下，发送方和接收方要同步就绪，只有在两者都 ready 的情况下，数据才能在两者间传输（后面会看到，实际上就是内存拷贝）。否则，任意一方先行进行发送或接收操作，都会被挂起，等待另一方的出现才能被唤醒。

异步模式下，在缓冲槽可用的情况下（有剩余容量），发送和接收操作都可以顺利进行。否则，操作的一方（如写入）同样会被挂起，直到出现相反操作（如接收）才会被唤醒。
(中断问题,sysmon)

#### Channel使用
```golang
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg = &sync.WaitGroup{}
	wg.Add(2)

	ch := make(chan int)
	go goroutineA(ch, wg)
	go goroutineB(ch, wg)
	ch <- 3
	ch <- 4
	wg.Wait()
}

func goroutineA(a <-chan int, wg *sync.WaitGroup) {
	val := <-a
	fmt.Println("G1 received data: ", val)
	wg.Done()
}

func goroutineB(b <-chan int, wg *sync.WaitGroup) {
	val := <-b
	fmt.Println("G2 received data: ", val)
	wg.Done()
}
```

#### Channel本质
channel 的发送和接收操作本质上都是 “值的拷贝”，无论是从 sender goroutine 的栈到 chan buf，还是从 chan buf 到 receiver goroutine，或者是直接从 sender goroutine 到 receiver goroutine。


lock or channel？
<img src="https://user-images.githubusercontent.com/10111580/112921524-824c4080-913d-11eb-87a0-6e9ce730417e.png" width="480">

### Channel原理

***LV1***
```golang
type hchan struct {
	// chan 里元素数量
	qcount   uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf      unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed   uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx    uint   // send index
	// 已接收元素在循环数组中的索引
	recvx    uint   // receive index
	// 等待接收的 goroutine 队列
	recvq    waitq  // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq    waitq  // list of send waiters

	// 保护 hchan 中所有字段
	lock mutex
}
type waitq struct {
	first *sudog
	last  *sudog
}
```

<img src="https://user-images.githubusercontent.com/10111580/112921593-9f810f00-913d-11eb-83c4-239e1e2f3bdb.png" width="580">

<img src="https://user-images.githubusercontent.com/10111580/112922412-09e67f00-913f-11eb-9693-d678afea7c19.png" width="580">

***LV2***
带缓冲Channel的缓冲区处理
| make(chan T, 3)| ch <- T | ch <- T *2 | <- ch |
| --- | --- | --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112947197-17b0fa00-9169-11eb-9f6b-d70239077a7c.png)| ![image](https://user-images.githubusercontent.com/10111580/112947749-c6edd100-9169-11eb-9ff7-dde171d998fd.png) | ![image](https://user-images.githubusercontent.com/10111580/112947808-d8cf7400-9169-11eb-8006-53676c3b3dc2.png)|![image](https://user-images.githubusercontent.com/10111580/112947854-e553cc80-9169-11eb-858b-de92b6c2a84b.png)|

内存地址补充

发送和接收带缓冲数据

| G1 | G2 |
| --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112950069-77f56b00-916c-11eb-8817-634afc047985.png)| ![image](https://user-images.githubusercontent.com/10111580/112950094-7d52b580-916c-11eb-8179-33862cfb8ddb.png) |

##### Sender ： G1
| tashCH <- task0 | 1. acquire | 2. enqueue task0(copy) | 3. release |
| --- | --- | --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112950391-d15d9a00-916c-11eb-8a4e-f7c459664bb9.png) | ![image](https://user-images.githubusercontent.com/10111580/112950435-dae70200-916c-11eb-89a4-326f1744eb4c.png) | ![image](https://user-images.githubusercontent.com/10111580/112950466-e6d2c400-916c-11eb-8517-1200da4f25fc.png) | ![image](https://user-images.githubusercontent.com/10111580/112950514-f520e000-916c-11eb-9afe-71260dd23ce0.png)|

##### Recevier ： G2
| t := <-ch task0（copy) | 1. acquire | 2. dequeue | 3. release  |
| --- | --- | --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112950931-69f41a00-916d-11eb-88a2-6e0ea87c5ea9.png) | ![image](https://user-images.githubusercontent.com/10111580/112950961-711b2800-916d-11eb-8a69-176527d49fc9.png) | ![image](https://user-images.githubusercontent.com/10111580/112951092-9445d780-916d-11eb-9726-17380283c977.png) | ![image](https://user-images.githubusercontent.com/10111580/112951140-9f990300-916d-11eb-9a90-1f5cb0e31e54.png) |


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
