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

# 五、Elasticsearch的写入请求
![blockchain](/resource/images/elasticsearch写入流程图.jpg "elasticsearch写入流程图")
    红色：Client Node；  
    绿色：Primary Node；  
    蓝色：Replica Node；  
## 1. Client Node
  - Ingest Pipeline  
        对原始文档做一些处理，如HTML解析、自定义的处理，具体逻辑可以通过插件实现，在Elasticsearch中，Ingest Pipeline会比较消耗CPU等资源，可以设置专门的Ingest Node，专门来处理Ingest Pipeline逻辑。
        如果当前Node不能执行Ingest Pipeline，则会将请求发给另一台可以执行Ingest Pipeline的Node。
  - Auto Create Index  
        判断当前index是否存在，如果不存在，则需要自动创建index，这里需要和master交互，也可以通过配置关闭自动创建Index的功能。
  - Set Routing  
        设置路由条件，如果Request中指定了路由条件，则直接使用Request中的Routing，否则使用Mapping配置的，如果Mapping没有配置，则使用默认的_id字段值；
  - Construct BulkShardRequest  
        由于BulkRequest中会包含多个（index/update/delete）请求，这些请求根据routing可能会落在多个shard上执行，这一步会按shard挑拣single write request，同一个Shard的请求聚集在一起，构建BulkShardRequest，每个BulkShardRequest对应一个Shard。
  - Send Request To Primary  
        将每个BulkShardRequest请求发送给相应的Shard的Primary Node。

## 2. Primary Node
  - Index or Update or Delete  
        循环执行每个Single Write Request,对于每个Request，根据操作类型（CREATE/INDEX/UPDATE/DELETE）选择不同的处理逻辑；
        其中，Create/Index是直接新增Doc，Delete是直接根据_id删除Doc，Update比较复杂；
  - Translate Update to Index or Delete   
        这一步是Update特有的步骤，在这里，会将Update请求转换为Index或者Delete请求。首先，会通过GetRequest查询到已经存在的相同_id Doc的完整字段和值（依赖_source），然后和请求中的Doc合并。同时，这里会获取到Doc版本号，记做v1；
  - parse Doc  
        解析Doc各个字段，生成ParsedDocument对象，同时生成uid Term。在Elasticsearch中，_uid = type # _id，对用户_id可见，而Elasticsearch中存的是_uid，这一部分生成的ParsedDocument中也有Elasticsearch系统字段，大部分会根据当前内容填充，部分未知的会在后面继续填充。
  - Update Mapping  
        Elasticsearch有自动更新Mapping的功能，如果开启，会挑选出mapping中未包含的新Field，然后更新mapping。
  - Get Sequence Id and Version  
        由于当前是Primary Shard，则会从SequenceNumber Service获取一个sequenceId和Version，SequenceId在Shard级别每次递增1，SequenceId在写入Doc成功后，会用来初始化LocalCheckPoint，Version则是根据当前Doc的最大Version增1。
  - Add Doc to Lucene  
        这一步开始的时候会给特定_uid加锁，然后判断该_uid对应的Version是否等于之前的Translate Update toIndex步骤里获取的Version，如果不相等，说明刚才读取Doc后，该Doc发生了变化，出现了版本冲突。需要重新从Transport Update to Index or Delete开始执行。
        如果version相等，则继续执行，如果已经存在同id的Doc，则会调用Lucene的updateDocument(uid, doc)接口，先根据uid删除doc，然后在Index新Doc。如果是首次写入，则直接调用lucenee的addDocument接口完成doc的index。
  - Write to TransLog  
        写完Lucene的Segment后，会以k-v的形式写TransLog，key是_id，value是doc内容。当查询的时候，如果请求是getDocbyId，则可以直接根据_id从transLog中读取到，满足NOSQL的实时性要求。
        需要注意的是，这里只是写入到内存的TransLog，是否sync到磁盘的逻辑还在后面。
  - Renew Bulk Request  
        这里会重新构造Bulk Request，原因是之前已经将UpdateRequest翻译成了Index或Delete请求，则后续所有的Replica中只需要执行Index或Delete就可以了，不需要再执行Update逻辑。
  - Flush Translog  
        这里会根据TransLog的策略，选择不同的执行方式，要么是立即Flush到磁盘，要么是等到以后再flush，flush的频率越高，可靠性越高，对写入性能影响越大。
  - Send Requests to Replicas  
        将刚才构建的新的Bulk Request并行发送给多个Replica，然后等待Replica返回，这里需要等待所有的Replica返回后，Primary Node才会返回用户。如果某个Replica失败了，则Primary会给Master发送一个Remove Shard请求，要求Master将该Replica Shard从节点中移除。
        这里，同时会将SequenceId、Primary Term、GlobalCheckPoint等传递给Replica。
  - Review Response From Replicas  
        Replica中请求都处理完后，会更新Primary Node的LocalCheckPoint。

## 3. Replica Node
  - Index Or Delete  
        根据请求类型是Index还是Delete，选择不同的执行逻辑。
  - Parse Doc  
        跟Primary Node逻辑一致；
  - Update Mapping  
        跟Primary Node逻辑一致；
  - Get Sequence Id and Version  
        Primary Node会生成Sequ和Version，然后放入ReplicaRequest中，这里只需要取出来就行。
  - Add Doc to Lucene  
        根据请求类型是Index还是Delete，选择不同的执行逻辑。
  - Write Translog  
        跟Primary Node逻辑一致；
  - Flush Translog  
        跟Primary Node逻辑一致；