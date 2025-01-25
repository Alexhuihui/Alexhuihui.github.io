---
title: 【Mybatis】Mybatis源码解析—基础功能包
date: 2023-06-08 19:54:06
categories: Mybatis
tags: Mybatis
---

阅读`mybatis`源码可以先从外围的基础功能包开始，剥洋葱一样一层一层深入

<!-- more --> 

### type 包

#### 归类总结

- 类型处理器
- 类型注册器
- 注解类

#### 设计模式

1. 模板模式

 模板中定义了大体的处理框架，留下一些细节供使用者来完善。在设计模式中，使用一个抽象类定义一整套的操作流程，而抽象类的子类则完成每个操作步骤的实现。

#### 类型处理器

用于ORM框架中处理Java类型和数据库类型，使用模板方式定义了一个`BaseTypeHandler`，`getResult`方法完成了异常处理等统一的工作，而与具体类型的操作则通过抽象方法由具体的子类类型处理器实现。

#### 类型注册表

光有类型处理器还不够，还需要快速查找数据类型对应的类型处理器

根据一个Java类型和JDBC类型就能确定一个类型处理器

```
// TypeHandlerRegistry.class
private <T> TypeHandler<T> getTypeHandler(Type type, JdbcType jdbcType) {
    if (ParamMap.class.equals(type)) {
      return null;
    }
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = getJdbcHandlerMap(type);
    TypeHandler<?> handler = null;
    if (jdbcHandlerMap != null) {
      handler = jdbcHandlerMap.get(jdbcType);
      if (handler == null) {
        handler = jdbcHandlerMap.get(null);
      }
      if (handler == null) {
        // #591
        handler = pickSoleHandler(jdbcHandlerMap);
      }
    }
    // type drives generics here
    return (TypeHandler<T>) handler;
  }
```

### io 包

io包即输入/输出包，负责完成 MyBatis中与输入/输出相关的操作

#### 设计模式

##### 单例模式

单例模式（Singleton Pattern）是一种非常简单的设计模式。使 用了单例模式的类提供一个方法得到该类的对象，并且总保证这个对 象是唯一的。

```
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    
    private Singleton() {};
    
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

##### 代理模式

代理模式（Proxy Pattern）是指建立某一个对象的代理对象，并且由代理对象控制对原对象的引用。

### logging 包

logging包负责完成 MyBatis操作中的日志记录工作。

#### 设计模式

##### 适配器模式

适配器模式（Adapter Pattern）是一种结构型模式，基于该模式 设计的类能够在两个或者多个不兼容的类之间起到沟通桥梁的作用。 转换插头就是一个适配器的典型例子。不同的转换插头能够适配 不同国家的插座标准，从而使得一个电器能在各个国家使用。