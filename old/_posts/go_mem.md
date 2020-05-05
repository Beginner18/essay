---
title: go 内存
date: 2019-06-13 21:55:41
tags: go,内存
---

go内存模型
===
1. **内存模型剖析**（同步模型）
---
>- 一个goroutine内compilers and processors 可能会reorder读写的执行顺序，仅当reorder不会改变coder语言定义的语义执行顺序；
>
```
For example, if one goroutine executes a = 1; b = 2;, 
another might observe the updated value of b before the updated value of a.
由于compiler与processors的reorder一个goroutine内的具体读写执行顺序并不一定与coder的语义顺序一致；
```
>- e1事件发生在e2事件前，标明e2事件发生在e1事件后，若e1事件即不发生在e2事件前又不发生在e2事件后，则e1和e2 happen concurrently.
>- 单个goroutine内事件发生顺序与编码一致(Within a single goroutine, the happens-before order is the order expressed by the program.)
>
```
r事件(读v) 可以观察到w事件写的v（w写事件(写v) ）需要满足以下条件：
1 r does not happen before w.
2 There is no other write w' to v that happens after w but before r.
保证r事件观察到特殊的w写v事件需要满足以下:（This pair of conditions is stronger than the first pair; it requires that there are no other writes happening concurrently with w or r.）
1 w happend before r
2 其他对v的写要么在w前要么在r后
```
>- 单个goroutine内不存在concurrency,不同goroutine共享变量需要控制同步机制
>- 变量v的初始化在内存模型中为，v的对应类型0值写事件
>- 读写值超过一个机器字的读写行为是：非特定顺序的多个机器字size的读写操作
>```
Reads and writes of values larger than a single machine word behave as multiple machine-word-sized operations in an unspecified order.
```
>- 初始化：
>
```
Program initialization runs in a single goroutine, but that goroutine may create other goroutines, which run concurrently.
If a package p imports package q, the completion of q's init functions happens before the start of any of p's.
The start of the function main.main happens after all init functions have finished.
```
>- goroutine的创建: go语句创建goroutine发生在goroutine's的执行开始
>
```
The go statement that starts a new goroutine happens before the goroutine's execution begins.
For example, in this program:
var a string
func f() {
	print(a)
}
func hello() {
	a = "hello, world"
	go f()
}
calling hello will print "hello, world" at some point in the future (perhaps after hello has returned).
```
>- goroutine的desctruction:goroutine的退出无法保证happend before任何事件，若goroutine的effects需要被其他goroutine观察到需要用同步机制
>
```
The exit of a goroutine is not guaranteed to happen before any event in the program. For example, in this program:
var a string
func hello() {
	go func() { a = "hello" }()
	print(a)
}
the assignment to a is not followed by any synchronization event, so it is not guaranteed to be observed by any other goroutine. In fact, an aggressive compiler might delete the entire go statement.
If the effects of a goroutine must be observed by another goroutine, use a synchronization mechanism such as a lock or channel communication to establish a relative ordering.
```
>- Channel communication:goroutine间通信
>
>>- goroutine间通信的主要方法
>>- Each send on a particular channel is matched to a corresponding receive from that channel, usually in a different goroutine.
>>- A send on a channel happens before the corresponding receive from that channel completes.
>>- The closing of a channel happens before a receive that returns a zero value because the channel is closed.
>>- A receive from an unbuffered channel happens before the send on that channel completes.
>>
```
This program:
var c = make(chan int, 10)
var a string
func f() {
	a = "hello, world"
	c <- 0
}
func main() {
	go f()
	<-c
	print(a)
}
一定打印出"hello, world". The write to a happens before the send on c, which happens before the corresponding receive on c completes, which happens before the print.
In this example, replacing c <- 0 with close(c) yields a program with the same guaranteed behavior.
example2: 
var c = make(chan int)
var a string
func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
is also guaranteed to print "hello, world". The write to a happens before the receive on c, which happens before the corresponding send on c completes, which happens before the print.
If the channel were buffered (e.g., c = make(chan int, 1)) then the program would not be guaranteed to print "hello, world". (It might print the empty string, crash, or do something else.)
```
>>- The kth receive on a channel with capacity C happens before the k+Cth send from that channel completes.限制并发量的通用方法，模拟信号量，缓冲channel中的item量与active use的数目一致，channel的容量与同时使用的最大数量限制一致，channel发送数据获取一个信号，接收一个项释放一个信号(The kth receive on a channel with capacity C happens before the k+Cth send from that channel completes. This rule generalizes the previous rule to buffered channels. It allows a counting semaphore to be modeled by a buffered channel: the number of items in the channel corresponds to the number of active uses, the capacity of the channel corresponds to the maximum number of simultaneous uses, sending an item acquires the semaphore, and receiving an item releases the semaphore. This is a common idiom for limiting concurrency.)
>>
```
var limit = make(chan int, 3)
func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

>- 锁： sync.Mutex and sync.RWMutex
>
>>- For any sync.Mutex or sync.RWMutex variable l and n < m, call n of l.Unlock() happens before call m of l.Lock() returns.
>>- For any call to l.RLock on a sync.RWMutex variable l, there is an n such that the l.RLock happens (returns) after call n to l.Unlock and the matching l.RUnlock happens before call n+1 to l.Lock. l.rlock发生在第n个l.unlock之后，l.runlock发生在n+1 l.lock前；（读锁发生在上一个写锁解锁后，读锁解锁发生在lock锁之前）
>>
```
This program:
var l sync.Mutex
var a string
func f() {
	a = "hello, world"
	l.Unlock()
}
func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
is guaranteed to print "hello, world". The first call to l.Unlock() (in f) happens before the second call to l.Lock() (in main) returns, which happens before the print.
```

>- once: 用于多goroutine初始化仅初始化一次，多个goroutine执行once.do(f)，只有一个goroutine可以执行f，其他调用会堵塞知道执行f的goroutine完成f的执行
>
>>- The sync package provides a safe mechanism for initialization in the presence of multiple goroutines through the use of the Once type. Multiple threads can execute once.Do(f) for a particular f, but only one will run f(), and the other calls block until f() has returned.
>>- A single call of f() from once.Do(f) happens (returns) before any call of once.Do(f) returns.
>>
```
In this program:
var a string
var once sync.Once
func setup() {
	a = "hello, world"
}
func doprint() {
	once.Do(setup)
	print(a)
}
func twoprint() {
	go doprint()//执行f的goroutine执行完，另一个才会解除堵塞，print时两者均可拿到a
	go doprint()
}
calling twoprint will call setup exactly once. The setup function will complete before either call of print. The result will be that "hello, world" will be printed twice.
```

>- 不正确的同步：显式同步解决此问题； In all these examples, the solution is the same: use explicit synchronization.

>
```
Note that a read r may observe the value written by a write w that happens concurrently with r. Even if this occurs, it does not imply that reads happening after r will observe writes that happened before w.
In this program:
var a, b int
func f() {
	a = 1
	b = 2
}
func g() {
	print(b)
	print(a)
}
func main() {
	go f()
	g()
}
it can happen that g prints 2 and then 0.
This fact invalidates a few common idioms.
```
>
```
Double-checked locking is an attempt to avoid the overhead of synchronization. For example, the twoprint program might be incorrectly written as:
var a string
var done bool
func setup() {
	a = "hello, world"
	done = true
}
func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}
func twoprint() {
	go doprint()
	go doprint()
}
but there is no guarantee that, in doprint, observing the write to done implies observing the write to a. This version can (incorrectly) print an empty string instead of "hello, world".
```
>
```
Another incorrect idiom is busy waiting for a value, as in:
var a string
var done bool
func setup() {
	a = "hello, world"
	done = true
}
func main() {
	go setup()
	for !done {
	}
	print(a)
}
As before, there is no guarantee that, in main, observing the write to done implies observing the write to a, so this program could print an empty string too. Worse, there is no guarantee that the write to done will ever be observed by main, since there are no synchronization events between the two threads. The loop in main is not guaranteed to finish.
There are subtler variants on this theme, such as this program.
type T struct {
	msg string
}
var g *T
func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}
func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
Even if main observes g != nil and exits its loop, there is no guarantee that it will observe the initialized value for g.msg.
```

>- 显示同步：channel sync/atomic rwlock, mutex lock
>
[参考](https://studygolang.com/articles/1597)
[官网参考](https://golang.org/ref/mem)

2. [**逃逸分析**](https://www.jianshu.com/p/72e1a2571f01)
---
>- 逃逸分析用处：
>
```
最大的好处应该是减少gc的压力，不逃逸的对象分配在栈上，当函数返回时就回收了资源，不需要gc标记清除。
因为逃逸分析完后可以确定哪些变量可以分配在栈上，栈的分配比堆快，性能好
同步消除，如果你定义的对象的方法上有同步锁，但在运行时，却只有一个线程在访问，此时逃逸分析后的机器码，会去掉同步锁运行。
```

>- 编译参数加入 -gcflags '-m -l'
>- go消除堆栈区别  
>- 变量生命周期超出其所在的栈，则逃逸至堆
>- 逃逸分析确定变量的储存区域为堆/栈，减少gc压力，栈可以直接回收，栈分配空间及回收空间较快
>- 代码段

>> ```package main
import (
	"time"
	"fmt"
)
func main(){
	//go_parser.TestParser("a==1 && b!=2")
	res := make(map[string]int)
	fmt.Println(res["res"])
	fmt.Println(int(time.Now().Weekday()))
	// test grpc dispatch get scenes
	//grpc.GetScene(20351, imkfdispatch.Scene_BuyCarConsult)
	res1 := taoyi()
	fmt.Println("2st addr of i: ", res1)
	i := *res1
	fmt.Println("the res is: %d", i)
	fmt.Println("3st addr of i: ", &i)
}
func taoyi()(*int){
	var i int = 1
	fmt.Println("1st addr of i: ", &i)
	return &i
}
```

>- 输出内容：

>> ./main.go:25:14: "1st addr of i: " escapes to heap
./main.go:25:33: &i escapes to heap
./main.go:25:33: &i escapes to heap
./main.go:24:6: moved to heap: i
./main.go:26:9: &i escapes to heap
./main.go:25:13: taoyi ... argument does not escape
./main.go:11:17: res["res"] escapes to heap
./main.go:13:17: int(time.Now().Weekday()) escapes to heap
./main.go:17:14: "2st addr of i: " escapes to heap
./main.go:17:14: res1 escapes to heap
./main.go:19:14: "the res is: %d" escapes to heap
./main.go:19:14: i escapes to heap
./main.go:20:14: "3st addr of i: " escapes to heap
./main.go:20:33: &i escapes to heap
./main.go:20:33: &i escapes to heap
./main.go:18:2: moved to heap: i
./main.go:10:13: main make(map[string]int) does not escape
./main.go:11:13: main ... argument does not escape
./main.go:13:13: main ... argument does not escape
./main.go:17:13: main ... argument does not escape
./main.go:19:13: main ... argument does not escape
./main.go:20:13: main ... argument does not escape

[参考1](https://studygolang.com/articles/10026)

3. 垃圾回收
---
>- 标记清除
>- 白色：未被引用
>- 灰色：被引用，待查询灰色的引用
>- 黑色：被引用，其引用已标记 
>- 最终白色被清楚
>- stop the world
>- 更新版 ：清除可以跟用户线程同步（我们知道Golang三色标记法中最后只剩下的黑白两种对象，黑色对象是程序恢复后接着使用的对象，如果不碰触黑色对象，只清除白色的对象，肯定不会影响程序逻辑。所以：清除操作和用户逻辑可以并发。）
>- 标记操作与用户逻辑的并发保证：存在短暂stw
>
```
标记阶段，堆中新增数据：若标记为白色会被错误清除
Golang为了解决这个问题，引入了写屏障这个机制。
写屏障：该屏障之前的写操作和之后的写操作相比，先被系统其它组件感知。
通俗的讲：就是在gc跑的过程中，可以监控对象的内存修改，并对对象进行重新标记。(实际上也是超短暂的stw，然后对对象进行标记)
在上述情况中，新生成的对象，一律都标位灰色！
灰色或者黑色对象的引用改为白色对象的时候，Golang是该如何操作的？
看如下图，一个黑色对象引用了曾经标记的白色对象。
这时候，写屏障机制被触发，向GC发送信号，GC重新扫描对象并标位灰色。
```
>- 因此，gc一旦开始，无论是创建对象还是对象的引用改变，都会先变为灰色。
[gc回收](https://blog.csdn.net/u010649766/article/details/80582153)
[三色标记法](https://i6448038.github.io/2019/03/04/golang-garbage-collector/)

