  	Golang中存在很多分层啊，果然在计算机里没有什么是是加一层解决不了的，如果有那就加两层。
### Golang 底层的内存模型是借鉴了 Thread-Caching-malloc模型的。
![[Pasted image 20250312130217.png]]
上面这个模型就是 go 锁借鉴的模型，右边是用户的某个程序的协程，左边是虚拟内存。
### Go 内存模型
![[Pasted image 20250312130344.png]]
golang 的内存模型是这样的三层，mcache、mcentral、mheap
当协程需要分配内存的时候，先去对应p的mcache 中获取，若没有，去 mcentral 中获取，如果没有就要去 mheap 中获取，如果 mheap 也没有，就要去和真正的虚拟内存索取。
#### 1. 为什么分层？
因为锁在 golang 中是一个相对来说比较消耗资源的事情，而分配内存的时候一般都需要上锁来保证安全，因此，我们的程序肯定是不希望频繁的上锁来获取内存的，会影响程序的效率。因此，golang 的这个模型，从上到下锁的粒度也是升高的，对应的上锁解锁带来的开销也比较小。当从mcache中能获取到内存的时候，不用上锁，直接分配（mcache 是每个 P 独有的缓存，因此交互无锁）。当 cache 中没有，再去 mcentral 中获取，此时这个锁的粒度也是比较细的，最后再去 mheap 甚至是 virtual memory 中获取。
#### 2. 内存管理的最小单位
Golang 中内存管理的最小单位是 mspan，是全局的内存起源，访问要加全局锁
![[Pasted image 20250312132720.png]]
在 golang 中的 page 是虚拟地址上的 page，而不是虚拟地址的页，这是两个概念。page是 go 中最小的存储单元，大小为 8B。而 mspan 基于这个，从 8B 到 80 KB 被划分为 67 种不同的规格。
#### 3. mcache——mcentral——mheap
mcache：是每个 p 拥有一个，其中将每个等级的 mspan 各存储了一个，由于是 p所持有，不用考虑共享，所以不用锁
mcentral：每个mcentral对应了一种spanclass，每个 mcentral 下聚合了该 spanClass 下的 mspan，每个mcentral对应一把锁
mheap：对于 golang 上层程序而言，mheap 是 os 虚拟内存的抽象，其负责将golang 中的页组装成mspan。是 mcentral 的持有者，持有所有 spanClass 下的 mcentral，作为自身的缓存。当其内存不够时，向操作系统申请，申请单位为 heapArena（64M）
# Golang-内存逃逸
golang 内存逃逸分析是指，golang 编译器会分析对象的生命周期，对于生命周期短的对象会分配在方法栈上，比如一些在方法结束后就可以回收的对象，对于一些生命周期比较长的对象，会分配在堆上。在 GC 的时候以栈的单位来进行回收。