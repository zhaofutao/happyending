# 一、字符串前缀查询 PrefixQuery
        Lucene通过FST的前缀共享属性，实现前缀查询逻辑。
        它会将共享同样前缀的一些关键词都找出来，然后再merge他们的文档列表。

# 二、数字范围查询 NumericRangeQuery
        Lucene将数字进行特殊处理，内部使用BKD-Tree来组织数字键。
        BKD简单理解就是将数值比较接近的数字联合起来作为一个节点共享一个PostingList，同时在PostingList的元素需要存储数字键的值，在查询时会额外多出一个过滤步骤。
        BKD树会将一个大的数值范围进行二分，然后再继续二分，一直分到几层后发现关联的文档数量小于设定的阈值，就不再继续拆分了。
        BKD树还可以索引多维数值，这样就可以应用于地理位置查询。

# 三、字符串范围查询 TermRangeQuery
        字符串范围查询是通过遍历关键词前缀树FST来实现的，它会按照字典序将范围内的所有词汇都列出来，然后merge所有关键词的文档列表。

# 四、短语查询 PhraseQuery

# 五、通配符匹配 WildcardQuery

# 六、模糊查询 FuzzyQuery
        搜索整个关键词的FST-Tree，然后计算他们之间的编辑距离，再挑选出最大编辑距离范围内的词汇。

# 七、全表遍历 MatchAlldocsQuery
        不走倒排索引；