# 一、Lucene代码目录结构

- analysis：负责语法分析及语言处理而形成Term；
- codecs：负责一些数据结构的实现，和一些编码压缩算法，包括skipList、docValues等；
- document：包括了Lucene各类数据类型的定义实现；
- index：负责索引的创建，里面有IndexWriter；
- store：负责索引的读写；
- search：负责对索引的搜索；
- geo：负责geo查询相关的类实现；
- util：bkd、fst等数据结构的实现；