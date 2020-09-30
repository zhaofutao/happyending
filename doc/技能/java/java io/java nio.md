# 一、NIO基本概念  
    Java NIO由以下几个核心部分组成：  
  - Channels  
  - Buffers  
  - Selectors  

        传统IO基于字节流和字符流进行操作，而NIO基于Channel和Buffer进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。
        Selector用于监听多个通道的事件（如连接打开、数据到达等），因此单个线程可以监听多个数据通道。
        NIO和IO的第一个最大区别就是，IO是面向流的、NIO是面向缓冲区的。Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方。
        IO的各种流都是阻塞的，这意味着，当一个线程调用read()或write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入，该线程在此期间不能再干任何事情了。NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞。


# 二、Channel
    NIO中的Channel主要实现有：
  - FileChannel，对应File IO

  - DatagramChannel
  - SocketChannel
  - ServerSocketChannel

# 三、Buffer
    NIO中的关键Buffer有：
  - ByteBuffer，对应byte
  - CharBuffer，对应char
  - DoubleBuffer，对应double
  - FloatBuffer，对应float
  - IntBuffer，对应int
  - LongBuffer，对应long
  - ShortBuffer，对应short

# 四、Selector
    Selector运行单线程处理多个Channel。
