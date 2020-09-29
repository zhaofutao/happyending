# 一、JVM简介
    JVM是一种规范，是虚构出来的一台计算机，包括字节码指令集和内存管理（方法区、堆、栈、本地方法栈）。
    JVM与Java无关，任何语言文件被编译为.class文件后，都能直接在JVM上执行。
    常见的JVM实现：
  - Hotspot：oracle官方；
  - Jrockit：BEA，曾经号称世界上最快的JVM，被Oracle收购合并于Hotspot；
  - J9：IBM;
  - Microsoft VM;
  - TaobaoVM hostspot淘宝深度定制版；
  - LiquidVM 直接针对硬件；
  - azul zing 最新垃圾回收的业界标杆；


# 二、JVM运行流程
![blockchain](/resource/images/jvm运行流程.png "JVM运行流程")  

        Java程序经过javac编译后，将java代码编译为字节码也就是.class文件，然后在不同的操作系统上依靠不同的Java虚拟机进行解释，最后再转换为不同平台的机器码，最终得到执行。
        对于一段简单的代码：
```
public class HelloWorld {
	public static void main(String[] args) {
		System.out.print("Hello world");
	}
}
```
        这段程序从编译到运行，最终大运出“Hello world”，需经过以下步骤：
![blockchain](/resource/images/helloworld执行过程.png "Hello world执行过程")
        Java代码通过编译之后生成字节码文件（.class文件）
        通过java HelloWorld命令执行
        Java根据系统版本找到jvm.cfg，然后根据配置找到jvm.dll，JVM的具体实现；
        然后初始化JVM，获取JNI接口；
        最后找到class文件后加载进JVM，找到main方法，最后执行。

# 三、JVM内部结构
![blockchain](/resource/images/jvm内部结构.png "JVM内部结构")
        .class文件被JVM加载以后，经过JVM的内存空间调配，最终是由执行引擎完成class文件的执行。

## 1. 内存空间
        JVM内存空间包含：方法区、Java堆、Java栈、本地方法栈；
        方法区是各个现场共享的区域，存放类信息、常量、静态变量；
        Java堆也是线程共享的区域，我们的类的实例就是放在这个区域，Java堆的空间是最大的，如果Java堆空间不足了，程序会抛出OutOfMemory异常。
        Java栈是每个线程私有的区域，它的生命周期与线程相同，一个线程对应一个Java栈，没执行一个方法就会往栈中压一个元素，这个元素叫栈帧。栈帧中包含了方法的局部变量、用于存放中间状态值的操作栈。如果Java栈空间不足了，程序会抛出StackOverflowError异常。
        本地方法栈角色和Java栈类似，只不过它是用来表示执行本地方法的，本地方法栈存放的方法调用本地方法接口，最终调用本地方法库，实现与操作系统、硬件交互的目的。