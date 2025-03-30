## Project-2
### P2 整体结构
以下是整个 Project-B 的 Part-A 的整个调用关系，上层引用通过调用 RawNode 的 Ready() 和 Advance() 方法，以实现存取日志、调节节点关系等功能。Raft 则实现了一众消息处理方法，比如说当 Follower 收到选举消息时，进行选举处理的方法。当 Raft 接收到涉及到 Log 存取时，则通过调用其结构体成员 RaftLog 的有关消息存取的方法。Storage 则是对 Log 进行持久化存储的对象。从上往下呈包含关系。
```
strcut RawNode {
	Raft *Raft
	...
}

struct Raft {
	Raftlog *RaftLog
	...
}

struct RaftLog {
	Storage *Storage
	...
}
```
![[Pasted image 20250212103024.png]]

### P2的关键结构体说明
```
type Message struct {
MsgType MessageType
To uint64
From uint64
Term uint64
LogTerm uint64
Index uint64
Entries []*Entry
Commit uint64
Snapshot *Snapshot
Reject bool
...
}
```
这是 Raft 节点之间相互联系所发送的 msg，至关重要
Term 是 msg 发送者的任期。LogTerm 和 Index 以及 Enter 是 Leader 向 Follower 发送日志同步请求时会用到的关键参数。SnapShot 是要发送的快照请求，Commit 是当前节点的最后一条提交的日志的 Index。
### P2 各之间的包含关系
![[Pasted image 20250228170538.png]]
![[Pasted image 20250228171009.png]]
![[Pasted image 20250302144751.png]]
