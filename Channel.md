* [一、前言](#一、前言：并发问题的挑战)
* [二、Channel介绍](#二、Channel介绍)
   * [2.1 CSP](#2.1 CSP)
   * [2.2 Channel](#2.2 Channel)
   * [2.3 Channel优点](#2.3 Channel优点)
   * [2.4 Channel声明](#2.4 Channel声明)
   * [2.5 Channel使用](#2.5 Channel使用)
   * [2.6 Channel本质](#2.6 Channel本质)
* [Goroutine调度算法](#goroutine调度算法)
   * [work-stealing](#work-stealing)
   * [syscall](#syscall)
   * [spining](#spining)
   * [network poller](#network-poller)
   * [sysmon](#sysmon)
   * [scheduler affinity](#scheduler-affinity)


# Channel
Channel 是 Go 语言中被用来实现并行计算方程之间通信的类型。其功能是允许线程内/间通过发送和接收来传输指定类型的数据。

> 线程内：runtime.GOMAXPROCS(1) 
> 线程间：runtime.GOMAXPROCS(2+) 发送/接收端在同一个M，其他M任务偷取

### 一、前言：并发问题的挑战

<strong>并发问题一般有下面这几种： </strong>

* 数据竞争。简单来说就是两个或多个线程同时读写某个变量，造成了预料之外的结果。

* 原子性。在一个定义好的上下文里，原子性操作不可分割。上下文的定义非常重要。有些代码，你在程序里看起来是原子的，如最简单的 i++，但在机器层面看来，这条语句通常需要几条指令来完成（Load，Incr，Store），不是不可分割的，也就不是原子性的。原子性可以让我们放心地构造并发安全的程序。

* 内存访问同步。代码中需要控制同时只有一个线程访问的区域称为临界区。Go 语言中一般使用 sync 包里的 Mutex 来完成同步访问控制。锁一般会带来比较大的性能开销，因此一般要考虑加锁的区域是否会频繁进入、锁的粒度如何控制等问题。
 
* 死锁。在一个死锁的程序里，每个线程都在等待其他线程，形成了一个首尾相连的尴尬局面，程序无法继续运行下去。

* 活锁。想象一下，你走在一条小路上，一个人迎面走来。你往左边走，想避开他；他做了相反的事情，他往右边走，结果两个都过不了。之后，两个人又都想从原来自己相反的方向走，还是同样的结果。这就是活锁，看起来都像在工作，但工作进度就是无法前进。
 
* 饥饿。并发的线程不能获取它所需要的资源以进行下一步的工作。通常是有一个非常贪婪的线程，长时间占据资源不释放，导致其他线程无法获得资源。

大多数的编程语言的并发编程模型是基于线程和内存同步访问控制，Go 的并发编程的模型则用 goroutine 和 channel 来替代。Goroutine 和线程类似，channel 和 mutex (用于内存同步访问控制)类似。
Channel解决 数据竞争和内存访问同步。

### 二、Channel介绍

#### 2.1 CSP
CSP 全称是 “Communicating Sequential Processes”, 中文可以叫做通信顺序进程，是 Go 在并发编程上成功的关键因素。
它是一种并发编程模型，由 Tony Hoare 于 1977 年提出。简单来说，CSP 模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。CSP 模型的关键是关注 channel，而不关注发送消息的实体。
Go 语言实现了 CSP 部分理论，goroutine 对应 CSP 中并发执行的实体，channel 也就对应着 CSP 中的 channel。

> Do not communicate by sharing memory; instead, share memory by communicating.<br>
> 不要通过共享内存来通信，而要通过通信来实现内存共享。

#### 2.2 Channel
Goroutine 和 channel 是 Go 语言并发编程的两大基石。Goroutine用于执行并发任务，channel用于goroutine 之间的同步、通信。
Channel 在gouroutine间架起了一条管道，在管道里传输数据，实现 gouroutine 间的通信；由于它是线程安全的，所以用起来非常方便；channel还提供FIFO“先进先出”的特性；它还能影响 goroutine 的阻塞和唤醒。


#### 2.3 Channel优点
Go 通过 channel 实现 CSP 通信模型，主要用于 goroutine 之间的消息传递和事件通知。

有了 channel 和 goroutine 之后，Go 的并发编程变得异常容易和安全，得以让程序员把注意力留到业务上去，实现开发效率的提升。
(用同步代码解决异步问题)

#### 2.4 Channel声明
Channel 分为两种：带缓冲、不带缓冲。

对不带缓冲的 channel 进行的操作实际上可以看作“同步模式”，带缓冲的则称为“异步模式”。

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

#### 2.5 Channel使用
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
	// 创建Goroutine A&B 并接收ch数据
	go goroutineA(ch, wg)
	go goroutineB(ch, wg)
	// 发送 3&4 到ch
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

#### 2.6 Channel本质
channel 的发送和接收操作本质上都是 “值的拷贝”，无论是从 sender goroutine 的栈到 chan buf，还是从 chan buf 到 receiver goroutine，或者是直接从 sender goroutine 到 receiver goroutine。


lock or channel？
<img src="https://user-images.githubusercontent.com/10111580/112921524-824c4080-913d-11eb-87a0-6e9ce730417e.png" width="480">

### 三、Channel原理

#### 3.1 hchan数据结构说明
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

#### 3.2 带缓冲Channel的缓冲区处理

| make(chan Task, 3)| ch <- Task | ch <- Task *2 | <- ch |
| --- | --- | --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112947197-17b0fa00-9169-11eb-9f6b-d70239077a7c.png)| ![image](https://user-images.githubusercontent.com/10111580/112947749-c6edd100-9169-11eb-9ff7-dde171d998fd.png) | ![image](https://user-images.githubusercontent.com/10111580/112947808-d8cf7400-9169-11eb-8006-53676c3b3dc2.png)|![image](https://user-images.githubusercontent.com/10111580/112947854-e553cc80-9169-11eb-858b-de92b6c2a84b.png)|

> 补充chan的heap地址

#### 3.2 带缓冲Channel步骤说明

| G1 | G2 |
| --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112950069-77f56b00-916c-11eb-8817-634afc047985.png)| ![image](https://user-images.githubusercontent.com/10111580/112950094-7d52b580-916c-11eb-8179-33862cfb8ddb.png) |

##### 3.2.1 Sender ： G1
| tashCH <- task0 | 1. acquire | 2. enqueue task0(copy) | 3. release |
| --- | --- | --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112950391-d15d9a00-916c-11eb-8a4e-f7c459664bb9.png) | ![image](https://user-images.githubusercontent.com/10111580/112950435-dae70200-916c-11eb-89a4-326f1744eb4c.png) | ![image](https://user-images.githubusercontent.com/10111580/112950466-e6d2c400-916c-11eb-8517-1200da4f25fc.png) | ![image](https://user-images.githubusercontent.com/10111580/112950514-f520e000-916c-11eb-9afe-71260dd23ce0.png)|

##### 3.2.2 Recevier ： G2
| t := <-ch task0（copy) | 1. acquire | 2. dequeue | 3. release  |
| --- | --- | --- | --- |
| ![image](https://user-images.githubusercontent.com/10111580/112950931-69f41a00-916d-11eb-88a2-6e0ea87c5ea9.png) | ![image](https://user-images.githubusercontent.com/10111580/112950961-711b2800-916d-11eb-8a69-176527d49fc9.png) | ![image](https://user-images.githubusercontent.com/10111580/112951092-9445d780-916d-11eb-9726-17380283c977.png) | ![image](https://user-images.githubusercontent.com/10111580/112951140-9f990300-916d-11eb-9a90-1f5cb0e31e54.png) |

举例: 如果没有receiver, sender发送第四个task的情况
| Code | Graph |
| --- | --- |
| ch <- task1 <br> ch <- task2 <br> ch <- task3 <br>...<br>ch <- task3| <img src="https://user-images.githubusercontent.com/10111580/112955782-5d25f500-9172-11eb-8342-009169f38ce1.png" width="380">|

task将放入sendq队列中
`	c.sendq.enqueue(mysg)`


#### 3.3 不带缓冲Channel步骤说明

待补充


#### 3.4 chan.go源码说明 - send部分

```golang
// 位于 src/runtime/chan.go

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 如果 channel 是 nil
	if c == nil {
		// 不能阻塞，直接返回 false，表示未发送成功
		if !block {
			return false
		}
		// 当前 goroutine 被挂起
		gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}

	// 省略 debug 相关……

	// 对于不阻塞的 send，快速检测失败场景
	//
	// 如果 channel 未关闭且 channel 没有多余的缓冲空间。这可能是：
	// 1. channel 是非缓冲型的，且等待接收队列里没有 goroutine
	// 2. channel 是缓冲型的，但循环数组已经装满了元素
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 锁住 channel，并发安全
	lock(&c.lock)

	// 如果 channel 关闭了
	if c.closed != 0 {
		// 解锁
		unlock(&c.lock)
		// 直接 panic
		panic(plainError("send on closed channel"))
	}

	// 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 对于缓冲型的 channel，如果还有缓冲空间
	if c.qcount < c.dataqsiz {
		// qp 指向 buf 的 sendx 位置
		qp := chanbuf(c, c.sendx)

		// ……

		// 将数据从 ep 处拷贝到 qp
		typedmemmove(c.elemtype, qp, ep)
		// 发送游标值加 1
		c.sendx++
		// 如果发送游标值等于容量值，游标值归 0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		// 缓冲区的元素数量加一
		c.qcount++

		// 解锁
		unlock(&c.lock)
		return true
	}

	// 如果不需要阻塞，则直接返回错误
	if !block {
		unlock(&c.lock)
		return false
	}

	// channel 满了，发送方会被阻塞。接下来会构造一个 sudog

	// 获取当前 goroutine 的指针
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil

	// 当前 goroutine 进入发送等待队列
	c.sendq.enqueue(mysg)

	// 当前 goroutine 被挂起
	goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

	// 从这里开始被唤醒了（channel 有机会可以发送了）
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		// 被唤醒后，channel 关闭了。坑爹啊，panic
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 去掉 mysg 上绑定的 channel
	mysg.c = nil
	releaseSudog(mysg)
	return true
}

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 省略一些用不到的
	// ……

	// sg.elem 指向接收到的值存放的位置，如 val <- ch，指的就是 &val
	if sg.elem != nil {
		// 直接拷贝内存（从发送者到接收者）
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	// sudog 上绑定的 goroutine
	gp := sg.g
	// 解锁
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 唤醒接收的 goroutine. skip 和打印栈相关，暂时不理会
	goready(gp, skip+1)
}

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src 在当前 goroutine 的栈上，dst 是另一个 goroutine 的栈

	// 直接进行内存"搬迁"
	// 如果目标地址的栈发生了栈收缩，当我们读出了 sg.elem 后
	// 就不能修改真正的 dst 位置的值了
	// 因此需要在读和写之前加上一个屏障
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}
```


#### 3.5 chan.go源码说明 - receive部分

```golang
// 位于 src/runtime/chan.go

// chanrecv 函数接收 channel c 的元素并将其写入 ep 所指向的内存地址。
// 如果 ep 是 nil，说明忽略了接收值。
// 如果 block == false，即非阻塞型接收，在没有数据可接收的情况下，返回 (false, false)
// 否则，如果 c 处于关闭状态，将 ep 指向的地址清零，返回 (true, false)
// 否则，用返回值填充 ep 指向的内存地址。返回 (true, true)
// 如果 ep 非空，则应该指向堆或者函数调用者的栈

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 省略 debug 内容 …………

	// 如果是一个 nil 的 channel
	if c == nil {
		// 如果不阻塞，直接返回 (false, false)
		if !block {
			return
		}
		// 否则，接收一个 nil 的 channel，goroutine 挂起
		gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
		// 不会执行到这里
		throw("unreachable")
	}

	// 在非阻塞模式下，快速检测到失败，不用获取锁，快速返回
	// 当我们观察到 channel 没准备好接收：
	// 1. 非缓冲型，等待发送列队 sendq 里没有 goroutine 在等待
	// 2. 缓冲型，但 buf 里没有元素
	// 之后，又观察到 closed == 0，即 channel 未关闭。
	// 因为 channel 不可能被重复打开，所以前一个观测的时候 channel 也是未关闭的，
	// 因此在这种情况下可以直接宣布接收失败，返回 (false, false)
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 加锁
	lock(&c.lock)

	// channel 已关闭，并且循环数组 buf 里没有元素
	// 这里可以处理非缓冲型关闭 和 缓冲型关闭但 buf 无元素的情况
	// 也就是说即使是关闭状态，但在缓冲型的 channel，
	// buf 里有元素的情况下还能接收到元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		// 解锁
		unlock(&c.lock)
		if ep != nil {
			// 从一个已关闭的 channel 执行接收操作，且未忽略返回值
			// 那么接收的值将是一个该类型的零值
			// typedmemclr 根据类型清理相应地址的内存
			typedmemclr(c.elemtype, ep)
		}
		// 从一个已关闭的 channel 接收，selected 会返回true
		return true, false
	}

	// 等待发送队列里有 goroutine 存在，说明 buf 是满的
	// 这有可能是：
	// 1. 非缓冲型的 channel
	// 2. 缓冲型的 channel，但 buf 满了
	// 针对 1，直接进行内存拷贝（从 sender goroutine -> receiver goroutine）
	// 针对 2，接收到循环数组头部的元素，并将发送者的元素放到循环数组尾部
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 缓冲型，buf 里有元素，可以正常接收
	if c.qcount > 0 {
		// 直接从循环数组里找到要接收的元素
		qp := chanbuf(c, c.recvx)

		// …………

		// 代码里，没有忽略要接收的值，不是 "<- ch"，而是 "val <- ch"，ep 指向 val
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 清理掉循环数组里相应位置的值
		typedmemclr(c.elemtype, qp)
		// 接收游标向前移动
		c.recvx++
		// 接收游标归零
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// buf 数组里的元素个数减 1
		c.qcount--
		// 解锁
		unlock(&c.lock)
		return true, true
	}

	if !block {
		// 非阻塞接收，解锁。selected 返回 false，因为没有接收到值
		unlock(&c.lock)
		return false, false
	}

	// 接下来就是要被阻塞的情况了
	// 构造一个 sudog
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	// 待接收数据的地址保存下来
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.param = nil
	// 进入channel 的等待接收队列
	c.recvq.enqueue(mysg)
	// 将当前 goroutine 挂起
	goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

	// 被唤醒了，接着从这里继续执行一些扫尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}


func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 如果是非缓冲型的 channel
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		// 未忽略接收的数据
		if ep != nil {
			// 直接拷贝数据，从 sender goroutine -> receiver goroutine
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 缓冲型的 channel，但 buf 已满。
		// 将循环数组 buf 队首的元素拷贝到接收数据的地址
		// 将发送者的数据入队。实际上这时 revx 和 sendx 值相等
		// 找到接收游标
		qp := chanbuf(c, c.recvx)
		// …………
		// 将接收游标处的数据拷贝给接收者
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}

		// 将发送者数据拷贝到 buf
		typedmemmove(c.elemtype, qp, sg.elem)
		// 更新游标值
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx
	}
	sg.elem = nil
	gp := sg.g

	// 解锁
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}

	// 唤醒发送的 goroutine。需要等到调度器的光临
	goready(gp, skip+1)
}

// 无缓冲或缓冲区满
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}

// 有缓冲
func chanbuf(c *hchan, i uint) unsafe.Pointer {
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}
```

#### 3.6 补充G的切换和唤醒：goPark & goReady
goPart -> mcall(pc/sp->m.sched) -> parm_m (schedule -> execute -> gogo(m.sched -> pc/sp))
goReady -> 


#### 3.7 资源泄漏

Channel 可能会引发 goroutine 泄漏。

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，不见天日。

###四、Channel使用问题
关于 channel 的使用，有几点不方便的地方：
* 在不改变 channel 自身状态的情况下，无法获知一个 channel 是否关闭。
* 关闭一个 closed channel 会导致 panic。所以，如果关闭 channel 的一方在不知道 channel 是否处于关闭状态时就去贸然关闭 channel 是很危险的事情。
* 向一个 closed channel 发送数据会导致 panic。所以，如果向 channel 发送数据的一方不知道 channel 是否处于关闭状态时就去贸然向 channel 发送数据是很危险的事情。

#### 4.1 原则

* don’t close a channel from the receiver side and don’t close a channel if the channel has multiple concurrent senders.
* don’t close (or send values to) closed channels.

FIX：

* 使用 defer-recover 机制，放心大胆地关闭 channel 或者向 channel 发送数据。即使发生了 panic，有 defer-recover 在兜底。
* 使用 sync.Once 来保证只关闭一次。
