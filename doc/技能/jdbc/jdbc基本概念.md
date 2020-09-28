# 一、什么是JDBC
        JDBC(Java Data Base Connectivity，Java数据库连接)是一种用于执行SQL语句的Java API，可以为多种关系数据库提供统一访问，它由一组用Java编写的类和接口组成。
        JDBC提供了一种基准，据此可以构建更高级的工具和接口，使数据库开发人员能够编写数据库应用程序。

# 二、常用接口
 ## 1、Driver
         Driver接口由数据库厂家提供，作为Java开发人员，只需要使用Driver接口就行了，在编程中要连接数据库，必须先装载特定厂商的数据库驱动程序，不同的数据库有不同的状态方法。如：
         Mysql驱动：Class.forName("com.mysql.jdbc.Driver");
         Oracle驱动：Class.forName("oracle.jdbc.driver.OracleDriver");

 ## 2、Connection
         Connection与特定数据库建立连接（会话），在连接上下文中执行SQL语句并返回结果。
 ### 2.1 建立连接
         Mysql数据库：Connection conn = DriverManager.getConnection("jdbc:mysql://host:port/databasese", "user", "password");
         Oracle数据库：Connection conn = DriverManager.getConnection("jdbc:oracle:thin:@host:port:database", "host", "password");
### 2.2 常用方法
- createStatement()：创建向数据库发送SQL的statement对象；
- prepareStatement(sql)：创建向数据库发送预编译SQL的PrepareStatement对象；
- prepareCall(sql)：创建执行存储过程的callableStatement对象；
- setAutoCommit(boolean autoCommit)：设置事务是否自动提交；
- commit()：在连接上提交事务；
- rollback()：在连接上回滚事务；

## 3、Statement
        用于执行静态SQL语句并返回所生成结果的对象。
### 3.1 Statement类
        有三种Statement类：
- Statement：由createStatement创建，用于发送简单的SQL语句（不带参数）；
- PreparedStatement：继承自Statement接口，由preparedStatement创建，用于发送含有一个或多个参数的SQL语句。PreparedStatement对象比Statement对象的效率更高，并且可以防止SQL注入，所以我们一般都使用PreparedStatement。
- CallableStatement：继承自PreparedStatement接口，由方法perpareCall创建，用于调用存储过程。
### 3.2 常用方法
- execute(String sql)：运行语句，返回是否有结果集；
- executeQuery(String sql)：运行select语句，返回ResultSet结果集；
- executeUpdate(String sql)：运行inser/update/delete操作，返回更新的行数；
- addBatch(String sql)：把多条sql语句放到一个批处理中；
- executeBatch()：想数据库发送一批sql语句执行；
  
## 4、ResultSet
        ResultSet提供检索不同类型字段的方法。
### 4.1 常用获取数据方法
- getString(int index)、getString(String columnName)：获得在数据库里是varchar、char等类型的数据对象；
- getFloat(int index)、getFloat(String columnName)：获得在数据库里是float类型的数据对象；
- getDate(int index)、getDate(String columnName)：获得在数据库里是date类型的数据对象；
- getBoolean(int index)、getBoolean(String columnName)：获得在数据库里是boolean类型的数据对象；
- getObject(int index)、getObject(String columnName)：获得在数据库里任意类型的数据。
### 4.2 结果滚动方法
- next()：移动到下一行
- previous()：移动到前一行
- absolute(int row)：移动到指定行
- beforeFirst()：移动到resultSet的最前面；
- afterLast()：移动到resultSet的最后面；

# 三、使用JDBC的步骤
## 1、加载JDBC驱动程序
## 2、简历数据库连接Connection
## 3、创建执行SQL语句的Statement
## 4、处理执行结果ResultSet
## 5、释放资源
        资源释放顺序：ResultSet -> Statement -> Connection

# 四、事务
## 1、事务基本概念
## 2、事务的特点


# 五、时间处理

# 六、CLOB、BLOB文件大对象操作