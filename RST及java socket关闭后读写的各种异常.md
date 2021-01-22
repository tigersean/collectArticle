## 1. RST (Reset)

TCP连接的断开有两种方式：

1. 连接正常关闭时双方会发送FIN，经历4次挥手过程；
2. 通过RST包异常退出，此时会丢弃缓冲区内的数据，也不会对RST响应ACK。

java中，调用`Socket#close()`可以关闭Socket，该方法类似Unix网络编程中的close方法，将Socket的 **读写** 都关闭，已经排队等待发送的数据会被尝试发送，最后（默认）发送FIN。考虑一个典型的网络事务，A向B发送数据，A发送完毕后`close()`，FIN发送出去；B一直read直到返回了-1，也通过`close()`发送FIN，4次挥手，连接关闭，一切都很和谐。

那什么时候会用RST而非FIN关闭连接呢？

1. `Socket#setSoLinger(true,0)`，则close时会发送RST
2. 如果主动关闭方缓冲区还有数据没有被应用层消费掉，close会发送RST并忽略这些数据
3. A向B发送数据，B已经通过`close()`方法关闭了Socket，虽然TCP规定半关闭状态下B仍然可以接收数据，但close动作关闭了该socket上的任何数据操作，如果此时A继续write，B将返回RST，A的该次write无法立即通知应用层（因为write仅把数据写入发送缓冲区），只会把状态保存在tcp协议栈内，下次write时才会抛出`SocketException`。

## 2. 对已关闭socket读写会产生的异常

### 2.1 主动关闭方

`close()`后，无论是发送FIN/RST关闭的，之后再读写均会抛`java.net.SocketException:socket is closed`.

### 2.2 被动关闭方

#### 被FIN关闭

1. 写（即向”已被对方关闭的Socket”写）
   如上所说，第一次write得到RST响应但不抛异常，第二次write抛异常，ubuntu下是`broken pipe (断开的管道)`，win7下是`Software caused connection abort: socket write error`
2. 读 – 始终返回 -1

#### 被RST关闭

读写都会抛出异常：`connection reset (by peer)`

#### 重点在于：

1. `connection reset`：另一端用RST主动关闭连接
2. `broken pipe / Software caused connection abort: socket write error` ： 对方已调用Socket#close()关闭连接，己方还在写数据

java中网络编程时很大一部分代码在做各种fail时的处理，了解各种异常发生时背后的逻辑才能正确地处理之。以上列举的只是连接关闭的异常，还有其他各种异常没有提及，以后有机会再补上。

## 3. 怎么避免意外的RST？

针对几种出现RST的情况：

1. 利用应用层协议定义结构化的数据，双方对何时数据发送/接收完毕/可以安全关闭连接有明确一致的契约；
2. close之前消费掉数据；
3. 需要在半关闭状态下读数据时，使用`shutdownOutput()`，它会发送FIN但依然可以读取数据；等对方发送FIN，`read()`返回-1后再调用`close()`释放socket。

## 参考资料

- [Orderly Versus Abortive Connection Release in Java](http://docs.oracle.com/javase/6/docs/technotes/guides/net/articles/connection_release.html)
- [UNIX网络编程——shutdown 与 close 函数 的区别](http://blog.csdn.net/ctthuangcheng/article/details/9430587)
- [几种TCP连接中出现RST的情况](http://my.oschina.net/costaxu/blog/127394)