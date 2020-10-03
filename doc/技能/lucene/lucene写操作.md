# 一、IndexWriter
```
// initialization
Directory index = new NIOFSDirectory(Paths.get("/index"));
IndexWriterConfig config = new IndexWriterConfig();
IndexWriter writer = new IndexWriter(index, config);
// create a document
Document doc = new Document();
doc.add(new TextField("title", "Lucene - IndexWriter", Field.Store.YES));
doc.add(new StringField("content", "招人，求私信", Field.Store.YES));
// index the document
writer.addDocument(doc);
writer.commit();
```
    IndexWriter写数据，主要分三个步骤：
1. 初始化，初始化IndexWriter必要的两个元素是Directory和IndexWriterConfig，Directory是Lucene中数据持久层的抽象接口，通过这层接口可以实现很多不同类型的数据持久层，例如本地文件系统、网络文件系统、数据库或者是分布式文件系统。IndexWriterConfig内提供了很多可配置的高级参数，提供给高级玩家进行性能调优和功能定制。
2. 构造文档，Lucene中文档由Document表示，Document由Field构成。Lucene提供多种不同类型的Field，其FieldType决定了它所支持的索引模式，也支持自定义Field。
3. 写入文档，通过IndexWriter的addDocument函数写入文档，写入同时根据FieldType创建不同的索引。文档写入完成后，还不可被搜索，最后需要调用IndexWriter的commit，在commit完后Lucene才保住文档被持久化并且是searchable的。

# 二、IndexWriterConfig
    IndexWriterConfig提供了一些供高级玩家做性能调优和功能定制的核心参数，如：
  - IndexDeletionPolicy：Lucene开放对commit point的管理，通过对commit point的管理可以实现例如snapshot等功能。
  - Similarity：搜索的核心是相关性，Similarity是相关性算法的抽象接口，Lucene默认实现了TF-IDF和BM25算法。相关性计算在数据写入和搜索时都会发生，数据写入时的相关性计算成为Index-time boosting，计算Normalization并写入索引，搜索时的相关性计算成为query-time boosting。
  - MergePolicy：Lucene内部数据写入会产生很多Segment，查询时会对多个Segment查询并合并结果。所以Segment的数量一定程度上会影响查询的效率，所以需要对Segment进行合并，合并的过程就成为Merge，而何时出发Merge由MergePolicy决定。
  - MergeScheduler：当MergePolicy触发Merge后，执行Merge会由MergeScheduler来管理。Merge统筹是比较耗CPU和IO的过程，MergeScheduler提供了对Merge过程定制管理的能力。
  - Codec：Codec是Lucene中最核心的部分，定义了Lucene内部所有类型索引的Encoder和Decoder。Lucene在Config这一层将Codec配置话，主要目的是提供不同版本数据的处理能力。
  - IndexerThreadPool：管理IndexWriter内部索引线程（DocumentsWriterPerThread）池。
  - FlushPolicy：FlushPolicy决定了In-memory buffer何时被flush，默认的实现会根据RAM大小和文档个数来判断flush的时机，FlushPolicy会在每次文档add/update/delete时调用判定。
  - MaxBufferedDoc：Lucene提供的默认FlushPolicy的实现FlushRamOrCountsPolicy中允许DocumentsWriterPerThread使用的最大内存上线，超过则触发flush。
  - RAMPerThreadHardLimitMB：除了FlushPolicy能决定flush外，Lucene还会有一个指标强制限制DocumentsWriterPerThread占用的内存大小，当超过阈值则强制flush。
  - Analyzer：分词器；
  
# 三、IndexWriter核心接口
  - addDocument  
    新增一个文档，Lucene内部没有主键索引，所有新增文档都会被认为一个新的文档，分配一个独立的DocId；
  - updateDocuments  
    更新文档，Lucene的更新是查询后删除再新增，流程是先delete by term，后add document。
  - deleteDocument  
    删除文档，支持两种删除：by term和by query。
  - flush  
    触发强制fulsh，将所有Thread的In-momory buffer flush成segment文件，这个动作可以清理内存，强制对数据持久化。
  - prepareCommit/commit/rollback  
    commit后数据才可内搜索，commit是一个二阶段才做,prepareCommit是二阶段操作的第一个阶段，也可以通过调用commit一步完成，rollback提供了回滚到last commit的操作；
  - maybeMerge/forceMerge  
    maybeMerge触发一次MergePolicy判定，而forceMerge触发一次强制merge。

# 四、IndexWriter流程架构
![blockchain](/resource/images/IndexWriter流程图.png) 

## 1. 并发模型
        IndexWriter提供的核心接口都是线程安全的，并且内部做了特殊的并发优化来优化线程写入的性能。IndexWriter内部为每个线程都会单独开辟一个空间来写入，这块空间由DocumentsWritePerThread来控制。整个多线程数据处理流程为：  
        1. 多线程并发调用IndexWriter的写接口，在IndexWriter内部具体请求会由DocumentWriter来执行。DocumentWriter内部在处理请求之前，会先根据当前执行操作的Thread来分配DocumentsWriterPerThread。
        2. 每个线程在其独立的DocumentsWriterPerThread空间内部进行数据处理，包括分词、相关性计算、索引构建等。
        3. 数据处理完毕后，在DocumentsWriterPerThread层执行一些后续操作，例如触发FlushPolicy的判定等。
        引入DocumentsWriterPerThread后，Lucene内部在处理数据时，整个处理步骤只需要对以上第一步和第三步加锁，第二步完全不用加锁，每个线程都在自己独立的空间内处理数据。而通常来说，第一步和第三步都是非常轻量级的，而第二步是对计算和内存资源消耗最大的。这样做之后，能够将加锁的时间大大缩短，提高并发的效率。每个DWPT内单独包含一个In-memory buffer，这个buffer最终会flush成不同的独立的segment文件。

## 2. add/update
        add接口用于新增文档，update接口用于更新文档，但lucene的update是查询后删除再新增，不支持更新文档内部分列。流程是先delete by term,后add document。
        IndexWriter提高的add和update接口，都会映射到DocumentsWriter的update接口。
```
long updateDocument(final Iterable<? extends IndexableField> doc, final Analyzer analyzer, final Term delTerm) thorws IOException, AbortinException;
```
    这个函数的处理流程是：
    1. 根据Thread分配DWPT
    2. 在DWPT内执行delete
    3. 在DWPT内执行add
    add操作会直接将文档写入DWPT内的In-Memory buffer。

## 3. delete
        delete相对add和update来说，是完全不同的数据路径。而且update和delete虽然内部会执行数据删除，但这两者又是不同的数据路径。文档删除不会直接影响In-memory buffer内的数据，而是会有另外的方式来达到删除的目的。
![blockchain](/resource/images/lucene%20dwpt%20delete%20queue.jpg)  
        在Delete路径上关键的数据结构是Deletion queue，在IndexWriter内部会有一个全局的Deletion queue，被称为Global Deletion Queue。而在每个DWPT内部，还会有一个独立的Deletion Queue，被称为Pending Updates, DWPT Pending Updates会与Global Deletion Queue进行双向同步，因为文档删除是全局范围的，不应该只发生在DWPT范围内。
        Pending Updates内部会按顺序记录每个删除操作，并且标记该删除影响的文档范围，文档影响范围通过记录当前已写入的最大DocId来标记，即表示这个删除动作只删除小于等于该DocId的文档。
        update接口和delete接口都可以进行文档删除，但是有一些差异：
  - update只能进行by term的文档删除，而delete除了by term,还支持by query。
  - update的删除会先作用于DWPT内部，后作用于Global，再由Global同步到其他DWPT。
  - delete的删除会作用在Global级别，后异步同步到DWPT级别。
        update和delete流程上的差异也决定了他们行为上的一些差异，update的删除操作会先发生在DWPT内部，并且是和add同时发生，所以能够保证该DWPT内部的delete和add的原子性，即保证在add之前的所有符合条件的文档一定被删除。
        DWPT Pending Updates里的删除操作什么时候会真正作用于数据呢，在Lucene Segment内部，数据实际上并不会被真正删除。Segment中有一个特殊的文件叫live docs，内部是一个位图的数据结构，记录了这个Segment内部哪些DocId是存活的，哪些DocId是被删除的。所以删除的过程就是构建live docs标记位图的过程，数据实际上不会被真正删除，只是在live docs里会被标记删除。Term删除和Query删除会在不同阶段构建live docs，Term删除要求先根据Term查询出它所关联的所有doc，所以这个会发生在倒排索引构建时。而Query删除要求执行一次完整的查询后才能拿到对应的DocId，所以会发生在Segment被flush完成后，基于flush后的索引文件构建IndexReader后执行搜索才能完成。
        live docs只影响倒排，所以在live docs里被标记删除的文档没有办法通过倒排索引检索出，但是还能够通过doc id查询到store fields。当然文档数据Uzi中是会被真正删除，这个过程会发生在merge时。

## 4. flush
        flush是将DWPT内In-memory buffer里的数据持久化到文件的过程，flush会在每次新增文档后由FlushPolicy判定自动触发，也可以通过IndexWriter的flush接口手动触发。
        每个DWPT会flush成一个segment文件，flush完成后这个segment文件是不可被搜索的，只有在commit之后，所有的commit之前flush的文件才可被搜索。

## 5. commit
        commit时会触发数据的一次强制flush，commit完成后在此之前flush的数据才可被搜索。commit动作会触发生成一个commit point，commit point是一个文件，由IndexDeletionPolicy管理。

## 6. merge
        merge是对segment文件合并的操作，合并的好处是能够提升查询的效率以及回收一些删除的文档。merge会在segment文件flush时触发mergePolicy来判定自动触发，也可以通过IndexWriter进行一次force merge.

# 五、IndexWriter索引流程
![blockchain](/resource/images/IndexWriter索引流程.jpg "IndexWriter索引流程")  
        Lucene内部索引构建最关键的概念是IndexingChain。IndexingChain索引构建的顺序是invert index、store fields、doc values和point values.这些索引类型处理文档后会将索引内容直接写入文件（主要是store field和term vector），而有些索引类型会先将文档内容写入memory buffer，最后在flush的时候再写入文件。能直接写入文件的索引，通常是文档级的索引，索引构建可以文档级的增量构建。而不能写入文件的索引，例如倒排，则必须等Segment内所有文档全部写入完毕后，会先对Term进行一个全排序，之后才能构建索引，所以必须要有一个memory-buffer先缓存所有文档。

