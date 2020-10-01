# 一、Lucene的数据模型
    Lucene中包含了四种基本数据类型，分别是:
  - Index：索引，由许多的Document组成；
  - Document：由很多的Field组成，是Index和Search的最小单位；
  - Field：由许多的Term组成，包含Field Name和Field Value；
  - Term：由很多的字节组成，可以分词。  
    Lucene中存储的索引主要有三种类型：
  - Invert Index：倒排索引，简称索引，通过Term可以查询到拥有该Term的文档，可以配置是否分词，并选择不同的分词器。
  - DocValues：正排索引，通过DocID可以快速读取到该Doc的特定字段的值。采用列式列式存储，性能比较好，一般用于sort、agg等需要高频读取Doc字段值的场景；
  - Store：字段原始内容存储，同一篇文章的多个Field的Store会存储在一起，适用于一次读取少量且多个字段内存的场景，比如摘要等。
  
          Lucene中提供Index和Search的最小组织形式是Segment，Segment中按照索引类型不同，分成了Invert Index、DocValues、Store三大类，每一类里面都是按照Doc为最小单位存储。
          Invert Index中存储的key是Term、Value是DocId的链表；
          Doc Value中key是Doc Id和Field Name， value是Field Value；
          Store的key是Doc Id， Value是Field Name 和Field Value。

          Lucene中没有主键概念和更新逻辑，所有对Lucene的更新都是append一个新的doc，类似于一个只能append的队列。
    
    Lucene的不足：
  - Lucene是一个单机搜索库，不支持分布式海量数据；
  - Lucene没有更新，每次都是append一个新的文档；
  - Lucene没有主键索引，不支持同一个Doc的多次写入；
  - Lucene中生成完整Segment后，该Segment就不能再被更改，此时Segment才能被搜索，无法支持实时搜索；