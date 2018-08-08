---
layout:     post                    # 使用的布局（不需要改）
title:      Java 字节码             # 标题
subtitle:   从入门到放弃            #副标题
date:       2018-07-08              # 时间
author:     west                    # 作者
header-img: img/post_green.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - bytecode
---



#### 简单的例子


```
# 一段简单的代码
println(3+4);
```

```
# 词法分析
3 4 + println
```
 
```
3       // push 3 on the stack
4       // push 4 on the stack
+       // consume 3 and 4 and push the result on the stack
println // consume the result and print it
```

***问题:如何将上面这段代码转化成为字节码？***


---
##### 相关基础知识

- JVM栈模型

![](http://java8.in/wp-content/uploads/2014/07/JVM_Structure.png)

- Object Model

```
                                        
 *                                   Hotspot java object model(normal)
 *                                   /
 *            +--------------------+
 *            |      markWord      | 
 *            +--------------------+  
 *            |      metaData      | \
 *            +--------------------+  \
 *            |       field 0      |   Point to the Runtime  Class Object
 *            +--------------------+
 *            |       ....         |
 *            +--------------------+
 *            |       field N      |
 *            +--------------------+ 
 *            |       padding      |
 *            +--------------------+
 
//  markWord bits layout :
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
```

#### ByteCode简介

- 字节码设计理念：***小而紧凑,面向对象***

##### 字节码格式
```
<index> <opcode> [ <operand1> [ <operand2>... ]] [<comment>]
```
- opcode 的大小只有 1 byte(目前字节码总数 xx)；
- 针对一些字节码操作做出优化,使得字节码更短;

##### 字节码类型
- 分类型：数值类型，引用类型 => [字节码助记符](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-7.html)
- 总结： **针对不同的的运行时数据结构，VM 定义了相关的字节码，比如局部变量表的load+store两种字节码，再结合相关的类型，就是完整的局部变量表字节码集合** 

- **所以，经过上面的分析，printf(3+4) 转化为字节码是不是很简单了呢？**
  - 大概的字节码如下(伪码)

```
# 假设 3 和 4 为 int 类型: 

iconst_3       // push 3 on the stack
iconst_4       // push 4 on the stack
iadd           // consume 3 and 4 and push the result on the stack
invokestatic  println // consume the result and print it
```


#### Example

- 复杂代码

```
 ## ConcurrentLinkedQueen.offer(E e)

 public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
          if (p.casNext(null, newNode)) {
             if (p != t)                   
                 casTail(t, newNode);
                 return true;
             }
        } else if (p == q)
            p = (t != (t = tail)) ? t : head;          // 这行代码有点复杂，对应的字节码如下
        else
           p = (p != t && t != (t = tail)) ? t : q;
    }
 }
```

```
public boolean offer(E);
    Code:
      ...

      58: aload         4                  // localvarTable[4] = p;
      60: aload         5                  // localvarTable[5] = q;
      62: if_acmpne     88                 // if (q==p) 
      65: aload_3                          // load t
      66: aload_0                          // load this
      67: getfield      #4                 // Field tail:Ljava/util/concurrent/ConcurrentLinkedQueue$Node;
      70: dup                              // operadStack :... t(old value),tail,tail
      71: astore_3                         // operadStack :... t(old value), tail   => update t
      72: if_acmpeq     79                 // if( t(old value )== tail)

      ....

```
---

- **NoSuchMethodError导致生产环境Cat OOM**

> - 由于Java8 编译，Java7 环境运行，导致NoSuchMethodError：
> - 下图是cat server 消费消息的流程：
>    - NosuchMehodError 最终导致DumpAnalyzer 挂掉，消息无法及时刷回磁盘，最终导致OOM。
> - 得出的教训：
>   - 各个环境的JDK运行环境要一致；
>   - 有时环境问题并不能选择，那么面向接口编程是最好的选择；

```
graph LR
Receiver--> realComsumer
realComsumer--> PeriodTask
PeriodTask--> Analyzer
Analyzer-->makeReport
Analyzer-->dumpFile
```
- 罪魁祸首

```
    // 原先的代码
    ConcurrentHashMap<String, String> map = new ConcurrentHashMap<String, String>();
    map.keySet();
        
        //修改后的代码
    ConcurrentMap<String, String> map1 = new ConcurrentHashMap<String, String>();
    map1.keySet();
```

```
    Code:
       0: new           #2                  // class java/util/concurrent/ConcurrentHashMap
       3: dup
       4: invokespecial #3                  // Method java/util/concurrent/ConcurrentHashMap."<init>":()V
       7: astore_1
       8: aload_1
       9: invokevirtual #4                  // Method java/util/concurrent/ConcurrentHashMap.keySet:()Ljava/util/concurrent/ConcurrentHashMap$KeySetView;
      12: pop
      13: new           #2                  // class java/util/concurrent/ConcurrentHashMap
      16: dup
      17: invokespecial #3                  // Method java/util/concurrent/ConcurrentHashMap."<init>":()V
      20: astore_2
      21: aload_2
      22: invokeinterface #5,  1            // InterfaceMethod java/util/concurrent/ConcurrentMap.keySet:()Ljava/util/Set;
      27: pop
      28: return

```

#### Finally

献上 [slidesPPT之Java字节码从入门到放弃](https://slides.com/heizhan/deck#/)

```
 _____________ 
< ThankYou ~  >
 ------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

```


#### 参考

- [The Java® Virtual Machine Specification Java SE 8 Edition](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)
- [wiki-Java virtual machine](https://en.wikipedia.org/wiki/Java_virtual_machine)
- [markOop](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp)
