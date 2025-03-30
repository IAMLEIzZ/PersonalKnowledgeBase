	小而美
### 依旧从示例出发，了解 Gin 在底层做了什么
```
import "github.com/gin-gonic/gin"
func main() {    
	// 创建一个 gin Engine，本质上是一个 http Handler    
	mux := gin.Default()    
	// 注册中间件    
	mux.Use(myMiddleWare)    
	// 注册一个 path 为 /ping 的处理函数   
	mux.POST("/ping", func(c *gin.Context) {        
		c.JSON(http.StatusOK, "pone")    
	})    // 运行 http 服务    
	if err := mux.Run(":8080"); err != nil {        
		panic(err)    
	}
}
```
### 从三个流程（定义、注册和接收客户端请求）考虑，便能理清
##### gin.Default()
这段代码做了什么？
深入底层会发现，这个代码返回了一个 new 了 engine 对象，这个 engine 对象是贯彻整个 gin  服务生命周期始终的。
```
type Engine struct {   
	// 路由组    
	RouterGroup    
	// ...    
	// context 对象池    
	pool  sync.Pool    
	// 方法路由树    
	trees methodTrees    
	// ...
}
```
不难看出，实际上 engine 中包含了这个完整服务的所有路由组，Context 对象池以及所有的方法路由树（按照不同的请求 Get、Post 等，有九种），因此，可以理解为所有的路由相关内容都注册在 engine 中。同时 engine 是一个实现了 net/http 的 Handler interface 的，所以engine 启动的时候，实际上是底层启动了 net/http 的 ListenAndServer，然后ListenAndServer在没有传入自定义控件的情况下会进入 golang 默认的全局控件，但是这里engine实现了Handler interface，于是将 engine 传入。也就是说，当服务开始时，端口在 net/http 中被注册，然后后续 net/http 监听到的该端口请求，都交由 engine 也就是 gin 去操作。
```
type RouterGroup struct {    
	Handlers HandlersChain    
	basePath string    
	engine *Engine    
	root bool
}
```
HandlersChain 是由多个路由处理函数 HandlerFunc 构成的处理函数链. 在使用的时候，会按照索引的先后顺序依次调用 HandlerFunc.
##### RaftGroup
```
package main
import (
	"github.com/gin-gonic/gin"
)
func main() {
	r := gin.Default()
	// 创建一个 "/api" 前缀的路由组
	apiGroup := r.Group("/api")
	{
		apiGroup.GET("/users", func(c *gin.Context) {
			c.JSON(200, gin.H{"message": "Get all users"})
		})
		apiGroup.POST("/users", func(c *gin.Context) {
			c.JSON(201, gin.H{"message": "Create user"})
		})
	}
	// 启动服务器
	r.Run(":8080")
}
```
包含关系是看起来 Raftgroup 包含的对应的 engine，如果设置了组，调用的都是组这个对象
##### 路由注册
```
mux.POST("/ping", func(c *gin.Context) {        
	c.JSON(http.StatusOK, "pone")    
})
```
![[Pasted image 20250314222558.png]]
上图讲解了路由注册流程，当开始注册时，gin 会判断其是属于哪个路由组的，然后将路由组的前缀地址拼接给他，然后将路由组中的中间件传递给他，接在其 HandlersChain 中。然后去找压缩前缀树对应的节点，如果为空，则创建，并且挂载对应的调用链，如果不为空，发生一样的节点挂载不同的调用链，则直接 panic（这一段也就是在路由树中注册对应的节点）。
##### 压缩前缀树增删改查
压缩前缀树是在普通前缀树的基础上，增加了一点
	1.  倘若某个子节点是其父节点的唯一孩子，则与父节点进行合并
golang 在压缩前缀树的基础上进行了小优化，每一个节点存其子节点的数量，子节点数量高的节点会被接到树的靠左边，提高检索效率。（因为是多叉树）
对于 Gin，根据 9 中不同的 http 请求，一共有9 个树。每次请求到达后，会先根据请求定位对应的树，然后在对应的树中进行增删改。
	这里为什么不用 map 呢？
###### 查
查是普通的查。
###### 增
但是不一样的点是增，如果当前增加的节点比如说出现以下情况，根 ABC， 左 ZF，现在要插插入BC，此时会进行分裂。根变成 A，左边变成 BCZF，右边变成 BC（这里还要把分裂后的信息赋值给 BCZF）。如果不是这种情况，那就继续往下检查，如果根 ABC，左ZF，插入D，则直接插在右边。如果根 ABC，左 ZF，插入 A，则分裂为根 A，左 BCZF。
##### Gin.Context
![[Pasted image 20250314224659.png]]
context 贯穿了 Gin 一次请求的生命周期，当他死亡后，不立刻释放资源，而是放入对象池，等待回收利用。
###### 复用策略
![[Pasted image 20250314224757.png]]
http 请求到达时，从 pool 中获取 Context，倘若池子已空，通过 pool.New 方法构造新的 Context 补上空缺。
http 请求处理完成后，将 Context 放回 pool 中，用以后续复用。
###### 调用流程next()
![[Pasted image 20250314224942.png]]
这个可以看我实现的 gee 的 Readme 文件
```
func myHandleFunc(c *gin.Context){    
// 前处理    
	preHandle()      
	c.Next()    
	// 后处理    
	postHandle()}
```

```
func (c *Context) Next() {    
	c.index++    
	for c.index < int8(len(c.handlers)) {        
		c.handlers[c.index](c)        
		c.index++    
	}
}
```

- gin 将 Engine 作为 http.Handler 的实现类进行注入，从而融入 Golang net/http 标准库的框架之内
- gin 中基于 handler 链的方式实现中间件和处理函数的协调使用
- gin 中基于压缩前缀树的方式作为路由树的数据结构，对应于 9 种 http 方法共有 9 棵树
- gin 中基于 gin.Context 作为一次 http 请求贯穿整条 handler chain 的核心数据结构
- gin.Context 是一种会被频繁创建销毁的资源对象，因此使用对象池 sync.Pool 进行缓存复用