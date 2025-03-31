	值传递和引用传递，傻傻搞不清楚？
#### Slice 的定义
```
s1 := make([]int)
s2 := make([]int, x, y)
s3 := make([]int, x)
```
对于上述三种定义，这里先引出 slice_header 的定义
```
// runtime/slice.go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
其实每当 make 的时候，就是创建了一个 slice 结构体，其中 array 是指开辟出的数组空间的头部地址（你知道的，数组在虚拟内存中的地址是连续的），len 是指当前 slice 中的元素数量，cap 是指给这个 slice 的元素容量（cap >= len 哦）。
s1 没有分配空间；s2 分配了 y 的空间，并填充了 x 个默认值（数组定义类型的零值）；s3 cap 和 len 都是 x。
### Slice 引用
其实这里只要搞清楚值传递和引用传递就行。
```
s := make([]int, 5, 10)
s1 := s
```
我们直到每次创建 slice 时都会创建一个 slice-header 结构体。
当我们进行上面的操作的时候，实际上会给 s1 也开辟一个 slice-header 结构体，其中的 len 和 cap 大小和 s 一致，但 array这个指针，是直接指向 s的 array 的。也就是说 s1 和 s 两个 slice 的内部存储指向同一块地址。
那假设，现在给 s1 添加元素，让 s1 变成len = 6，cap不变；那是不是 s[5] 就能访问到 s1 添加的元素了呢？答案是否定的，逻辑上确实是可以，但是每次访问前 s 要判断数组访问下标有没有超过 len，虽然在s1 中 len 是 6，但因为 len 和 cap 实际上只是值一样，地址不同，所以 s 的 len 还是 5，这样依旧越界。但是如果 s1 将 s1[0] = 123，那么 s[0]也会变成 123。
```
s := make([]int, 4, 6)
for i := 0; i < 4; i ++ {
	s[i] = i + 2
}
// s => [2, 3, 4, 5]
s1 := s[5:]
```
上面这种复制，s1 的 array 会指到 s 数组的第五个元素开头，len 为5，cap 为 5
![[Pasted image 20250313155432.png]]
### Slice 扩容
1. 如果需要的容量 > 原来容量 * 2 ==> 直接分配需要的容量
2. 如果不需要，则先检查当前容量是否 < 256，小于直接翻倍
3. 如果当前容量 > 256，则进行这样的操作 oldmem += (oldmem + 3 * 256) / 4
**当分配好内存后，然后将久的内存中的元素复制过去**
也就是说，在分配内存后，slice 的内存地址会发生变化
那么在上面的例子中
```
s := make([]int, 5, 10)
s1 := s
s = append(s, s[:]) 
s = append(s, 10) // 发生扩容
```
此时，s 发生扩容后，他 slice_header 中的 array 发生变化，于是不再和 s1 共享一块内存，次后 s 和 s1 的增删再无关联，s1 依然指向 s 不要的旧内存空间。
分配内存这里还有一个坑，就是假设我现在分配一个 60 的容量，但是由于 golang 的内存管理和内存分配机制，内存是按照mspan 分配的，mspan 是分等级的（每一级大小一般是 2 的 n 次方），所以会向上取整，直接分配一个 64 过去（减少碎片）。如下例：
```
func Test_slice(t *testing.T){    
	s := make([]int,512)      
	s = append(s,1)    
	t.Logf("len of s: %d, cap of s: %d",len(s),cap(s))
}
```
	答案：len: 513, cap: 848
	问题11的内容看起来平平无奇，为什么我会选择将其作为压轴呢？原因在于其中暗藏了两个细节，使得这个问题远没有其表面上看上去的那么简单.

	首先，如 2.7 小节中谈到的，由于切片 s 原有容量为 512，已经超过了阈值 256，因此对其进行扩容操作会采用的计算共识为 512 * (512 + 3*256)/4 = 832

	其次，在真正申请内存空间时，我们会根据切片元素大小乘以容量计算出所需的总空间大小，得出所需的空间为 8byte * 832 = 6656 byte

	再进一步，结合分配内存的 mallocgc 流程，为了更好地进行内存空间对其，golang 允许产生一些有限的内部碎片，对拟申请空间的 object 进行大小补齐，最终 6656 byte 会被补齐到 6784 byte 的这一档次. （内存分配时，对象分档以及与 mspan 映射细节可以参考 golang 标准库 runtime/sizeclasses.go 文件，也可以阅读我的文章了解更多细节——golang 内存模型与分配机制）
### Copy
如果我就要 s1 和 s 不共享一片内存呢？
这里要用到 copy 关键字
```
func Test_slice(t *testing.T) {    
	s := []int{0, 1, 2, 3, 4}    
	s1 := make([]int, len(s))    
	copy(s1, s)    
	t.Logf("s: %v, s1: %v", s, s1)    
	t.Logf("address of s: %p, address of s1: %p", s, s1)
}
```