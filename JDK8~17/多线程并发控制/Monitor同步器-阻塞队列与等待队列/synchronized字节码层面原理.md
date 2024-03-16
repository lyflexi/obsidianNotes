# synchronized 同步语句块原理
测试程序：
```java
package org.lyflexi.monitor_synchronized.classLevel;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/15 13:15  
 */public class SynBlockTest {  
    static final Object lock = new Object();  
    static int counter = 0;  
    public static void main(String[] args) {  
        synchronized (lock) {  
            counter++;  
        }  
    }  
}
```
使用IDEA插件JClassViewer查看字节码信息来验证synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中：
- 当执行 monitorenter 指令时，线程试图获取 对象监视器 monitor 的持有权。
- 因为加锁之后MarkWord中的hashcode不见了，所以执行monitorexit指令解锁的同时，也会取回Monitor中的hashcode以及age等信息将lock对象的MarkWord恢复重置, 并唤醒 EntryList
```java
// class version 52.0 (52)
// access flags 0x21
public class org/lyflexi/monitor_synchronized/classLevel/SynBlockTest {

  // compiled from: SynBlockTest.java

  // access flags 0x18
  final static Ljava/lang/Object; lock

  // access flags 0x8
  static I counter

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 7 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lorg/lyflexi/monitor_synchronized/classLevel/SynBlockTest; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x9
  public static main([Ljava/lang/String;)V
    TRYCATCHBLOCK L0 L1 L2 null
    TRYCATCHBLOCK L2 L3 L2 null
   L4
    LINENUMBER 11 L4
    GETSTATIC org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.lock : Ljava/lang/Object;
    DUP
    ASTORE 1
    MONITORENTER
   L0
    LINENUMBER 12 L0
    GETSTATIC org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.counter : I
    ICONST_1
    IADD
    PUTSTATIC org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.counter : I
   L5
    LINENUMBER 13 L5
    ALOAD 1
    MONITOREXIT
   L1
    GOTO L6
   L2
   FRAME FULL [[Ljava/lang/String; java/lang/Object] [java/lang/Throwable]
    ASTORE 2
    ALOAD 1
    MONITOREXIT
   L3
    ALOAD 2
    ATHROW
   L6
    LINENUMBER 14 L6
   FRAME CHOP 1
    RETURN
   L7
    LOCALVARIABLE args [Ljava/lang/String; L4 L7 0
    MAXSTACK = 2
    MAXLOCALS = 3

  // access flags 0x8
  static <clinit>()V
   L0
    LINENUMBER 8 L0
    NEW java/lang/Object
    DUP
    INVOKESPECIAL java/lang/Object.<init> ()V
    PUTSTATIC org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.lock : Ljava/lang/Object;
   L1
    LINENUMBER 9 L1
    ICONST_0
    PUTSTATIC org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.counter : I
    RETURN
    MAXSTACK = 2
    MAXLOCALS = 0
}

```
或者下面通过 JDK 自带的反编译命令javap来查看对应的字节码
```shell
#编译后的.class 文件
javac .\SynBlockTest.java 
#然后执行反编译命令查看 SynBlockTest 类的相关字节码信息。
javap -c -s -v -l  .\SynBlockTest.class
```
信息如下，跟JClassViewer查看的信息类似
```java
PS E:\github\debuginfo_jdkToFramework\debug_jdk\alonejdk\src\main\java\org\lyflexi\monitor_synchronized\classLevel> javap -c -s -v -l  .\SynBlockTest.class
Classfile /E:/github/debuginfo_jdkToFramework/debug_jdk/alonejdk/src/main/java/org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.class
  Last modified 2024-3-15; size 607 bytes
  MD5 checksum abf4195a062238c9641f1cacd205fc91
  Compiled from "SynBlockTest.java"
public class org.lyflexi.monitor_synchronized.classLevel.SynBlockTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#23         // java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#24         // org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.lock:Ljava/lang/Object;
   #3 = Fieldref           #5.#25         // org/lyflexi/monitor_synchronized/classLevel/SynBlockTest.counter:I
   #4 = Class              #26            // java/lang/Object
   #5 = Class              #27            // org/lyflexi/monitor_synchronized/classLevel/SynBlockTest
   #6 = Utf8               lock
   #7 = Utf8               Ljava/lang/Object;
   #8 = Utf8               counter
   #9 = Utf8               I
  #10 = Utf8               <init>
  #11 = Utf8               ()V
  #12 = Utf8               Code
  #13 = Utf8               LineNumberTable
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               StackMapTable
  #17 = Class              #28            // "[Ljava/lang/String;"
  #18 = Class              #26            // java/lang/Object
  #19 = Class              #29            // java/lang/Throwable
  #20 = Utf8               <clinit>
  #21 = Utf8               SourceFile
  #22 = Utf8               SynBlockTest.java
  #23 = NameAndType        #10:#11        // "<init>":()V
  #24 = NameAndType        #6:#7          // lock:Ljava/lang/Object;
  #25 = NameAndType        #8:#9          // counter:I
  #26 = Utf8               java/lang/Object
  #27 = Utf8               org/lyflexi/monitor_synchronized/classLevel/SynBlockTest
  #28 = Utf8               [Ljava/lang/String;
  #29 = Utf8               java/lang/Throwable
{
  static final java.lang.Object lock;
    descriptor: Ljava/lang/Object;
    flags: ACC_STATIC, ACC_FINAL

  static int counter;
    descriptor: I
    flags: ACC_STATIC

  public org.lyflexi.monitor_synchronized.classLevel.SynBlockTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                  // <- lock引用 （synchronized开始）
         3: dup                               // 复制一份lock引用
         4: astore_1                          // lock引用暂存至 -> slot 1
         5: monitorenter                      // 将 lock对象 MarkWord 置为 Monitor 指针
         6: getstatic     #3                  // <- i
         9: iconst_1                          // 准备常数 1
        10: iadd                              // +1
        11: putstatic     #3                  // -> i
        14: aload_1                           // <- 从暂存区取回lock引用
        15: monitorexit                       // 取回Monitor中的hashcode以及age等信息将lock对象的MarkWord重置, 并唤醒 EntryList
        16: goto          24
        19: astore_2                          //19~23是synchronized异常兜底策略
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             6    16    19   any
            19    22    19   any
      LineNumberTable:
        line 11: 0
        line 12: 6
        line 13: 14
        line 14: 24
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: new           #4                  // class java/lang/Object
         3: dup
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: putstatic     #2                  // Field lock:Ljava/lang/Object;
        10: iconst_0
        11: putstatic     #3                  // Field counter:I
        14: return
      LineNumberTable:
        line 8: 0
        line 9: 10
}
SourceFile: "SynBlockTest.java"

```

# synchronized 同步方法原理
测试程序
```Java
package org.lyflexi.monitor_synchronized.classLevel;  
  
/**  
 * @Author: ly  
 * @Date: 2024/3/15 13:36  
 */public class SynMethodTest {  
    public static void main(String[] args) {  
  
    }  
    public synchronized void method() {  
        System.out.println("synchronized 方法");  
    }  
}
```
通过JClassViewer查看，没有monitor指令
```java
// class version 52.0 (52)
// access flags 0x21
public class org/lyflexi/monitor_synchronized/classLevel/SynMethodTest {

  // compiled from: SynMethodTest.java

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 7 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lorg/lyflexi/monitor_synchronized/classLevel/SynMethodTest; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x9
  public static main([Ljava/lang/String;)V
   L0
    LINENUMBER 10 L0
    RETURN
   L1
    LOCALVARIABLE args [Ljava/lang/String; L0 L1 0
    MAXSTACK = 0
    MAXLOCALS = 1

  // access flags 0x21
  public synchronized method()V
   L0
    LINENUMBER 12 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "synchronized \u65b9\u6cd5"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L1
    LINENUMBER 13 L1
    RETURN
   L2
    LOCALVARIABLE this Lorg/lyflexi/monitor_synchronized/classLevel/SynMethodTest; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1
}

```
但是通过JDK 自带的反编译命令javap来查看对应的字节码
```shell
#编译后的.class 文件
javac .\SynMethodTest.java 
#然后执行反编译命令查看 SynBlockTest 类的相关字节码信息。
javap -c -s -v -l  .\SynMethodTest.class
```
发现存在flags: ACC_PUBLIC, ACC_SYNCHRONIZED方法级别的标志
```java
PS E:\github\debuginfo_jdkToFramework\debug_jdk\alonejdk\src\main\java\org\lyflexi\monitor_synchronized\classLevel> javap -c -s -v -l  .\SynMethodTest.class
Classfile /E:/github/debuginfo_jdkToFramework/debug_jdk/alonejdk/src/main/java/org/lyflexi/monitor_synchronized/classLevel/SynMethodTest.class
  Last modified 2024-3-15; size 534 bytes
  MD5 checksum 8879687073fdb261917546f76b06da37
  Compiled from "SynMethodTest.java"
public class org.lyflexi.monitor_synchronized.classLevel.SynMethodTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#16         // java/lang/Object."<init>":()V
   #2 = Fieldref           #17.#18        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #19            // synchronized 方法
   #4 = Methodref          #20.#21        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #22            // org/lyflexi/monitor_synchronized/classLevel/SynMethodTest
   #6 = Class              #23            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               method
  #14 = Utf8               SourceFile
  #15 = Utf8               SynMethodTest.java
  #16 = NameAndType        #7:#8          // "<init>":()V
  #17 = Class              #24            // java/lang/System
  #18 = NameAndType        #25:#26        // out:Ljava/io/PrintStream;
  #19 = Utf8               synchronized 方法
  #20 = Class              #27            // java/io/PrintStream
  #21 = NameAndType        #28:#29        // println:(Ljava/lang/String;)V
  #22 = Utf8               org/lyflexi/monitor_synchronized/classLevel/SynMethodTest
  #23 = Utf8               java/lang/Object
  #24 = Utf8               java/lang/System
  #25 = Utf8               out
  #26 = Utf8               Ljava/io/PrintStream;
  #27 = Utf8               java/io/PrintStream
  #28 = Utf8               println
  #29 = Utf8               (Ljava/lang/String;)V
{
  public org.lyflexi.monitor_synchronized.classLevel.SynMethodTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 10: 0

  public synchronized void method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String synchronized 方法
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 12: 0
        line 13: 8
}
SourceFile: "SynMethodTest.java"

```

