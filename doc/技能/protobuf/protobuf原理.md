# 一、什么是PorotoBuf
        Protocol buffers是一种语言无关、平台无关、可扩展的序列化数据结构的方法，它可用于（数据）通信协议、数据存储等。

# 二、Protobuf优缺点
## 1. 优点
        Protobuf有如XML，不过他更小、更快、也更简单。你可以定义自己的数据结构，然后使用代码生成器生成的代码来读写这个数据结构。你甚至可以在无需重新部署程序的情况下更新数据结构。只需使用Protobuf对数据结构进行一次描述，即可利用各种不同语言或从各种不同数据流中对你的结构化数据轻松读写。
        它有一个非常棒的特性，即向后兼容性好，人们不必破坏已部署的、依靠老数据格式的程序就可以对数据结构进行升级。
        Protobuf语义更清晰，无需类似XML解析器的东西（因为Protobuf编译器会将.proto文件编译成对应的数据访问类以对Protobuf数据进行序列化、反序列化操作）。

## 2. 缺点
        Protobuf可读性不好；

# 三、Protobuf编解码
## 1. Varint
        Varint的每个byte的最高位bit有特殊的含义，如果该位是1，表示后续的byte也是该数字的一部分；如果是0，则结束。
        Varint用没个byte的后7位表示数值。
        小于128的数字都可以用1个byte表示；大于128的数字，会用2个byte表示；
![image](/resource/images/varint编码.jpg "VarInt编码")  

## 2. Message
        消息经过序列化后会成为一个二进制数据流，该流中的数据为一系列的key-value对。
![image](/resource/images/message%20buffer.jpg "Message Buffer")   
        
        采用这种key-value结构无需使用分隔符来分隔不同的Field。  
        对于可选的Field，如果消息中不存在该Field，那么在最终的Message Buffer中就没有该Field。

        Key用来标识具体的Field，在解包的时候，Protobufcol Buffer根据Key就可以知道相应的Value应该对应于消息中的哪一个Field。
        Key的定义如下：
```
    (field_number << 3) | wire_type
```
        key由2部分组成，第一部分是field_number,在定义.proto文件时指定，第二部分为wire_type，表示value的传输类型。
        Wire Type的类型如下：
  - 0：Varint，表示int32、int64、uint32、uint64、sint32、sint64、bool、enum类型
  - 1：64-bit，表示fixed64、sfixed64、double类型
  - 2：Length-delimi，表示string、bytes、embedded messages、packed repeated fields类型
  - 3：Start group，废弃
  - 4：End group，废弃
  - 5：32-bit，表示fixed32、sfixed32、float类型

## 3. ZigZag编码
        使用ZigZag编码，绝对值小的数字，无论正负都可以采用较小的byte表示，充分利用了Varint技术。
        ZigZag编码按照数字的绝对值进行升序排列，将整数通过一个hash函数h(n) = (n << 1)^(n >> 31)(如果是sint64 h(n) = (n << 1) ^ (n >> 63))转换为递增的32位bit流。
        通过ZigZag编码，主要绝对值小的数字，都可以用较少位的byte表示，解决了负数的Varint位数会比较长的问题。
![image](/resource/images/zigzag.jpg "ZigZag编码")  

## 4. 字符串编码
        采用类似数据库中的varchar的表示方法，即用一个varint表示长度，然后将其余部分紧跟着这个长度部分即可。