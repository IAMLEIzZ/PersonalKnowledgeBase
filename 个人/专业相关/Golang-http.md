	原本就支持网络开发的语言，舒服
net/http 的包我选择通过代码执行流程来梳理，而不是直接走定义
#### 示例程序
```
func main() {
	http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		 w.Write([]byte("pong"))
	})

	http.ListenAndServe(":8080", nil)
}
```
当启动这个后，程序会持续监听 8080 端口，接下来我们从 `http.HandleFunc` 和 `http.ListenAndServe` 来进入 net/http。
#### Server
整个 http 库被封装在 Server 对象中，当调用ListenAndServer("port", handler)
![[Pasted image 20250326165728.png]]
当监听到了消息，会转发到指定的 handler，如果没有指定 handler，则会转发到一个全局默认的 handler（ServeMux）。ServeMux是 handler 的一个具体的实现。golang本身的 net/http 库并没有使用前缀树，而是使用 map[string]muxEntry 来实现的。string是对应的 pattern，muxEntry 包含了对应的方法。对于普通路径，使用map[string]muxEntry；对于以 '/' 结尾的 path，根据 path 长度将 muxEntry 有序插入到数组 ServeMux.es 中（**模糊匹配时，会找到一个与请求路径 path 前缀完全匹配且长度最长的 pattern，其对应的handler 会作为本次请求的处理函数**）.
#### 服务端server
##### http.HandleFunc
这个过程即使上是一个注册过程，当服务器执行这个的时候，会在以下这个结构体中注册对应的路径和 handler
```
type ServeMux struct {    
	mu sync.RWMutex    
	m map[string]muxEntry    
	es []muxEntry // slice of entries sorted from longest to shortest.    
	hosts bool // whether any patterns contain hostnames
}
type muxEntry struct {    
	h Handler    
	pattern string 
}
```
其中 m 映射了从 path 到 handler 的映射关系。
![[Pasted image 20250314165157.png]]
当注册服务时，说白了就是将方法和对应的请求路径包装成一个 muxEntry 然后注册到全局的路径映射 map 中。
##### http.ListenAndServe
![[Pasted image 20250314165455.png]]
当启动时，主 go 程会启动监听对应的端口，然后自身陷入 for 循环自旋状态，当有请求来把他唤醒的时候，他会立刻创建一个单独的 go-routine ，让这个 go-routine 去进行路径匹配和处理请求，而自身又会立刻陷入自旋中。