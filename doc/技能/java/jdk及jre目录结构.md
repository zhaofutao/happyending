# 一、JDK目录
## 1. bin
    包含在Java开发包的可执行文件，PATH环境变量设置在此目录。
    包含有java、javac、jar、jmap、jps、jstack、jstat等常用命令；

## 2. db
    包含Java DB，一个局域Java编程语言和SQL关系数据库管理系统；

## 3. include
    支持使用本机代码变成的C语言头文件，Java本地接口（JNI）和Java虚拟机调试程序接口（JPDA）；
    JNI（Java Native Interface）Java本地接口是一个标准的编程接口，用于编写Java本地方法或者嵌入Java虚拟机到本地应用程序中；

## 4. lib
    JDK使用的依赖包，如:
  - tools.jar JDK的非核心工具支撑类；
  - dt.jar 告诉IDE设计时存档如何显示Java组件以及如何让开发者自定义他们的应用程序；

# 二、JRE目录
## 1. bin
    JavaP平台工具所使用的的可执行文件，一般在jdk/bin下也存在；

## 2. lib
    代码树、树形设置以及JRE使用的源文件，如
  - rt.jar Bootstrap类（构成Java平台核心API的运行时类）
  - charsets.jar 字符转换类

# 三、JDK包含的组件
- javc：编译器，将后缀名为.java的源代码编译成后缀名为.class的字节码；
- java：运行工具，运行.class的字节码；
- jar：打包工具，将相关的类文件打包成一个文件；
- javadoc：文档生成器，从源码注释中提取文档，注释需匹配规范；
- jdb：调试工具；
- jps：显示当前Java程序运行的进程状态；
- jhat：Java堆分析工具；
- jstack：JVM检测统计工具；
- jinfo：获取正在运行或崩溃的Java程序配置信息；
- jmap：获取Java进程内存映射信息；