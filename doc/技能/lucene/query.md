# 一、TermQuery
  - 成员
    - term
  - 操作
    - createWeight(IndexSearcher searcher, ScoreMode scoreMode, float boost)
    - rewrite(IndexReader reader)  
        不需要重写

# 二、BooleanQuery
  - 成员
    - minimumNumberShouldMatch
    - clauses
  - 操作
    - createWeight(IndexSearcher searcher, ScoreMode scoreMode, float boost)
    - rewrite(IndexReader reader)  
        需要重写

# 三、MultiTermQuery
  - 成员
    - field
    - rewriteMethod
## 3.1 AutomatonQuery
  - 成员
    - automaton（自动机）
    - compiled
    - term
    - automatonIsBinary
### 3.1.1 WildcardQuery
    支持*(匹配0个或多个字符)和?(匹配一个字符)两个通配符。
  - 操作
    - toAutomation(Term wildcardQuery);  
        碰到*,生成makeAnyString
        碰到？，生成makeAnyChar
        碰到\\及正常字符，生成对应的makeChar
        然后，将上述所有自动机串联起来。
### 3.1.2 PrefixQuery
    PrefixQuery指定前缀模糊查询，查询时会匹配所有前缀开头的term。
### 3.1.3 RegexpQuery
    RegexpQuery支持正则表达式；
### 3.1.4 TermRangeQuery
  - 成员
    - BytesRef lowerTerm    下限term
    - BytesRef upperTerm    上限term
    - boolean includeLower  是否包含下限
    - boolean includeUpper 是否包含上限
  - 操作
    - toAutomation(BytesRef lowerTerm, BytesRef upperTerm, boolean includeLower, boolean includeupper);  
        如果最小值、最大值都为空，匹配所有term;
        如果最大值和最小值相等，且都不包含，匹配空term；如果包含，返回单值自动机；
        如果最小值大于最大致，匹配空term；
        否则，返回一个正常的自动机；
## 3.2 FuzzyQuery

# 四、PhraseQuery
    PhraseQuery用于搜索term序列。
    PhraseQuery查询需要term的position信息，如果索引阶段没有保存position信息，就无法使用phrase类的查询；
  - 成员
    - slop：两个term或者两个term序列的编辑距。
    - field
    - terms
    - positions
  - 操作
    - rewrite(IndexReader reader)  
        当term数为1，改写为TermQuery;
        当首个position不为0，所有position前移相应位置，改写为新的PhraseQuery

# 六、PointRangeQuery
  - 成员
  - 操作
    - rewrite(IndexReader reader)  
        不需要重写

# 八、ConstantScoreQuery
    ConstantScoreQuery的功能是将其他查询包装起来，并将它们的查询结果的评分改为一个常量值（默认1.0）。

# 九、DisjunctionMaxQuery
    DisjunctionMaxQuery使用子查询语句中分数最高的那个子查询的评分作为文档的最终评分。
  - 成员
    - Query[] disjuncts;
    - float tieBreakerMultiplier 其他非最高分的权重；

# 十、DocValuesFieldExistsQuery
  - 成员
    - field


# 十一、NormsFieldExistsQuery

# 十二、MatchAllDocsQuery
    匹配所有文档。
