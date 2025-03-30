### sync.map 数据结构
既然这是个面向并发的map，那么就需要锁
```
type Map struct {
    mu Mutex    
    read atomic.Value     
    dirty map[any]*entry  
    misses int
}
```
mu 是互斥锁，read 是只读 map，dirty 是全量数据，miss 是访问 read 未命中的次数。
```
type entry struct {    
	p unsafe.Pointer 
}

type readOnly struct {    
	m       map[any]*entry    
	amended bool // true if the dirty map contains some key not in m.
}
```
amended 是指当前 readOnly 与 dirty 是否存在数据不一致性。
当 miss 次数达到一定程度，会直接用 dirty 直接将 readOnly 覆盖一次。
### 三种数据状态
entry 有三种状态：
1. 有数据状态
2. 软删除状态（逻辑上的删除，物理上实际上没有 delete，而且只会存在于 readOnly 中）
3. 硬删除状态（实际上的删除）
### 读
![[Pasted image 20250311113606.png]]
#### 读未命中次数达到阈值会触发覆盖 missLock()，这里的细节要注意，dirty 全量覆盖readOnly 后将 dirty 置 nil
### 写
![[Pasted image 20250311113623.png]]
#### 这里要注意从 read 反同步回 dirty 的细节，要少触发dirtylock()，因为这会将 readOnly 数据一个一个判断（软删除改为硬删除，赢删除不读入）读入 dirty 中，时间复杂度会高
### 删
![[Pasted image 20250311113633.png]]
