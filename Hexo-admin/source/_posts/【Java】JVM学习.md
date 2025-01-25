---
title: 【Java】JVM学习
date: 2023-02-24 19:54:06
categories: JVM
tags: JVM
---

[JVM 的垃圾回收](https://tobebetterjavaer.com/jvm/gc.html)，其实就是收拾那些不再使用的 Java 对象，把他们曾经占用的内存重新释放出来。所以我们要搞清楚：

- [对象是如何创建的](https://tobebetterjavaer.com/jvm/whereis-the-object.html)？对象是如何被访问的？到底哪些对象是废弃的？于是我们就需要搞清楚对象的生和死。
- 这些废弃了的对象到底放在哪？于是就需要了解[JVM 的内存结构](https://tobebetterjavaer.com/jvm/neicun-jiegou.html)：方法区、堆、程序计数器、虚拟机栈和本地方法栈。
- 这些废弃了的对象会不会造成内存泄露（OOM，OutOfMemoryError）？于是我们就需要了解每个分区的 OOM。
- 这些废弃了对象什么时候被回收？于是我们就需要了解垃圾回收算法，比如说清除算法、复制算法、标记整理算法和分代收集算法。

知道了一个对象在内存中的生和死，我们还需要知道类是如何在内存中变成对象的？对象的方法是如何执行的？

<!-- more --> 

于是我们开始学习 Java 虚拟机的执行过程，学习[字节码文件](https://tobebetterjavaer.com/jvm/class-file-jiegou.html)（ .class 文件），学习[类的加载机制](https://tobebetterjavaer.com/jvm/class-load.html)，学习[虚拟机栈的栈帧结构](https://tobebetterjavaer.com/jvm/how-jvm-run-zijiema-zhiling.html)，学习方法的调用过程，学习[字节码指令](https://tobebetterjavaer.com/jvm/zijiema-zhiling.html)等等。

为了监控虚拟机和故障排查，我们需要学习[常用的 JDK 命令行工具](https://tobebetterjavaer.com/jvm/problem-tools.html)，掌握必要的线上问题排查方法；此外，还需要了解 JIT (Just In Time)并不是简单的将热点代码编译成机器码就收工的，它还会对代码的执行进行优化（[方法内联和逃逸分析](https://tobebetterjavaer.com/jvm/jit.html)）。

JVM 相关的知识已经成为面试必考的科目了，但老实讲，JVM 相关的知识还真的不太好用在项目中，或者说不太好在项目中体现出来。

那这里给大家推荐一个实战项目，基于 Spring Boot 的在线 Java IDE，可以远程执行 Java 代码并将程序的运行结果反馈出来。涉及了 Java 类文件的结构、Java 类加载器和 Java 类的热替换等 JVM 相关的技术。

> https://github.com/TangBean/OnlineExecutor

听我这么一说，是不是一下子就清晰多了！