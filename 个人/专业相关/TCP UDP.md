### TCP 和 UDP 的区别
### TCP 三次握手
客户端向送连接请求后，进入 syn_send 状态，当服务器接收到 sync 请求后，向客户端发送 sync_ack 确认，然后服务器进入syn_recv状态，然后客户端收到 sync_ack 后向服务器发送 ack，并开启连接，服务器收到 ack 后也开启链接
##### 为什么是三次握手，而不是两次
假设 A 与 B 现在要建立连接，A 发送一个 syn = x，此时 x 丢失，A 重新发送 syn = x，B 成功收到 syn = x，并且接受，然后发送ack，开始数据传输。注意此时，如果第一个 x 又回来了，B 收到了，然后B 再次发送一个 ack 于是又建立了一个连接。
### TCP 四次挥手
TCP 四次挥手：A 向 B 发送 fin，后开始状态 fin1，然后 B 收到后向 A 发送fin1_ack，a收到后进入状态 fin2等待，等 B 发送fin2 时，A 发送 ack，并且进入 time—wait阶段，time-wait 阶段无事发生则关闭A，当 B 收到 fin2 时，则关闭连接。（这是优雅的关闭）
time-wait 一般设置为 2 * 报文最大生存时间（MSL）

### TCP 重传机制
1. 超时重传
2. 快速重传
3. SACK（选择性重传）
4. Duplicate SACK
### TCP 滑动窗口
ACK 的累计确认机制，当收到 ACK 时，代表这个 ack 之前的数据全被收到
### TCP 的流量控制
在 TCP 头部，有一个 Window Size（窗口大小）字段，表示接收端还能接收多少数据，发送端必须遵守这个值。
### TCP 的拥塞控制
1. 慢启动
   TCP 在刚建立连接完成后，首先是有个慢启动的过程，这个慢启动的意思就是一点一点的提高发送数据包的数量（**当发送方每收到一个 ACK，拥塞窗口 cwnd 的大小就会 * 2）
2. 拥塞避免（当慢启动 cwnd > slow start threshold慢启动门限）
   **当发送方每收到一个 ACK，拥塞窗口 cwnd 的大小就会 + 1
3. 拥塞发生 
	1. **超时丢包（Timeout）** → **重传超时（RTO）**：ssthresh 设为 cwnd 一半，cwnd = 1
	2. **三次重复 ACK（Fast Retransmit）**：cwnd 设为 cwnd 一半，然后进入快速恢复阶段
4. 快速恢复
	1. ssthresh = cwnd
	2. cwnd + 3
	3. 然后转到拥塞避免阶段
   