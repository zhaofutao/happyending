# 一、Spring-Data-Elasticsearch是什么
## 1. Spring Data
        Spring Data是基于Spring为数据访问提供一种相似且一致性的编程模型，并保存底层数据存储的。
## 2. Spring-Data-Elasticsearch
        Spring-Data-Elasticsearch是是Spring Data的Community Modules之一，是Spring Data对Elasticsearch引擎的实现。

# 二、Spring-Data-Elasticsearch基本概念
## 1. ElasticSearchRepository
        ES通用的存储接口的一种默认实现，Spring根据接口定义的方法名，具体执行对应的数据存储实现。
        ElasticsearchRepository继承ElasticsearchCrudRepository，ElasticsearchCrudRepository继承PagingAndSortingRepository，所以CRUD带分页已经支持。
![blockchain](/resource/images/elasticsearchrepository.png)

## 2. ElasticsearchTemplate
        ES数据操作的中心支持类，和JdbcTemplate一样，几乎所有操作都可以使用ElasticsearchTemplate来完成。
        ElasticsearchTemplate实现了ElasticsearchOperations和ApplicationContextAware接口。ElasticsearchOperations接口提供了ES相关的操作，并将ElasticsearchTemplate加入到Spring上下文。
![blockchain](/resource/images/elasticsearch%20template.png)