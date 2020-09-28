# 一、Spring JDBC基本概念
        Spring JDBC抽象框架提供了对JDBC操作的完整封装，包括：
- 指定数据库连接参数；
- 打开数据库连接；
- 预编译并执行SQL语句；
- 遍历查询结果；
- 处理抛出的任何异常；
- 处理事务；
- 关闭数据库连接； 

# 二、JdbcTemplate类
        JdbcTemplate是Spring JDBC的核心类，它替我们完成了资源的创建以及释放工作，从而简化了我们对JDBC的使用。
        我们可以在DAO实现类通过传递一个DataSource引用来完成JdbcTemplate的实例化，也可以在Spring的IOC容器中配置一个JdbcTemplate的bean并赋予DAO实现类作为一个实例。