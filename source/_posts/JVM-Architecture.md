---
title: JVM-体系结构
tags: 
  - JVM
categories: JVM
---

# JVM体系结构

JVM（Java Virtual Machine，Java虚拟机 ）是一种抽象的计算机，它能够运行Java编译后生成的字节码文件（.class后缀的文件）。JVM充当了一个翻译官的角色，将开发者编写的Java代码转换为底层操作系统能够理解的指令。最为核心的特性是“一次编写，到处运行”，即通过JVM，Java程序可以在任何支持JVM的平台上运行，而无需修改代码。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250312140051.png)

如图所示，可以知道JVM主要由四个部分组成：

- 类加载子系统（ClassLoader）

- 运行时数据区 （Runtime Data Area）

- 执行引擎 （Execution Engine）

- 本地库接口 （Native Method Library）

  

## 1.类加载子系统

### 1.1.类的生命周期

类加载过程包括了`加载`、`验证`、`准备`、`解析`、`初始化`五个阶段。其中，`验证`、`准备` 和 `解析` 这三个部分统称为连接。注意这几个阶段是**按顺序开始，而不是按顺序进行或完成**，通常都是交叉地混合进行的，通常在一个阶段执行的过程中调用或激活另一个阶段。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250315173231.png)

**加载：查找并加载类的二进制数据**

- 通过类的全限定名，获取二进制字节流。

- 将这个二进制字节流所代表的静态存储结构转化为方法区的运行时数据结构。

- 在Java堆中创建一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

  ![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250315183702.png)

**验证：确保被加载类的正确性**

验证主要是确保Class文件的字节流包含的信息符合当前虚拟机的要求，并且对虚拟机不会造成危害。

主要分为：

- 文件格式验证：是否符合Class文件的规范。 例如：文件开头是否是`0xCAFEBABE`

- 元数据验证：

  - 这个类是否有父类。（除了Object这个类之外，其余的类都应该有父类）
  - 这个类是否继承（extends）了被final修饰过的类。（被final修饰过的类表示类不能被继承）
  - 类中的字段、方法是否与父类产生矛盾。（被final修饰过的方法或字段是不能被覆盖的）

- 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。

- 符号引用验证：符号引用一组符号来描述引用的目标，符号可以是任何形式的字面量。

  ```
  int i = 3; // 字面量:3 符号引用：i
  ```

**准备：为类的静态变量分配内存，并将其初始化为默认值**

为类变量（static）分配内存并设置类变量的初始值，这些内存都将在方法区中分配。

- static变量，分配空间在准备阶段完成（设置默认零值：0,null,false等），赋值在初始化阶段完成
- static变量是final的基本类型，以及字符串常量，值已确认，赋值在准备阶段完成。
- static变量是final的引用类型，那么赋值也会在初始化阶段完成。

```java
static int b = 10; //准备阶段是默认0值，初始化时赋值10
static final int c = 20; //准备阶段直接赋值
static final String d = "hello"; //准备阶段直接赋值
static final Object obj = new Object(); //准备阶段是null,初始化阶段赋值
```

**解析：把类中的符号引用转换为直接引用**

直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

比如：方法中调用了其他方法，方法名可以理解为符号引用，而直接引用就是使用指针直接指向方法。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250315231205.png)

**初始化**

对类的静态变量，静态代码块执行初始化操作

- 如果该类的直接父类尚未初始化，则优先初始化其父类。
- 如果同时包含多个静态变量和静态代码块，则按照自上而下的顺序依此执行。

**使用**

JVM开始从入口方法（前面加载阶段说的方法区的数据结构接口）开始执行用户的程序代码。

- 调用静态类成员信息。（比如：静态字段、静态方法）
- 使用new关键字为其创建对象实例。

**卸载**

当用户程序代码执行完毕后，JVM便销毁创建的Class对象，最后负责运行的JVM也退出内存。

### 1.2.类加载器

我们知道JVM只会运行二进制文件，而类加载器（ClassLoader）的主要作用就是将字节码文件加载到JVM中。现有的类加载器基本上都是java.lang.ClassLoader的子类，该类的只要职责就是用于将指定的类找到或生成对应的字节码文件，同时类加载器还会负责加载程序所需要的资源。

**类加载器种类**

<img src="https://gitee.com/Luyseon/blogimage/raw/master/img/20250315232708.png" style="zoom: 50%;" />

`启动类加载器`：Bootstrp ClassLoader，负责加载存放在JAVA_HOME\jre\lib下，启动类加载器是无法被java程序直接引用的，它由C++编写实现。

`扩展类加载器`：ExtClassLoader，负责加载JAVA_HOME\jre\lib\ext目录中的类库。

`应用程序类加载器`：Application ClassLoader，加载开发者自己编写的Java类。

`自定义类加载器`：CustomizaClassLoader，开发者自定义类继承ClassLoader，实现自定义加载规则。

类加载器的体系并**不是继承**体系，而是**委派体系**，类加载器加载一个类时首先会先在自己的parent中查找类或资源，如果找不到才回到自己的本地查找。这种委托行为是为了避免相同的类被加载多次。

**寻找类加载器** 

一个类加载器的例子：（这个例子是对于java8）

```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        System.out.println(loader);
        System.out.println(loader.getParent());
        System.out.println(loader.getParent().getParent());
    }
}
```

输出结果：

```java
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@27c170f0
null
```

从上面结果我们可以看到比没有获取到ExtClassLoader的父类，原因是BootstrapClassLoader是C++实现的，找不到确定的返回父Loader的方式，所有返回null。

**双亲委派机制**

当一个类加载器在接到加载类的请求时，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类

- 当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。

- 当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。

- 如果BootStrapClassLoader加载失败(例如在JAVA_HOME/jre/lib里未查找到该class)，会使用ExtClassLoader来尝试加载；

- 若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。



## 2.运行时数据区 

### 2.1.程序计数器

程序计数器，线程私有的，是一块较小的内存空间，内部保存的字节码的行号。用于记录正在执行的字节码指令的地址。**程序计数器是JVM规范中唯一一个没有规定出现OOM的区域**，所以这个区域也不会进行GC回收。

**作用**

- 在方法调用、分支、循环等控制流操作中，程序计数器用于定位下一条需要执行的指令。
- 存储指向下一条指令的地址（即即将执行的字节码指令的地址）。由执行引擎读取下一条指令。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250317104804.png)

**为什么程序计数器会设为线程私有？**

JVM虚拟机对于多线程是通过线程轮流切换并分配线程执行时间，在任何一个时间点只会执行一个线程，当这个线程时间片用完，就会线程挂起，然后处理器会切换到另外一个线程进行执行。这期间线程来回切换，每次切换回来就需要知道当前线程执行到哪里，所以为每个线程分配一个程序计数器，每个线程都独立计算，不会互相影响。

### 2.2.虚拟机栈

Java 虚拟机栈(Java Virtual Machine Stacks)，每个线程在创建的时候都会创建一个虚拟机栈，其内部保存一个个的栈帧(Stack Frame），对应着一次次 Java 方法调用，是**线程私有**的，生命周期和线程一致。（**栈不存在垃圾回收问题**)

**作用**

主管 Java 程序的运行，它保存**方法的参数和局部变量、返回地址，并参与方法的调用和返回。**

**栈运行原理**

- 每次调用一个方法时，JVM会为该方法创建一个新的栈帧，并将其压入当前线程的虚拟机栈中。在一条活动线程中，一个时间点上，只会有一个活动的栈帧。

  ```
  public void methodA() {
      methodB();
  }
  
  public void methodB() {
      // 方法体
  }
  ```

  当methodA调用methodB时，methodB的栈帧会被压入虚拟机栈中。

- 当方法执行完成，其对应的栈帧就会被弹出虚拟机栈（出栈）,并释放相关资源。如果有返回值，则返回值会被传递给调用者。

- 虚拟机栈可能抛出两种异常：

  - `StackOverflowError`：当栈深度超过限制时抛出。典型问题：递归调用

    ```java
    public static void m(){
    	m(); // 抛出异常： java.lang.StackOverflowError
    }
    ```

  - `OutOfMemoryError`：如果虚拟机栈允许动态扩展，但在尝试扩展时无法申请到足够的内存，则会抛出这个异常。

**栈帧的内部结构**

每个栈帧中存储着：

- 局部变量表
  - 用于存储方法的参数和方法内部定义的局部变量。
  - 他是一个数据结构，按索引访问，支持基本数据类型和对象引用的存储。
- 操作数栈
  - 是一个后进先出的栈活动，用于保存**计算过程中的中间结果**。
  - 操作数栈作为方法执行的工作区，所有的计算操作（如加法、赋值等）都在这里完成。
- 动态链路：
  - 指向运行时常量池中该栈帧所属方法的引用。
  - **动态链路的作用就是为了将符号引用转换为调用方法的直接引用**。
- 方法返回值：方法正常退出或异常退出的地址，如果正常返回，返回值会被传递给调用者。
- 一些附加信息

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250318110059.png)

### 2.3.本地方法栈

- 本地方法栈与虚拟机栈类似，Java虚拟机栈用于管理 Java 方法的本地调用，而本地方法栈用于管理本地方法的调用。

- 本地方法栈也是线程私有的。
- 可能抛出两种异常：
  - `StackOverflowError`：当栈深度超过限制时抛出。
  - `OutOfMemoryError`：如果虚拟机栈允许动态扩展，但在尝试扩展时无法申请到足够的内存，则会抛出这个异常。
- 本地方法是使用 C 语言实现的。

### 2.4.堆

**内存划分**

Java堆是 Java 虚拟机管理的内存中最大的一块，被所有线程共享。堆中唯一的目的就是存放对象的实例，几乎所有对象的实例以及数据都在堆内存中。

虚拟机把堆内存逻辑上划分为两块区域（分代的唯一理由就是优化GC性能）：

- 新生代（年轻代）：新对象和没达到一定年龄的对象都在年轻代。

- 老年代（养老区）：被长时间使用的对象。java1.8时，老年代和新生代比例：1：2。
- 元空间（JDK1.8 之前叫永久代）：类信息、静态变量、常量、编译后的代码，JDK1.8 之前占用 JVM 内存，JDK1.8 之后直接使用本地内存。

为了避免方法区出现OOM，所以在Java1.8 中将堆上的**方法区/永久代**给移动到了本地内存上，重新开辟一块空间，叫做**元空间**。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250317171555.png)

**年轻代**

年轻代是所有对象创建的地方。当填充年轻代时，执行垃圾收集，这种垃圾收集称为Minor CG。年轻代被划分为三个部分：**伊甸园**（Eden Memory）和两个**幸存区**（Survivor Memory，被称为from/to或s0/s1），默认比例为：**8：1：1**

- 大多数新创建的对象都位于 Eden 内存空间中。
- 当 Eden 空间填满时，触发**Minor GC**（Young GC），并将所有幸存者对象移动到（**复制算法**）一个非空的Survivor区（如S0）中，另一个Survivor区保持为空。
- 下一次Minor GC时，存活对象从当前使用的Survivor（如S0）移动到另一个Survivor区（如S1），并清空原来的Survivor区。所有每次，一个幸存区总是空的。
- 如果对象的年龄达到阈值（默认15次）或Survivor区空间不足，则将其晋升到老年代。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250317183453.png)

**老年代**

老年代中包含那些经过许多轮小型GC（Minor GC）后仍然存活的对象。通常，垃圾收集是在老年代内存满时执行。老年代垃圾收集通过主GC（Major GC），一般需要更长的时间。

大对象直接晋升为老年代，这样做的目的时避免Eden区和Survivor区之间发生大量的内存拷贝。

**元空间**

不管是 JDK8 之前的永久代，还是 JDK8 之后的元空间，本质都是**对 Java虚拟机规范中方法区的实现**。

在**HotSpot JVM**中，永久代（ ≈ 方法区）中用于存放类和方法的元数据以及常量池，比如Class和Method。每当一个类初次被加载时，它的元数据都会放到永久代中。

永久代是有大小限制的，由于永久代是放在 JVM管理的内存中的很容易造成内存溢出，所有Java8之后取消了永久代，而是在本地内存中开辟了一个元空间，元空间的大小仅受本地内存限制。

**设置堆内存大小和OOM**

Java堆内存大小在 JVM 启动的时候就确定了，我们可以通过`-Xmx`和`-Xms`设定：

- `-Xmx`：用于设置最大堆内存大小。（默认大小为电脑内存大小/4）
- `Xms`：用于设置初始堆内存大小。（默认为电脑内存大小/64）

这两个参数一般会设为相同值，其目的是为了能够在垃圾回收机制清理完堆区后不再需要重新分配计算堆的大小，从而提高性能。

如果堆的内存大小超过了`Xmx`设定的最大内存，就会抛出`outOfMemoryError`。

### 2.5.方法区

- 方法区与Java堆一样是线程共享的内存区域。
- 主要存储类的信息、运行时常量池。

- 方法区的大小和堆空间一样，可以固定大小也可以选择可扩展，方法区的大小决定了系统可以放多少个类，如果系统类太多，导致方法区溢出，Java虚拟机也会抛出内存溢出错误。
- 虚拟机启动时创建，关闭虚拟机释放。

**方法区内部结构**

方法区用于存储已经被虚拟机加载的类型信息、运行时常量池、静态变量，即时编译器编译后的代码缓存等。

- **类型信息**：
  - 类的全限定名、父类名 、实现的接口列表等。
  - 类的访问修饰符（如`public`、`final`等）和字段、方法的相关信息。
- **运行时常量池**：存储字面量和符号引用。
- **静态变量**：存储类的静态变量。
- **即时编译器编译后的代码缓存**： 即时编译器（JIT）将热点代码编译为本地机器码后，缓存到方法区中。

**常量池**

在字节码为文件（.class后缀）中除了包含类的版本信息，字段，方法以及接口等信息之外，还包含常量池表（Constant Pool Table），它包含各种字面量和对类型、域和方法的符号引用。

查看字节码结构（类的基本信息、常量池、方法定义） javap -v xx.class

比如下面类的字节码结构：

```java
public class helloTest {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

字节码：

```
Classfile /D:/idea_java_projects/MyBlog-Code/test/target/classes/com/luyseon/testcode/test/helloTest.class
  Last modified 2025-3-17; size 612 bytes  //最后修改的时间
  MD5 checksum aa0f3151d1ddf5dc2c6858279573e939  //签名
  Compiled from "helloTest.java"
public class com.luyseon.testcode.test.helloTest
  minor version: 0
  major version: 52  //jdk版本 
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:  //常量池表
   #1 = Methodref          #6.#21         // java/lang/Object."<init>":()V
   #2 = Fieldref           #22.#23        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #24            // hello world
   #4 = Methodref          #25.#26        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #27            // com/luyseon/testcode/test/helloTest
   #6 = Class              #28            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/luyseon/testcode/test/helloTest;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               MethodParameters
  #19 = Utf8               SourceFile
  #20 = Utf8               helloTest.java
  #21 = NameAndType        #7:#8          // "<init>":()V
  #22 = Class              #29            // java/lang/System
  #23 = NameAndType        #30:#31        // out:Ljava/io/PrintStream;
  #24 = Utf8               hello world
  #25 = Class              #32            // java/io/PrintStream
  #26 = NameAndType        #33:#34        // println:(Ljava/lang/String;)V
  #27 = Utf8               com/luyseon/testcode/test/helloTest
  #28 = Utf8               java/lang/Object
  #29 = Utf8               java/lang/System
  #30 = Utf8               out
  #31 = Utf8               Ljava/io/PrintStream;
  #32 = Utf8               java/io/PrintStream
  #33 = Utf8               println
  #34 = Utf8               (Ljava/lang/String;)V
{
  public com.luyseon.testcode.test.helloTest();  //构造方法
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/luyseon/testcode/test/helloTest;

  public static void main(java.lang.String[]);  //main方法
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 5: 0
        line 6: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
    MethodParameters:
      Name                           Flags
      args
}
SourceFile: "helloTest.java"
```

**运行时常量池**

常量池时 *.class 文件中，当该类被加载，它的常量池信息就会放入运行时常量池，并把里面的符号地址变为真实地址。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250317231536.png)

**元空间和永久代**

-  **方法区只是JVM规范中定义的一个逻辑上的概念**，并没有规定如何去实现它，不同的厂商有不同的实现方式。而**永久代是 Hotspot 虚拟机特有的概念**，**Java8开始被元空间取代**，永久代和元空间都可以理解为方法区的实现。
- JVM 规范说方法区在逻辑上是堆的一部分，但目前实际上是与 Java 堆分开的（Non-Heap）。
- 存储内容不同，元空间存储类的元信息和运行时常量池，而静态变量与字符串常量池等并入堆中。相当于**永久代的数据被分到了堆和元空间中**。

![](https://gitee.com/Luyseon/blogimage/raw/master/img/20250317233035.png)



## **3. 执行引擎**

执行引擎是JVM的核心组件之一，其主要任务是执行由`javac`编译生成的`.class`文件中的字节码。

**原理**

- **输入**：字节码指令（Bytecode）。  
- **输出**：对应平台上的本地机器指令。  
- **目标**：通过解释或编译的方式，将字节码转换为底层硬件可以直接执行的二进制指令。

**作用**

执行引擎充当了字节码与底层硬件之间的桥梁，确保Java程序能够在不同平台上运行。



## **4. 本地库接口**

本地库接口（Native Interface）是JVM与底层操作系统或其他语言交互的桥梁。

**主要功能**

1. 提供调用C/C++等本地方法的能力，使得Java可以利用底层系统的功能（如文件操作、网络通信等）。  
2. 支持通过JNI（Java Native Interface）实现Java与其他语言的融合。

**作用**

本地库接口扩展了Java的功能范围，弥补了Java在某些底层操作上的不足。
