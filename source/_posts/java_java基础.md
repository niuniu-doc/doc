---
title: java基础
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
#### bean的作用域
```
Spring的`scope` :
`Singleton` 一个spring容器中、只有一个`bean` 的实例、spring默认配置
`Prototype` 每次调用新建一个bean
`Request` web项目中、给每个http request请求一个bean实例
`Session` web项目中、给每个http session一个bean实例
`GlobalSession` 只有在Portal应用中有、给每个global http request一个bean实例
```
#### postConstructor
```
@PostConstruct 在 construct之后执行、
相当于java配置方式的 @Bean(initMethod='xxx')、
相当于xml配置方式的 init-method

@PreDestroy 在bean销毁之前执行、
相当于java配置方式的 @Bean(deatroyMethod='xxx')
相当于xml配置方式的 destroy-method
``` 

#### profile 
```
提供不同环境不同配置文件的支持
```

#### 事件
```
Spring的事件为Bean与Bean之间的通信提供了支持、
1) 自定义事件  继承ApplicationEvent
2) 定义事件监听器  实现 ApplicationListener
3) 使用容器发布事件
```

#### 任务
```
1. 异步任务 @Async 
2. 规划任务 @Scheduled 每隔固定时间执行

```

#### 桟
```
主要用来存放函数调用所需要的数据(函数参数、返回地址及函数内部的局部变量)、但、返回值不在桟中、会有一个专门的返回值存储器、
```

#### 变量的生命周期
```
1. 函数中的参数和函数内定义的变量、都分配在桟上、在函数调用时被分配、调用结束释放
2. 数组和对象 存放变量地址的空间是分配在桟上的、存放变量内容的空间是分配在堆上的
函数调用结束、存放变量地址的空间会被立即释放、而存放内容的空间不会、它会因为没有变量引用、而被垃圾回收机制回收掉
```

#### 类和对象的声明周期
```
类加载进内存后、一般不会释放、直到程序结束、一般情况下、类只会加载1次、所以、静态变量在内存中只有一份

对象：每次new创建一个对象的时候、对象产生、在内存中、会存储这个对象的实例变量值、每new一次、对象就会产生一个、就会有一份独立的实例变量
```
