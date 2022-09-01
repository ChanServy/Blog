---
title: JVM 学习笔记
date: 2021-12-25 22:08:12
toc: true
categories: JVM
tags: JVM
urlname: jvm
---

## 什么是 JVM

### 定义

Java Virtual Machine，JAVA程序的**运行环境**（JAVA二进制字节码的运行环境）也就是 Java 虚拟机。

### 好处

- 一次编写，到处运行
- 自动内存管理，垃圾回收机制
- 数组下标越界检查
- 多态，面向对象

### 比较

JVM、JRE、JDK的区别

<!--more-->

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image05.png)

## 内存结构

### 整体架构

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image01.png)

JVM 大致上可分为：

1. 类加载器模块
2. JVM 内存结构
3. 执行引擎

JVM被分为三个主要的子系统：类加载器子系统、运行时数据区和执行引擎 。

![](https://img-blog.csdnimg.cn/img_convert/6422795210912307f4956a9d08bd9791.png)

首先，大致概述一下整个流程：

一个类从 Java 源代码编译成了二进制字节码以后，必须经过类加载器才能被加载到 JVM 中运行。类都是放在方法区的部分，类将来创建的实例对象放在堆，而堆中的对象在调用方法时，又会用到虚拟机栈、程序计数器以及本地方法栈。

方法执行时，每行代码是由执行引擎中的解释器逐行进行执行，方法中的一些热点代码（被频繁调用的代码）会由一个 JIT 即时编译器来编译，优化执行。执行引擎中还有一个很重要的模块：GC 垃圾回收。GC 会对堆中的一些不再引用的对象进行垃圾回收。当然这里还会有一些 Java 代码不方便实现的功能必须调用底层操作系统的功能，所以我们要和操作系统的一些功能打交道的话，就需要借用本地方法接口来调用操作系统的一些方法，这就是整个 JVM 的组成。

### 程序计数器

#### 作用

是记住下一条jvm指令的执行地址。

#### 特点

- 线程私有，每个线程都有自己的*程序计数器* ，随着线程的创建而创建，随着线程的销毁而销毁
  - CPU会为每个线程分配时间片，当当前线程的时间片使用完以后，CPU就会去执行另一个线程中的代码
  - 程序计数器是**每个线程**所**私有**的，当另一个线程的时间片用完，又返回来执行当前线程的代码时，通过程序计数器可以知道应该执行哪一句指令
- 不会存在内存溢出（程序计数器在 JVM 规范中是唯一一个不会存在内存溢出的区域）

计数器是 java 对物理硬件的屏蔽和抽象；计数器在物理上是通过寄存器来实现的。

寄存器是整个 CPU 的组件里读取速度最快的一个单元，因为我们读写指令地址的动作是非常频繁的，所以 Java 虚拟机在设计的时候，就将 CPU 中的寄存器当做了程序计数器，用它来存储地址，将来从寄存器读取地址。

Java 是支持多线程的，多线程运行时，CPU 会有一个调度器组件，给线程们分配时间片。比如线程 1 和线程 2，线程 1 先分到了时间片，在时间内代码未执行完，执行到了一定程度，然后会将线程 1 的状态进行一个暂存，切换到线程 2 执行，等线程 2 执行到一定程度，线程 2 的时间片用尽，再切换回来，再继续执行线程 1 剩余的代码。这便是时间片的概念。

线程切换的过程中，我们的程序计数器会记住下一条指令执行到哪里了，程序计数器是线程私有的，每个线程都有自己的程序计数器。

### 虚拟机栈

#### 定义

- 每个**线程**运行需要的内存空间，称为**虚拟机栈**
- 每个栈由多个**栈帧**组成，对应着每次调用方法时所占用的内存
- 每个线程只能有**一个活动栈帧**，对应着**当前正在执行的方法**

符合数据结构栈内存的特点，先进后出。其实栈就是线程运行时需要的内存空间。一个线程就需要一个栈，多个线程就需要多个栈。

一个栈内是由多个栈帧组成的。一个栈帧对应一次方法的调用。线程最终的目的是为了执行代码的，代码都是由一个个的方法来组成的，所以在线程运行时，每个方法它需要的内存，我们便称之为栈帧。那么方法需要的内存指的是什么呢？参数、局部变量、返回地址这些都需要占用内存。

栈帧与栈是如何联系起来的？比如调用第一个方法时，会给这个方法划分一段栈帧空间，并且压入栈内，当这个方法执行完了，就会将这个方法对应的栈帧让它出栈，也就是释放这个方法所占用的内存，这就是栈与栈帧的联系。

一个栈内可能有多个栈帧。比如调用方法 1，方法 1 间接调用方法 2，方法 2 就会产生一个新的栈帧，方法 2 执行完，会将方法 2 对应的栈帧出栈，最后方法 1 执行完，也是将方法 1 对应的栈帧出栈。

#### 演示

代码

```java
public class Main {
    public static void main(String[] args) {
        method1();
    }

    private static void method1() {
        method2(1, 2);
    }

    private static int method2(int a, int b) {
        int c = a + b;
        return c;
    }
}
```

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150534.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150534.png)

在控制台中可以看到，主类中的方法在进入虚拟机栈的时候，符合栈的特点

#### 问题辨析

- 垃圾回收是否涉及栈内存？
  - **不需要**。因为虚拟机栈中是由一个个栈帧组成的，而栈帧内存对应着每一次方法的调用，在方法执行完毕后，对应的栈帧就会被弹出栈，也就是会自动的被回收掉。所以无需通过垃圾回收机制去回收栈内存，垃圾回收只是回收堆内存中的无用对象。
- 栈内存的分配越大越好吗？
  - 不是。因为**物理内存是一定的**，栈内存越大，虽然可以支持更多的递归调用，但是可执行的线程数就会越少。
- 方法内的局部变量是否是线程安全的？
  - 如果方法内**局部变量没有逃离方法的作用范围**，则是**线程安全**的
  - 如果如果**局部变量引用了对象**，并**逃离了方法的作用范围**，则需要考虑线程安全问题

#### 内存溢出

**Java.lang.stackOverflowError** 栈内存溢出

**发生原因**

- 虚拟机栈中，**栈帧过多**（无限递归）
- 每个栈帧**所占用过大**

#### 线程运行诊断

CPU占用过高

- Linux环境下运行某些程序的时候，可能导致CPU的占用过高，这时需要定位占用CPU过高的线程
  - **top**命令，查看是哪个**进程**占用CPU过高
  - **ps H -eo pid, tid（线程id）, %cpu | grep 刚才通过top查到的进程id** 通过ps命令进一步查看是哪个线程占用CPU过高
  - **jstack 进程id** 通过查看进程中线程的nid，与刚才通过ps命令看到的tid来**对比定位**，进一步定位到问题代码的源码行号。注意jstack查找出的线程id是**16进制的**，**需要转换**

### 本地方法栈

一些带有**native关键字**的方法就是需要JAVA去调用本地的C或者C++方法，因为JAVA有时候没法直接和操作系统底层交互，所以需要用到本地方法

### 堆

#### 定义

通过new关键字**创建的对象**都会被放在堆内存

#### 特点

- **所有线程共享**，堆内存中的对象都需要**考虑线程安全问题**
- 有垃圾回收机制

#### 堆内存溢出

**java.lang.OutofMemoryError** ：java heap space. 堆内存溢出

#### 堆内存诊断

**jps**：查看当前系统中有哪些 java 进程

**jmap**：查看堆内存占用情况 jmap - heap 进程id

**jconsole**：jconsole 是 jdk 自带的堆诊断工具，在 bin 目录下，或者在 idea 控制台旁边的 Terminal 输入 jconsole 就打开了。图形界面的，多功能的监测工具，可以连续监测。

**jvirsalvm**：也是可以进行堆内存诊断的，并且 jvisualvm 有堆 Dump 的功能，就是可以抓取堆的当前快照，我们可以进一步的对堆的一些详细内容进行分析。查找保留前 20 个最大的对象。在 idea 控制台旁边的 Terminal 输入 jvisualvm 就打开了。

 **jstat**：查看jvm的GC情况。[jstat命令查看jvm的GC情况 （以Linux为例）](https://www.cnblogs.com/yjd_hycf_space/p/7755633.html)

### 方法区

和堆一样也是线程共享的区域。方法区中存储了跟类的结构相关的一些信息，比如有成员变量，成员方法以及构造器方法。在虚拟机启动时被创建。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image06.png)

#### 结构

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image07.png)

#### 方法区内存溢出

- 1.8以前会导致**永久代**内存溢出

  ```
  * 演示永久代内存溢出 java.lang.OutOfMemoryError: PermGen space
  * -XX:MaxPermSize=8m
  ```

- 1.8以后会导致**元空间**内存溢出

  ```
  * 演示元空间内存溢出 java.lang.OutOfMemoryError: Metaspace
  * -XX:MaxMetaspaceSize=8m
  ```

#### 常量池

二进制字节码的组成：类的基本信息、常量池、类的方法定义（包含了虚拟机指令）

常量池的作用就是给我们的这些指令提供一些常量符号，根据常量符号找到下一步的指令。

eg：

```java
package com.chan.concurrent.lianxi;

public class Test {
    public static void main(String[] args) {
        String a = "a";
        String b = "b";
        String ab = "ab";
        String ab2 = a+b;
        //使用拼接字符串的方法创建字符串
        String ab3 = "a" + "b";
    }
}
```

**通过反编译来查看类的信息**

- 获得对应类的.class文件

  - 在JDK对应的bin目录下运行cmd，**也可以在IDEA控制台输入**

  -  **javac 对应类的绝对路径**输入完成后，对应的目录下就会出现类的.class文件

    ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image08.png)

- 在控制台输入 javap -v 类的绝对路径

  ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image09.png)

- 然后能在控制台看到反编译以后类的信息了

  - 类的基本信息

    ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image10.png)

  - 常量池

    ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image11.png)

    ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image12.png)

  - 虚拟机中执行编译的方法（框内的是真正编译执行的内容，**#号的内容需要在常量池中查找**）

    ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image13.png)

#### 运行时常量池

- 常量池
  - 就是一张表（如上图中的constant pool），虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量信息
- 运行时常量池
  - 常量池是*.class文件中的，当该**类被加载以后**，它的常量池信息就会**放入运行时常量池**，并把里面的**符号地址变为真实地址**

#### 常量池与串池的关系

##### 串池StringTable

**特征**

- 常量池中的字符串仅是符号，只有**在被用到时才会转化为对象**
- 利用串池的机制，来避免重复创建字符串对象
- 字符串**变量**拼接的原理是**StringBuilder**
- 字符串**常量**拼接的原理是**编译器优化**
- 可以使用**intern方法**，主动将串池中还没有的字符串对象放入串池中
- **注意**：无论是串池还是堆里面的字符串，都是对象

用来放字符串对象且里面的**元素不重复**

```java
public class StringTableStudy {
    public static void main(String[] args) {
        String a = "a"; 
        String b = "b";
        String ab = "ab";
    }
}
```

常量池中的信息，都会被加载到运行时常量池中，但这是a b ab 仅是常量池中的符号，**还没有成为java字符串**

```java
0: ldc           #2                  // String a
2: astore_1
3: ldc           #3                  // String b
5: astore_2
6: ldc           #4                  // String ab
8: astore_3
9: return
```

当执行到 ldc #2 时，会把符号 a 变为 “a” 字符串对象，**并放入串池中**（hashtable结构 不可扩容）

当执行到 ldc #3 时，会把符号 b 变为 “b” 字符串对象，并放入串池中

当执行到 ldc #4 时，会把符号 ab 变为 “ab” 字符串对象，并放入串池中

最终**StringTable [“a”, “b”, “ab”]**

**注意**：字符串对象的创建都是**懒惰的**，只有当运行到那一行字符串且在串池中不存在的时候（如 ldc #2）时，该字符串才会被创建并放入串池中。

使用拼接**字符串变量对象**创建字符串的过程

```java
public class StringTableStudy {
    public static void main(String[] args) {
        String a = "a";
        String b = "b";
        String ab = "ab";
        //拼接字符串对象来创建新的字符串
        String ab2 = a+b; 
    }
}
```

反编译后的结果

```java
	 Code:
      stack=2, locals=5, args_size=1
         0: ldc           #2                  // String a
         2: astore_1
         3: ldc           #3                  // String b
         5: astore_2
         6: ldc           #4                  // String ab
         8: astore_3
         9: new           #5                  // class java/lang/StringBuilder
        12: dup
        13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        16: aload_1
        17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String
;)Ljava/lang/StringBuilder;
        20: aload_2
        21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String
;)Ljava/lang/StringBuilder;
        24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/Str
ing;
        27: astore        4
        29: return
```

通过拼接的方式来创建字符串的**过程**是：StringBuilder().append(“a”).append(“b”).toString() 也就是 new String("ab")，因为 StringBuilder 的 toString 方法的底层就是 new String。

```java
// StringBuilder类中的方法
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

最后的toString方法的返回值是一个**新的字符串**，但字符串的**值**和拼接的字符串一致，但是两个不同的字符串，**一个存在于串池之中，一个存在于堆内存之中**

```java
String ab = "ab";
String ab2 = a+b;
//结果为false,因为ab是存在于串池之中，ab2是由StringBuffer的toString方法所返回的一个对象，存在于堆内存之中
System.out.println(ab == ab2);
```

使用**拼接字符串常量对象**的方法创建字符串

```java
public class StringTableStudy {
    public static void main(String[] args) {
        String a = "a";
        String b = "b";
        String ab = "ab";
        String ab2 = a+b;
        //使用拼接字符串的方法创建字符串
        String ab3 = "a" + "b";
    }
}
```

反编译后的结果

```java
 	  Code:
      stack=2, locals=6, args_size=1
         0: ldc           #2                  // String a
         2: astore_1
         3: ldc           #3                  // String b
         5: astore_2
         6: ldc           #4                  // String ab
         8: astore_3
         9: new           #5                  // class java/lang/StringBuilder
        12: dup
        13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        16: aload_1
        17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String
;)Ljava/lang/StringBuilder;
        20: aload_2
        21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String
;)Ljava/lang/StringBuilder;
        24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/Str
ing;
        27: astore        4
        //ab3初始化时直接从串池中获取字符串
        29: ldc           #4                  // String ab
        31: astore        5
        33: returnCopy
```

- 使用**拼接字符串常量**的方法来创建新的字符串时，因为**内容是常量，javac在编译期会进行优化，结果已在编译期确定为ab**，而创建ab的时候已经在串池中放入了“ab”，所以ab3直接从串池中获取值，所以进行的操作和 ab = “ab” 一致。
- 使用**拼接字符串变量**的方法来创建新的字符串时，因为内容是变量，只能**在运行期确定它的值，所以需要使用StringBuilder来创建**，在堆中

##### intern方法 1.8

调用字符串对象的intern方法，会将该字符串对象尝试放入到串池中

- 如果串池中没有该字符串对象，则放入成功
- 如果有该字符串对象，则放入失败

无论放入是否成功，都会返回**串池中**的字符串对象

**注意**：此时如果调用intern方法成功，堆内存与串池中的字符串对象是同一个对象；如果失败，则不是同一个对象

**例1**

```java
public class Main {
    public static void main(String[] args) {
        //"a" "b" 被放入串池中，str则存在于堆内存之中
        String str = new String("a") + new String("b");
        //调用str的intern方法，这时串池中没有"ab"，则会将该字符串对象str放入到串池中，并且返回"ab"
        String st2 = str.intern();
        //给str3赋值，因为此时串池中已有"ab"，则直接将串池中的内容返回
        String str3 = "ab";
        //因为str被放入了串池，所以以下两条语句打印的都为true
        System.out.println(str == st2);
        System.out.println(str == str3);
    }
}
```

**例2**

```java
public class Main {
    public static void main(String[] args) {
        //此处创建字符串对象"ab"，因为串池中还没有"ab"，所以将其放入串池中
        String str3 = "ab";
        //"a" "b" 被放入串池中，str则存在于堆内存之中
        String str = new String("a") + new String("b");
        //此时因为在创建str3时，"ab"已存在于串池中，所以str放入串池失败，但是会返回串池中的"ab"
        String str2 = str.intern();
        //false str是堆中的,str2是串池中的
        System.out.println(str == str2);
        //false str是堆中的,str3是串池中的
        System.out.println(str == str3);
        //true
        System.out.println(str2 == str3);
    }
}
```

##### intern方法 1.6

调用字符串对象的intern方法，会将该字符串对象尝试放入到串池中

- 如果串池中没有该字符串对象，会将该字符串对象复制一份，再**将副本**放入到串池中
- 如果有该字符串对象，则放入失败

无论放入是否成功，都会返回**串池中**的字符串对象

面试题

```java
String s1 = "a"; //"a"放入串池
String s2 = "b"; //"b"放入串池
String s3 = "a" + "b";//"ab"放入串池
String s4 = s1 + s2;//s4为"ab",但是在堆中
String s5 = "ab";//直接从串池中拿到"ab"
String s6 = s4.intern();//将s4放入串池,但是串池中有了"ab"因此放入失败,因此s4仍然在堆中
// 问
System.out.println(s3 == s4);//false
System.out.println(s3 == s5);//true
System.out.println(s3 == s6);//false
String x2 = new String("c") + new String("d");//"c"和"d"放入串池,x2为"cd"在堆中
String x1 = "cd";//将"cd"放入串池
x2.intern();//将x2放入串池,但是串池中有了"cd"因此放入失败,因此x2仍在堆中
System.out.println(x1 == x2);//false
// 问，如果调换了【倒数2,3行代码】的位置呢?
// 答:x2能放入串池,返回true
// 问: 如果是jdk1.6呢
// 答:肯定是false,因为1.6中,如果串池中没有那么会复制一份x2的副本,将副本放入串池 x1==x2返回false,如果串池中之前就有也返回false
```

**总结**

StringTable 特性：

- 常量池中的字符串仅是符号，第一次用到时才变为对象
- 利用串池的机制，来避免重复创建字符串对象
- 字符串变量拼接的原理是 StringBuilder （1.8）
- 字符串常量拼接的原理是编译期优化
- 可以使用 intern 方法，主动将串池中还没有的字符串对象放入串池
  - 1.8 将这个字符串对象尝试放入串池，如果有则并不会放入，如果没有则放入串池， 会把串池中的对象返回
  - 1.6 将这个字符串对象尝试放入串池，如果有则并不会放入，如果没有会把此对象复制一份，放入串池， 会把串池中的对象返回

#### StringTable 垃圾回收

StringTable在内存紧张时，会发生垃圾回收

#### StringTable调优

- 因为StringTable是由HashTable实现的，所以可以**适当增加HashTable桶的个数**，来减少字符串放入串池所需要的时间

  ```
  -XX:StringTableSize=xxxxCopy
  ```

- 考虑是否需要将字符串对象入池

  可以通过**intern方法减少重复入池**

StringTable 的底层是类似于 Hashtable 的实现，也就是哈希表，数组加链表的结构，数组上的每个索引的位置称为桶，它是以哈希表的结构来存储数据的。

适当的将 StringTable 的桶的总个数调大也就可以理解为让数组长度适当变长，让它有一个更好的哈希分布，减少哈希冲突，让我们的 StringTable 串池字符串常量池的性能得到提升。

如果你的应用里有大量的字符串，而且这些字符串可能会存在重复的问题，那么我们可以让字符串入池来减少字符串对象的个数，节约我们的堆内存的使用。

### 直接内存

- 属于操作系统，常见于NIO操作时，**用于数据缓冲区**
- 分配回收成本较高，但读写性能高
- 不属于 JVM 内存结构中，不受 JVM 内存回收管理

#### 文件读写流程

**普通的 IO 操作**：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image15.png)

**使用了DirectBuffer**：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image16.png)

直接内存是操作系统和Java代码**都可以访问的一块区域**，无需将代码从系统内存复制到Java堆内存，少了一次 copy 操作，从而提高了效率。

#### 释放原理

直接内存的回收不是通过 JVM 的垃圾回收来释放的，而底层是通过 **unsafe.freeMemory** 来释放

通过

```java
//通过ByteBuffer申请分配1G的直接内存
ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_1G);
// ... 一些操作
// 显式回收 ByteBuffer
System.gc();
```

上面代码中使用了 System.gc() ，让 JVM 回收了 ByteBuffer。此时我们查看电脑的内存占用，发现直接内存没了。。。wtf？？不是说 JVM 不能回收直接内存吗？那调用了 gc() 之后为何会释放直接内存？

这就要从底层一步一步解释了。。。

**allocateDirect的源码实现**

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

DirectByteBuffer类

源码中的 unsafe 对象解析：这个对象通过反射可以得到，是极其底层的，记得 cas 中也用到了 unsafe 对象。其实直接内存的分配和释放，核心就是靠这个 unsafe 对象。

```java
DirectByteBuffer(int cap) {   // package-private
   
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 底层：直接内存的申请必须依靠unsafe
        base = unsafe.allocateMemory(size); //申请内存 返回直接内存的内存地址
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);// 这行代码和前面的try中的代码必须组合到一起使用才可完成直接内存的分配
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // Cleaner是一个虚引用类型，this就为虚引用的实际对象，在此也就是ByteBuffer
    // 在此可以看到 cleaner 有一个回调函数Deallocator，这个回调函数其实是一个任务，因为底层实现了Runnable，在run方法中主动调用了unsafe的freeMemory方法。注：直接内存的释放必须主动调用unsafe的freeMemory方法！！
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); //通过虚引用，来实现直接内存的释放，this为虚引用的实际对象
    att = null;
}
```

这里就解释了为何调用了 gc() ，不受 JVM 内存管理的直接内存会释放：当调用了 gc 后，不被引用的 ByteBuffer 对象会被 JVM 垃圾回收，当它被回收掉时，它就会触发虚引用对象 cleaner 中的 clean 方法，clean() 方法中又调用了 run 方法，run 方法中又主动调用了 unsafe 的 freeMemory 方法，进而直接内存会释放。

```java
public void clean() {
    if (remove(this)) {
        try {
            this.thunk.run(); //调用run方法
        } catch (final Throwable var2) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    if (System.err != null) {
                        (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                    }

                    System.exit(1);
                    return null;
                }
            });
        }
    }
}
```

这个 clean 方法不是在主线程中被执行的，它是在后台的一个叫 ReferenceHandler 的这个线程中执行的，这个线程是专门去检测虚引用对象的，一旦虚引用对象所关联的那个实际对象也就是那个 ByteBuffer 被 gc 回收掉之后，它就会调用到虚引用对象里的 clean 方法，然后去执行任务对象也就是 run 方法里的内容，run 方法里面就会主动调用 unsafe 的 freeMemory 方法，从而释放直接内存。

对应对象的run方法

```java
public void run() {
    if (address == 0) {
        // Paranoia
        return;
    }
    unsafe.freeMemory(address); //释放直接内存中占用的内存
    address = 0;
    Bits.unreserveMemory(size, capacity);
}
```

##### 直接内存的回收机制总结

- 使用了Unsafe类来完成直接内存的分配回收，回收需要主动调用freeMemory方法
- ByteBuffer的实现内部使用了Cleaner（虚引用）来检测ByteBuffer。一旦ByteBuffer被垃圾回收，那么ReferenceHandler线程会检测到，并且调用Cleaner的clean方法，clean方法又调用run方法，然后run中又调用freeMemory来释放内存。

所以直接内存的释放借助 Java 中的虚引用机制（四大引用之一）。

#### 禁用显式回收

在涉及到 JVM 调优时，经常会在运行前在 idea 中加入`-XX:+DisableExplicitGC`，这个参数的作用就是禁用显式回收，也就是让代码中的`System.gc()` 无效。因为 System.gc() 是显式的垃圾回收，触发的是 Full GC，是比较影响性能的一种回收，因为 Full GC 不光回收新生代还要回收老年代，会造成程序的暂停时间较长。因此加这个参数，加这个参数的话对别的代码影响倒不大，但是会影响到直接内存的回收机制。

如果我们不能显式的回收掉 ByteBuffer 的话，那么 ByteBuffer 只能等到真正的垃圾回收时才会被清理，所以造成了直接内存不能及时被释放，会让直接内存占用较大，长时间得不到回收释放。

解决：直接用 unsafe 的 freeMemory() 来手动释放直接内存 Direct Memory，不等 ReferenceHandler 检测，直接手动管理。

## 垃圾回收

### 如何判断对象可以回收

#### 引用计数法

是指只要一个对象被其它变量所引用，那就让这个对象的计数 +1，如果被引用两次，那么它的计数就会加到 2，如果某个变量不再引用它了，那么让它的计数 -1，当这个对象的引用计数变为 0 的时候，就可以作为垃圾被回收。

弊端：循环引用时，两个对象的计数都为1，导致两个对象都无法被释放

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image14.png)

#### 可达性分析算法

JVM 采用了这种算法进行垃圾回收。这种算法首先会确定一系列的根对象（GC root），也就是肯定不能当成垃圾被回收的对象。在垃圾回收之前，会先对堆内存中的所有对象进行扫描，看每一个对象是否被根对象所直接或间接的引用，如果是，则不能被回收；反之，可作为垃圾将来被回收。

```java
// 在我么当前活动线程的执行过程中，局部变量所引用的对象可作为根对象
// 方法参数中引用的字符串数组对象也是一个根对象
public class Demo {
    public static void main(String[] args) throws IOException {
        // list 为局部对象，存在于栈帧中，它引用的new ArrayList<>()在堆中。
        List<Object> list = new ArrayList<>();
        list.add("a");
        list.add("b");
        System.out.println(1);
        System.in.read();// 让程序暂停运行 按enter继续
        
        list = null;
        System.out.println(2);
        System.in.read();
        System.out.println("end...");
    }
}
```

总结：

- JVM中的垃圾回收器通过**可达性分析**来探索所有存活的对象
- 扫描堆中的对象，看能否沿着GC Root对象为起点的引用链找到该对象，如果**找不到，则表示可以回收**
- 可以作为GC Root的对象
  - 虚拟机栈（栈帧中的本地变量表）中引用的对象。　
  - 方法区中类静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中JNI（即一般说的Native方法）引用的对象

#### 五种引用

下图中实线代表强引用。

B、C 对象都是根对象。

软引用、弱引用、虚引用、终结器引用也都是对象。

A2 是被软引用对象引用的对象。

A3 是被弱引用对象引用的对象。

ByteBuffer 是被虚引用对象引用的对象。

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150800.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150800.png)

##### 强引用

只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收。

- 如上图B、C对象都不引用A1对象时，A1对象才能被回收

##### 软引用

当GC Root指向**软引用对象**时，在**内存不足时**，会**回收软引用对象所引用的对象**。

- 如上图如果B对象不再引用A2对象且内存不足时，软引用所引用的A2对象就会被回收。

###### 软引用的使用

```java
public class Demo {

    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) throws IOException {
        soft();
    }

    /**
     * 软引用
     */
    public static void soft() {
        // list-->SoftReference-->byte[]
		//使用软引用对象 list和SoftReference是强引用，而SoftReference和byte数组则是软引用
        List<SoftReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            SoftReference<byte[]> reference = new SoftReference<>(new byte[_4MB]);
            System.out.println(reference.get());
            list.add(reference);
            System.out.println(list.size());
        }
        System.out.println("循环结束：" + list.size());
        for (SoftReference<byte[]> reference : list) {
            System.out.println(reference.get());
        }
    }
}
```

如果在垃圾回收时发现内存不足，在回收软引用所指向的对象时，**软引用本身不会被清理**

如果想要**清理软引用**，需要使**用引用队列**

```java
public class Demo {

    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) throws IOException {
        soft();
    }

    /**
     * 软引用配合引用队列
     */
    public static void soft() {
        // list-->SoftReference-->byte[]
        List<SoftReference<byte[]>> list = new ArrayList<>();

        // 使用引用队列，用于移除引用为空的软引用对象
        ReferenceQueue<byte[]> queue = new ReferenceQueue<>();

        for (int i = 0; i < 5; i++) {
            // 关联了引用队列，当软引用所关联的 byte[] 被回收时，软引用自己会加入到 queue 中去
            SoftReference<byte[]> reference = new SoftReference<>(new byte[_4MB], queue);
            System.out.println(reference.get());
            list.add(reference);
            System.out.println(list.size());
        }
        // poll 方法：取到队列中最先放入的元素 将它移出队列
        // 遍历引用队列，如果有元素，则移除
        Reference<? extends byte[]> poll = queue.poll();
        while (poll != null) {
            // 引用队列不为空，则从集合中移除该元素
            list.remove(poll);
            // 移动到引用队列中的下一个元素
            poll = queue.poll();
        }

        for (SoftReference<byte[]> reference : list) {
            System.out.println(reference.get());
        }
    }
}

```

**大概思路为：**查看引用队列中有无软引用，如果有，则将该软引用从存放它的集合中移除（这里为一个list集合）

##### 弱引用

只有弱引用引用该对象时，在垃圾回收时，**无论内存是否充足**，都会回收弱引用所引用的对象

- 如上图如果B对象不再引用A3对象，则A3对象会被回收

**弱引用的使用和软引用类似**，只是将 **SoftReference 换为了 WeakReference**

```java
public class Demo {

    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) throws IOException {
        weak();
    }

    /**
     * 弱引用
     */
    public static void weak() {
        // list-->WeakReference-->byte[]

        List<WeakReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            WeakReference<byte[]> reference = new WeakReference<>(new byte[_4MB]);
            System.out.println(reference.get());
            list.add(reference);
            System.out.println(list.size());
        }
        System.out.println("循环结束：" + list.size());
        for (WeakReference<byte[]> reference : list) {
            System.out.println(reference.get());
        }
    }
}
```

> 小总结：软引用本身也是一个对象，也占用内存，软引用可以关联一个引用队列（可选），如果软引用对象引用的对象被垃圾回收了，软引用对象本身不会被回收，而是进入一个引用队列（如果这个软引用关联了引用队列的话）。如果想对软引用对象占用的内存进行释放，要结合引用队列，从队列中遍历并找到，然后进一步判断软引用对象是否被强引用引用，如果没有的话可回收释放。弱引用也是一样的道理，弱引用和软引用很相似，唯一的不同就是：软引用引用的对象（该对象没有被强引用），会在内存不足的时候被回收；而弱引用引用的对象（该对象没有被强引用），**无论内存是否充足**，都会回收弱引用所引用的对象。

##### 虚引用

必须关联一个引用队列。虚引用的一个典型案例就是直接内存的释放。

对于虚引用对象，前面直接内存的地方就提到过。申请直接内存的时候，创建 ByteBuffer 的实现类对象时，就会创建一个名为 cleaner 的虚引用对象，ByteBuffer 会被分配一块直接内存，并且会将直接内存的内存地址传递给虚引用对象，这样的一个操作是为何？将来我的 ByteBuffer 一旦没有强引用所引用它了，ByteBuffer 自己会被垃圾回收掉，但是只有它被垃圾回收了还不够，因为分配给它的直接内存并不能被垃圾回收管理，所以我们要在 ByteBuffer 被回收时，让虚引用对象进入到引用队列，而虚引用所在的引用队列由一个叫做 ReferenceHandler 的线程来定时的到这个引用队列中找有无一个新入队的 cleaner，如果有，这个线程就会调用虚引用对象 cleaner 的 clean() 方法，clean 方法就会根据前面记录的直接内存的地址，调用 unsafe.freeMemory()，将直接内存释放掉，这样就保证不会造成因为直接内存一直不释放而导致的内存泄漏。

当虚引用对象所引用的对象被回收以后，虚引用对象就会被放入引用队列中，调用虚引用的方法

- 虚引用的一个体现是**释放直接内存所分配的内存**，当引用的对象ByteBuffer被垃圾回收以后，虚引用对象Cleaner就会被放入引用队列中，然后调用Cleaner的clean方法来释放直接内存
- 如上图，B对象不再引用ByteBuffer对象，ByteBuffer就会被回收。但是直接内存中的内存还未被回收。这时需要将虚引用对象Cleaner放入引用队列中，然后调用它的clean方法来释放直接内存

总结：在虚引用引用的对象（比如 ByteBuffer）被垃圾回收时，虚引用对象（比如 cleaner）自己就会进入到引用队列，从而间接地由一个线程（ReferenceHandler）来调用虚引用对象 cleaner 的 clean 方法，然后调用 unsafe.freeMemory() 来释放直接内存。

##### 终结器引用

所有的类都继承自Object类，Object类有一个finalize方法。当某个对象不再被其他的对象所引用时，会先将终结器引用对象放入引用队列中，然后根据终结器引用对象找到它所引用的对象，然后调用该对象的finalize方法。调用以后，该对象就可以被垃圾回收了

- 如上图，B对象不再引用A4对象。这是终结器对象就会被放入引用队列中，引用队列会根据它，找到它所引用的对象。然后调用被引用对象的finalize方法。调用以后，该对象就可以被垃圾回收了

总结：终结器引用回收的效率低，因为在第一次回收时，不能真正的回收掉 A4 对象，而是先将终结器引用加入到引用队列，并且处理这个引用队列的线程优先级很低，被执行的机会很少，所以会造成 A4 对象的 finalize() 方法迟迟不被调用，这个队列占的内存也迟迟得不到释放，这也是为何我们不推荐使用 finalize() 来释放资源的原因。

##### 引用队列

- 软引用和弱引用**可以配合**引用队列
  - 在**弱引用**和**虚引用**所引用的对象被回收以后，会将这些引用放入引用队列中，方便一起回收这些软/弱引用对象
- 虚引用和终结器引用**必须配合**引用队列
  - 虚引用和终结器引用在使用时会关联一个引用队列

**大总结：**

1. 强引用
   - 只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收

2. 软引用（SoftReference）
   - 仅有软引用引用该对象时，在垃圾回收后，内存仍不足时会再次出发垃圾回收，回收软引用对象
   - 可以配合引用队列来释放软引用自身

3. 弱引用（WeakReference）
   - 仅有弱引用引用该对象时，在垃圾回收时，无论内存是否充足，都会回收弱引用对象
   - 可以配合引用队列来释放弱引用自身

4. 虚引用（PhantomReference）
   - 必须配合引用队列使用，主要配合 ByteBuffer 使用，被引用对象回收时，会将虚引用入队，由 Reference Handler 线程调用虚引用相关方法释放直接内存

5. 终结器引用（FinalReference）
   - 无需手动编码，但其内部配合引用队列使用，在垃圾回收时，终结器引用入队（被引用对象暂时没有被回收），再由 Finalizer 线程通过终结器引用找到被引用对象并调用它的 finalize 方法，第二次 GC 时才能回收被引用对象。

### 垃圾回收算法

#### 标记-清除

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image18.png)

**定义**：标记清除算法顾名思义，是指在虚拟机执行垃圾回收的过程中，先采用标记算法确定可回收对象，然后垃圾收集器根据标识清除相应的内容，给堆内存腾出相应的空间

- 这里的腾出内存空间并不是将内存空间的字节清0，而是记录下这段内存的起始结束地址，下次分配内存的时候，会直接**覆盖**这段内存

**缺点**：**容易产生大量的内存碎片**，可能无法满足大对象的内存分配，一旦导致无法分配对象，那就会导致jvm启动gc，一旦启动gc，我们的应用程序就会暂停，这就导致应用的响应速度变慢

#### 标记-整理

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image19.png)

标记-整理 会将不被GC Root引用的对象回收，清楚其占用的内存空间。然后整理剩余的对象，可以有效避免因内存碎片而导致的问题，但是因为整体需要消耗一定的时间，所以效率较低

#### 复制

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image20.png)

大致流程：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image21.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image22.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image23.png)

将内存分为等大小的两个区域，FROM和TO（TO中为空）。先将被GC Root引用的对象从FROM放入TO中，再回收不被GC Root引用的对象。然后交换FROM和TO。这样也可以避免内存碎片的问题，但是会占用双倍的内存空间。

### 分代回收

前面说的几种垃圾回收算法，并不是单独工作的，它们是协同工作的，具体实现是 JVM 中的分代垃圾回收机制，它将整个堆内存大区域划分成 2 块：新生代、老年代，不同区域的不同的垃圾回收算法就可以更有效的进行垃圾回收。其中新生代一般存放一些用完了就可以丢掉的对象；老年代一般存放一些长时间使用的对象，这里的长时间就是指那些经历了新生代阈值次数的垃圾回收还幸存下来的对象，阈值一般为 15，但是有时候可能会不到 15 就提前放入老年代。新生代中又分了三个区域，分别为伊甸园，幸存区 From、幸存区 To，它们仨的默认比例为 8 : 1 : 1，新创建的对象都被放在了**新生代的伊甸园**中，当伊甸园的内存不足的时候，就会进行一次垃圾回收，这时的回收叫做 Minor GC，Minor GC 触发后，会采用可达性分析算法沿着 GC root（根对象）的引用链找，看这些对象是有引用的还是可以作为垃圾进行一次标记动作的，标记成功之后，采用复制算法，将伊甸园和幸存区 From 中存活的对象复制到幸存区 To 中，并且让幸存的对象寿命 +1（最开始的时候寿命为 0）。然后将伊甸园和幸存区 From 中的垃圾清除。现在的状况就是：伊甸园是干净的，幸存区 From 是干净的，幸存区 To 中存放了 Minor GC 之后存活的对象，然后将幸存区 From 和幸存区 To 的位置调换。这就是复制回收算法的思想。这样新创建的对象就可以继续放入伊甸园了。

注：minor gc 会引发 stop the world，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行。原因就是一个线程在垃圾回收过程中，会涉及到对象的复制，可能对象的地址会发生变化，如果其它线程不阻塞的话就可能得到一个错误的地址。

但是如果经过了几轮的垃圾回收，达到了一定的阈值，仍然幸存的对象，那么这些对象就会放入老年代中，等对象越来越多老年代内存满了，当老年代空间不足，会先尝试触发 minor gc，如果之后空间仍不足，那么触发 full gc，STW 的时间更长。

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150931.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150931.png)

#### 回收流程

新创建的对象都被放在了**新生代的伊甸园**中

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150939.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150939.png)

当伊甸园中的内存不足时，就会进行一次垃圾回收，这时的回收叫做 **Minor GC**

Minor GC 会将**伊甸园和幸存区FROM**存活的对象**先**复制到**幸存区 TO**中， 并让其**寿命加1**，再**交换两个幸存区**

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150946.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150946.png)

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150955.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608150955.png)

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151002.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151002.png)

再次创建对象，若新生代的伊甸园又满了，则会**再次触发 Minor GC**（会触发 **stop the world**， 暂停其他用户线程，只让垃圾回收线程工作），这时不仅会回收伊甸园中的垃圾，**还会回收幸存区中的垃圾**，再将活跃对象复制到幸存区TO中。回收以后会交换两个幸存区，并让幸存区中的对象**寿命加1**。

如果幸存区中的对象的**寿命超过某个阈值**（最大为15，4bit），就会被**放入老年代**中

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151018.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151018.png)

**总结：**

- 对象首先分配在伊甸园区域
- 新生代空间不足时，触发 minor gc，伊甸园和 from 存活的对象使用 copy 复制到 to 中，存活的对象年龄加 1并且交换 from to
- minor gc 会引发 stop the world，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行
- 当对象寿命超过阈值时，会晋升至老年代，最大寿命是15（4bit）
- 当老年代空间不足，会先尝试触发 minor gc，如果之后空间仍不足，那么触发 full gc，STW 的时间更长

#### 老年代空间分配担保规则

参考原文：https://blog.csdn.net/qq_21588061/article/details/114290212

如果年轻代里大量对象存活，确实自己的Survivor区放不下了，必须转移到老年代去

但是如果老年代里空间也不够放这些对象，改怎么办呢？

首先，在执行任何一次Minor GC之前，JVM都会先检查一些老年代可用的内存空间，是否大于年轻代所有对象的总大小，为什么呢？因为最极端的情况下，可能年轻代Minor GC之后，所有对象都存活下来了，那岂不是年轻代所有对象全部进入老年代

![](https://img-blog.csdnimg.cn/img_convert/5534854bd4a73aa7ca89ec1ebe875637.png)

如果说发现老年代内存大小是大于年轻代所有对象的，此时就可以放心大胆地对年轻代发起一次Minor GC了，因为即使 Minor GC 之后所有对象都存活，Survivor 区放不下了，也可以转移到老年代去。

但是假如执行Minor GC之前，发现老年代的可用内存已经小于了年轻代的全部对象大小了。

这个时候是不是有可能在Minor GC之后年轻代的对象全部存活下来，全部需要转移到老年代中去，但是老年代内存空间又不够？理论上是有可能的。

所以假如Minor GC之前，发现老年代的可用内存已经小于了年轻代的全部对象大小，就会看一个“-XX:-HandlePromotionFailure”的参数是否设置了

如果设置了这个参数，那么就会继续尝试进行下一步判断。下一步判断，就会看看老年代的内存大小，是否大于之前每一次Minor GC后进去老年代对象的平均大小。

![](https://img-blog.csdnimg.cn/img_convert/63b4c455669455600d9aea000fdbe9b5.png)

但是如果上面步骤判断失败了，或者是“-XX:-HandlePromotionFailure”参数没设置，此时就会直接触发一次“Full GC”，

就是对老年代进行垃圾回收，尽量腾出来一些内存空间，然后再执行Minor GC。

如果上面两个步骤都判断成功，那么就可以冒点风险尝试一下Minor GC，此时进行Minor GC有几种可能：

①Minor GC过后，剩余的存活对象的大小，小于Survivor区的大小，那么此时存活对象进入Survivor区即可

②Minor GC过后，剩余的存活对象的大小，大于Survivor区的大小，但是小于老年代可用内存大小，就直接进入老年代即可

③Minor GC过后，剩余的存活对象的大小，大于Survivor区的大小，同时大于老年代可用内存大小，就是比以往Minor GC之后剩余对象大小的平均值大，此时就会发生“Handle Promotion Failure”的情况，这个时候就会出发一次“Full GC”。

Full GC就是对老年代进行垃圾回收，同时也一般会对年轻代进行垃圾回收。

如果Full GC之后，老年代还是没有足够空间存放Minor GC过后的剩余存活对象，此时就会导致所谓的“OOM”内存溢出了。

所以，所谓的JVM优化，`就是尽可能的让对象都在年轻代里分配和回收，尽量别让太多对象频繁进入老年代，避免频繁对老年代进行垃圾回收，同时给系统充足的内存大小，避免年轻代频繁的进行垃圾回收`。

#### GC 分析

##### 大对象处理策略

当遇到一个**较大的对象**时，就算新生代的**伊甸园**为空，也**无法容纳该对象**时，会将该对象**直接晋升为老年代**，这就是阈值 15 的例外情况。

##### 线程内存溢出

某个线程的内存溢出了而抛异常（out of memory），不会让其他的线程结束运行

这是因为当一个线程**抛出OOM异常后**，**它所占据的内存资源会全部被释放掉**，从而不会影响其他线程的运行，**进程依然正常**

### 垃圾回收器

#### 相关概念

**并行收集**：指多条垃圾收集线程并行工作，但此时**用户线程仍处于等待状态**。

**并发收集**：指用户线程与垃圾收集线程**同时工作**（在同一时间段内交替执行）。**用户程序在继续运行**，而垃圾收集程序运行在另一个CPU上

**吞吐量**：即CPU用于**运行用户代码的时间**与CPU**总消耗时间**的比值（吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 垃圾收集时间 )），也就是。例如：虚拟机共运行100分钟，垃圾收集器花掉1分钟，那么吞吐量就是99%

#### 串行

串行指的是垃圾回收线程和用户工作线程串行，在垃圾回收的时候，工作线程全部暂停。

- 单线程
- 内存较小，个人电脑（CPU核数较少）

`-XX:+UseSerialGC = Serial + SerialOld`

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image25.png)

**安全点**：让其他线程都在这个点停下来，以免垃圾回收时移动对象地址，使得其他线程找不到被移动的对象

因为是串行的，所以只有一个垃圾回收线程。且在该线程执行回收工作时，其他线程进入**阻塞**状态

##### Serial 收集器

Serial收集器是最基本的、发展历史最悠久的收集器

**特点：**单线程、简单高效（与其他收集器的单线程相比），采用**复制算法**。对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程手机效率。收集器进行垃圾回收时，必须暂停其他所有的工作线程，直到它结束（Stop The World）

##### ParNew 收集器

ParNew收集器其实就是Serial收集器的多线程版本

**特点**：多线程、ParNew收集器默认开启的收集线程数与CPU的数量相同，在CPU非常多的环境中，可以使用-XX:ParallelGCThreads参数来限制垃圾收集的线程数。和Serial收集器一样存在Stop The World问题

##### Serial Old 收集器

Serial Old是Serial收集器的老年代版本

**特点**：同样是单线程收集器，采用**标记-整理算法**

#### 吞吐量优先

- 多线程垃圾回收
- 堆内存较大，多核CPU
- 单位时间内，STW（stop the world，停掉其他所有工作线程）时间最短
- **JDK1.8默认使用**的垃圾回收器

`-XX:+UseParallelGC ~ -XX:+UseParallelOldGC`

`-XX:+UseAdaptiveSizePolicy`

`-XX:GCTimeRatio=ratio`

`-XX:MaxGCPauseMillis=ms`

`-XX:ParallelGCThreads=n`

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image26.png)

##### Parallel Scavenge 收集器

与吞吐量关系密切，故也称为吞吐量优先收集器

**特点**：属于新生代收集器也是采用**复制算法**的收集器（用到了新生代的幸存区），又是并行的多线程收集器（与ParNew收集器类似）

该收集器的目标是达到一个可控制的吞吐量。还有一个值得关注的点是：**GC自适应调节策略**（与ParNew收集器最重要的一个区别）

**GC自适应调节策略**：Parallel Scavenge收集器可设置-XX:+UseAdptiveSizePolicy参数。当开关打开时**不需要**手动指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRation）、晋升老年代的对象年龄（-XX:PretenureSizeThreshold）等，虚拟机会根据系统的运行状况收集性能监控信息，动态设置这些参数以提供最优的停顿时间和最高的吞吐量，这种调节方式称为GC的自适应调节策略。

Parallel Scavenge收集器使用两个参数控制吞吐量：

- XX:MaxGCPauseMillis 控制最大的垃圾收集停顿时间
- XX:GCRatio 直接设置吞吐量的大小

##### Parallel Old 收集器

是Parallel Scavenge收集器的老年代版本

**特点**：多线程，采用**标记-整理算法**（老年代没有幸存区）

#### 响应时间优先

- 多线程
- 堆内存较大，多核CPU
- 尽可能让单次STW时间变短（尽量不影响其他线程运行）

`-XX:+UseConcMarkSweepGC ~ -XX:+UseParNewGC ~ SerialOld`

`-XX:ParallelGCThreads=n ~ -XX:ConcGCThreads=threads`

`-XX:CMSInitiatingOccupancyFraction=percent`

`-XX:+CMSScavengeBeforeRemark`

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image27.png)

##### CMS 收集器

Concurrent Mark Sweep，一种以获取**最短回收停顿时间**为目标的**老年代**收集器

**特点**：基于**标记-清除算法**实现。并发收集、低停顿，但是会产生内存碎片

**应用场景**：适用于注重服务的响应速度，希望系统停顿时间最短，给用户带来更好的体验等场景下。如web程序、b/s服务

**CMS收集器的运行过程分为下列4步：**

**初始标记**：标记GC Roots能直接到的对象。速度很快但是**仍存在Stop The World问题**

**并发标记**：进行GC Roots Tracing 的过程，找出存活对象且用户线程可并发执行

**重新标记**：为了**修正并发标记期间**因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在Stop The World问题

**并发清除**：对标记的对象进行清除回收

CMS为了减轻STW带来的影响，采用了如上四个阶段来进行垃圾回收，其中初始标记和重新标记的耗时很短，会STW但是影响不大，然后并发标记和并发清理阶段的耗时很长，但是可以和工作线程并发运行的，因此对系统的影响不大。这就是CMS的大致的工作原理。

CMS收集器的垃圾回收过程（垃圾回收线程）是与用户线程一起**并发执行**的。

但是CMS的问题也显而易见：

- 并发回收垃圾导致CPU资源紧张。
- Concurrent Mode Failure问题：就是CMS由于是允许垃圾回收线程和工作线程并发执行的，因此需要给工作线程预留一定的内存，因此CMS并不是等到老年代满了才开始回收垃圾的，而是当老年代中的占用内存达到了老年代内存总大小的92%时就会自动开始使用标记清理算法回收垃圾，因此，还剩8%的剩余空间，预留8%的空间给并发清理期间，系统程序将一些新对象放入老年代中，但如果工作线程生成的对象经过Minor GC晋升到老年代的速度太快（大于并行清除阶段垃圾回收的速度并且占满剩余的8%内存）的极端情况发生，也就是说如果CMS并发清理期间，系统程序要放入老年代的对象大于了此时老年代的可用内存空间，这个时候会发生Concurrent Mode Failure，会造成 CMS 垃圾回收器并发失败，就是说并发垃圾回收失败了，我一边回收，你一边把对象放入老年代，内存都不够了，从而导致 CMS 垃圾回收器不能正常工作，会退化为 Serial Old 串行的老年代垃圾回收器。这样效率会很低，强行把系统程序STW，重新进行长时间的GC Roots追踪，标记出全部的存活对象，不允许新的对象产生。响应时间变长。

在并发清理阶段，CMS是回收之前没标记的对象，也就是回收之前的垃圾，但是由于并发清理阶段，工作线程和垃圾回收线程并行运行，工作线程会产生新的对象和新的垃圾，这些新产生的可能也进入到老年代了，但是本轮GC不管。因此不必担心没被标记的新对象到了老年代中被误清理的情况，但是这也意味着可能也会产生新的垃圾到老年代中，也即是我们提到的“浮动垃圾”。

为什么说本轮GC不管？是因为并发清理阶段新晋升到老年代的对象会被分配在FreeList中指定的合适区域，因此不会参与本轮的清理。因为CMS内部维护着一个数据结构，用来记录可以安全分配新对象的内存空间地址。

>  内存碎片问题：CMS的标记清理算法会造成大量的内存碎片，这样容易因为内存不连贯而频繁触发Full GC，因此CMS不是完全就仅仅用标记清理算法的，CMS有一个参数是“-XX:+UseCMSCompactAtFullCollection”，默认就是打开的。它的意思是在Full GC之后要再次进行“Stop the World”，停止工作线程，然后进行碎片整理，就是将存活的对象挪到一起，空出来大片连续的内存空间，避免内存碎片。
>
> 还有一个参数是“-XX:CMSFullGCsBeforeCompaction”，这个意思是执行多少次Full GC之后再执行一次内存碎片整理的工作，默认是0，意思就是每次Full GC之后都会进行一次内存整理。
>
> 内存碎片整理完之后，存活的对象都放在一起，然后空出来大片连续的内存空间可供使用。

#### G1

##### **定义**

Garbage First

JDK 9以后默认使用，而且替代了CMS 收集器

##### 适用场景

- 同时注重吞吐量和低延迟（响应时间）
- 超大堆内存（内存大的），会将堆内存划分为多个**大小相等**的区域
- 整体上是**标记-整理**算法，两个区域之间是**复制**算法

**相关参数**：JDK8 并不是默认开启的，所需要参数开启

```java
-XX:+UseG1GC
-XX:G1HeapRegionSize=size
-XX:MaxGCPauseMillis=time
```

##### G1垃圾回收阶段

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image28.png)

##### Young Collection

**分区算法region**：分代是按对象的生命周期划分，分区是将堆空间划分连续几个不同小区间，每一个小区间独立回收，可以控制一次回收多少个小区间，方便控制 GC 产生的停顿时间

E：伊甸园 S：幸存区 O：老年代

- 会 STW

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image29.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image30.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image31.png)

##### Young Collection + CM

注：CM：ConcurrentMark，并发标记

- 在 Young GC 时会**对 GC Root 进行初始标记**
- 在老年代**占用堆内存的比例**达到阈值时，对进行**并发标记**（不会 STW），阈值可以根据用户来进行设定

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image32.png)

##### Mixed Collection

会对 E、S、O 进行**全面的回收**，优先回收垃圾最多的区域。

- 最终标记：因为垃圾回收线程和工作线程可并发执行，垃圾回收时不会阻塞工作线程，所以可能会有浮动垃圾，因此需要最终标记
- **拷贝**存活

`-XX:MaxGCPauseMills:xxx` 用于指定最长的停顿时间

**问**：为什么有的老年代被拷贝了，有的没拷贝？

因为指定了最大停顿时间，如果对所有老年代都进行回收，耗时可能过高。为了保证时间不超过设定的停顿时间，会**回收最无价值的老年代**（回收后，能够得到更多内存）

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image33.png)

##### 垃圾回收器总结

1. SerialGC（垃圾回收线程串行，其它线程阻塞）
   - 新生代内存不足发生的垃圾收集 - minor gc
   - 老年代内存不足发生的垃圾收集 - full gc
2. ParallelGC（垃圾回收线程并行，其它线程阻塞）
   - 新生代内存不足发生的垃圾收集 - minor gc
   - 老年代内存不足发生的垃圾收集 - full gc
3. CMS（垃圾回收线程和用户的工作线程并发交替运行）
   - 新生代内存不足发生的垃圾收集 - minor gc
   - 老年代内存不足
4. G1（垃圾回收线程和用户的工作线程并发交替运行）
   - 新生代内存不足发生的垃圾收集 - minor gc
   - 老年代内存不足的时候，触发的垃圾回收要分两种情况：
     - G1在老年代内存不足时（老年代所占内存超过阈值）

       1. 如果垃圾产生速度慢于垃圾回收速度，不会触发Full GC
       2. 如果垃圾产生速度快于垃圾回收速度，便会触发Full GC

**看完儒猿技术窝JVM专栏中这部分的内容，对G1简单总结一下：**

首先强调一下，前面学习的几种垃圾回收器需要搭配使用，比如ParNew和CMS搭配使用，并且需要设置堆内存的大小（Xms和Xmx，一般设置成一样的值）和指定新生代的大小，那么剩下的就是老年代的大小。现在我们了解的G1垃圾回收器，是不需要设置新生代的大小的，只需要设置堆内存的大小就行，关于堆内存中的新生代和老年代，G1可以自动计算和设置，当然也可以手动更改，但是一般都用默认值就好。

其实G1垃圾回收器设计的思想，主要是将堆内存拆分为很多个小的Region，然后新生代和老年代各自对应一些Region，回收的时候尽可能的挑选停顿时间最短以及回收对象最多的Region，尽量保证达到我们指定的垃圾回收系统停顿时间。其实大家通过之前的学习，有关JVM的优化思路，都明白一点，我们对内存的合理分配，优化一些参数，就是为了尽可能的减少Minor GC和Full GC，尽量减少GC带来的系统停顿，避免影响系统处理请求。但是现在我们可以直接给G1指定，在一个时间内，垃圾回收导致的系统卡顿时间不能超过多久，由G1全权给你负责保证达到这个目标。G1怎么做到垃圾回收导致的系统停顿可控的呢？就是G1会追踪每个Region里的回收价值，G1会搞清楚每个Region里的对象有多少是垃圾，如果对这个Region进行垃圾回收需要耗费多长时间，可以回收掉多少垃圾。因此简单来说，G1可以做到让我们自己设定垃圾回收对系统的影响，G1通过将内存拆分为大量的小Region，以及追踪每个Region中可以回收的对象大小和预估时间，最后在垃圾回收的时候，尽量把垃圾回收对系统造成的影响控制在我们指定的时间范围内，同时在有限的时间内尽量回收尽可能多的垃圾。这就是G1的核心设计思路。

可以使用“`-XX:+UseG1GC`”来指定使用G1垃圾回收器，然后JVM启动的时候一旦发现使用的是G1垃圾回收器，此时就会自动用堆内存大小除以2048。因为JVM最多可以有2048个Region，并且Region的大小必须是2的倍数，比如说2M、4M之类的。当然可以通过手动方式设定，则是“`-XX:G1HeapRegionSize`”。刚开始的时候，默认新生代在堆内存的占比是5%，可以通过“`-XX:G1NewSizePrecent`”来设置新生代的初始占比的，维持默认即可。在系统运行中，JVM会不停的给新生代增加更多的Region，但是最多新生代的占比不会超过60%，可以通过“`-XX:G1MaxNewSizePercent`”来设置。一旦Region进行了垃圾回收，此时新生代的Region数量还会减少，这些其实都是动态的。

其实在G1中虽然将内存划分为了很多的Region，但是其实还是有新生代、老年代之分的，并且在新生代中还是有Eden和Survivor划分的。其实前面学习的很多的技术原理在G1时期都是有用的。并且Eden和Survivor的默认比依然是8：1：1。只不过前面是按照内存大小来比，现在是依靠Region的数量来比值。随着系统运行，不停地在新生代的Eden对应的Region中放对象，JVM就会不停的给新生代加入更多的Region，直到新生代占据堆大小的最大比例60%，一旦占了60%，并且Eden区还占满了对象，那么就会触发新生代的GC（Minor GC），这时只是回收新生代的垃圾对象。G1会用复制算法来进行垃圾回收，同时系统进入STW状态。这个过程和之前的区别就是G1可以设定目标GC停顿时间，也就是G1指定GC的时候最多允许系统停顿多长时间，可以通过“`-XX:MaxGCPauseMills`”参数来设定，默认值是200ms。

按照前面的推算，新生代最大占60%，那么老年代就是40%，在G1中对象从新生代进入老年代的条件和前面几乎一样，对象在新生代躲过了几次的垃圾回收，到达一定年龄了；或者根据动态年龄判定的规则，一旦发现某次新生代的GC过后，存活对象超过了Survivor的50%，此时就会判断一下，比如年龄为1、2、3、4岁的对象的大小总和超过了Survivor的50%，此时4岁以上的老对象全部会进入到老年代，这就是动态年龄判定规则。这时候又会有人想起大对象，大对象不是直接进入到老年代吗，是的没错，但是只有在G1的这套内存模式下是特别的。因为G1专门提供了Region来存放大对象，而不是让大对象直接进入到老年代中。在G1中，大对象的判定规则是如果一个对象大小超过了一个Region大小的50%，那就会被放入大对象专门的Region中。而且一个对象如果太太大，也可能会横跨多个Region来存放。有人会问，不是新生代60%老年代40%吗，那√8还有哪些Region给大对象？其实前面提到了，在G1里面，新生代和老年代的Region是动态的是不停变化的。比如说现在新生代占据了1200个Region，但是一次垃圾回收之后，就让里面的1000个Region都空了，那么此时那1000个空着的Region就可以不属于新生代了，里面的很多Region可以用来存放大对象。那还会有人问：既然大对象不属于新生代也不属于老年代，那什么时候会触发垃圾回收呢？也很简单，其实新生代、老年代在回收的时候，会顺带带着大对象Region一起回收。所以这就是在G1内存模型下对大对象的分配和回收策略。

G1垃圾回收的过程：

1. 初始标记阶段，会STW，仅仅标记GC Roots直接能引用的对象，速度快。
   - 类的静态变量可以作为根对象
   - 方法中的局部变量可以作为根对象
   - 类的成员变量不可
2. 并发标记阶段，允许系统程序运行，不会STW，从GC Roots开始沿着引用链追踪所有的存活对象，耗时但是影响不大因为可以和用户线程并行。
3. 最终标记阶段，会STW，因为并发标记阶段垃圾回收线程和工作线程可并发执行，垃圾回收时不会阻塞工作线程，所以可能会有浮动垃圾，因此需要最终标记，和CMS垃圾回收器中的重新标记阶段类似。会根据并发标记阶段记录的那些对象的修改，最终标记一下有哪些存活的对象，有哪些是垃圾对象。
4. 混合回收阶段，是最后一个阶段，这个阶段会计算新生代、老年代、大对象中每个Region中的存活对象数量，存活对象占比，还有执行垃圾回收的预期性能和效率，接着会STW，然后全力以赴尽快进行垃圾回收，此时它可能会从新生代、老年代、大对象里各自挑选部分的Region进行回收，因为必须让垃圾回收的停顿时间控制在我们指定的范围内。这里会STW，虽然会停止系统程序，但是不必担心，因为在这个阶段中，G1是允许执行多次混合回收的，每次混合回收虽然会让系统STW，但不会让系统停顿时间过长。

什么时候触发新生代+老年代的混合垃圾回收？G1有一个参数，是“`-XX:InitiatingHeapOccupancyPercent`”，它的默认值是45%，意思就是说：如果老年代占据了堆内存的45%的Region的时候（新生代最大已经达不到占堆内存60%），此时就会尝试触发一个新生代+老年代+大对象一起回收的混合回收阶段，也就是说，此时垃圾回收不仅仅是回收老年代，还会回收新生代，以及大对象。那么到底是回收这些区域的哪些Region呢？那就要看情况了。因为我们设定了对GC停顿时间的目标，因此它会从新生代、老年代、大对象里面各自挑选一些Region，保证用指定的时间（比如默认的200ms）回收尽可能多的垃圾，这就是混合回收Mixed GC，在最后一个阶段（混合回收）的时候，其实会停止所有程序运行，所以说G1是允许执行多次混合回收的。比如先停止工作，执行一次混合回收回收掉一些Region，接着恢复系统运行，然后再次停止系统运行，再执行一次混合回收回收掉一些Region，接着恢复系统运行。

有些参数可以控制这个，比如：

- “`-XX:G1MixedGCCountTarget`”参数，就是在一次混合回收的过程中，这个阶段执行几次混合回收，默认值是8次。先停止系统运行，执行一次混合回收回收掉一些Region，接着恢复系统运行，然后再次停止系统运行，再执行一次混合回收回收掉一些Region，反复8次。这样可以不让系统停顿时间过长。
- “`-XX:G1HeapWastePercent`”参数，默认值是5%。意思是，在混合回收的时候，对Region回收都是基于复制算法进行的，不会出现碎片问题，都是将要回收的Region中存活的对象放入其它的Region，然后这个Region中的垃圾对象全部清理掉。这样的话就会不断的空出来新的Region，一旦空闲出来的Region数量达到了堆内存的5%，此时就会立刻停止混合回收。意味着本次混合回收结束。
- “`-XX:G1MixedGCLiveThresholdPercent`”参数，默认值是85%，意思是确定要回收的Region的时候，必须是存活对象低于85%的Region才可以进行回收。否则还回收干什么？人家里面的存活对象那么多，还要把85%的对象都拷贝到别的Region，成本很高。

回收失败时的Full GC：在进行混合回收的时候，无论是年轻代还是老年代都是基于复制算法进行回收，都要把各个Region的存活对象拷贝到别的Region里，此时万一出现拷贝过程中发现没有空闲的Region可以承载自己的存活对象了，就会触发一次失败。一旦失败立马就会切换为停止系统程序，然后采用单线程进行标记、清理和压缩整理，空闲出来一批Region，这个过程极慢。stop the world 的时间变长。

G1和ParNew+CMS调优原则都是尽可能Minor GC，不做老年代的Full GC，G1相对来说更加智能，但也意味着JVM要用更多的资源去判断每个region的使用情况，而ParNew+CMS更加纯粹和直接。虽然G1在GC的时候不会产生内存碎片（复制算法），但是由于每个region存在存活对象85%不清理机制，会导致内存没有被充分释放的问题，而CMS虽然会有内存碎片（标记-清理），但是清理完之后后面会有一个对内存空间的压缩（由两个默认的参数设置决定，可以调节在多少次清理之后进行内存压缩，默认为0，每次都压缩），以减少内存碎片，是空间更加连贯。

因此对于CPU性能高的、内存容量大的、对应用响应度要求高的系统更推荐使用G1，而内存小一些的、CPU性能相对一般的甚至低下的更推荐使用ParNew+CMS。

我们知道，新创建的对象都是优先在伊甸园中的，那么伊甸园Eden区快满了的时候，会触发Minor GC，在触发Minor GC之前，会根据空间担保机制判断是否需要先进行Full GC，那么我们知道理论上，当老年代快占满了的时候，会触发Full GC。但是要知道：在使用ParNew+CMS的时候，老年代垃圾回收触发的时机有所变化，因为CMS是并发垃圾回收器，意思是CMS允许垃圾回收线程和用户工作线程并发执行，在CMS回收垃圾的同时，工作线程可能会在老年代中产生新的对象以及浮动垃圾，因此老年代需要预留一些空间，所以CMS中规定老年代占满92%就开始使用CMS进行老年代的垃圾回收，也就是说老年代可用空间只有原来的92%，8%预留给并发运行的工作线程，CMS本轮不会清理新晋升到老年代中的对象和垃圾。当8%内存不够用时，意思是晋升新对象以及浮动垃圾的速度大于垃圾回收线程回收垃圾的速度，并且已经占满了8%，导致放不下了，那么CMS并发回收失败，并且此时老年代的垃圾回收器从CMS退化为 Serial Old 串行的老年代垃圾回收器。这样效率会很低，强行把系统程序STW，重新进行长时间的GC Roots追踪，标记出全部的存活对象，不允许新的对象产生。响应时间变长。对于G1，有了region的概念，G1是在新生代region占了整个堆内存的60%（可调节），并且Eden满了的情况下才会触发Minor GC（复制算法）。在老年代region占了整个堆内存的45%，开始进行Mixed GC，混合回收阶段会回收新生代、老年代、大对象的region（复制算法），当Mixed GC时，没有新的空闲region用来装存活的对象时，会触发Full GC，Mixed GC就会切换为采用单线程进行标记、清理和压缩整理，空闲出来一批Region，这个过程极慢。stop the world 的时间变长。

##### Young Collection 跨代引用

- 新生代回收的跨代引用（老年代引用新生代）问题

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image34.png)

- 卡表与Remembered Set
  - Remembered Set 存在于E中，用于保存新生代对象对应的脏卡
    - 脏卡：O被划分为多个区域（一个区域512K），如果该区域引用了新生代对象，则该区域被称为脏卡
- 在引用变更时通过post-write barried + dirty card queue
- concurrent refinement threads 更新 Remembered Set

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image35.png)

> **跨代引用的总结：**
>
> 对于新生代垃圾回收时的跨代引用问题，首先我们要回忆新生代的垃圾回收，第一步就是找根对象（GC root），然后根据可达性分析算法，找到根对象的引用链，从而确定幸存下来的对象。那么我们要知道，根对象有一部分是来自于老年代的，也就是需要从老年代中找，但是老年代中通常存活的对象非常多，遍历整个老年代及其耗时，所以 JVM 会采用一种卡表（cardTable）的方式，将老年代区域再划分成一个个的 card，每个大约 512k。这样如果老年代中的某一个对象是引用了新生代对象的根对象（例子：可能集合中插入新对象，集合在老年代，新对象在新生代。），那么就标记这个根对象所在的 card 为 dirty card（脏对象）。这样在新生代垃圾回收的时候，查找根对象的时候，就不用遍历老年代中所有的对象了，而是去找那个标记为 dirty card 的小区域，只遍历小区域中的对象。
>
> 这时就会有人问了：如何知道这个 card 中有引用了新生代对象的根对象呢，标记 dirty card 不是也需要遍历整个老年代吗？
>
> 其实是不用的！因为在进行对象引用的创建时，会有一个查找过程，查找该引用是否是被其它区域的对象所引用，如果是，则在 Remembered Set （注：新生代这边有一个 Remembered Set，其中会记录外部对我的一些引用，也就是记录都有哪些 dirty card。）中集中标注，也就是这里的标记脏 card。所以说，引用的时候就自动标记了。
>
> 将来对新生代伊甸区做垃圾回收时，先通过 Remember Set 知道它对应哪些 dirty card，再到这些 dirty card 区域遍历查找 GC root，减少了 GC root 的查找时间。
>
> 那么具体如何标记脏卡的？
>
> 标记脏卡是通过一个叫 post-write barrier 的写屏障，在每次对象的引用发生变更时，都要去更新脏卡，将卡表中的卡标记为脏卡的动作是异步操作，不会立刻去完成卡表的更新，它会将更新的指令放在一个 dirty card queue 中，将来由一个线程去完成脏卡的更新操作。



##### Remark

重新标记阶段

在垃圾回收时，收集器处理对象的过程中

黑色：已被处理，需要保留的 灰色：正在处理中的 白色：还未处理的

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151229.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151229.png)

但是在**并发标记过程中**，有可能A被处理了以后未引用C，但该处理过程还未结束，在处理过程结束之前A引用了C，这时就会用到remark

过程如下

- 之前C未被引用，这时A引用了C，就会给C加一个写屏障，写屏障的指令会被执行，将C放入一个队列当中，并将C变为 处理中 状态
- 在**并发标记**阶段结束以后，重新标记阶段会STW，然后将放在该队列中的对象重新处理，发现有强引用引用它，就会处理它

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151239.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151239.png)

> Remark 总结：
>
> 在说 CMS 和 G1 这种并发垃圾回收器时，对于标记动作而言，有两个不同的阶段：并发标记阶段和重新标记阶段，后者也就是所谓的 Remark 阶段。并发标记阶段意味着在垃圾回收线程进行垃圾回收时，用户线程不会阻塞，也可以运行，所以当某个对象被当做垃圾进行清理但还未清理的时候，可能同时会有用户线程在对这个对象的引用做修改。意思就是可能已经被并发标记为可作为垃圾回收的对象，又有了新的引用，所以我们需要重新标记阶段。注意不要和最终标记弄混了，最终标记针对的是并发时可能出现的浮动垃圾的标记。
>
> 重新标记阶段的具体做法：
>
> 当我的对象的引用发生改变时，JVM 会给它加入一个写屏障 pre-write barrier ，只要对象的引用发生了改变，这个写屏障的指令就会被执行，这指令会将引用发生改变的这个对象加入到一个队列中 satb_mark_queue，并且将这个对象标记为未处理完，等到整个并发标记阶段结束了，接下来进入重新标记阶段，会 STW，这时重新标记阶段的线程会将对象从队列中取出再做一次检查。

##### JDK 8u20 字符串去重

过程

- 将所有新分配的字符串（底层是char[]）放入一个队列
- 当新生代回收时，G1并发检查是否有重复的字符串
- 如果字符串的值一样，就让他们**引用同一个字符串对象**
- 注意，其与String.intern的区别
  - intern 方法关注的是字符串对象本身，让字符串本身不重复，因为串池的底层就是 HashTable。
  - 字符串去重关注的是char[]
  - 在JVM内部，使用了不同的字符串标

优点与缺点

- 节省了大量内存
- 新生代回收时间略微增加，导致略微多占用CPU

##### JDK 8u40 并发标记类卸载

在并发标记阶段结束以后，就能知道哪些类不再被使用。如果一个类加载器的所有类都不在使用，则卸载它所加载的所有类

##### JDK 8u60 回收巨型对象

- 一个对象大于region的一半时，就称为巨型对象
- G1不会对巨型对象进行拷贝
- 回收时被优先考虑
- G1会跟踪老年代所有incoming引用，如果老年代incoming引用为0的巨型对象就可以在新生代垃圾回收时处理掉

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151249.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200608151249.png)

### GC 调优

查看虚拟机参数命令

```
"F:\JAVA\JDK8.0\bin\java" -XX:+PrintFlagsFinal -version | findstr "GC"Copy
```

可以根据参数去查询具体的信息

#### 调优领域

- 内存（GC）
- 锁竞争
- CPU占用
- IO

#### 确定目标

低延迟/高吞吐量？ 选择合适的GC

- CMS G1 ZGC
- ParallelGC
- Zing

#### 最快的GC是不发生GC

首先排除减少因为自身编写的代码而引发的内存问题

- 查看Full GC前后的内存占用，考虑以下几个问题
  - 数据是不是太多？
  - 数据表示是否太臃肿
    - 对象图
    - 对象大小
  - 是否存在内存泄漏

#### 新生代调优

- 新生代的特点
  - 所有的new操作分配内存都是非常廉价的
    - TLAB：每个线程都会在伊甸园中分配一块私有区域，也就是 TLAB（Thread-local allocation buffer）缓冲区，是每个线程局部的、私有的。当线程中 new 一个对象的时候，首先会检查这个 TLAB 缓冲区中有没有可用的内存，如果有，优先会在 TLAB 中进行对象的分配，原因是因为堆内存是线程间共享的区域，我们对象的分配可能会有线程安全问题，所以在做对象的分配的时候，也要做个线程并发安全的保护，这个操作就是 JVM 帮我们做的，所以 TLAB 的作用就是每个线程用自己私有的这块伊甸园内存来进行对象的分配。
  - 死亡对象回收零代价
  - 大部分对象用过即死（朝生夕死）
  - MInor GC 所用时间远小于Full GC
- 新生代内存越大越好么？
  - 不是
    - 新生代内存太小：频繁触发Minor GC，会STW，会使得吞吐量下降
    - 新生代内存太大：因为堆内存大小固定，所以老年代内存占比有所降低，会更频繁地触发Full GC。而且触发Minor GC时，清理新生代所花费的时间会更长
  - 新生代内存设置为内容纳[并发量*(请求-响应)]的数据为宜

> 随着新生代的内存空间变大，吞吐量变大，因为新生代内存空间大了，新生代垃圾回收占整个 CPU 计算的时间比例就会变小，这就代表吞吐量会变大，但是并不是新生代内存空间一直变大，吞吐量也会一直变大，反而是达到了一个点之后吞吐量会逐渐变小，因为老年代内存占比有所降低，会更频繁地触发Full GC。而且触发Minor GC时，清理新生代所花费的时间会更长

#### 幸存区调优

- 幸存区需要能够保存 **当前活跃对象**+**需要晋升的对象**！
- 晋升阈值配置得当，让长时间存活的对象尽快晋升

幸存区的晋升规则：如果幸存区分配的比较小，它就会由 JVM 动态的去调整晋升阈值，这个时候阈值就不一定为默认的 15 了，也许本来轮不到晋升的对象，但是由于幸存区的内存不够，导致 JVM 会提前将一些对象晋升到老年代，也就是有可能存活时间较短的对象提前进入了老年代，那么问题来了，如果一个本来存活时间短的对象被提前晋升到老年代的话，那么可能它得等到老年代的内存不足的时候，（暂且不考虑 CMS 和 G1 中 Full GC 之前的并发标记以及回收。）触发 Full GC 的时候才能把它回收，这是不好的，白白占用老年代的宝贵内存资源。所以我们要尽量保证存活时间短的对象在下次新生代垃圾回收的时候就回收掉，那些真正需要长时间存活的对象才晋升到老年代。

#### 老年代调优

以 CMS 并发垃圾回收器为例：CMS 是一种低响应时间的，并发的垃圾回收器，垃圾回收线程在工作的同时，其它的用户线程也能并发地执行。这样虽然减少了前面几种收集器阻塞线程时而浪费的时间，但是有个缺点，因为在垃圾回收线程回收垃圾的同时，用户线程可能会产生新的垃圾，也就是浮动垃圾。如果浮动垃圾产生的速度大于垃圾回收线程清理垃圾的速度，也就是如果浮动垃圾的产生又导致内存不足，这时会造成 CMS 垃圾回收器并发失败，从而导致 CMS 垃圾回收器不能正常工作，会退化为 Serial Old 串行的老年代垃圾回收器。这样效率会很低，STW，响应时间变长。所以我们在给老年代规划内存大小的时候，需要把它规划的大一些，避免浮动垃圾引起的并发失败，从而导致 Full GC 严重影响效率。

## 类加载与字节码技术

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image36.png)

### 类文件结构

一个简单的 HelloWorld.java

```java
// HelloWorld 示例
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

执行 javac -parameters -d . HellowWorld.java

编译为 HelloWorld.class 后是这个样子的：

以下是字节码文件

```java
[root@localhost ~]# od -t xC HelloWorld.class
0000000 ca fe ba be 00 00 00 34 00 23 0a 00 06 00 15 09
0000020 00 16 00 17 08 00 18 0a 00 19 00 1a 07 00 1b 07
0000040 00 1c 01 00 06 3c 69 6e 69 74 3e 01 00 03 28 29
0000060 56 01 00 04 43 6f 64 65 01 00 0f 4c 69 6e 65 4e
0000100 75 6d 62 65 72 54 61 62 6c 65 01 00 12 4c 6f 63
0000120 61 6c 56 61 72 69 61 62 6c 65 54 61 62 6c 65 01
0000140 00 04 74 68 69 73 01 00 1d 4c 63 6e 2f 69 74 63
0000160 61 73 74 2f 6a 76 6d 2f 74 35 2f 48 65 6c 6c 6f
0000200 57 6f 72 6c 64 3b 01 00 04 6d 61 69 6e 01 00 16
0000220 28 5b 4c 6a 61 76 61 2f 6c 61 6e 67 2f 53 74 72
0000240 69 6e 67 3b 29 56 01 00 04 61 72 67 73 01 00 13
0000260 5b 4c 6a 61 76 61 2f 6c 61 6e 67 2f 53 74 72 69
0000300 6e 67 3b 01 00 10 4d 65 74 68 6f 64 50 61 72 61
0000320 6d 65 74 65 72 73 01 00 0a 53 6f 75 72 63 65 46
0000340 69 6c 65 01 00 0f 48 65 6c 6c 6f 57 6f 72 6c 64
0000360 2e 6a 61 76 61 0c 00 07 00 08 07 00 1d 0c 00 1e
0000400 00 1f 01 00 0b 68 65 6c 6c 6f 20 77 6f 72 6c 64
0000420 07 00 20 0c 00 21 00 22 01 00 1b 63 6e 2f 69 74
0000440 63 61 73 74 2f 6a 76 6d 2f 74 35 2f 48 65 6c 6c
0000460 6f 57 6f 72 6c 64 01 00 10 6a 61 76 61 2f 6c 61
0000500 6e 67 2f 4f 62 6a 65 63 74 01 00 10 6a 61 76 61
0000520 2f 6c 61 6e 67 2f 53 79 73 74 65 6d 01 00 03 6f
0000540 75 74 01 00 15 4c 6a 61 76 61 2f 69 6f 2f 50 72
0000560 69 6e 74 53 74 72 65 61 6d 3b 01 00 13 6a 61 76
0000600 61 2f 69 6f 2f 50 72 69 6e 74 53 74 72 65 61 6d
0000620 01 00 07 70 72 69 6e 74 6c 6e 01 00 15 28 4c 6a
0000640 61 76 61 2f 6c 61 6e 67 2f 53 74 72 69 6e 67 3b
0000660 29 56 00 21 00 05 00 06 00 00 00 00 00 02 00 01
0000700 00 07 00 08 00 01 00 09 00 00 00 2f 00 01 00 01
0000720 00 00 00 05 2a b7 00 01 b1 00 00 00 02 00 0a 00
0000740 00 00 06 00 01 00 00 00 04 00 0b 00 00 00 0c 00
0000760 01 00 00 00 05 00 0c 00 0d 00 00 00 09 00 0e 00
0001000 0f 00 02 00 09 00 00 00 37 00 02 00 01 00 00 00
0001020 09 b2 00 02 12 03 b6 00 04 b1 00 00 00 02 00 0a
0001040 00 00 00 0a 00 02 00 00 00 06 00 08 00 07 00 0b
0001060 00 00 00 0c 00 01 00 00 00 09 00 10 00 11 00 00
0001100 00 12 00 00 00 05 01 00 10 00 00 00 01 00 13 00
0001120 00 00 02 00 14
```

根据 JVM 规范，**类文件结构**如下

```java
ClassFile {
    u4 magic;
    u2 minor_version;
    u2 major_version;
    u2 constant_pool_count;
    cp_info constant_pool[constant_pool_count-1];
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[interfaces_count];
    u2 fields_count;
    field_info fields[fields_count];
    u2 methods_count;
    method_info methods[methods_count];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

#### 魔数

u4 magic

对应字节码文件的0~3个字节

0000000 **ca fe ba be** 00 00 00 34 00 23 0a 00 06 00 15 09

#### 版本

u2 minor_version;

u2 major_version;

0000000 ca fe ba be **00 00 00 34** 00 23 0a 00 06 00 15 09

4~7 字节，表示类的版本 00 34（52） 表示是 Java 8

### 字节码指令

可参考

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5

#### javap工具

Oracle 提供了 **javap** 工具来反编译 class 文件

```java
Classfile /home/chanservy/idea/projects/concurrent/src/main/java/com/chan/jvm/HelloWorld.class
  Last modified 2021-10-27; size 438 bytes
  MD5 checksum 5ab6bab2c9d3036ba334e71f619ace9a
  Compiled from "HelloWorld.java"
public class com.chan.jvm.HelloWorld
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#15         // java/lang/Object."<init>":()V
   #2 = Fieldref           #16.#17        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #18            // hello world
   #4 = Methodref          #19.#20        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #21            // com/chan/jvm/HelloWorld
   #6 = Class              #22            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               SourceFile
  #14 = Utf8               HelloWorld.java
  #15 = NameAndType        #7:#8          // "<init>":()V
  #16 = Class              #23            // java/lang/System
  #17 = NameAndType        #24:#25        // out:Ljava/io/PrintStream;
  #18 = Utf8               hello world
  #19 = Class              #26            // java/io/PrintStream
  #20 = NameAndType        #27:#28        // println:(Ljava/lang/String;)V
  #21 = Utf8               com/chan/jvm/HelloWorld
  #22 = Utf8               java/lang/Object
  #23 = Utf8               java/lang/System
  #24 = Utf8               out
  #25 = Utf8               Ljava/io/PrintStream;
  #26 = Utf8               java/io/PrintStream
  #27 = Utf8               println
  #28 = Utf8               (Ljava/lang/String;)V
{
  public com.chan.jvm.HelloWorld();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 6: 0
        line 7: 8
}
SourceFile: "HelloWorld.java"

```

#### 图解方法执行流程

1）原始 java 代码

注意：当一段 Java 代码（.Java）被执行的时候，先编译成字节码文件，也就是我们熟知的 .class 文件，然后会由 Java 虚拟机的类加载器把我们刚才 main 方法所在的类进行一个类加载的操作，类加载实际上是将刚才的这些 class 的字节数据读取到内存中，其中常量池的这部分数据会放入运行时常量池（运行时常量池：之前我们说内存结构的时候，说过整个 JVM 分为堆、栈、方法区三部分，运行时常量池从某种意义上讲其实就属于方法区。），就是把我们 class 文件常量池中的数据都放入运行时常量池。

```java
/**
    * 演示 字节码指令 和 操作数栈、常量池的关系
    */
public class Demo3_1 {
    public static void main(String[] args) {
        int a = 10;
        int b = Short.MAX_VALUE + 1;
        int c = a + b;
        System.out.println(c);
    }
}
```

在 Java 源代码中，像 a 那样比较小的数字，其实并不是存在于常量池中的，而是跟着方法的字节码指令存放在一起，而一旦数字的范围超过了整数的最大值，比如 b ，则会存在于常量池中，也就是说：在 short 范围内的数字，都和字节码指令存放在一起，一旦超过了 short 整数的范围则会存在于常量池中，short 的最大值为 32767。

2）编译后的字节码文件

```java
Classfile /home/chanservy/idea/projects/concurrent/src/main/java/com/chan/jvm/Demo3_1.class
  Last modified 2021-10-27; size 445 bytes
  MD5 checksum 3e7e729c66dad33e41c46dc43bb1bc81
  Compiled from "Demo3_1.java"
public class com.chan.jvm.Demo3_1
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#16         // java/lang/Object."<init>":()V
   #2 = Class              #17            // java/lang/Short
   #3 = Integer            32768
   #4 = Fieldref           #18.#19        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = Methodref          #20.#21        // java/io/PrintStream.println:(I)V
   #6 = Class              #22            // com/chan/jvm/Demo3_1
   #7 = Class              #23            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               SourceFile
  #15 = Utf8               Demo3_1.java
  #16 = NameAndType        #8:#9          // "<init>":()V
  #17 = Utf8               java/lang/Short
  #18 = Class              #24            // java/lang/System
  #19 = NameAndType        #25:#26        // out:Ljava/io/PrintStream;
  #20 = Class              #27            // java/io/PrintStream
  #21 = NameAndType        #28:#29        // println:(I)V
  #22 = Utf8               com/chan/jvm/Demo3_1
  #23 = Utf8               java/lang/Object
  #24 = Utf8               java/lang/System
  #25 = Utf8               out
  #26 = Utf8               Ljava/io/PrintStream;
  #27 = Utf8               java/io/PrintStream
  #28 = Utf8               println
  #29 = Utf8               (I)V
{
  public com.chan.jvm.Demo3_1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: bipush        10
         2: istore_1
         3: ldc           #3                  // int 32768
         5: istore_2
         6: iload_1
         7: iload_2
         8: iadd
         9: istore_3
        10: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        13: iload_3
        14: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        17: return
      LineNumberTable:
        line 5: 0
        line 6: 3
        line 7: 6
        line 8: 10
        line 9: 17
}
SourceFile: "Demo3_1.java"
```

**3）字节码中的常量池载入到运行时常量池**

我们都知道，方法区是概念，在 java7 中，方法区的具体实现就是永久代，常量池存在于永久带中，永久带存在于方法区中；而在 java8 中，方法区的具体实现就是元空间，常量池存在于元空间中，虽然元空间独立存在与本地内存，但是也叫方法区的实现，因此从某种意义上讲，常量池也属于方法区，只不过这里单独提出来说明了。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image37.png)

**4）方法字节码载入方法区**

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image38.png)

**5）main 线程开始运行，分配栈帧内存**

栈帧中有局部变量表和操作数栈

（stack=2，locals=4） 对应操作数栈有2个空间（每个空间4个字节），局部变量表中有4个槽位

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image39.png)

**6）执行引擎开始执行字节码**

**bipush 10**

- 将一个 byte 压入操作数栈（操作数栈大小都为 4 字节，其长度会补齐 4 个字节），类似的指令还有
- sipush 将一个 short 压入操作数栈（其长度会补齐 4 个字节）
- ldc 将一个 int 压入操作数栈
- ldc2_w 将一个 long 压入操作数栈（**分两次压入**，因为 long 是 8 个字节）
- 这里小的数字都是和字节码指令存在一起，**超过 short 范围的数字存入了常量池**

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image40.png)

**istore 1**

将操作数栈栈顶元素弹出，放入局部变量表的slot 1中

对应代码中的

```
a = 10Copy
```

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image41.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image42.png)

**ldc #3**

读取运行时常量池中 #3 数据到操作数栈，即32768(超过short最大值范围的数会被放到运行时常量池中)，将其加载到操作数栈中

注意 Short.MAX_VALUE 是 32767，所以 32768 = Short.MAX_VALUE + 1 实际是在编译期间计算好的

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image43.png)

**istore 2**

将操作数栈中的元素弹出，放到局部变量表的2号位置

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image44.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image45.png)

**iload1 iload2**

将局部变量表中1号位置和2号位置的元素放入操作数栈中

- 因为只能在操作数栈中执行运算操作

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image46.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image47.png)

**iadd**

将操作数栈中的两个元素**弹出栈**并相加，结果在压入操作数栈中

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image48.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image49.png)

**istore 3**

将操作数栈中的元素弹出，放入局部变量表的3号位置

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image50.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image51.png)

**getstatic #4**

在运行时常量池中找到#4，发现是一个对象，它存在于堆内存中

在堆内存中找到该对象，并将其**引用**放入操作数栈中

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image52.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image53.png)

**iload 3**

将局部变量表中3号位置的元素压入操作数栈中

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image54.png)

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image55.png)

**invokevirtual 5**

- 找到常量池 #5 项
- 定位到方法区 `java/io/PrintStream.println:(I)V` 方法
- 生成新的栈帧（分配 locals、stack等）
- **传递参数**，执行新栈帧中的字节码

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image56.png)

- 执行完毕，弹出栈帧
- 清除 main 操作数栈内容

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image57.png)

**return**
完成 main 方法调用，弹出 main 栈帧，程序结束

练习：

```java
/**
    * 从字节码角度分析 a++ 相关题目
    */
public class Demo3_1 {
    public static void main(String[] args) {
        int a = 10;
        int b = a++ + ++a + a--;
        System.out.println(a);// 11
        System.out.println(b);// 34
    }
}
```

```java
Classfile /home/chanservy/idea/projects/concurrent/src/main/java/com/chan/jvm/Demo3_1.class
  Last modified 2021-10-27; size 434 bytes
  MD5 checksum fd4b0448bd8dd4239c5e5119332ffe79
  Compiled from "Demo3_1.java"
public class com.chan.jvm.Demo3_1
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #5.#14         // java/lang/Object."<init>":()V
   #2 = Fieldref           #15.#16        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = Methodref          #17.#18        // java/io/PrintStream.println:(I)V
   #4 = Class              #19            // com/chan/jvm/Demo3_1
   #5 = Class              #20            // java/lang/Object
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               main
  #11 = Utf8               ([Ljava/lang/String;)V
  #12 = Utf8               SourceFile
  #13 = Utf8               Demo3_1.java
  #14 = NameAndType        #6:#7          // "<init>":()V
  #15 = Class              #21            // java/lang/System
  #16 = NameAndType        #22:#23        // out:Ljava/io/PrintStream;
  #17 = Class              #24            // java/io/PrintStream
  #18 = NameAndType        #25:#26        // println:(I)V
  #19 = Utf8               com/chan/jvm/Demo3_1
  #20 = Utf8               java/lang/Object
  #21 = Utf8               java/lang/System
  #22 = Utf8               out
  #23 = Utf8               Ljava/io/PrintStream;
  #24 = Utf8               java/io/PrintStream
  #25 = Utf8               println
  #26 = Utf8               (I)V
{
  public com.chan.jvm.Demo3_1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: bipush        10
         2: istore_1
         3: iload_1
         4: iinc          1, 1
         7: iinc          1, 1
        10: iload_1
        11: iadd
        12: iload_1
        13: iinc          1, -1
        16: iadd
        17: istore_2
        18: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        21: iload_1
        22: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        25: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        28: iload_2
        29: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        32: return
      LineNumberTable:
        line 5: 0
        line 6: 3
        line 7: 18
        line 8: 25
        line 9: 32
}
SourceFile: "Demo3_1.java"
```

分析：

- **注意 iinc 指令是直接在局部变量表 slot 上进行运算**
- a++ 和 ++a 的区别是先执行 iload 还是先执行 iinc

#### 通过字节码指令来分析问题

代码

```java
public class Demo2 {
	public static void main(String[] args) {
        // 0号槽位放的是args
		int i=0;// 局部变量表1号槽位
		int x=0;// 局部变量表2号槽位
		while(i<10) {
            // 这里的 x++,是先 iload x,结果就是把局部变量表中的0读进操作数栈
            //			   后 iinc x 1:注意这里是将局部变量表中 x 槽位的值 +1
            // 这里的++操作的是局部变量表中的值
            // 赋值操作“=” 是将操作数栈中的值弹出赋值到x
			x = x++;
			i++;
		}
		System.out.println(x); //结果为0
	}
}
```

为什么最终的x结果为0呢？ 其实不管循环多少次，最终 x 的结果都是 0。通过分析字节码指令即可知晓：

```java
Code:
     stack=2, locals=3, args_size=1	//操作数栈分配2个空间，局部变量表分配3个空间
        0: iconst_0	//准备一个常数0
        1: istore_1	//将常数0放入局部变量表的1号槽位 i=0
        2: iconst_0	//准备一个常数0
        3: istore_2	//将常数0放入局部变量的2号槽位 x=0	
        4: iload_1		//将局部变量表1号槽位的数放入操作数栈中
        5: bipush        10	//将数字10放入操作数栈中，此时操作数栈中有2个数
        7: if_icmpge     21	//比较操作数栈中的两个数，如果下面的数大于上面的数，就跳转到21。这里的比较是将两个数做减法。因为涉及运算操作，所以会将两个数弹出操作数栈来进行运算。运算结束后操作数栈为空
       10: iload_2		//将局部变量2号槽位的数放入操作数栈中，放入的值是0
       11: iinc          2, 1	//将局部变量2号槽位的数加1，自增后，槽位中的值为1
       14: istore_2	//将操作数栈中的数放入到局部变量表的2号槽位，2号槽位的值又变为了0
       15: iinc          1, 1 //1号槽位的值自增1
       18: goto          4 //跳转到第4条指令
       21: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       24: iload_2
       25: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
       28: return
```

#### 构造方法

##### cinit()V

```java
public class Demo3 {
	static int i = 10;

	static {
		i = 20;
	}

	static {
		i = 30;
	}

	public static void main(String[] args) {
		System.out.println(i); //结果为30
	}
}
```

编译器会按**从上至下**的顺序，收集所有 static 静态代码块和静态成员赋值的代码，**合并**为一个特殊的方法 cinit()V ：

```java
stack=1, locals=0, args_size=0
         0: bipush        10
         2: putstatic     #3                  // Field i:I
         5: bipush        20
         7: putstatic     #3                  // Field i:I
        10: bipush        30
        12: putstatic     #3                  // Field i:I
        15: return
```

##### init()V

```java
public class Demo4 {
	private String a = "s1";

	{
		b = 20;
	}

	private int b = 10;

	{
		a = "s2";
	}

	public Demo4(String a, int b) {
		this.a = a;
		this.b = b;
	}

	public static void main(String[] args) {
		Demo4 d = new Demo4("s3", 30);
		System.out.println(d.a);
		System.out.println(d.b);
	}
}
```

编译器会按**从上至下**的顺序，收集所有 {} 代码块和成员变量赋值的代码，**形成新的构造方法**，但**原始构造方法**内的代码**总是在后**

```java
Code:
     stack=2, locals=3, args_size=3
        0: aload_0
        1: invokespecial #1                  // Method java/lang/Object."<init>":()V
        4: aload_0
        5: ldc           #2                  // String s1
        7: putfield      #3                  // Field a:Ljava/lang/String;
       10: aload_0
       11: bipush        20
       13: putfield      #4                  // Field b:I
       16: aload_0
       17: bipush        10
       19: putfield      #4                  // Field b:I
       22: aload_0
       23: ldc           #5                  // String s2
       25: putfield      #3                  // Field a:Ljava/lang/String;
       //原始构造方法在最后执行
       28: aload_0
       29: aload_1
       30: putfield      #3                  // Field a:Ljava/lang/String;
       33: aload_0
       34: iload_2
       35: putfield      #4                  // Field b:I
       38: return
```

静态代码块 > 普通代码块 > 构造方法，初始化。

总结：从字节码层面分析，编译器会按照从上至下的顺序，收集所有初始化代码块和成员变量赋值的代码，最后合并成一个新的构造方法。原始的构造方法里的代码也会附加到新的构造方法中，不过是放在最后的位置。

#### 方法调用

```java
public class Demo5 {
	public Demo5() {

	}

	private void test1() {

	}

	private final void test2() {

	}

	public void test3() {

	}

	public static void test4() {

	}

	public static void main(String[] args) {
		Demo5 demo5 = new Demo5();
		demo5.test1();
		demo5.test2();
		demo5.test3();
		Demo5.test4();
	}
}
```

不同方法在调用时，对应的虚拟机指令有所区别

- 私有、构造、被final修饰的方法，在调用时都使用**invokespecial**指令
- 普通成员方法在调用时，使用invokespecial指令。因为编译期间无法确定该方法的内容，只有在运行期间才能确定
- 静态方法在调用时使用invokestatic指令

```java
Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class com/nyima/JVM/day5/Demo5 
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: invokespecial #4                  // Method test1:()V
        12: aload_1
        13: invokespecial #5                  // Method test2:()V
        16: aload_1
        17: invokevirtual #6                  // Method test3:()V
        20: invokestatic  #7                  // Method test4:()V
        23: returnCopy
```

- new 是创建【对象】，给对象分配堆内存，执行成功会将【**对象引用**】压入操作数栈
- dup 是赋值操作数栈栈顶的内容，本例即为【**对象引用**】，为什么需要两份引用呢，一个是要配合 invokespecial 调用该对象的构造方法 “init”:()V （会消耗掉栈顶一个引用），另一个要 配合 astore_1 赋值给局部变量
- 终方法（ﬁnal），私有方法（private），构造方法都是由 invokespecial 指令来调用，属于静态绑定
- 普通成员方法是由 invokevirtual 调用，属于**动态绑定**，即支持多态 成员方法与静态方法调用的另一个区别是，执行方法前是否需要【对象引用】

#### 多态原理

因为普通成员方法需要在运行时才能确定具体的内容，所以虚拟机需要调用**invokevirtual**指令

在执行invokevirtual指令时，经历了以下几个步骤

- 先通过栈帧中对象的引用找到对象
- 分析对象头，找到对象实际的Class
- Class结构中有**vtable**
- 查询vtable找到方法的具体地址
- 执行方法的字节码

#### 异常处理

##### try-catch

```java
public class Demo1 {
	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		}catch (Exception e) {
			i = 20;
		}
	}
}
```

对应字节码指令

```
Code:
     stack=1, locals=3, args_size=1
        0: iconst_0
        1: istore_1
        2: bipush        10
        4: istore_1
        5: goto          12
        8: astore_2
        9: bipush        20
       11: istore_1
       12: return
     //多出来一个异常表
     Exception table:
        from    to  target type
            2     5     8   Class java/lang/Exception
```

- 可以看到多出来一个 Exception table 的结构，[from, to) 是**前闭后开**（也就是检测2~4行）的检测范围，一旦这个范围内的字节码执行出现异常，则通过 type 匹配异常类型，如果一致，进入 target 所指示行号
- 8行的字节码指令 astore_2 是将异常对象引用存入局部变量表的2号位置（为e）

##### 多个single-catch

```java
public class Demo1 {
	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		}catch (ArithmeticException e) {
			i = 20;
		}catch (Exception e) {
			i = 30;
		}
	}
}
```

对应的字节码

```
Code:
     stack=1, locals=3, args_size=1
        0: iconst_0
        1: istore_1
        2: bipush        10
        4: istore_1
        5: goto          19
        8: astore_2
        9: bipush        20
       11: istore_1
       12: goto          19
       15: astore_2
       16: bipush        30
       18: istore_1
       19: return
     Exception table:
        from    to  target type
            2     5     8   Class java/lang/ArithmeticException
            2     5    15   Class java/lang/Exception
```

- 因为异常出现时，**只能进入** Exception table 中**一个分支**，所以局部变量表 slot 2 位置**被共用**

##### finally

```java
public class Demo2 {
	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		} catch (Exception e) {
			i = 20;
		} finally {
			i = 30;
		}
	}
}
```

对应字节码

```
Code:
     stack=1, locals=4, args_size=1
        0: iconst_0
        1: istore_1
        //try块
        2: bipush        10
        4: istore_1
        //try块执行完后，会执行finally    
        5: bipush        30
        7: istore_1
        8: goto          27
       //catch块     
       11: astore_2 //异常信息放入局部变量表的2号槽位
       12: bipush        20
       14: istore_1
       //catch块执行完后，会执行finally        
       15: bipush        30
       17: istore_1
       18: goto          27
       //出现异常，但未被Exception捕获，会抛出其他异常，这时也需要执行finally块中的代码   
       21: astore_3
       22: bipush        30
       24: istore_1
       25: aload_3
       26: athrow  //抛出异常
       27: return
     Exception table:
        from    to  target type
            2     5    11   Class java/lang/Exception
            2     5    21   any
           11    15    21   any
```

可以看到 ﬁnally 中的代码被**复制了 3 份**，分别放入 try 流程，catch 流程以及 catch剩余的异常类型流程

所以在字节码层面，finally 块中的代码会复制一式三份，会被复制到 try 分支的末尾；会被复制到 catch 分支的末尾；会被复制到 catch 分支未捕获到的异常分支的末尾。

**注意**：虽然从字节码指令看来，每个块中都有finally块，但是finally块中的代码**只会被执行一次**

##### finally中的return

```java
public class Demo3 {
	public static void main(String[] args) {
		int i = Demo3.test();
        //结果为20
		System.out.println(i);
	}

	public static int test() {
		int i;
		try {
			i = 10;
			return i;
		} finally {
			i = 20;
			return i;
		}
	}
}
```

对应字节码

```
Code:
     stack=1, locals=3, args_size=0
        0: bipush        10
        2: istore_0
        3: iload_0
        4: istore_1  //暂存返回值
        5: bipush        20
        7: istore_0
        8: iload_0
        9: ireturn	//ireturn会返回操作数栈顶的整型值20
       //如果出现异常，还是会执行finally块中的内容，没有抛出异常
       10: astore_2
       11: bipush        20
       13: istore_0
       14: iload_0
       15: ireturn	//这里没有athrow了，也就是如果在finally块中如果有返回操作的话，且try块中出现异常，会吞掉异常！
     Exception table:
        from    to  target type
            0     5    10   any
```

- 由于 ﬁnally 中的 **ireturn** 被插入了所有可能的流程，因此返回结果肯定以ﬁnally的为准
- 至于字节码中第 2 行，似乎没啥用，且留个伏笔，看下个例子
- 跟上例中的 ﬁnally 相比，发现**没有 athrow 了**，这告诉我们：如果在 ﬁnally 中出现了 return，会**吞掉异常**
- 所以**不要在finally中进行返回操作**

##### 被吞掉的异常

```java
public class Demo3 {
   public static void main(String[] args) {
      int i = Demo3.test();
      //最终结果为20
      System.out.println(i);
   }

   public static int test() {
      int i;
      try {
         i = 10;
         //这里应该会抛出异常
         i = i/0;
         return i;
      } finally {
         i = 20;
         return i;
      }
   }
}
```

会发现打印结果为20，并未抛出异常

##### finally不带return

```java
public class Demo4 {
	public static void main(String[] args) {
		int i = Demo4.test();
		System.out.println(i);
	}

	public static int test() {
		int i = 10;
		try {
			return i;
		} finally {
			i = 20;
		}
	}
}
// 结果为 10
```

返回的结果一定是 10，因为 finally 块中的代码会复制到 try 块的末尾，也就是 return i 的后面，那 i 的值已经返回了 10，因此 finally 中无论对 i 如何赋值返回的结果都是 10。finally 块中不要写 return 返回操作，否则会捕获不到异常。

对应字节码

```
Code:
     stack=1, locals=3, args_size=0
        0: bipush        10
        2: istore_0 //赋值给i 10
        3: iload_0	//加载到操作数栈顶
        4: istore_1 //加载到局部变量表的1号位置
        5: bipush        20
        7: istore_0 //赋值给i 20
        8: iload_1 //加载局部变量表1号位置的数10到操作数栈
        9: ireturn //返回操作数栈顶元素 10
       10: astore_2
       11: bipush        20
       13: istore_0
       14: aload_2 //加载异常
       15: athrow //抛出异常
     Exception table:
        from    to  target type
            3     5    10   any
```

#### Synchronized

```java
public class Demo5 {
	public static void main(String[] args) {
		int i = 10;
		Lock lock = new Lock();
		synchronized (lock) {
			System.out.println(i);
		}
	}
}

class Lock{}
```

对应字节码

```
Code:
     stack=2, locals=5, args_size=1
        0: bipush        10
        2: istore_1
        3: new           #2                  // class com/nyima/JVM/day06/Lock
        6: dup //复制一份，放到操作数栈顶，用于构造函数消耗
        7: invokespecial #3                  // Method com/nyima/JVM/day06/Lock."<init>":()V
       10: astore_2 //剩下的一份放到局部变量表的2号位置
       11: aload_2 //加载到操作数栈
       12: dup //复制一份，放到操作数栈，用于加锁时消耗
       13: astore_3 //将操作数栈顶元素弹出，暂存到局部变量表的三号槽位。这时操作数栈中有一份对象的引用
       14: monitorenter //加锁
       //锁住后代码块中的操作    
       15: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
       18: iload_1
       19: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
       //加载局部变量表中三号槽位对象的引用，用于解锁    
       22: aload_3    
       23: monitorexit //解锁
       24: goto          34
       //异常操作    
       27: astore        4
       29: aload_3
       30: monitorexit //解锁
       31: aload         4
       33: athrow
       34: return
     //可以看出，无论何时出现异常，都会跳转到27行，将异常放入局部变量中，并进行解锁操作，然后加载异常并抛出异常。      
     Exception table:
        from    to  target type
           15    24    27   any
           27    31    27   any
```

### 编译期处理

所谓的 **语法糖** ，其实就是指 java 编译器把 *.java 源码编译为 \*.class 字节码的过程中，**自动生成**和**转换**的一些代码，主要是为了减轻程序员的负担，算是 java 编译器给我们的一个额外福利

**注意**，以下代码的分析，借助了 javap 工具，idea 的反编译功能，idea 插件 jclasslib 等工具。另外， 编译器转换的**结果直接就是 class 字节码**，只是为了便于阅读，给出了 几乎等价 的 java 源码方式，并不是编译器还会转换出中间的 java 源码，切记。

#### 默认构造函数

```java
public class Candy1 {

}
```

经过编译期优化后

```java
public class Candy1 {
   //这个无参构造器是java编译器帮我们加上的
   public Candy1() {
      //即调用父类 Object 的无参构造方法，即调用 java/lang/Object." <init>":()V
      super();
   }
}
```

语法糖一：当自己的类没有实现任何构造器的情况下，编译器会帮我们默认实现一个无参构造，也就是调用父类 Object 的无参构造 super()。

#### 自动拆装箱

基本类型和其包装类型的相互转换过程，称为拆装箱

在JDK 5以后，它们的转换可以在编译期自动完成

```java
public class Demo2 {
   public static void main(String[] args) {
      Integer x = 1;
      int y = x;
   }
}
```

转换过程如下

```java
public class Demo2 {
   public static void main(String[] args) {
      //基本类型赋值给包装类型，称为装箱
      Integer x = Integer.valueOf(1);
      //包装类型赋值给基本类型，称谓拆箱
      int y = x.intValue();
   }
}
```

#### 泛型集合取值

泛型也是在 JDK 5 开始加入的特性，但 java 在**编译泛型代码后**会执行 **泛型擦除** 的动作，即泛型信息在编译为字节码之后就**丢失**了，实际的类型都当做了 **Object** 类型来处理：

```java
public class Demo3 {
   public static void main(String[] args) {
      List<Integer> list = new ArrayList<>();
      list.add(10);// 实际调用的是 List.add(Object e)
      Integer x = list.get(0);// 实际调用的是 Object obj = List.get(int index);
   }
}
```

所以调用get函数取值时，编译器真正生成的字节码中，还要额外做一个类型转换的操作：

```java
// 需要将 Object 转为 Integer
Integer x = (Integer) list.get(0);
```

如果要将返回结果赋值给一个int类型的变量，则还有**自动拆箱**的操作

```java
// 需要将 Object 转为 Integer, 并执行拆箱操作
int x = ((Integer)list.get(0)).intValue();
```

还好这些麻烦事都不用自己做。

擦除的是字节码上的泛型信息，可以看到 LocalVariableTypeTable 仍然保留了方法参数泛型的信息

对应字节码

```
Code:
    stack=2, locals=3, args_size=1
       0: new           #2                  // class java/util/ArrayList
       3: dup
       4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: bipush        10
      11: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      //这里进行了泛型擦除，实际调用的是add(Objcet o)
      14: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z

      19: pop
      20: aload_1
      21: iconst_0 
      22: invokeinterface #6,  2            // InterfaceMethod java/util/List.get:(I)Ljava/lang/Object;
//这里进行了类型转换，将Object转换成了Integer
      27: checkcast     #7                  // class java/lang/Integer
      30: astore_2
      31: returnCopy
```

#### 可变参数

```java
public class Demo4 {
   public static void foo(String... args) {
      //将args赋值给arr，可以看出String...实际就是String[] 
      String[] arr = args;
      System.out.println(arr.length);
   }

   public static void main(String[] args) {
      foo("hello", "world");
   }
}
```

可变参数 **String…** args 其实是一个 **String[]** args ，从代码中的赋值语句中就可以看出来。 同 样 java 编译器会在编译期间将上述代码变换为：

```java
public class Demo4 {
   public Demo4 {}

    
   public static void foo(String[] args) {
      String[] arr = args;
      System.out.println(arr.length);
   }

   public static void main(String[] args) {
      foo(new String[]{"hello", "world"});
   }
}
```

注意，如果调用的是foo()，即未传递参数时，等价代码为foo(new String[]{})，**创建了一个空数组**，而不是直接传递的null

#### foreach

```java
public class Demo5 {
	public static void main(String[] args) {
        //数组赋初值的简化写法也是一种语法糖。
		int[] arr = {1, 2, 3, 4, 5};
		for(int x : arr) {
			System.out.println(x);
		}
	}
}
```

编译器会帮我们转换为

```java
public class Demo5 {
    public Demo5 {}

	public static void main(String[] args) {
		int[] arr = new int[]{1, 2, 3, 4, 5};
		for(int i=0; i<arr.length; ++i) {
			int x = arr[i];
			System.out.println(x);
		}
	}
}
```

**如果是集合使用foreach**

```java
public class Demo5 {
   public static void main(String[] args) {
      List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
      for (Integer x : list) {
         System.out.println(x);
      }
   }
}
```

集合要使用foreach，需要该集合类实现了**Iterable接口**，因为集合的遍历需要用到**迭代器Iterator**

```java
public class Demo5 {
    public Demo5 {}
    
   public static void main(String[] args) {
      List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
      //获得该集合的迭代器
      Iterator<Integer> iterator = list.iterator();
      while(iterator.hasNext()) {
         Integer x = iterator.next();
         System.out.println(x);
      }
   }
}
```

#### switch字符串

```java
public class Demo6 {
   public static void main(String[] args) {
      String str = "hello";
      switch (str) {
         case "hello" :
            System.out.println("h");
            break;
         case "world" :
            System.out.println("w");
            break;
         default:
            break;
      }
   }
}
```

在编译器中执行的操作

```java
public class Demo6 {
   public Demo6() {
      
   }
   public static void main(String[] args) {
      String str = "hello";
      int x = -1;
      //通过字符串的hashCode+value来判断是否匹配
      switch (str.hashCode()) {
         //hello的hashCode
         case 99162322 :
            //再次比较，因为字符串的hashCode有可能相等
            if(str.equals("hello")) {
               x = 0;
            }
            break;
         //world的hashCode
         case 11331880 :
            if(str.equals("world")) {
               x = 1;
            }
            break;
         default:
            break;
      }

      //用第二个switch在进行输出判断
      switch (x) {
         case 0:
            System.out.println("h");
            break;
         case 1:
            System.out.println("w");
            break;
         default:
            break;
      }
   }
}
```

过程说明：

- 在编译期间，单个的switch被分为了两个
  - 第一个用来匹配字符串，并给x赋值
    - 字符串的匹配用到了字符串的hashCode，还用到了equals方法
    - 使用hashCode是为了提高比较效率，使用equals是防止有hashCode冲突（如BM和C.）
  - 第二个用来根据x的值来决定输出语句

#### switch枚举

```java
public class Demo7 {
   public static void main(String[] args) {
      SEX sex = SEX.MALE;
      switch (sex) {
         case MALE:
            System.out.println("man");
            break;
         case FEMALE:
            System.out.println("woman");
            break;
         default:
            break;
      }
   }
}

enum SEX {
   MALE, FEMALE;
}
```

编译器中执行的代码如下

```java
public class Demo7 {
   /**     
    * 定义一个合成类（仅 jvm 使用，对我们不可见）     
    * 用来映射枚举的 ordinal 与数组元素的关系     
    * 枚举的 ordinal 表示枚举对象的序号，从 0 开始     
    * 即 MALE 的 ordinal()=0，FEMALE 的 ordinal()=1     
    */ 
   static class $MAP {
      //数组大小即为枚举元素个数，里面存放了case用于比较的数字
      static int[] map = new int[2];
      static {
         //ordinal即枚举元素对应所在的位置，MALE为0，FEMALE为1
         map[SEX.MALE.ordinal()] = 1;
         map[SEX.FEMALE.ordinal()] = 2;
      }
   }

   public static void main(String[] args) {
      SEX sex = SEX.MALE;
      //将对应位置枚举元素的值赋给x，用于case操作
      int x = $MAP.map[sex.ordinal()];
      switch (x) {
         case 1:
            System.out.println("man");
            break;
         case 2:
            System.out.println("woman");
            break;
         default:
            break;
      }
   }
}

enum SEX {
   MALE, FEMALE;
}
```

#### 枚举类

```java
enum SEX {
   MALE, FEMALE;
}
```

转换后的代码

```java
public final class Sex extends Enum<Sex> {   
   //对应枚举类中的元素
   public static final Sex MALE;    
   public static final Sex FEMALE;    
   private static final Sex[] $VALUES;
   
    static {       
    	//调用构造函数，传入枚举元素的值及ordinal
    	MALE = new Sex("MALE", 0);    
        FEMALE = new Sex("FEMALE", 1);   
        $VALUES = new Sex[]{MALE, FEMALE}; 
   }
 	
   //调用父类中的方法
    private Sex(String name, int ordinal) {     
        super(name, ordinal);    
    }
   
    public static Sex[] values() {  
        return $VALUES.clone();  
    }
    public static Sex valueOf(String name) { 
        return Enum.valueOf(Sex.class, name);  
    } 
   
}
```

#### 匿名内部类

```java
blic class Candy11 {
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("ok");
            }
        };
    }
}
```

转换后的代码

```java
// 额外生成的类
final class Candy11$1 implements Runnable {
    Candy11$1() {
    }
    public void run() {
        System.out.println("ok");
    }
}

public class Candy11 {
    public static void main(String[] args) {
        Runnable runnable = new Candy11$1();
    }
}
```

引用**局部变量**的匿名内部类，源代码：

```java
public class Candy11 {
    public static void test(final int x) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("ok:" + x);
            }
        };
    }
}
```

转化后代码

```java
// 额外生成的类
final class Candy11$1 implements Runnable {
    int val$x;
    Candy11$1(int x) {
        this.val$x = x;
    }
    public void run() {
        System.out.println("ok:" + this.val$x);
    }
}

public class Candy11 {
    public static void test(final int x) {
        Runnable runnable = new Candy11$1(x);
    }
}
```

**注意**：这同时解释了为什么匿名内部类引用局部变量时，局部变量必须是 final 的：因为在创建 Candy11$1 对象时，将 x 的值赋值给了 Candy11$1 对象的 val，如果不是 final 的话，那么可能局部变量发生变化，但是 Candy11$1 对象的 val$x 没办法随着变化，不一致会出现问题。我感觉这点和 lambda 中用的参数不能是一个变量是一个道理。

#### try-with-resources

JDK 7 开始新增了对需要关闭的资源处理的特殊语法 `try-with-resources`：它的主要作用就是帮助简化资源关闭的，原来写一个资源关闭的代码都需要自己写一个 finally 块来确保资源一定会关闭，在 JDK7 和之后的版本中，有了 try-with-resources，就不用那么麻烦了。只需要按照它的语法格式（如下代码块）去创建资源对象，可以省略 finally 的书写。

格式：

```java
try(资源变量 = 创建资源对象){
    
} catch( ) {
    
}
```

但是这个简化，有个前提，就是我们的这个资源对象必须要实现一个叫 AutoCloseable 的接口。例如 InputStream、OutputStream、Connection、Statement 、 ResultSet 等接口都实现了 AutoCloseable，使用 try-with-resources 的格式创建它们的资源对象就可以不用写 finally 语句块，编译器会帮助生成关闭资源代码，例如：

```java
public class Candy9 {
    public static void main(String[] args) {
        try(InputStream is = new FileInputStream("d:\\1.txt")) {
            System.out.println(is);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

会被转换为：

```java
public class Candy9 {
    public Candy9() {
    }
    public static void main(String[] args) {
        try {
            InputStream is = new FileInputStream("d:\\1.txt");
            Throwable t = null;
            try {
                System.out.println(is);
            } catch (Throwable e1) {
                // t 是我们代码出现的异常
                t = e1;
                throw e1;
            } finally {
                // 判断了资源不为空
                if (is != null) {
                    // 如果我们代码有异常
                    if (t != null) {
                        try {
                            is.close();
                        } catch (Throwable e2) {
                            // 如果 close 出现异常，作为被压制异常添加
                            t.addSuppressed(e2);
                        }
                    } else {
                        // 如果我们代码没有异常，close 出现的异常就是最后 catch 块中的 e
                        is.close();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

为什么要设计一个 `addSuppressed(Throwable e) `（添加被压制异常）的方法呢？是为了防止异常信息的丢失！这个方法也是 JDK7 中新的方法，为了在释放资源的时候，希望 try 块中我们自己写的代码的异常和关闭资源的异常都保留下来，不要丢失掉。（代码中出现的异常、关闭资源出现的异常全部都捕获到。）

想想 try-with-resources 生成的 fianlly 中如果抛出了异常：

```java
public class Test6 {
    public static void main(String[] args) {
        try (MyResource resource = new MyResource()) {
            int i = 1/0;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
class MyResource implements AutoCloseable {
    public void close() throws Exception {
        throw new Exception("close 异常");
    }
}
```

输出：

```
java.lang.ArithmeticException: / by zero
at test.Test6.main(Test6.java:7)
Suppressed: java.lang.Exception: close 异常
at test.MyResource.close(Test6.java:18)
at test.Test6.main(Test6.java:6)
```

#### 方法重写时的桥接方法

我们都知道，方法重写时对返回值分两种情况：

- 父子类的返回值完全一致
- 子类返回值可以是父类返回值的子类（比较绕口，见下面的例子）

```java
class A {
    public Number m() {
        return 1;
    }
}
class B extends A {
    @Override
    // 子类 m 方法的返回值是 Integer 是父类 m 方法返回值 Number 的子类
    public Integer m() {
        return 2;
    }
}
```

对于子类，java 编译器会做如下处理：

```java
class B extends A {
    public Integer m() {
        return 2;
    }
    // 此方法才是真正重写了父类 public Number m() 方法
    public synthetic bridge Number m() {
        // 调用 public Integer m()
        return m();
    }
}
```

其中桥接方法比较特殊，仅对 java 虚拟机可见，并且与原来的 public Integer m() 没有命名冲突，可以用下面反射代码来验证：

```java
for (Method m : B.class.getDeclaredMethods()) {
    System.out.println(m);
}
```

会输出：

```java
public java.lang.Integer test.candy.B.m()
public java.lang.Number test.candy.B.m()
```



### 类加载阶段

**java类的生命周期：**

指一个class文件从加载到卸载的全过程，类的完整生命周期包括7个部分：加载——验证——准备——解析——初始化——使用——卸载，如下图所示：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image59.png)

其中，验证——准备——解析 称为连接阶段，除了解析外，其他阶段是顺序发生的，而解析可以与这些阶段交叉进行，因为Java支持动态绑定（晚期绑定），需要运行时才能确定具体类型；在使用阶段实例化对象。

**类的初始化：**

是完成程序执行的准备工作。在这个阶段，***静态的***（变量，方法，代码块）会被执行。同时在会开辟一块存储空间用来存放静态的数据。初始化只在类加载的时候执行一次

**类的实例化（实例化对象）：**

是指创建（new）一个对象的过程。这个过程中会在堆中开辟内存，将一些**非静态**的 new 出的对象存放在里面。在程序执行的过程中，可以创建多个对象，既多次实例化。每次实例化都会开辟一块新的内存。（就是调用构造函数）

在每个类初始化使用前，都会先对该类进行加载。

类加载有几个步骤，加载 -> 链接-验证 -> 链接-准备 -> 链接-解析 -> 初始化

在编译过程会把**常量的值放入类的常量池中**，在**准备过程会对类变量（static修饰的变量）赋初始值**，也就是零值，同时会将常量的值赋予常量；在**初始化过程会按照类文件中的声明顺序执行类变量的赋值和静态语句块（static{}块）**，如果**父类还没有初始化会先进行父类的初始化**，完成后才会进行子类的初始化。

可以看到在初始化阶段就会执行 static{} 块的语句，而每一个类在运行过程中一般只会被加载一次，只会完成一次初始化的过程，因此也就只会执行 static{} 块一次。

#### 加载

通过类名获取类的二进制字节码，这是通过类加载器来完成的。其加载过程使用“**双亲委派模型**”。

- 将类的字节码载入方法区（1.8后为元空间，在本地内存中）中，内部采用 C++ 的 instanceKlass 描述 java 类，它的重要 ﬁeld 有：
  - _java_mirror 即 java 的类镜像，例如对 String 来说，它的镜像类就是 String.class，作用是把 klass 暴露给 java 使用
  - _super 即父类
  - _ﬁelds 即成员变量
  - _methods 即方法
  - _constants 即常量池
  - _class_loader 即类加载器
  - _vtable 虚方法表
  - _itable 接口方法
- 如果这个类还有父类没有加载，**先加载父类**
- 加载和链接（解析）可能是**交替运行**的

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image58.png)

- instanceKlass保存在**方法区**。JDK 8以后，方法区位于元空间中，而元空间又位于本地内存中
- _java_mirror则是保存在**堆内存**中
- InstanceKlass和*.class(JAVA镜像类)互相保存了对方的地址
- 类的对象在对象头中保存了*.class的地址。让对象可以通过其找到方法区中的instanceKlass，从而获取类的各种信息

#### 链接

##### 验证

当一个类被加载之后，必须要验证一下这个类是否合法，验证类是否符合 JVM规范，安全性检查，比如这个类是不是符合字节码的格式、变量与方法是不是有重复、数据类型是不是有效、继承与实现是否合乎标准等等。总之，这个阶段的目的就是保证加载的类是能够被 jvm 所运行。

##### 准备

为类变量（静态变量）在方法区分配内存，并设置零值。注意：这里是类变量，不是实例变量，实例变量是对象分配到堆内存时根据运行时动态生成的。

为 static 变量分配空间，设置默认值

- static变量在JDK 7以前是存储与instanceKlass末尾。但在JDK 7以后就存储在_java_mirror末尾了
- static变量在分配空间和赋值是在两个阶段完成的。分配空间在准备阶段完成，赋值在初始化阶段完成
- 如果 static 变量是 ﬁnal 的**基本类型**，以及**字符串常量**，那么编译阶段值就确定了，**就是说赋值在准备阶段完成**
- 如果 static 变量是 ﬁnal 的，但属于**引用类型**，那么赋值也会在**初始化阶段完成**

##### 解析

把常量池中的符号引用解析为直接引用：根据符号引用所作的描述，在内存中找到符合描述的目标并把目标指针指针返回。因为仅仅是符号的话，并不知道类、方法、属性到底在内存的何处，解析之后就知道它们在内存中的位置了。

**HSDB的使用**

- 先获得要查看的进程ID

```
jps
```

- 打开HSDB

```
java -cp F:\JAVA\JDK8.0\lib\sa-jdi.jar sun.jvm.hotspot.HSDB
```

- 运行时可能会报错，是因为**缺少一个.dll的文件**，我们在JDK的安装目录中找到该文件，复制到缺失的文件下即可

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611221703.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611221703.png)

- 定位需要的进程

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611221857.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611221857.png)

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611222029.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611222029.png)

**解析的含义**

将常量池中的符号引用解析为直接引用

- 未解析时，常量池中的看到的对象仅是符号，未真正的存在于内存中

```java
public class Demo1 {
   public static void main(String[] args) throws IOException, ClassNotFoundException {
      ClassLoader loader = Demo1.class.getClassLoader();
      //只加载不解析
      Class<?> c = loader.loadClass("com.nyima.JVM.day8.C");
      //用于阻塞主线程
      System.in.read();
   }
}

class C {
   D d = new D();
}

class D {

}
```

- 打开HSDB
  - 可以看到此时只加载了类C

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611223153.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611223153.png)

查看类C的常量池，可以看到类D**未被解析**，只是存在于常量池中的符号

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611230658.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611230658.png)

- 解析以后，会将常量池中的符号引用解析为直接引用

  - 可以看到，此时已加载并解析了类C和类D

  [![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611223441.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200611223441.png)

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200613104723.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200613104723.png)

#### 初始化

类的初始化过程是这样的：**按照顺序自上而下运行类中的变量赋值语句和静态语句，如果有父类，则首先按照顺序运行父类中的变量赋值语句和静态语句。在类的初始化阶段，只会初始化与类相关的静态赋值语句和静态语句，也就是有static关键字修饰的信息，而没有static修饰的赋值语句和执行语句在实例化对象的时候才会运行。执行类构造器clinit()方法的过程**，虚拟机会保证这个类的『构造方法』的线程安全（clinit是class initialize的简写）

- clinit()方法是由编译器自动收集类中的所有类变量的**赋值动作和静态语句块**（static{}块）中的语句合并产生的

**实例化：**在堆区分配内存空间，执行实例对象初始化，设置引用变量a指向刚分配的内存地址

![](https://img-blog.csdnimg.cn/img_convert/d18b3a81e0a2b17d41556f133dda8b92.png)

**注意**

编译器收集的顺序是由语句在源文件中**出现的顺序决定**的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它**之后**的变量，在前面的静态语句块**可以赋值，但是不能访问**，如

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20201118204542.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20201118204542.png)

##### 发生时机

**类的初始化的懒惰的**，以下情况会初始化

- main 方法所在的类，总会被首先初始化
- 首次访问这个类的静态变量或静态方法时
- 子类初始化，如果父类还没初始化，会引发
- 子类访问父类的静态变量，只会触发父类的初始化
- Class.forName
- new 会导致初始化

以下情况不会初始化

- 访问类的 static ﬁnal 静态常量（基本类型和字符串）
- 类对象.class 不会触发初始化
- 创建该类对象的数组
- 类加载器的.loadClass方法
- Class.forNamed的参数2为false时

**验证类是否被初始化，可以看改类的静态代码块是否被执行**

### 类加载器

Java虚拟机设计团队有意把类加载阶段中的**“通过一个类的全限定名来获取描述该类的二进制字节流”**这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需的类。实现这个动作的代码被称为**“类加载器”**（ClassLoader）

以JDK 8为例

| 名称                                      | 加载的类              | 说明                            |
| ----------------------------------------- | --------------------- | ------------------------------- |
| Bootstrap ClassLoader（启动类加载器）     | JAVA_HOME/jre/lib     | 无法直接访问                    |
| Extension ClassLoader(拓展类加载器)       | JAVA_HOME/jre/lib/ext | 上级为Bootstrap，**显示为null** |
| Application ClassLoader(应用程序类加载器) | classpath             | 上级为Extension                 |
| 自定义类加载器                            | 自定义                | 上级为Application               |

注：下面的描述中将 Bootstrap ClassLoader 简写为 B，Extension ClassLoader 简写为 E，Application ClassLoader 简写为 A。

它们几个实际上是有层级关系的。并且它们各司其职。B 只负责加载 JAVA_HOME/jre/lib 目录下的所有的类，不在这个目录下的类它不闻不问。同理 E 只加载 JAVA_HOME/jre/lib/ext ，A 只加载 classpath（类路径下的类，比如我们自定义的类，jdk 中没有的类）

当类加载器加载一个类的时候，首先经过 A ，A 会查找该类是否已经被自己曾经加载过，会有个缓存，如果自己没有加载过，那么 A 就会找到它的上级 E，问 E 是否加载过这个类，这个时候 E 也会查看缓存，看看自己曾经是否加载过这个类，如果 E 也没有加载过，那么 A 会让 E 再去问 B，看 B 有没有加载过这个类，如果 B 也没有加载过这个类，也就是说 A 的两个上级都没有加载过这个类，那么才轮到 A 自己去加载这个类。

比如加载 String 类，先 A ，A没加载过，A 问 E，E 没加载过，E 问 B ，因为 String 类就在 JAVA_HOME/jre/lib 目录下，所以 B 加载了 String 类，A 和 E 就都不用操心了。如果是一个自定义的 Student 类的话，那么问了一圈，最后还是 A 来加载。

可以看出双亲委派模式，就是先从下到上询问，再从上到下加载。

#### 启动类加载器

可通过在控制台输入指令，使得类被启动类加器加载

#### 拓展类加载器

如果classpath和JAVA_HOME/jre/lib/ext 下有同名类，加载时会使用**拓展类加载器**加载。当应用程序类加载器发现拓展类加载器已将该同名类加载过了，则不会再次加载

#### 双亲委派模式

双亲委派模式，即调用类加载器ClassLoader 的 loadClass 方法时，查找类的规则

loadClass源码

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 1.首先查找该类是否已经被当前类加载器加载过了，这里查一个缓存，加载器加载过的类的缓存记录。
        Class<?> c = findLoadedClass(name);
        //如果没有被加载过
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                
                if (parent != null) {
                    // 2. 有上级的话，委派上级 loadClass
                    c = parent.loadClass(name, false);
                } else {
                    // 3. 如果没有上级了（到ExtClassLoader了），则委派BootstrapClassLoader，这里解释一下，ExtClassLoader的上级其实是BootstrapClassLoader，但是这显示为null，原因是BootstrapClassLoader底层是c++，不是java，因此java访问不到，所以为null
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
                //捕获异常，但不做任何处理
            }

            if (c == null) {
                // 4. 每一层都找不到，调用 findClass 方法（每个类加载器自己扩展）来加载
                //先让拓展类加载器调用findClass方法去找到该类，如果还是没找到，就抛出异常，抛出的异常会在应用类加载器中catch到
                //然后让应用类加载器去找classpath我们的类路径下找该类
                long t1 = System.nanoTime();
                c = findClass(name);

                // 5.记录时间
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

例如：

```java
public class Load5_3 {
    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> aClass = Load5_3.class.getClassLoader()
            .loadClass("cn.itcast.jvm.t3.load.H");
        System.out.println(aClass.getClassLoader());
    }
}
```

执行流程为：

1. sun.misc.Launcher$AppClassLoader //1 处， 开始查看已加载的类，结果没有
2. sun.misc.Launcher$AppClassLoader // 2 处，委派上级 sun.misc.Launcher$ExtClassLoader.loadClass()
3. sun.misc.Launcher$ExtClassLoader // 1 处，查看已加载的类，结果没有
4. sun.misc.Launcher$ExtClassLoader // 3 处，没有上级了，则委派 BootstrapClassLoader 查找
5. BootstrapClassLoader 是在 JAVA_HOME/jre/lib 下找 H 这个类，显然没有
6. sun.misc.Launcher$ExtClassLoader // 4 处，调用自己的 findClass 方法，是在 JAVA_HOME/jre/lib/ext 下找 H 这个类，显然没有，回到 sun.misc.Launcher$AppClassLoader 的 // 2 处
7. 继续执行到 sun.misc.Launcher$AppClassLoader // 4 处，调用它自己的 findClass 方法，在 classpath 下查找，找到了

#### 线程上下文类加载器

我们在使用 JDBC 时，都需要加载 Driver 驱动，不知道你注意到没有，不写

```java
Class.forName("com.mysql.jdbc.Driver")
```

也是可以让 com.mysql.jdbc.Driver 正确加载的，你知道是怎么做的吗？

让我们追踪一下源码：

```java
public class DriverManager {
    // 注册驱动的集合
    private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers
        = new CopyOnWriteArrayList<>();
    // 初始化驱动
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
    ...
}
```

先不看别的，看看 DriverManager 的类加载器：

```java
System.out.println(DriverManager.class.getClassLoader());
```

打印 null，表示它的类加载器是 Bootstrap ClassLoader，会到 JAVA_HOME/jre/lib 下搜索类，但 JAVA_HOME/jre/lib 下显然没有 mysql-connector-java-5.1.47.jar 包，这样问题来了，在 DriverManager 的静态代码块中，怎么能正确加载 com.mysql.jdbc.Driver 呢？

继续看核心 loadInitialDrivers() 方法：

```java
private static void loadInitialDrivers() {
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }
    // 1）使用 ServiceLoader 机制加载驱动，即 SPI
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            ServiceLoader<Driver> loadedDrivers =
                ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
                // Do nothing
            }
            return null;
        }
    });
    println("DriverManager.initialize: jdbc.drivers = " + drivers);
    // 2）使用 jdbc.drivers 定义的驱动名加载驱动
    if (drivers == null || drivers.equals("")) {
        return;
    }
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            // 这里的 ClassLoader.getSystemClassLoader() 就是应用程序类加载器
            Class.forName(aDriver, true,
                          ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```

我们看到，loadInitialDrivers() 方法中，实现类加载的方式有两个：

1. 使用 ServiceLoader 机制加载驱动，即 SPI
2. 使用 jdbc.drivers 定义的驱动名加载驱动

先看 2 发现它最后是使用 Class.forName 完成类的加载和初始化，

```java
 Class.forName(aDriver, true, ClassLoader.getSystemClassLoader());
```

其中用 `ClassLoader.getSystemClassLoader()` 就代表关联的是应用程序类加载器来加载，也就是说，JDK 打破了双亲委派模式，其实 JDK 在某些情况下确实需要打破双亲委派模式。因此可以顺利完成类加载。

再看 1 它就是大名鼎鼎的 Service Provider Interface （SPI）使用 ServiceLoader 机制加载驱动，面向接口编程 + 解耦。

**SPI 的使用规则**：要在 jar 包下建一个 META-INF/services 包，包里新建一个文件，文件的命名就是接口的全限定名，是一个普通的文本文件，文件内部的内容就是这个接口的实现类，只要按照这个约定去设计 jar 包，那么将来我们便可以配合 ServiceLoader 机制，来根据接口找到它的实现类并加以实例化，实现解耦。

约定：在 jar 包的 META-INF/services 包下，以接口全限定名名为文件，文件内容是实现类名称。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm/image60.png)

这样就可以使用

```java
ServiceLoader<接口类型> allImpls = ServiceLoader.load(接口类型.class);
Iterator<接口类型> iter = allImpls.iterator();
while(iter.hasNext()) {
    iter.next();
}
```

来得到实现类，体现的是【面向接口编程+解耦】的思想，在下面一些框架中都运用了此思想：

- JDBC
- Servlet
- Spring 容器
- Dubbo（但是 Dubbo 中对 SPI 进行了扩展）

接着看 ServiceLoader.load 方法：

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    // 获取线程上下文类加载器
    // 通过如下这种方式获得的加载器称为线程上下文加载器，其实就是每个当前线程的应用程序类加载器A，说到底也是用A实现的类加载
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

线程上下文类加载器是当前线程使用的类加载器，默认就是应用程序类加载器，它内部又是由 Class.forName 调用了线程上下文类加载器完成类加载，具体代码在 ServiceLoader 的内部类 LazyIterator 中：

```java
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn + " not a subtype");
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error(); // This cannot happen
}
```

#### 自定义类加载器

##### 使用场景

- 想加载非 classpath 随意路径中的类文件
- 通过接口来使用实现，希望解耦时，常用在框架设计
- 这些类希望予以隔离，不同应用的同名类都可以加载，不冲突，常见于 tomcat 容器

##### 步骤

- 继承ClassLoader父类
- 要遵从双亲委派机制，重写 ﬁndClass 方法
  - 不是重写loadClass方法，否则不会走双亲委派机制
- 读取类文件的字节码
- 调用父类的 deﬁneClass 方法来加载类
- 使用者调用该类加载器的 loadClass 方法

#### 破坏双亲委派模式

- 双亲委派模型的第一次“被破坏”其实发生在双亲委派模型出现之前——即JDK1.2面世以前的“远古”时代
  - 建议用户重写findClass()方法，在类加载器中的loadClass()方法中也会调用该方法
- 双亲委派模型的第二次“被破坏”是由这个模型自身的缺陷导致的
  - 如果有基础类型又要调用回用户的代码，此时也会破坏双亲委派模式
- 双亲委派模型的第三次“被破坏”是由于用户对程序动态性的追求而导致的
  - 这里所说的“动态性”指的是一些非常“热”门的名词：代码热替换（Hot Swap）、模块热部署（Hot Deployment）等

### 运行期优化

#### 分层编译

JVM 将执行状态分成了 5 个层次：

- 0层：解释执行，用解释器将字节码翻译为机器码
- 1层：使用 C1 **即时编译器**编译执行（不带 proﬁling）
- 2层：使用 C1 即时编译器编译执行（带基本的profiling）
- 3层：使用 C1 即时编译器编译执行（带完全的profiling）
- 4层：使用 C2 即时编译器编译执行

proﬁling 是指在运行过程中收集一些程序执行状态的数据，例如【方法的调用次数】，【循环的回边次数】等。

JVM 将整个字节码执行分为 5 个层次。解释执行：就是字节码被加载到 JVM 后，靠一个解释器一个字节一个字节解释执行。解释器就是会将字节码**解释**为真正的机器码，从而让 CPU 可以识别并执行。注意，当字节码被反复调用的时候，反复调用达到一定的阈值后，它就会启用即时编译器对字节码进行**编译**，编译为机器码并存入 Code Cache，下次遇到相同的代码，直接执行，无需再编译，提升效率。

##### 即时编译器（JIT）与解释器的区别

- 解释器
  - 将字节码**解释**为机器码，下次即使遇到相同的字节码，仍会执行重复的解释
  - 是将字节码解释为针对所有平台都通用的机器码
- 即时编译器
  - 将一些字节码**编译**为机器码，**并存入 Code Cache**，下次遇到相同的代码，直接执行，无需再编译
  - 根据平台类型，生成平台特定的机器码

对于大部分的不常用的代码，我们无需耗费时间将其编译成机器码，而是采取解释执行的方式运行；另一方面，对于仅占据小部分的热点代码，我们则可以将其编译成机器码，以达到理想的运行速度。 执行效率上简单比较一下 Interpreter < C1 < C2，总的目标是发现热点代码（hotspot名称的由来），并优化这些热点代码。

上面的优化手段 C2 可以称之为【逃逸分析】，发现新建的对象是否逃逸。可以使用 -XX:-DoEscapeAnalysis 关闭逃逸分析。

[参考资料](https://docs.oracle.com/en/java/javase/12/vm/java-hotspot-virtual-machine-performance-enhancements.html#GUID-D2E3DC58-D18B-4A6C-8167-4A1DFB4888E4)

##### 逃逸分析

逃逸分析（Escape Analysis）简单来讲就是，Java Hotspot 虚拟机可以分析新创建对象的使用范围，并决定是否在 Java 堆上分配内存的一项技术。

逃逸分析的 JVM 参数如下：

- 开启逃逸分析：-XX:+DoEscapeAnalysis
- 关闭逃逸分析：-XX:-DoEscapeAnalysis
- 显示分析结果：-XX:+PrintEscapeAnalysis

逃逸分析技术在 Java SE 6u23+ 开始支持，并默认设置为启用状态，可以不用额外加这个参数

**对象逃逸状态**

**全局逃逸（GlobalEscape）**

- 即一个对象的作用范围逃出了当前方法或者当前线程，有以下几种场景：
  - 对象是一个静态变量
  - 对象是一个已经发生逃逸的对象
  - 对象作为当前方法的返回值

**参数逃逸（ArgEscape）**

- 即一个对象被作为方法参数传递或者被参数引用，但在调用过程中不会发生全局逃逸，这个状态是通过被调方法的字节码确定的

**没有逃逸**

- 即方法中的对象没有发生逃逸

**逃逸分析优化**

针对上面第三点，当一个对象**没有逃逸**时，可以得到以下几个虚拟机的优化

**锁消除**

我们知道线程同步锁是非常牺牲性能的，当编译器确定当前对象只有当前线程使用，那么就会移除该对象的同步锁

例如，StringBuffer 和 Vector 都是用 synchronized 修饰线程安全的，但大部分情况下，它们都只是在当前线程中用到，这样编译器就会优化移除掉这些锁操作

锁消除的 JVM 参数如下：

- 开启锁消除：-XX:+EliminateLocks
- 关闭锁消除：-XX:-EliminateLocks

锁消除在 JDK8 中都是默认开启的，并且锁消除都要建立在逃逸分析的基础上

**标量替换**

首先要明白标量和聚合量，**基础类型**和**对象的引用**可以理解为**标量**，它们不能被进一步分解。而能被进一步分解的量就是聚合量，比如：对象

对象是聚合量，它又可以被进一步分解成标量，将其成员变量分解为分散的变量，这就叫做**标量替换**。

这样，如果一个对象没有发生逃逸，那压根就不用创建它，只会在栈或者寄存器上创建它用到的成员标量，节省了内存空间，也提升了应用程序性能

标量替换的 JVM 参数如下：

- 开启标量替换：-XX:+EliminateAllocations
- 关闭标量替换：-XX:-EliminateAllocations
- 显示标量替换详情：-XX:+PrintEliminateAllocations

标量替换同样在 JDK8 中都是默认开启的，并且都要建立在逃逸分析的基础上

**栈上分配**

当对象没有发生逃逸时，该**对象**就可以通过标量替换分解成成员标量分配在**栈内存**中，和方法的生命周期一致，随着栈帧出栈时销毁，减少了 GC 压力，提高了应用程序性能

#### 方法内联

##### **内联函数**

内联函数就是在程序编译时，编译器将程序中出现的内联函数的调用表达式用内联函数的函数体来直接进行替换

##### **JVM内联函数**

C++是否为内联函数由自己决定，Java由**编译器决定**。Java不支持直接声明为内联函数的，如果想让他内联，你只能够向编译器提出请求: 关键字**final修饰** 用来指明那个函数是希望被JVM内联的，如

```java
public final void doSomething() {  
        // to do something  
}
```

总的来说，一般的函数都不会被当做内联函数，只有声明了final后，编译器才会考虑是不是要把你的函数变成内联函数

JVM内建有许多运行时优化。首先**短方法**更利于JVM推断。流程更明显，作用域更短，副作用也更明显。如果是长方法JVM可能直接就跪了。

第二个原因则更重要：**方法内联**

如果JVM监测到一些**小方法被频繁的执行**，它会把方法的调用替换成方法体本身，如：

```java
	private int add4(int x1, int x2, int x3, int x4) { 
		//这里调用了add2方法
        return add2(x1, x2) + add2(x3, x4);  
    }  

    private int add2(int x1, int x2) {  
        return x1 + x2;  
    }
```

方法调用被替换后

```java
	private int add4(int x1, int x2, int x3, int x4) {  
    	//被替换为了方法本身
        return x1 + x2 + x3 + x4;  
    }
```

#### 反射优化

```java
public class Reflect1 {
   public static void foo() {
      System.out.println("foo...");
   }

   public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
      Method foo = Demo3.class.getMethod("foo");
      for(int i = 0; i<=16; i++) {
         foo.invoke(null);
      }
   }
}
```

foo.invoke 前面 0 ~ 15 次调用使用的是 MethodAccessor 的 NativeMethodAccessorImpl 实现

invoke方法源码

```java
@CallerSensitive
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    //MethodAccessor是一个接口，有3个实现类，其中有一个是抽象类
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
```

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200614133554.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200614133554.png)

会由DelegatingMehodAccessorImpl去调用NativeMethodAccessorImpl

NativeMethodAccessorImpl源码

```java
class NativeMethodAccessorImpl extends MethodAccessorImpl {
    private final Method method;
    private DelegatingMethodAccessorImpl parent;
    private int numInvocations;

    NativeMethodAccessorImpl(Method var1) {
        this.method = var1;
    }
	
	//每次进行反射调用，会让numInvocation与ReflectionFactory.inflationThreshold的值（15）进行比较，并使使得numInvocation的值加一
	//如果numInvocation>ReflectionFactory.inflationThreshold，则会调用本地方法invoke0方法
    public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException {
        if (++this.numInvocations > ReflectionFactory.inflationThreshold() && !ReflectUtil.isVMAnonymousClass(this.method.getDeclaringClass())) {
            MethodAccessorImpl var3 = (MethodAccessorImpl)(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());
            this.parent.setDelegate(var3);
        }

        return invoke0(this.method, var1, var2);
    }

    void setParent(DelegatingMethodAccessorImpl var1) {
        this.parent = var1;
    }

    private static native Object invoke0(Method var0, Object var1, Object[] var2);
}
```

```java
//ReflectionFactory.inflationThreshold()方法的返回值
private static int inflationThreshold = 15;
```

- 一开始if条件不满足，就会调用本地方法invoke0
- 随着numInvocation的增大，当它大于ReflectionFactory.inflationThreshold的值16时，就会本地方法访问器替换为一个运行时动态生成的访问器，来提高效率
  - 这时会从反射调用变为**正常调用**，即直接调用 Reflect1.foo()

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200614135011.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/jvm-n/20200614135011.png)

## 内存模型

内存模型内容详见 [JAVA并发：共享模型之内存](https://chanservy.github.io/posts/20211121/java-concurrent-3.html)

