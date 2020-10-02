# 一、Lucene介绍
        Apache Lucene是一个开源的高性能、可扩展的信息检索引擎，提供了强大的数据检索能力。
        基于Lucene的开源项目有很多，最知名的要数Elasticsearch和Solr，如果说Elasticsearch和Solr是一辆设计精美、性能卓业的跑车，那么Lucene就是为其提供强大动力的引擎。
        Lucene的官方对自己的优势总结为几点：
        1、Scalable、High-Performance Index；
        2、Powerful、Accurate and Efficient Search Algorithms；

# 二、Lucene基本概念
![blockchain](/resource/images/lucene抽象架构图.png "Lucene抽象架构图")

## 1. Index（索引）  
        Lucene的Index类似数据库的表的概念，但与传统表的概念会有很大的不同。传统关系型数据库或者NOSQL数据库的表，在创建时至少要定义表的Scheme，定义表的主键或列等，会有一些明确定义的约束。而Lucene的Index，则完全没有约束。
        Lucene的Index可以理解为一个文档收纳箱，你可以往内部塞入新的文档，或者从里面拿出文档 ，但如果你要修改里面的某个文档，则必须先拿出来修改后再塞进去。
## 2. Document（文档）  
        类似于数据库内的行或者文档数据库内文档的概念，一个Index内会包含多个Document。写入Index的Document会被分配一个唯一的ID，即Sequence Number（更多被叫做DocId）。
## 3. Field（字段）
        一个Document会由一个或多个Field组成，Field是Lucene中数据索引的最小定义单位。Lucene提供多种不同类型的Field，如StringField、TextField、LongField或NumericDocValuesField等，Lucene根据Field的类型（FieldType）来判断要采用哪种类型的索引方式（Invert Index、Store Field、DocValues或N-dimensional等）。
## 4. Term和Term Dictionary
        Term是Lucene中索引和搜索的最小单位，一个Field会由一个或多个Term组成，Term由Field经过Analyzer（分词）组成。
        Term Dictionary 是Term的词典，是根据条件查找Term的基本索引。
## 5. Segment
        一个Index会由一个或多个sub-index构成，sub-index被称为Segment。
        Lucene中的数据写入会先写内存的一个Buffer，当Buffer内数据到一定量后会被flush成一个Segment，每个Segment有自己独立的索引，可独立被查询，但数据永远不能被更改。Segment中写入的文档不可被修改，但可被删除，删除的方式是由另外一个文件保存需要被删除的文档的DocId，保证数据文件不可被修改。Index的查询需要对多个Segment进行查询并对结果进行合并，还要处理被删除的文档，为了对查询进行优化，Lucene会有策略对多个Segment进行合并。
        Segment在被flush或commit之前，数据保存在内存中，是不可被搜索的，这也就是为什么Lucene被称为近实时而非实时查询的原因。
## 6. Sequence Number
        Sequence Number是Lucene中一个很重要的概念，数据库内通过主键来标识一行，而Lucene的Index通过DocId来唯一标识一个Doc。
        DocId并不在Index内唯一，而是Segment内唯一，这样是为了做写入和压缩优化。如果一个Index内有两个Segment，每个Segment内分别有100个Doc，在Segment内DocId都是0-100，转换到Index级别的DocId，需要将第二个Segment的DocId范围转换为100-200。
        DocId在Segment内唯一，取值从0开始递增，但不代表一定是连续的，如果有Doc被删除，可能会存在空值。
        一个文档对应的DocId可能会发生变化，主要是发生在Segment合并时。
        Lucene内最核心的倒排索引，本质上就是Term到所有包含该Term的文档的DocId列表的映射。所有Lucene内部在搜索的时候会是一个两阶段的查询，第一阶段是通过给定的Term的条件找到所有的Doc的DocId列表，第二阶段是根据DocId查找Doc。Lucene提供基于Term的搜索功能，也提供基于DocId的查询功能。
        DocId采用一个从0开始递增的int32值。

# 三、索引类型
        Lucene中支持丰富的字段类型，每种字段类型确定了支持的数据类型及索引方式，目前支持的字段类型包括LongPoint、TextField、StringField、NumericDocValues等。
![blockchain](/resource/images/lucene%20field.png "Lucene Field")  
        如图是Lucene中对于不同类型Field定义的一个基本关系，所有字段都会继承自Field这个类。
        Field包含3个重要属性：name(String)、fieldsData(BytesRef)和type(FieldType)。name即字段的名称，fieldsData即字段值，所有类型的字段的值最终都会转换为二进制字节流来表示,type是字段类型，确定了该字段被索引的方式。
        FieldType包含了多个重要属性，这些属性的值共同决定了该字段被索引的方式。
  - stored：代表是否需要保存该字段，如果是false，则lucene不会保存这个字段的值，而搜索结果中返回的文档只包含保存了的字段；
  - tokenized：代表是否做分词，在Lucene中只有TextField这个字段需要做分词。
  - termVector：保存了一个文档内所有的term的相关信息，包括term值、出现次数（frequencies）以及位置(positions)等，是一个per-document inverted vector。term vector的用途只有2个：1、关键词高亮；2、文档间相似匹配度计算；
  - omitNorms：Lucene允许每个文档的每个字段都存储一个normalization factor，是和搜索时的相关性计算相关的一个系数。norms的存储只占一个字段，但是每个文档的每个字段都会独立存储一份，且norms数据会全部加载到内存。索引若开启了norms，会消耗额外的存储空间和内存。
  - indexOptions：Lucene提供倒排索引的5中可选参数（NONE、DOCS、DOCS_AND_FREQS、DOCS_AND_FREQS_AND_POSITIONS、DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS）,用于选择该字段是否需要被索引，以及索引哪些内容。
  - docValuesType：DocValue是Lucene4.0引入的一个正排索引(docid到field的一个列存)，大大优化了sorting、faceting或aggregation的效率。DocValues是一个强Schema的存储结构，开启DocValues的字段必须拥有严格一致的类型，目前Lucene只提供NUMERIC、BINARY、SORTED、SORTED_NUMERIC和SORTED_SET五种类型。
  - dimension：Lucene支持多维数据的索引，采用特殊的索引来优化对多维数据的查询，这类数据最典型的应用场景是地理位置索引，一般经纬数据会采取这种索引格式。

# 四、常见字段类型
  - StoredField  
    一个只存值的字段  
    stored：true  

  - SortedDocValuesField  
    一个只做正排索引的字段  
    docValuesType：SORTED

  - SortedNumericDocValuesField  
    一个只做正排索引的数据字段  
    docValuesType：SORTED_NUMERIC

  - SortedSetDocValuesField  
    一个只做正排索引的集合字段  
    docValuesType：SORTED_SET
  
  - StringField  
    包含两种类型：TYPE_NOT_STORED和TYPE_STORED
    - TYPE_NOT_STORED  
    omitNorms：true，  需要normalization factor  
    indexOptions：DOCS，需要做倒排索引  
    tokenized：false，不做分词  
    - TYPE_STORED  
    omitNorms：true，需要normalization factor  
    indexOptions：DOCS，需要做倒排索引，只需要DOC信息  
    tokenized：false，不做分词 
    stored：true，保存字段

  - TextField  
    包含两种类型：TYPE_NOT_STORED和TYPE_STORED
    - TYPE_NOT_STORED  
    indexOptions：DOCS_AND_FREQS_AND_POSITION，需要做倒排索引，包括DOC、FREQ、POSITION等信息；  
    tokenization：true，需要做切词  
    - TYPE_STORED  
    indexOptions：DOCS_AND_FREQS_AND_POSITION，需要做倒排索引，包括DOC、FREQ、POSITION等信息；  
    tokenization：true，需要做切词  
    stored：true，保存字段  

  - NumericDocValuesField  
    docValuesType：NUMERIC，需要做正排索引；
    DoubleDocValuesField、FloatDocValuesField为其子类，属性一致；
  
  - BinaryDocValuesField
    docValuesType：BINARY，需要做正排索引


  - LongPoint、IntPoint、DoublePoint、FloatPoint、BinaryPoint  
    dimensions：根据构造决定，默认是1

 - LongRange、IntRange、DoubleRange、FloatRange  
    dimensions是*Point的2倍，也即min和max各算一个dimension。