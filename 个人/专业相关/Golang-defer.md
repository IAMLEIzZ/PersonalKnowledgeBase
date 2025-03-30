### 什么是闭包？
一个函数绑定了一个变量，只要这个函数还能使用，这个变量就不会被释放
### defer 执行流程
![[Pasted image 20250312125903.png]]
### golang的defer原理
在 goroutine 的数据结构里，有一个_defer指针，当有goroutine调用defer的时候，这个时候会进行注册 defer 操作，然后将 groutine 的_defer 指针指向被注册的 defer 函数的方法栈入口。如果有多个defer操作，如下
```
func main () {
	defer func0()
	defer func1()
	defer func2()
}
```
goroutine 的指针指向 func2，而 defer 其实也是有_defer 指针的，指向他下一个要调用的 defer。也就是说 main 在 走到return 时，先走func2->func1->func0。
### defer 的创建
当编译器遇到 defer 时，会进行 newdefer，然后将 newdefer 放到主协程的_defer 后