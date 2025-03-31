
| 对象\数据结构 |           |           |
| ------- | --------- | --------- |
| String  | SDS       |           |
| List    | QuickList |           |
| Hash    | Listpack  | HashTable |
| Set     | HashTable | Intset    |
| Zset    | Listpack  | SkipList  |

---
### String
	SDS（simple dynamic string），不要 '\0' 的字符串
```
// 数据结构
type struct SDS {
	len;
	alloc;
	flags;
	buf[];
}
```
len 记录字符串的长度，alloc 是分配给字符数组的空间大小，flags 标识不同类型的 SDS，buf 是字符数组，用来存储实际的数据。
#### 对比C 语言自己的字符串有什么优势？
1. O(1) 时间复杂度获得字符串的长度
2. 二进制安全
3. 不会发生缓冲区溢出，可以通过 alloc 和 len 计算出剩余缓冲区的大小
4. 通过多个 SDS 类型，灵活保存大小不同的字符串，从而节省内存空间
---
### List
	QuickList 简单来说就是 双向链表 + 压缩列表
在最早的时候 List 是用双向链表实现的，后来变成了双向链表 + 压缩列表
#### 压缩列表
内存紧密型的数据结构，占用一块连续的内存空间，适合存储少量的元素
![[Pasted image 20250330142101.png]]
zlbytes：记录整个压缩列表占用的内存字节数
zltail：记录列表尾部的偏移量（距离起始地址
zllen：记录节点数量
zlend：标记压缩列表的结束的地址
![[Pasted image 20250330142309.png]]
prevlen：记录前一个节点的长度（前一个节点小于254 字节，prevlen用 1 个字节空间，否则用 5 字节）
encoding：记录当前节点的数据类型和长度
data：记录数据
因为 prevlen 的分情况存储前一个节点大小的特性，因此在修改或者添加存在连锁更新的问题。
#### 双向链表 + 压缩列表
![[Pasted image 20250330143318.png]]
当添加一个节点的时候，不会直接 new 一个链表节点，而是先 check 插入位置的压缩列表是都超出限定值，如果超出，则创建新节点，否则原地插入。这样维护了压缩列表的规模，限制了连锁更新的影响。
#### 对比普通的双向链表有什么优势？
1. 普通的双向链表内存非连续，在 CPU 访存的时候，无法很好的利用 Cache
2. 普通双向链表每一个节点值都分配一个链表节点，内存利用率不高
#### 缺点
1. 没有完全解决连续更新的问题，所以在后来引入了 Listpack，彻底删除了 prevlen 字段，从而解决了连续跟新的问题
#### 为什么压缩列表要存储上一个元素的长度？
答：1. 为了支持反向遍历。2. 因为内存布局的连续性，没有指针的开销，所以需要一个字段来存储长度
#### Listpack
![[Pasted image 20250330144857.png]]
encoding：对应数据的类型
data：实际的数据
len：encoding字段 + data 字段的总长度

---
### Hash
	Redis 里的 HashTable 很像 golang 中的

![[Pasted image 20250330150454.png]]
**链式 Hash 解决冲突，增量式扩容解决性能劣化，渐进式 rehash 解决性能抖动**

---
### Set
	HashTable + 整数集合
#### 整数集合
```
typedef struct intset {
	uint32_t encoding;
	uint32_t length;
	int8_t contents[];
} intset;
```
encoding：编码方式
length：集合包含元素的数量
contents：保存元素的数组
contents 虽然被声明为 int8_t，但是 contents 数组的真正类型取决于encoding 的值
encoding 有16，32，64 三个规模
#### 整数集合升级
假设现在在一个全是 16 规模的 intset 中插入一个 32 的数字，则会进行整数集合升级，会将里面所有的数的规格都扩大为 32，然后分配多出来的空间，然后将每个数都进行规格升级，最后插入元素。
#### 优势：
1. 按需分配，提高内存的利用率
2. 自动升级
3. 查询快
#### 劣势
1. 数据量大的时候升级会导致性能抖动
**因为其升级的特性，为了避免发生性能抖动，所以当数据量过大的时候会升级为 HashTable**

---
### Zset
	log(N)的跳表，这里的跳表使用多层链表实现的
[[跳表]]
