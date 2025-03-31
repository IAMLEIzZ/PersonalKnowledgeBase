	链式调用，怎么退出？
#### cancel
如果父结点是 cancel 类型，则将父结点创建的子节点加入到自己的 children 队列中，如果父结点不是 cancel 类型，则子节点开一个守护 go 程监听父结点的 done。当父结点 cancel 时，会关闭 done，然后会 check children ，一个一个 cancel
#### timeContext
可以手动cancel