# 一、基本概念
    Java IO：即Java输入、输出系统；
    Stream：Java中数据的输入输出抽象成流，流是一组有顺序的、单向的、有起点和终点的数据集合，就像水流。按照流中的最小数据单元又分为字节流和字符流。
    字节流：以8bit作为一个数据单位，数据流中最小的数据单元是字节。
    字符流：以16bit作为一个数据单元，数据流中最小的数据单元是字符。

# 二、IO分类
![blockchain](/resource/images/io流.jpg "IO流")
    Java的IO主要包含两部分：
  - 流式部分：IO的主体部分，流式部分根据流向分为输入流（InputStrem/Reader）和输出流（OutputStream/Writer），根据数据不同的操作单元，分为字节流（InputStream/OutputStream）和字符流（Reader/Writer）。
      下面是Java IO体系中常用的流类：
![blockchain](/resource/images/常用的io流.jpg)
  - 非流式部分：主要包含一些辅助流式部分的类，如：SerializablePermission类、File类、RandomAccessFile类和FileDescriptor类等。
