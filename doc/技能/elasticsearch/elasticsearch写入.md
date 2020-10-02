# 一、搜索系统写操作特点
  - 实时性  
        搜索系统的Index一般都是NRT(Near Real Time)，近实时的，比如Elasticsearch中，Index的实时性是由refresh控制的，默认是1s，最快可到100ms。
  - 可靠性  
        搜索系统对可靠性要求都不高，一般数据的可靠性通过将原始数据存储在另一个存储系统来保证，当搜索系统的数据发生丢失时，再从其他存储系统导一份数据来重新rebuild就可以了。在Elasticsearch通过设置TransLog的Flush频率可以控制可靠性，要么是按要求，每次都flush，要么是按时间，每隔一段时间flush。一般为了性能考虑，会设置为每隔5s或者1分钟flush一次，flush间隔时间越长，可靠性就越低。

# 二、Lucene的写
    Lucene的写操作主要通过IndexWriter类实现，主要有三个接口：
```
public long addDocument();
public long updateDocuments();
public long deleteDocuments();
```
    通过这三个接口可以完成单个文档的写入、更新和删除功能，包括了分词、倒排创建、正排创建等所有搜索相关的流程。
    Lucene的写具有以下问题：
  - 文档写入Lucene后不是立刻可查询的，需要生成完整的Segment后才能被搜索；
  - Lucene不支持部分文档更新；

# 三、Elasticsearch的写
        Elasticsearch采用多Shard方式，通过配置routing规则将数据分成多个数据子集，每个数据子集提供独立的索引和搜索功能。
        每次写入请求的时候，写入请求都会先根据routing规则选择发送给哪个Sharding，Index Request可以设置使用哪个Field的值作为路由参数，如果没有设置，则使用Mapping的设置，如果mapping也没有设置，则使用_id作为路由参数。然后通过_routing的Hash值选择出Shard，最后从集群的Meta中找出该Shard的Primary。
        请求接着会发送给Primary Shard，在Primary Shard上执行成功后，再从Primary Shard上将请求同时发送给多个Replica Shard，请求在多个Replica Shard上执行成功并返回给Primary Shard后，写入请求执行成功，返回结果给客户端。
        这种模式下，写入操作的延时就等于latency = Latency(Primary Write) + Max(Latency(Replica Write))；只要有副本在，写入延迟做小也是两次单Shard的写入延时综合，写入效率会降低，但可以避免写入失败。
        采用多个副本后，避免了单机或磁盘发生故障时，对已经持久化后的数据造成损害。但是Elasticsearch里为了减少磁盘IO保证读写性能，一般是每隔一段时间（比如5分钟）才会把Lucene的Segment写入磁盘持久化，对于写入内存，但还未flush到磁盘的lucene数据，如果发生机器宕机或者断电，那么内存的数据也会丢失。对于这个问题，Elasticsearch采用了TransLog。
![blockchain](/resource/images/elasticsearch%20transport.jpg "Elasticsearch TransLog")  
        在每个Shard中，写入流程分为两部分，先写入Lucene，再写入TransLog。

# 四、Elasticsearch的更新
![blockchain](/resource/images/elasticsearch%20update.jpg)  
        lucene中不支持部分字段的update，所以需要在Elasticsearch中实现该功能，流程如下：
  - 收到update请求后，从segment或translog中读取同id的完整doc，记录版本好为v1；
  - 将版本v1的全量doc和请求中的部分字段doc合并为一个完整的doc，同时更新内存中的versionMap，获取得到完整doc后，update请求就变成了index请求；
  - 加锁；
  - 再次从version中读取该id的最大版本号v2，如果versionMap中没有，则从segment或者translog中读取；
  - 检查版本是否冲突，如果冲突，则回退到开始的update doc阶段，重新执行；如果不冲突，则执行最新的add请求。
  - 在index doc阶段，首先将version + 1得到v3,再讲doc加入到lucene中，lucene中会先删同id下的已存在的doc id，然后再增加新doc，写入lucene成功后，将当前v3更新到versionMap中；
  - 释放锁；