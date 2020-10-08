# 一、基本概念  
## 1、TokenStream
        一个由分词后的Token结果组成的流，能够不断得到下一个token。
## 2、抽象类Analyzer
        主要包括两个接口，用于生成TokenStream
  - TokenStream tokenStream(String fieldName, Reader reader);  
  - TokenStream reusableTokenStream(String fieldName, Reader reader);  
        为了提高性能，使得在同一个线程中无需再生成新的TokenStream对象，老的可以被复用，所以有resuableTokenStream接口；

# 二、几个具体的TokenStream
## 1、NumericTokenStream
        数字类型的Token
## 2、SingleTokenTokenStream
        此TokenStream仅仅包含一个Token，用于保存一个文档中仅有一个的信息，如id等。
## 3、Tokenizer
        包括CharTokenizer、LetterTokenizer、LowerCaseTokenizer、ChineseTokenizer、CJKTokenizer等
## 4、TokenFilter  
        将Tokenizer后的Token作过滤。
        包括ChineseFilter、LengthFilter、LowerCaseFilter等

# 三、几个具体的Analyzer
        不同的Analyzer就是组合不同的Tokenizer和TokenFilter得到最后的TokenStream
## 1、ChineseAnalyzer
        组合ChineseTokenizer和ChineseFilter组成，按字分词，并过滤停词、标点、英文
## 2、CLKAnalyzer
        组合CJKTokenizer和StopFilter，每两个字组成一个词，并去停用词
## 3、PorterStemAnalyzer
## 4、SmartChineseAnalyzer


