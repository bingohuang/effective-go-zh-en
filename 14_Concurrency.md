## Concurrency

## 并发

### Share by communicating

### 通过通信共享内存

Concurrent programming is a large topic and there is space only for some Go-specific highlights here.

并发编程是个很大的论题。但限于篇幅，这里仅讨论一些 Go 特有的东西。

Concurrent programming in many environments is made difficult by the subtleties required to implement correct access to shared variables. Go encourages a different approach in which shared values are passed around on channels and, in fact, never actively shared by separate threads of execution. Only one goroutine has access to the value at any given time. Data races cannot occur, by design. To encourage this way of thinking we have reduced it to a slogan:

在并发编程中，为实现对共享变量的正确访问需要精确的控制，这在多数环境下都很困难。 Go 语言另辟蹊径，它将共享的值通过信道传递，实际上，多个独立执行的线程从不会主动共享。 在任意给定的时间点，只有一个 goroutine 能够访问该值。数据竞争从设计上就被杜绝了。 为了提倡这种思考方式，我们将它简化为一句口号：

> Do not communicate by sharing memory; instead, share memory by communicating.

> 不要通过共享内存来通信，而应通过通信来共享内存。

This approach can be taken too far. Reference counts may be best done by putting a mutex around an integer variable, for instance. But as a high-level approach, using channels to control access makes it easier to write clear, correct programs.

这种方法意义深远。例如，引用计数通过为整数变量添加互斥锁来很好地实现。 但作为一种高级方法，通过信道来控制访问能够让你写出更简洁，正确的程序。

One way to think about this model is to consider a typical single-threaded program running on one CPU. It has no need for synchronization primitives. Now run another such instance; it too needs no synchronization. Now let those two communicate; if the communication is the synchronizer, there's still no need for other synchronization. Unix pipelines, for example, fit this model perfectly. Although Go's approach to concurrency originates in Hoare's Communicating Sequential Processes (CSP), it can also be seen as a type-safe generalization of Unix pipes.

我们可以从典型的单线程运行在单 CPU 之上的情形来审视这种模型。它无需提供同步原语。 现在再运行一个线程，它也无需同步。现在让它们俩进行通信。若将通信过程看做同步着， 那就完全不需要其它同步了。例如，Unix 管道就与这种模型完美契合。 尽管 Go 的并发处理方式来源于 Hoare 的通信顺序处理（CSP）， 它依然可以看做是类型安全的 Unix 管道的实现。

### Goroutines

### Goroutines

They're called goroutines because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. And the stacks start small, so they are cheap, and grow by allocating (and freeing) heap storage as required.

我们称之为 ** goroutine ** ，是因为现有的术语—线程、协程、进程等等—无法准确传达它的含义。 Goroutine 具有简单的模型：它是与其它 goroutine 并发运行在同一地址空间的函数。它是轻量级的， 所有消耗几乎就只有栈空间的分配。而且栈最开始是非常小的，所以它们很廉价， 仅在需要时才会随着堆空间的分配（和释放）而变化。

Goroutines are multiplexed onto multiple OS threads so if one should block, such as while waiting for I/O, others continue to run. Their design hides many of the complexities of thread creation and management.

Goroutine 在多线程操作系统上可实现多路复用，因此若一个线程阻塞，比如说等待 I/O， 那么其它的线程就会运行。Goroutine 的设计隐藏了线程创建和管理的诸多复杂性。

Prefix a function or method call with the go keyword to run the call in a new goroutine. When the call completes, the goroutine exits, silently. (The effect is similar to the Unix shell's & notation for running a command in the background.)

在函数或方法前添加 go 关键字能够在新的 goroutine 中调用它。当调用完成后， 该 goroutine 也会安静地退出。（效果有点像 Unix Shell 中的 & 符号，它能让命令在后台运行。）

```go
go list.Sort()  // run list.Sort concurrently; don't wait for it.
```
```go
go list.Sort()  // 并发运行 list.Sort，无需等它结束。
```
A function literal can be handy in a goroutine invocation.

函数字面在 goroutine 调用中非常有用。

```go
func Announce(message string, delay time.Duration) {
	go func() {
		time.Sleep(delay)
		fmt.Println(message)
	}()  // Note the parentheses - must call the function.
}
```
```go
func Announce(message string, delay time.Duration) {
	go func() {
		time.Sleep(delay)
		fmt.Println(message)
	}()  // 注意括号 - 必须调用该函数。
}
```
In Go, function literals are closures: the implementation makes sure the variables referred to by the function survive as long as they are active.

在 Go 中，函数字面都是闭包：其实现在保证了函数内引用变量的生命周期与函数的活动时间相同。

These examples aren't too practical because the functions have no way of signaling completion. For that, we need channels.

这些函数没什么实用性，因为它们没有实现完成时的信号处理。因此，我们需要信道。

### Channels

### 信道

Like maps, channels are allocated with make, and the resulting value acts as a reference to an underlying data structure. If an optional integer parameter is provided, it sets the buffer size for the channel. The default is zero, for an unbuffered or synchronous channel.

信道与映射一样，也需要通过 make 来分配内存。其结果值充当了对底层数据结构的引用。 若提供了一个可选的整数形参，它就会为该信道设置缓冲区大小。默认值是零，表示不带缓冲的或同步的信道。

```go
ci := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
```
```go
ci := make(chan int)            // 整数类型的无缓冲信道
cj := make(chan int, 0)         // 整数类型的无缓冲信道
cs := make(chan *os.File, 100)  // 指向文件指针的带缓冲信道
```
Unbuffered channels combine communication—the exchange of a value—with synchronization—guaranteeing that two calculations (goroutines) are in a known state.

无缓冲信道在通信时会同步交换数据，它能确保（两个 goroutine）计算处于确定状态。

There are lots of nice idioms using channels. Here's one to get us started. In the previous section we launched a sort in the background. A channel can allow the launching goroutine to wait for the sort to complete.

信道有很多惯用法，我们从这里开始了解。在上一节中，我们在后台启动了排序操作。 信道使得启动的 goroutine 等待排序完成。

```go
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
	list.Sort()
	c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```
```go
c := make(chan int)  // 分配一个信道
// 在 goroutine 中启动排序。当它完成后，在信道上发送信号。
go func() {
	list.Sort()
	c <- 1  // 发送信号，什么值无所谓。
}()
doSomethingForAWhile()
<-c   // 等待排序结束，丢弃发来的值。
```
Receivers always block until there is data to receive. If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.

接收者在收到数据前会一直阻塞。若信道是不带缓冲的，那么在接收者收到值前， 发送者会一直阻塞；若信道是带缓冲的，则发送者仅仅只需阻塞到值被复制到缓冲区； 若缓冲区已满，发送者会一直等待直到某个接收者取出一个值为止。

A buffered channel can be used like a semaphore, for instance to limit throughput. In this example, incoming requests are passed to handle, which sends a value into the channel, processes the request, and then receives a value from the channel to ready the “semaphore” for the next consumer. The capacity of the channel buffer limits the number of simultaneous calls to process.

带缓冲的信道可被用作信号量，例如限制吞吐量。在此例中，进入的请求会被传递给 handle，它从信道中接收值，处理请求后将值发回该信道中，以便让该 “信号量” 准备迎接下一次请求。信道缓冲区的容量决定了同时调用 process 的数量上限。

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
	sem <- 1    // Wait for active queue to drain.
	process(r)  // May take a long time.
	<-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
	for {
		req := <-queue
		go handle(req)  // Don't wait for handle to finish.
	}
}
```
```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
	sem <- 1 // 等待活动队列清空。
	process(r)  // 可能需要很长时间。
	<-sem    // 完成；使下一个请求可以运行。
}

func Serve(queue chan *Request) {
	for {
		req := <-queue
		go handle(req)  // 无需等待 handle 结束。
	}
}
```
Once MaxOutstanding handlers are executing process, any more will block trying to send into the filled channel buffer, until one of the existing handlers finishes and receives from the buffer.

一旦有 MaxOutstanding 个处理器进入运行状态，其他的所有处理器都会在试图发送值到信道缓冲区的时候阻塞，直到某个处理器完成处理并从缓冲区取回一个值为止。

This design has a problem, though: Serve creates a new goroutine for every incoming request, even though only MaxOutstanding of them can run at any moment. As a result, the program can consume unlimited resources if the requests come in too fast. We can address that deficiency by changing Serve to gate the creation of the goroutines. Here's an obvious solution, but beware it has a bug we'll fix subsequently:

然而，它却有个设计问题：尽管只有 MaxOutstanding 个 goroutine 能同时运行，但 Serve 还是为每个进入的请求都创建了新的 goroutine。其结果就是，若请求来得很快， 该程序就会无限地消耗资源。为了弥补这种不足，我们可以通过修改 Serve 来限制创建 Go 程，这是个明显的解决方案，但要当心我们修复后出现的 Bug。

```go
func Serve(queue chan *Request) {
	for req := range queue {
		sem <- 1
		go func() {
			process(req) // Buggy; see explanation below.
			<-sem
		}()
	}
}
```
```go
func Serve(queue chan *Request) {
	for req := range queue {
		sem <- 1
		go func() {
			process(req) // 这儿有 Bug，解释见下。
			<-sem
		}()
	}
}
```
The bug is that in a Go for loop, the loop variable is reused for each iteration, so the req variable is shared across all goroutines. That's not what we want. We need to make sure that req is unique for each goroutine. Here's one way to do that, passing the value of req as an argument to the closure in the goroutine:

Bug 出现在 Go 的 for 循环中，该循环变量在每次迭代时会被重用，因此 req 变量会在所有的 goroutine 间共享，这不是我们想要的。我们需要确保 req 对于每个 goroutine 来说都是唯一的。有一种方法能够做到，就是将 req 的值作为实参传入到该 goroutine 的闭包中：

```go
func Serve(queue chan *Request) {
	for req := range queue {
		sem <- 1
		go func(req *Request) {
			process(req)
			<-sem
		}(req)
	}
}
```
Compare this version with the previous to see the difference in how the closure is declared and run. Another solution is just to create a new variable with the same name, as in this example:

比较前后两个版本，观察该闭包声明和运行中的差别。 另一种解决方案就是以相同的名字创建新的变量，如例中所示：

```go
func Serve(queue chan *Request) {
	for req := range queue {
		req := req // Create new instance of req for the goroutine.
		sem <- 1
		go func() {
			process(req)
			<-sem
		}()
	}
}
```
```go
func Serve(queue chan *Request) {
	for req := range queue {
		req := req // 为该 Go 程创建 req 的新实例。
		sem <- 1
		go func() {
			process(req)
			<-sem
		}()
	}
}
```
It may seem odd to write

它的写法看起来有点奇怪

```go
req := req
```
but it's a legal and idiomatic in Go to do this. You get a fresh version of the variable with the same name, deliberately shadowing the loop variable locally but unique to each goroutine.

但在 Go 中这样做是合法且惯用的。你用相同的名字获得了该变量的一个新的版本， 以此来局部地刻意屏蔽循环变量，使它对每个 goroutine 保持唯一。

Going back to the general problem of writing the server, another approach that manages resources well is to start a fixed number of handle goroutines all reading from the request channel. The number of goroutines limits the number of simultaneous calls to process. This Serve function also accepts a channel on which it will be told to exit; after launching the goroutines it blocks receiving from that channel.

回到编写服务器的一般问题上来。另一种管理资源的好方法就是启动固定数量的 handle goroutine，一起从请求信道中读取数据。Goroutine 的数量限制了同时调用 process 的数量。Serve 同样会接收一个通知退出的信道， 在启动所有 goroutine 后，它将阻塞并暂停从信道中接收消息。

```go
func handle(queue chan *Request) {
	for r := range queue {
		process(r)
	}
}

func Serve(clientRequests chan *Request, quit chan bool) {
	// Start handlers
	for i := 0; i < MaxOutstanding; i++ {
		go handle(clientRequests)
	}
	<-quit  // Wait to be told to exit.
}
```
```go
func handle(queue chan *Request) {
	for r := range queue {
		process(r)
	}
}

func Serve(clientRequests chan *Request, quit chan bool) {
	// 启动处理程序
	for i := 0; i < MaxOutstanding; i++ {
		go handle(clientRequests)
	}
	<-quit  // 等待通知退出。
}
```
### Channels of channels

### 信道中的信道

One of the most important properties of Go is that a channel is a first-class value that can be allocated and passed around like any other. A common use of this property is to implement safe, parallel demultiplexing.

Go 最重要的特性就是信道是一等值，它可以被分配并像其它值到处传递。 这种特性通常被用来实现安全、并行的多路分解。

In the example in the previous section, handle was an idealized handler for a request but we didn't define the type it was handling. If that type includes a channel on which to reply, each client can provide its own path for the answer. Here's a schematic definition of type Request.

在上一节的例子中，handle 是个非常理想化的请求处理程序， 但我们并未定义它所处理的请求类型。若该类型包含一个可用于回复的信道， 那么每一个客户端都能为其回应提供自己的路径。以下为 Request 类型的大概定义。

```go
type Request struct {
	args        []int
	f           func([]int) int
	resultChan  chan int
}
```
The client provides a function and its arguments, as well as a channel inside the request object on which to receive the answer.

客户端提供了一个函数及其实参，此外在请求对象中还有个接收应答的信道。

```go
func sum(a []int) (s int) {
	for _, v := range a {
		s += v
	}
	return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)
```
```go
func sum(a []int) (s int) {
	for _, v := range a {
		s += v
	}
	return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// 发送请求
clientRequests <- request
// 等待回应
fmt.Printf("answer: %d\n", <-request.resultChan)
```
On the server side, the handler function is the only thing that changes.

在服务端，只需改动 handler 函数。

```go
func handle(queue chan *Request) {
	for req := range queue {
		req.resultChan <- req.f(req.args)
	}
}
```
There's clearly a lot more to do to make it realistic, but this code is a framework for a rate-limited, parallel, non-blocking RPC system, and there's not a mutex in sight.

要使其实际可用还有很多工作要做，这些代码仅能实现一个速率有限、并行、非阻塞 RPC 系统的框架，而且它并不包含互斥锁。

### Parallelization

### 并行化

Another application of these ideas is to parallelize a calculation across multiple CPU cores. If the calculation can be broken into separate pieces that can execute independently, it can be parallelized, with a channel to signal when each piece completes.

这些设计的另一个应用是在多 CPU 核心上实现并行计算。如果计算过程能够被分为几块可独立执行的过程，它就可以在每块计算结束时向信道发送信号，从而实现并行处理。

Let's say we have an expensive operation to perform on a vector of items, and that the value of the operation on each item is independent, as in this idealized example.

让我们看看这个理想化的例子。我们在对一系列向量项进行极耗资源的操作， 而每个项的值计算是完全独立的。

```go
type Vector []float64

// Apply the operation to v[i], v[i+1] ... up to v[n-1].
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
	for ; i < n; i++ {
		v[i] += u.Op(v[i])
	}
	c <- 1    // signal that this piece is done
}
```
```go
type Vector []float64

// 将此操应用至 v[i], v[i+1] ... 直到 v[n-1]
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
	for ; i < n; i++ {
		v[i] += u.Op(v[i])
	}
	c <- 1    // 发信号表示这一块计算完成。
}
```
We launch the pieces independently in a loop, one per CPU. They can complete in any order but it doesn't matter; we just count the completion signals by draining the channel after launching all the goroutines.

我们在循环中启动了独立的处理块，每个 CPU 将执行一个处理。 它们有可能以乱序的形式完成并结束，但这没有关系； 我们只需在所有 goroutine 开始后接收，并统计信道中的完成信号即可。

```go
const NCPU = 4  // number of CPU cores

func (v Vector) DoAll(u Vector) {
	c := make(chan int, NCPU)  // Buffering optional but sensible.
	for i := 0; i < NCPU; i++ {
		go v.DoSome(i*len(v)/NCPU, (i+1)*len(v)/NCPU, u, c)
	}
	// Drain the channel.
	for i := 0; i < NCPU; i++ {
		<-c    // wait for one task to complete
	}
	// All done.
}
```
```go
const NCPU = 4  // CPU 核心数

func (v Vector) DoAll(u Vector) {
	c := make(chan int, NCPU)  // 缓冲区是可选的，但明显用上更好
	for i := 0; i < NCPU; i++ {
		go v.DoSome(i*len(v)/NCPU, (i+1)*len(v)/NCPU, u, c)
	}
	// 排空信道。
	for i := 0; i < NCPU; i++ {
		<-c    // 等待任务完成
	}
	// 一切完成。
}
```
The current implementation of the Go runtime will not parallelize this code by default. It dedicates only a single core to user-level processing. An arbitrary number of goroutines can be blocked in system calls, but by default only one can be executing user-level code at any time. It should be smarter and one day it will be smarter, but until it is if you want CPU parallelism you must tell the run-time how many goroutines you want executing code simultaneously. There are two related ways to do this. Either run your job with environment variable GOMAXPROCS set to the number of cores to use or import the runtime package and call runtime.GOMAXPROCS(NCPU). A helpful value might be runtime.NumCPU(), which reports the number of logical CPUs on the local machine. Again, this requirement is expected to be retired as the scheduling and run-time improve.

目前 Go 运行时的实现默认并不会并行执行代码，它只为用户层代码提供单一的处理核心。 任意数量的 goroutine 都可能在系统调用中被阻塞，而在任意时刻默认只有一个会执行用户层代码。 它应当变得更智能，而且它将来肯定会变得更智能。但现在，若你希望 CPU 并行执行， 就必须告诉运行时你希望同时有多少 goroutine 能执行代码。有两种途径可达到这一目的，要么 在运行你的工作时将 GOMAXPROCS 环境变量设为你要使用的核心数， 要么导入 runtime 包并调用 runtime.GOMAXPROCS(NCPU)。 runtime.NumCPU() 的值可能很有用，它会返回当前机器的逻辑 CPU 核心数。 当然，随着调度算法和运行时的改进，将来会不再需要这种方法。

Be sure not to confuse the ideas of concurrency—structuring a program as independently executing components—and parallelism—executing calculations in parallel for efficiency on multiple CPUs. Although the concurrency features of Go can make some problems easy to structure as parallel computations, Go is a concurrent language, not a parallel one, and not all parallelization problems fit Go's model. For a discussion of the distinction, see the talk cited in [this blog post](https://blog.golang.org/2013/01/concurrency-is-not-parallelism.html).

注意不要混淆并发和并行的概念：并发是用可独立执行的组件构造程序的方法， 而并行则是为了效率在多 CPU 上平行地进行计算。尽管 Go 的并发特性能够让某些问题更易构造成并行计算， 但 Go 仍然是种并发而非并行的语言，且 Go 的模型并不适合所有的并行问题。 关于其中区别的讨论，见 [此博文](https://blog.golang.org/2013/01/concurrency-is-not-parallelism.html)。

### A leaky buffer

### 可能泄露的缓冲区

The tools of concurrent programming can even make non-concurrent ideas easier to express. Here's an example abstracted from an RPC package. The client goroutine loops receiving data from some source, perhaps a network. To avoid allocating and freeing buffers, it keeps a free list, and uses a buffered channel to represent it. If the channel is empty, a new buffer gets allocated. Once the message buffer is ready, it's sent to the server on serverChan.

并发编程的工具甚至能很容易地表达非并发的思想。这里有个提取自 RPC 包的例子。 客户端 Go 程从某些来源，可能是网络中循环接收数据。为避免分配和释放缓冲区， 它保存了一个空闲链表，使用一个带缓冲信道表示。若信道为空，就会分配新的缓冲区。 一旦消息缓冲区就绪，它将通过 serverChan 被发送到服务器。

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
	for {
		var b *Buffer
		// Grab a buffer if available; allocate if not.
		select {
		case b = <-freeList:
			// Got one; nothing more to do.
		default:
			// None free, so allocate a new one.
			b = new(Buffer)
		}
		load(b)              // Read next message from the net.
		serverChan <- b      // Send to server.
	}
}
```
```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
	for {
		var b *Buffer
		// 若缓冲区可用就用它，不可用就分配个新的。
		select {
		case b = <-freeList:
			// 获取一个，不做别的。
		default:
			// 非空闲，因此分配一个新的。
			b = new(Buffer)
		}
		load(b)              // 从网络中读取下一条消息。
		serverChan <- b   // 发送至服务器。
	}
}
```
The server loop receives each message from the client, processes it, and returns the buffer to the free list.

服务器从客户端循环接收每个消息，处理它们，并将缓冲区返回给空闲列表。

```go
func server() {
	for {
		b := <-serverChan    // Wait for work.
		process(b)
		// Reuse buffer if there's room.
		select {
		case freeList <- b:
			// Buffer on free list; nothing more to do.
		default:
			// Free list full, just carry on.
		}
	}
}
```
```go
func server() {
	for {
		b := <-serverChan    // 等待工作。
		process(b)
		// 若缓冲区有空间就重用它。
		select {
		case freeList <- b:
			// 将缓冲区放大空闲列表中，不做别的。
		default:
			// 空闲列表已满，保持就好。
		}
	}
}
```
The client attempts to retrieve a buffer from freeList; if none is available, it allocates a fresh one. The server's send to freeList puts b back on the free list unless the list is full, in which case the buffer is dropped on the floor to be reclaimed by the garbage collector. (The default clauses in the select statements execute when no other case is ready, meaning that the selects never block.) This implementation builds a leaky bucket free list in just a few lines, relying on the buffered channel and the garbage collector for bookkeeping.

客户端试图从 freeList 中获取缓冲区；若没有缓冲区可用， 它就将分配一个新的。服务器将 b 放回空闲列表 freeList 中直到列表已满，此时缓冲区将被丢弃，并被垃圾回收器回收。（select 语句中的 default 子句在没有条件符合时执行，这也就意味着 selects 永远不会被阻塞。）依靠带缓冲的信道和垃圾回收器的记录， 我们仅用短短几行代码就构建了一个可能导致缓冲区槽位泄露的空闲列表。
