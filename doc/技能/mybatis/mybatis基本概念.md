# 一、什么是MyBatis
    MyBatis是一个基于Java的持久层框架，它支持自定义SQL、存储过程以及高级映射。MyBatis是ORM的一种实现框架，是对JDBC的一种封装。  
    MyBatis免除了几乎所有的JDBC代码以及设置参数和获取结果集的参数。
   ![blockchain](/resource/images/ORM.jpg "ORM")

# 二、MyBatis主要内容
![blockchain](/resource/images/mybatis框架.jpg "mybatis框架")

## SqlSessionFactory、SqlSessionFactoryBuilder  
        每个基于MyBatis的应用都是以一个SqlSessionFactory的实例为核心的。SqlSessionFactory的实例可以通过SqlSessionFactoryBuilder获得。  
        而SqlSessionFactoryBuilder则可以从XML配置文件或一个预先配置的Configuration实例来构建出SqlSessionFactory实例。
        SqlSessionFactoryBuilder类可以被实例化、使用和丢弃，一旦创建了SqlSessionFactory就不再需要它了。SqlSessionFactoryBuilder的最佳作用域是方法作用域。
        SqlSessionFactory一旦被创建就应该在应用的运行期间一直存在，没有理由丢弃它或重新创建另一个实例。使用SQLSessionFactory的最佳实践是在应用运行期间不要重复创建多次。
        SqlSession每个线程都应该有它自己的SqlSession实例。SqlSession不是线程安全的，因此是不能被共享的，所以它的最佳作用域是请求或方法作用域。

## SqlSession
        有了SqlSessionFactory，我们就可以从中获得SqlSession，SqlSession提供了在数据库执行SQL命令所需的所有方法 
     

# 三、MyBatis的主要成员
- Configuration  MyBatis所有的配置信息都保存在Configuration对象中，配置文件中的大部分配置都会存储在该类中；
- SqlSession  作为MyBatis工作的主要顶层API，表示和数据库交互时的会话，完成必要数据库增删改查功能；
- Executor  MyBatis执行器，是MyBatis调度的核心，负责SQL语句的生成和查询缓存的维护；
- StatementHandler  封装了JDBC Statement操作，负责对JDBC statement的操作，如设置参数等；
- ParameterHandler 负责对用户传递的参数转换成JDBC Statement所对应的数据类型；
- ResultSetHandler  负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；
- TypeHandler  负责Java数据类型和JDBC数据类型之间的映射和转换；
- MappedStatement  维护一条<select|update|delete|insert>节点的封装；
- SQLSource  负责根据用户传递的parameterObject动态生成SQL语句，将信息封装到BoundSQL对象中，并返回；
- BoundSQL  表示动态生成的SQL语句以及相应的参数信息；

# 四、MyBatis层次结构
![blockchain](/resource/images/mybatis层次结构.png "MyBatis层次结构")
        


# 五、MyBatis缓存
        MyBatis提供查询缓存，用于减轻数据库压力，提高性能。MyBatis提供了一级缓存和二级缓存。
>> ![blockchain](/resource/images/mybatis缓存.png "MyBatis缓存")  

        一级缓存是SqlSession级别的缓存，每个SqlSession对象都有一个哈希表用于缓存数据，不同SqlSession对象之间缓存不共享。同一个SqlSession对象执行2遍相同的SQL，在第一次查询执行完毕后将结果缓存起来，这样第二遍查询就不用向数据库查询了，直接返回缓存结果即可。
        MyBatis默认是开启一级缓存的。
        二级缓存是mapper级别的缓存，是跨SqlSession的，多个SqlSession对象可以共享同一个二级缓存。不同的SqlSession对象执行两次相同的SQL语句，第一次会将查询结果进行缓存，第二次查询直接返回二级缓存中的结果即可。
        MyBatis默认是不开启二级缓存的。
        当SQL执行更新操作时（删除/添加/更新)时，会清空对应的缓存，保证缓存中存储的都是最新的数据。
