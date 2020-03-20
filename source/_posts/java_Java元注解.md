---
title: Java元注解
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
Java中元注解有4个：`@Retention`, `@Target`, `@Document`, `@Inherited`

`@Retention`注解的保留位置：
`@Retention(RetentionPolicy.SOURCE)` 注解仅存在于源码中、在class字节码文件中不存在
`@Retention(RetentionPolicy.CLASS)` 默认保留策略、注解会在class文件中存在、但、运行时无法获得
`@Retention(RetentionPolicy.RUNTIME)` 注解会在class文件中存在、并且可以通过反射得到

`@Target` 注解的作用目标
`@Target(ElementType.TYPE)` 接口、类、枚举、注解
`@Target(ElementType.FIELD)` 字段、枚举的常量
`@Target(ElementType.METHOD)` 方法
`@Target(ElementType.PARAMETER)` 方法参数
`@Target(ElementType.CONSTRUCTOR)` 构造函数
`@Target(ElementType.LOCAL_VARIABLE)` 局部变量
`@Target(ElementType.ANNTATION_TYPE)` 注解
`@Target(ElementType.PACKAGE)` 包

`@Document` 说明该注解将会被包含在javadoc中

`@Inherited` 说明子类可以继承父类中的该注解
