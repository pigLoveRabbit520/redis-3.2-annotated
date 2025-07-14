# redis-3.2源码注释
一部分，一部分看呗

# 全局变量
server.h中的server变量
```C
extern struct redisServer server;
```

# sds
```
struct sdshdr {
    int len;       // 当前字符串的长度
    int free;      // 剩余可用空间
    char buf[];    // 实际存储字符串内容的缓冲区
};
```

## 获取socket数据
`void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask)`  
`readQueryFromClient`函数负责读取socket数据，并将其追加到querybuf。  
readQueryFromClient函数中read(fd, c->querybuf+qblen, readlen) 并不能保证一次性读完客户端发送的所有数据。  

以下是原因和机制的详细说明：
1. TCP 数据流的特性 ：
TCP 是面向字节流的协议，数据可能会被分成多个包传输。
即使客户端一次性发送了完整请求，服务器端的 read 调用也可能只读取到部分数据。
2. Redis 的处理机制 ：
Redis 使用事件驱动模型，当客户端有新数据到达时，readQueryFromClient 会被调用。
如果一次 read 没有读完所有数据，剩余的数据会在下一次事件触发时继续读取。
这种机制确保了即使数据分段到达，Redis 也能正确处理。
3. 缓冲区管理和数据完整性 ：
Redis 使用 sds（简单动态字符串）作为查询缓冲区（c->querybuf）。
每次 read 读取的数据会被追加到 c->querybuf 中。
当 processInputBuffer 被调用时，Redis 会解析缓冲区中的数据，检查是否已经接收到完整的请求。如果请求不完整，Redis 会等待下一次事件触发，继续读取数据。
4. 极端情况下的保护 ：
如果客户端发送的数据量过大，超过了 server.client_max_querybuf_len（默认为 1GB），Redis 会关闭该客户端连接，以防止内存耗尽。