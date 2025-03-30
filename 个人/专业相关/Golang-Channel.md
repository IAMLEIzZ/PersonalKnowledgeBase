## 基于通讯实现共享内存，而不是基于共享内存实现通讯 
### 声明的不同
```
ch1 := make(chan int, 10) // 有缓冲区的 channel
ch2 := make(chan int) // 无缓冲区的 channel
var ch3 chan // 非初始化 channel，实际上是一个 nil
```
### 数据结构
一个 chan 是需要有类型的，chan 的数据结构里就是有这个 chan 对应的数据类型 **elemtype**，同时也会有这个类型的默认大小 **len**，这样才能算出初始化时需要为其开辟出多大的空间；
有缓冲区的 chan 还要有对应的 **buf**，这个 buf 在 chan 中是一个大小为 len 的环形队列，有环形队列就要维护环形队列的收尾指针 **head 和 tail**；
接着，因为当 go-routine 去读写一个 chan 时，这些 go-routine 可能会陷入阻塞，因此，需要有一个被当前 ch 所阻塞的写协程队列和读协程队列 **recvRoutineList 和 sendRotineList**，以便唤醒。
因为 chan 中的元素实际上属于临界元素，因为需要有锁 **lock** 来控制 chan 的读写。
因为需要判断当前 chan 是否被关闭，还需要一个 **close** 标识符来标记一个 chan 的关闭状态。
以上就是 chan 的数据结构的主要内容
```
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	timer    *timer // timer feeding this chan
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	lock mutex
}
```
### 构造函数
当构造有缓冲区的chan时，构造器会通过直接计算缓冲区大小和类型元素，malloc 开辟一个完整的内存空间（逻辑）（chan 结构体 + 缓冲区），以减少内存碎片
```
case !elem.Pointers():
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
```
如果是无缓冲区的chan，构造器仅仅 malloc 一个 chan 结构体的空间
```
case mem == 0:.
    c = (*hchan)(mallocgc(hchanSize, nil, true))
    c.buf = c.raceaddr()
```
如果是一个 chan 指针，hchan 结构体和 buf 需要分别分配（不重要）
```
default:
    // Elements contain pointers.
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
```
### 写流程
1. 如果写的是一个 chan 指针，则直接 panic
2. 如果写的 chan 已被关闭，则直接 panic
通过上述两道检查后，写 chan 的 routine 获得锁，此时检查 chan 的读队列，如果队列中有读阻塞协程，则唤醒，并将写的值传递给他；如果没有读阻塞协程，检查当前 buf 是否已满，如果未满，则将数据写入 buf 中，如果满，则进入写阻塞状态，加入到 chan 的写阻塞队列。最后解锁。
![[Pasted image 20250310103325.png]]
### 读流程
1. 如果读的 chan 是一个 nil，则直接 panic
通过检查，先获得锁如果读的 chan 已经 close，先判断 buf 中是否有数据，如果有，则取出值，如果没有，则返回这个 chan 的 elemType 对应的零值。如果没有 close，判断缓冲区是不是满了，如果满了，则取出一个元素并且唤醒一个写协程，否则直接取出即可。如果缓冲区空了，则进入阻塞队列
![[Pasted image 20250310104040.png]]
### 非阻塞模式
#### 注意哦，如果某个协程是通过 select 去进行读写 chan 的，那么这个chan会被标记为非阻塞模式
举例
```
package main
import "fmt"

func main() {
    ch := make(chan int)
    go func() {
        fmt.Println("开始发送数据...")
        ch <- 10 
        fmt.Println("发送成功")
    }()
    select {
	case ch <- 10:
	    fmt.Println("成功发送 10")
	default:
	    fmt.Println("通道已满或无人接收，放弃发送")
}
```
	1. 无非阻塞模式的情况下：
	• 当 ch <- 10 时，**如果 ch 是无缓冲的，并且没有协程在接收数据**，那么 ch <- 10 会导致当前 goroutine 阻塞。
	• 如果 main 只有一个 goroutine，并且这个 goroutine 被 ch <- 10 阻塞，那么 main 进程就无法继续执行，可能会导致程序死锁（如果没有其他地方读取 ch）。
	2. 有非阻塞模式（select + default）的情况下：
	• select 语句会尝试执行 case ch <- 10，但如果 ch 无法立即接受数据，它不会阻塞，而是执行 default 分支，继续向下执行代码。
	• 这就避免了 goroutine 进入阻塞状态，从而确保 main 不会卡死。
	3. select 不会让 main 进入真正的阻塞状态：
	• select 的作用是尝试多路监听多个 channel，而不是强制等待。
	• 如果 select 没有可以立即执行的 case，那么 default 分支会执行，保证 main 不被阻塞。
	• 如果没有 default 分支，select 会阻塞，直到其中一个 case 可以执行。
在上述例子中，如果 golang 没有非阻塞模式，那么当执行协程中的 `ch <- 10 ` 时，整个 main 会陷入阻塞，因为 ch 是无缓冲 chan，需要等协程来读，但是如果有了非阻塞模式，则在 select 中，ch 并不会导致整个 main 被加入到阻塞队列，而会执行 default，实现多路监听。换言之，select 不会导致调用 select 的主协程进入真正的阻塞态（因为阻塞了就无法监听，select 要实现同时多路监听，**这是 Golang 语法设计层面的东西，不要钻牛角尖**），而是主协程一直监听每一个 ch。
### 关闭
1. 如果 chan 是一个 nil，则直接 panic
加锁，如果 chan 已经被关闭，则直接 panic。将所有读阻塞队列中的协程加入 glist，将所有写阻塞队列中的协程加入 glist。解锁，最后统一唤醒 glist 中的协程。
![[Pasted image 20250310105812.png]]
###  高并发Map实现
	场景：
		1. 原生的 map 并不支持并发 changj
		2. 要求 Get 的时候，如果 map 中存在 k-v，则直接返回，如果不存在 k-v，则需要一直阻塞，直到 k-v 被存入或者等待时间超时
		3. put 的时候，直接存入即可
```Go
type Mymap struct {
	// 既然要并发，先来个锁
	sync.Mutex
	// 存储数据的 map
	kvmap map[int][int]
	// 存储 k 对应的 chan
	chanmap map[int]chan struct{}
}

func NewMymap() *Mymap {
	return &Mymap{
		kvmap: make(map[ing][int]),
		chanmap: make(map[int]chan struct{}),
	}
}

func (m *Mymap) Get (k, outtime int) (val int, err error) {
	// 获取数据的时候
	m.Lock()
	// 检查是否存在 k
	val, ok := m.kvmap[k]
	if ok {
		// 存在 k
		m.Unlock()
		return val, nil
	}
	// 不存在 key，需要阻塞
	ch, ok := m.chanmap[k]
	if !ok {
		ch = make(chan struct{})
		m.chanmap[k] = ch
	}
	// 先解锁防止死锁
	m.unlock()
	// 阻塞
	select {
	case: <-ch:
		m.lock()
		v = m.kvmap[k]
		m.unlock()
		return v, nil
	case: time.After(outtime * time.Second):
		// 超时返回错误
		return -1, Error.new("Get超时")
	}
}

func (m *Mymap) Put(k, v int) {
	// 先获得锁
	m.Lock()
	defer m.Unlock()
	m.kvmap[k] = v
	// 这个时候需要帮忙唤醒阻塞的 Get 协程
	ch, ok := m.chanmap[k]
	if !ok {
		// 如果不存在被当前 k 阻塞的协程，直接返回
		return
	}
	// if 存在，唤醒阻塞的 Get，因为 ch 关闭后，所有被阻塞的 read 和 write 都会被唤醒
	close(ch)
	delete(m.chanmap, k)
	// 或者这样写
	// select {
	// case <-ch:
		// 如果能读到，则代表已经关闭，因为读一个关闭的 chan，是不会阻塞的
	// 	return
	// default:
	//	 close(ch)
	// }
}
```