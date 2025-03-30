**如何避免 Raft 读脑裂？**
为了防止 Raft 旧 Leader 提供过时数据，常见的解决方案包括：
1. **Leader Lease（Leader 租约）**：
• Raft 允许 Leader 在一个超时时间内假定自己仍然是 Leader，并在此期间提供读服务。
• 只要 Lease 过期，Leader 就不能再提供读操作。
• 但如果网络分区发生，旧 Leader 可能无法检测到自己的 Lease 是否仍然有效，因此仍然可能出问题。
2. **强制 Read Index 机制（Linearizable Read）**：
• 在处理读请求之前，Leader 需要和大多数节点确认自己仍然是 Leader。
• 具体方法：Leader 发送 heartbeat 给大多数 Follower，确认自己仍然是有效 Leader，然后才允许处理读请求。
• 这样，即使网络分区发生，旧 Leader 也无法独自返回数据，从而避免脑裂。
3. **所有读写都走日志提交（Read Through Write Log）**：
• 让所有读操作也经过 Raft 日志提交，即让 Leader 把 **读请求** 也当作一个日志条目提交。
• 这样确保所有读操作都严格按照 Raft 的一致性规则执行，但会有较高的性能开销。