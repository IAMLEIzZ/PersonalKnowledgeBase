https://www.bilibili.com/video/BV1r54y1f7bU/?spm_id_from=333.337.search-card.all.click&vd_source=ec3e6adc144c0dda701e75fe9d38fa57
讲的不错嘻嘻
内核太遍历，没找到，自己阻塞（内核态）=>网卡收到数据，DMA 写到内存，然后给 CPU 中断，CPU 响应中断，根据 ip + port 找到对应 socket，将数据保存在 socket 的接受队列，唤醒被阻塞的用户程序，然后用户（内核）进程检查 fd 集合，有就绪，然后返回给用户态就绪的数量，然后用户态在遍历去寻找