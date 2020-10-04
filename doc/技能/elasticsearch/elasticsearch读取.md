# 一、Lucene的读
    Elasticsearch使用了Lucene作为搜索引擎库，通过Lucene完成特定字段的搜索等功能，在Lucene中这个功能主要是通过IndexSearcher的下列接口实现的；
```
public TopDocs search(Query query, int n);
public Document doc(int docId);
public int count(Query query);
```
        第一个search接口实现搜索功能，返回满足Query的N个结果；第二个doc接口通过docId查询doc内容；第三个count接口通过Query获取到命中数；
        这三个功能是搜索中最基本的三个功能点，对于大部分Elasticsearch的查询都是比较复杂的，直接用这个接口是无法满足需求的。

# 二、Elasticsearch的读
        Elasticsearch中每个Shard都会有多个Replica，主要是为了保证数据可靠性，除此之外，还可以增加读能力，因为写的时候虽然要写大部分的Replica Shard，但是查询的时候只需要查询Primary或Replica中的任何一个就可以了。
        ElasticSearch中的查询主要分为两类：Get请求：通过ID查询特定Doc；Search请求：通过Query查询匹配Doc。
![blockchain](/resource/images/elasticsearch读流程.jpg "Elasticsearch读流程")  

        对于Search类请求，查询的时候是一起查询内存和磁盘上的Segment，最后将结果合并后返回，这种查询是近实时的，主要是由于内存中的index数据需要一段时间后才会刷新为segment。
        对于Get请求，查询的时候是先查询内存中的TransLog，如果找到就立刻返回，如果没有找到再查询磁盘上的TransLog，如果还没有则再去查询磁盘上的Segment，这种查询是实时的。
        所有的搜索系统一般都是两阶段查询，第一阶段查询到匹配的DocId，第二阶段再查询DocId对应的完整文档，这种在Elasticsearch中成为query_then_fetch，还有一种是一阶段查询的时候就返回完整doc，在Elasticsearch中称作query_and_fetch，一般第二种适用于只需要查询一个Shard的请求。
        除了一阶段、二阶段外，还有一种三阶段查询的情况，搜索里面有一种算分逻辑是根据TF好DF计算基础分，但是Elasticsearch中查询的时候，是在每个Shard中独立查询的，每个Shard中的TF和DF也都是独立的，虽然在写入的时候通过_routing保证doc分布均匀，但是没法保证tf和df均匀，那么就会导致局部的tf和df不准的情况出现。为了解决这个问题，Elasticsearch引入了DFS查询，比如DFS_query_then_fetch，会先收集所有Shard中的TF和DF值，然后将这些值带入请求中，再次执行query_then_fetch，这样算分的时候TF和DF就是准确的。类似的还有DFS_query_and_fetch，这种查询的优势是算分更加精准，但是效率会更差。另一种选择是用bm25代替TF/DF模型。

# 三、Elasticsearch查询流程
![blockchain](/resource/images/elasticsearch查询流程.jpg "Elasticsearch查询流程")  

## 1. Ingest Node  
  - Get Remove Cluster Shard  
    判断是否需要跨集群访问，需要郭旭，则获取要访问的Shard列表；
  - Get Search Shard Iterator  
    获取当前Cluster要访问的Shard，和上一步中的Remove Cluster Shard合并，构建出最终要访问的完整Shard列表；  
    这一步会根据request请求中的参数从Primary Node和多个Replica Node中选择出一个要访问的Shard。
  - For Every Shard: Perform  
    遍历每个Shard，对每个Shard执行后面逻辑。
  - Send Request to Query Shard  
    将查询阶段请求发送给相应的shard。
  - Merge Docs  
    上一步将请求发送给多个Shard后，这一步就是异步等待返回结果，然后对结果合并。这里的合并策略是维护一个topN的优先级队列，每当收到一个shard的返回，就把结果放入优先级队列做一次排序，知道所有的Shard都返回。  
    翻页逻辑也在这里，如果要取top30 ~top40的结果，那么每个Shard需要返回top40的结果，然后在merge docs，计算出top40的结果，最后再除去top30，剩下的即时需要的结果。翻页逻辑有一个明显的缺点就是每次Shard返回的数据中包含了已经翻过的历史结果，如果翻页很深，则在这里需要排序的docs会很多，容易导致OOM。  
    如果有aggregatate，也会在这里做聚合。  
  - Send Request to Fetch Shard  
    选出topN个DocId后发送给这些DocId所在的Shard执行Fetch Phase，最后会返回topN的doc的内容。

## 2. Query Phase  
  - Create Search Context  
    创建Search Context，之后Search过程中所有中间状态都会存在Context中，这些状态总共有50多个。
  - Parse Query  
    接卸Query的Source，将结果存入Search Context，这里会根据请求中Query类型的不同创建不同的Query对象，比如TermQuery、FuzzyQuery等，最终真正执行TermQuery、FuzzyQuery等语义的地方是在Lucene中。  
    这里包含了dfsPhase、queryPhase和fetchPhase三个阶段的preProcess部分，只有queryPhase的preProcess有执行逻辑，其他两个都是空逻辑，执行完preProcess后，所有需要的参数都会设置完成。
  - Get From Cache  
    判断请求是否允许被Cache，如果允许，则检查Cache中是否已经有结果，如果有则直接读取Cache，否则继续后面的步骤，执行完后，再将结果加入Cache。
  - Add Collectors  
    Collector主要目标是收集查询结果，实现排序，对自定义结果集过滤和收集等。这一步会增加多个Collectors，组成一个List：
    - FilteredCollector：先判断请求是否有Post Filter，Post Filter用于Search、Aggregation等结束后再次对结果做Filter，希望Filter不影响Aggregation结果。如果有Post Filter则创建一个FilteredCollector，加入Collector List中。
    - PluginInMultiCollector：判断请求是否制定了自定义的一些Collector，如果有，则创建后加入Collector List。
    - MinimumScoreCollector：判断请求中是否制定了最小分数阈值，如果指定了，则创建MinimumScoreCollector加入Collector List，在后续收集结果时，会过滤掉得分小于最小分数的Doc。
    - EarlyTerminatingCollector：判断请求中是否提前结束Doc的Seek，如果是则创建EarlyTerminatingCollector，加入Collector List中。在后续Seek和收集Doc的过程中，当Seek的Doc数达到Early Terminating后会停止Seek后续倒排链。
    - CancellableCollector：判断当前操作是否可以被中断结束，比如是否已经超时等，如果是会抛出TaskCancelledException异常，该功能一般用来提前结束较长的查询请求，可以用来保护系统。
    - EarlyTerminatingSortingCollector：如果Index是排序的，那么可以提前结束对倒排链的Seek，相当于在一个排序递减链表上返回最大的N值，只需要直接返回前N个值就可以了。
    - TopDocsCollector：最核心的topN结果选择器，会加入到Collector List的头部，TopScoreDocCollector和TopFieldCollector都是TopDocsCollector的子类，TopScoreDocCollector会按照固定的方式算分，排序会按照分数+DocId的方式排列，而TopFieldCollector则是根据用户指定的Field的值排序。
  - Lucene::search  
    这一步会调用Lucene的IndexSearch的search接口，执行真正的搜索逻辑。每个Shard会有多个Segment，每个Segment对应一个LeafReaderContext，这里会遍历每个Segment，到每个Segment中去Search结果，然后计算分数。  
    搜索里面一般有两阶段算分，第一阶段是在这里算的，会对每个Seek到的Doc都计算分数，为了减少CPU消耗，一般是算一个基本分数，这一阶段完成后，会有一个排序。然后在第二阶段，再对top的结果做一次二阶段算分，在二阶段算分的时候会考虑更多的因子。
  - rescore  
    根据Request中是否包含rescore的配置觉得是否进行二阶段算分，如果有则执行二阶段算分逻辑，会考虑更多的算分因子，二阶段算分也是一种计算机中常见的多层设计，是一种资源消耗和效率的折中。  
    Elasticsearch支持配置多个Rescore，这些rescore逻辑会按顺序遍历执行，每个rescore内部会先按照请求参数window选择出top window的doc，然后对这些doc排序，拍完序再合并回原有的top结果顺序中。
  - suggest::execute()  
    如果有推荐请求，这里执行推荐请求。
  - aggregation::execute()  
    如果含有聚合统计请求，则在这里执行。Elasticsearch中的aggregation的处理逻辑也类似于search，通过多个Collector来实现。在Ingest Node也需要对aggregation做合并。
  - 

## 3. Fetch Phase  
        Elasticsearch作为操作系统时，除了Query阶段外，还会有一个Fetch阶段。因为Elasticsearch是分布式的，第一阶段查询的时候并不知道最终结果会在哪个Shard上，所以每个Shard中都需要查询完整结果，然后返回给Ingest Node。如果有100个Shard，取top10就需要返回100 * 10 = 1000个结果，而Fetch Doc内容的操作比较消耗IO和CPU，所以一般是当Ingest Node选择出最终TopN结果后，再对最终的topN读取Doc内容。  
        Fetch截断的目的是通过DocId获取用户需要的完整Doc内容，这些内容包括了DocValues、Store、Source、Script和Highlight等：
  - ExplainFetchSubPhase
  - DocValueFieldsFetchSubPhase
  - ScriptFieldsFetchSubPhase
  - FetchSourceSubPhase
  - VersionFetchSubPhase
  - MatchedQueriesFetchSubPhase
  - HightlightPhase
  - ParentFieldSubFetchPhase  
        除了系统默认的8种外，还有通过插件的形式注册自定义的功能，这些SubPhase中最重要的是Source和Highlight，Source是加载原文，Highlight是计算高亮显示的内容片段。
        上述多个SubPhase会针对每个Doc顺序执行，可能会产生多次的随即IO。