# 一、IndexSearch基本原理
```
Analyzer analyzer = new StandardAnalyzer();
    // Store the index in memory:
    Directory directory = new RAMDirectory();
    // To store an index on disk, use this instead:
    //Directory directory = FSDirectory.open("/tmp/testindex");
    IndexWriterConfig config = new IndexWriterConfig(analyzer);
    IndexWriter iwriter = new IndexWriter(directory, config);
    Document doc = new Document();
    String text = "This is the text to be indexed.";
    doc.add(new Field("fieldname", text, TextField.TYPE_STORED));
    iwriter.addDocument(doc);
    iwriter.close();

    // Now search the index:
    DirectoryReader ireader = DirectoryReader.open(directory);
    IndexSearcher isearcher = new IndexSearcher(ireader);
    // Parse a simple query that searches for "text":
    QueryParser parser = new QueryParser("fieldname", analyzer);
    Query query = parser.parse("text");
    ScoreDoc[] hits = isearcher.search(query, 1000).scoreDocs;
    //assertEquals(1, hits.length);
    // Iterate through the results:
    for (int i = 0; i < hits.length; i++) {
         Document hitDoc = isearcher.doc(hits[i].doc);
         System.out.println(hitDoc.get("fieldname"));
    }
    ireader.close();
    directory.close();
```
    IndexSearcher类使用的时候，需要一个DirectoryReader和QueryParser，其中DirectoryReader需要对应写入时候的Directory实现。QueryParser主要用来解析查询语句。

# 二、Lucene查询过程
        在Lucene中查询是基于Segment，每个Segment可以看做是一个独立的subindex,在建立索引的过程中，Lucene会不断的flush内存中的数据持久化形成新的Segment，多个Segment也会不断的被Merge成一个大的Segment，在老的Segment还有查询在读取的时候，不会被删除，没有被读取且被merge的Segment会被删除。
## 1. FST（Finite State Transducer，有限状态转移机）
        在Lucene中，如果term非常多，为了快速拿到倒排链，在Lucene中引入了Term Directory的概念，也就是term的词典。从Lucene4开始，为了方便实现RangeQuery或者前缀、后缀等复杂的查询语句，Lucene使用FST数据结构来存储term词典。
        FST最重要的功能是可以实现key到value的映射，相当于hashMap,FST的内存消耗要比hashmap小的多，但fst的查询速度比hashmap要慢。
        FST有两个优点：1、空间占用小，通过对词典中单词前缀和后缀的重复利用，压缩了存储空间；2—、查询速度快，O(len(str))的查询时间复杂度。
## 2. SkipList
        为了能够快速查找DocId，Lucene采用了SkipList这一数据结构，SkipList有以下几个特征：
  - 元素排序的，对应到我们的倒排链，Lucene是按照DocId进行排序，从小到大；
  - 跳跃有一个固定的健哥，这个需要简历SkipList的时候指定好；
  - SkipList的层次，是指整个SkipList有几层；

        有了FST和SkipList后，我们大体可以了解Lucene是如何实现整个倒排结构的：
![blockchain](/resource/images/lucene倒排结构.jpg " Lucene倒排结构")  


# 三、Lucene倒排合并
    假如有三个倒排需要进行合并：
## 1. 在termA开始遍历，得到第一个元素docId = 1
## 2. set currentDocId = 1
## 3. 在termB中search(currentDocId)，返回大于等于currentDocId的一个doc.
### 3.1 如果currentDocId == 1，继续
### 3.2 如果不相等，执行1,继续遍历，然后继续
## 4. 到termC依然符合，返回结果；
## 5. currentDocId = termC的nextItem；
## 6. 继续步骤3，依次循环，知道某个倒排链到末尾；  
        如果某个链很短，会大幅减少比对次数，并且由于SkipList结构的存在，在某个倒排中定位某个DocId的速度比较快不需要一个个遍历，可以很快的返回最终的结果。

# 四、BKDTree
        在Lucene中如果想做范围查找，根据FST模型来看，需要遍历FST找到包含这个range的一个点然后进入对应的倒排链，但是如果是数值类型，那么潜在的term可能会非常多，这样查询起来效率会很低。
        为了支持高效的数值类或者多维度查询，Lucene引入类BKDTree，BKDTree是基于KDTree，对数据进行按照维度划分简历一颗二叉树确保树两边的节点数目平衡。在一维的场景下，BKDTree会退化成一个二叉搜索树，在二叉搜索树中如果我们想查找一个区间，logN的复杂度就会访问到叶子结点得到对应的倒排链，如果是多维，KDTree的建立流程会发生一些变化：
## 1. 确定切分维度
        这里维度的选取顺序是数据在这个维度方法最大的维度有限，一个直接的理解就是，数据分散越开的维度，我们优先切分。
## 2. 确定切分点
        切分点选择这个维度最中间的点；
## 3. 递归进行步骤1、2
        我们设置一个阈值，点的数量少于多少后就不再切分，直到所有的点都切分好停止。
![blockchain](/resource/images/lucene%20KDTree.jpg "Lucene KDTree")  
        BKDTree是KDTree的变种，KDTree如果有新的节点加入，或者节点修改起来，消耗还是比较大。类似于LSM的merge思路，BKD也是多个KDTree，然后持续merge最终合并成一个。

# 五、Lucene排序
        新版本的Lucene中引入了DocValues，一个基于docId的列式存储。当我们拿到一系列的DocId后，进行排序就可以使用这个列式存储，结合一个堆排序进行。