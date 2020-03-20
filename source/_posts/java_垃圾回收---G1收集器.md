---
title: 垃圾回收---G1收集器
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
#### G1收集器 
上篇文章简单介绍了七种垃圾回收器、既然G1是最前沿的、有必要补充说明一下:
     1. `横跨整个堆内存`: 在G1之前的其它收集器收集范围都是整个新生代或者老年代
         G1在使用时、将整个Java堆划分为多个大小相等的`独立区域(Region)`
         保留新生代和老年代的概念、但不再物理隔离、都是一部分`Region`(不需要连续)的集合
     2. `建立可预测的时间模型`: G1跟踪各个Region里垃圾堆积的价值大小(回收可获得空间大小及回收所需时间的经验值)
         在后台维护一个优先级列表, 每次根据允许的收集时间、优先回收价值最大的Region(Garbage First)
         这种实验Region划分内存空间及有优先级的区域回收方式、保证了G1在有限时间内的收集效率
     3. `避免全堆扫描`: G1将堆分为多个Region、就是化整为零、单Region不可能是孤立的、一个对象分配在某个Region中
         可能与整个Java堆任意的对象发生引用关系、在做可达性分析确定对象是否存活时、需要扫整个Java堆保证准确性
         严重损耗GC效率，
        `注意：` 
为了避免全堆扫描、虚拟机为每个Region维护了一个对应的`Remembered Set`、虚拟机发现程序对在`Reference`类型的
         数据进行写操作时、会产生一个`Writer Barrier`暂时中断写操作、检测`Reference引用的对象`是否`在不同的Region中`
         (在分代的例子中就是检测老年代对象是否引用了新生代对象)、若是则通过`CardTable`把相关引用记录到被引用对象所属的
         Region的`Remembered Set`中、垃圾回收时，在GC根节点的枚举范围内`加入Remembered Set`即可保证不用全堆扫描也能不遗漏
#### 
```
         若不计算维护Remembered Set的操作、G1大致步骤为：
         1) 初始标记：initial mark仅标记GC Roots能直接关联的对象、并且修改TAMS(Nest top mark start)的值
            让下阶段用户程序并发运行时、可在正确可以的Region中创建对象、此阶段需要停顿用户线程、但时间较短
            
         2) 并发标记: concurrent Marking 从GC Root开始对堆中对象做可达性分析、找到存活对象
            耗时较长、但可与用户线程`并发执行`
         
         3) 最终标记: Final Marking 为了修正标记期间因用户程序继续运行导致标记产生变化的对象、
            虚拟机将这段时间内对象变化记录在线程的`Remembered Set Logs`里、最终标记阶段需要将这部分数据合并
            到 Remembered Set 中、需要`停顿线程`、但可`并行执行`
         
         4) 筛选回收: Live Data and Evacuation 先对各个Region中的回收价值和成本进行排序
            根据用户所期望的GC停顿时间来定制回收计划、此阶段也`可以`做到与用户线程`并发执行`、但因为只回收
            一部分Region、且时间是用户可控制的、`停顿用户线程`将大幅度`提高收集效率`
```
##### G1适用分析
    场景Scense：
       1. 多核多CPU而且大内存
       2. 要求低停顿
    
    如何实现-XX:MaxGCPauseMillis所设置的目标？
       1. 对于新生代来说，G1自动调整新生代的大小
       2. 对于老年代来说，每次Mixed Garbage Collecttion的时候回收的Rigion个数会基于
          Mixed Garbage Collection的目标次数，每个Region中的存活对象百分比，以及堆全局的垃圾比例来设定
  
##### 疑问  
    如何实现避免碎片？
       无论是新生代还是老年代，使用复制算法(将一批Region复制到另外一批Region中)
    
    如何避免可达性分析扫描整个堆？
       每一个Region都有自己的RSets(Remembered Set) 用来记录对自己Region内对象的引用
       
    在什么时机进行Mixed Garbage Collection?
       在堆的使用率达到-XX:InitiatingHeapOccupancyPercent所设置的使用率(老年代使用的内存/整个堆内存)时
       
       判断的时机是? 每次Young GC结束的时候
       
       因为老年代空间中的使用内存发生变化只有一个情形：Young GC的时候
    
    G1的哪些过程需要STW？
       1) young gc: 采用的是复制算法 - 会STW、多线程进行
       2) mixed gc: 也是复制算法、mixed gc也包括对新生代的收集
       3) global concurrent mark中的initial mark、remark、部分clean up
    
    既然有CMS，为什么要选择G1?
       1. STW更加可控
       2. 由于采用了标记-整理算法，不会产生内存碎片
    
    Humongous Objects的分配和回收:
       1. 怎么算？大于Region的空间的一半的算Humongous Objects
       2. 放哪里？Humongous Objects被分配在Humongous Region，直接在老年代
       3. 回收：不在mixed gc，而在 global concurrent marking的最后一步：clean up中
    
    可达性分析：
       找一组对象作为GC Root（根结点），并从根结点进行遍历，遍历结束后如果发现某个对象是不可达的,
       那么它就会被标记为不可达对象，等待GC
    
    哪些对象可以作为GC Root
        能作为GC Root的对象必定为可以存活的对象，
        eg. 全局性的引用（静态变量和常量）以及某些方法的局部变量（栈帧中的本地变量表）
        
        以下对象通常可以作为GC Root：
        1) 存活的线程
        2) 虚拟机栈(栈桢中的本地变量表)中的引用的对象
        3) 方法区中的类静态属性以及常量引用的对象
        4) 本地方法栈中JNI引用的局部变量以及全局变量
    


##### G1调优

```
1. -XX:MaxGCPauseMillis
   最大STW时间、参数设置过小会频繁GC、设置过大、吞吐量会降低、所以要合理的设置来保证 
   `最大停顿时间`和`最大吞吐量`的平衡
2. -XX:InitiatingHeapOccupancyPercent & -XX:+G1UseAdaptiveHop
   InitiatingHeapOccupancyPercent简称IHOP
   -XX:+G1UseAdaptiveIHOP是默认值，也就是说IHOP是adaptive的
   -XX:InitiatingHeapOccupancyPercent是初始值
```
 
##### 存在的问题
```
1. 跟cms一样、无法避免浮动垃圾的产生和溢出
    可以增加堆大小、或者GC线程`-XX:ConcGCThreads`来尽量避免
2. 同样存在晋升失败的问题
    可以提升`-XX:G1ReservePercent`同时同比例的增加堆大小、
    或者提前启动标记周期(减少-XX:InitiatingHeapOccupancyPercent)
``` 
##### GC 收集器总结:  
 
|收集器	| 串行、并行or并发 |	新生代/老年代 |	算法	 | 目标	| 适用场景 |
| ---- | ----| -----| -----| -----| ----|
|Serial|	串行	|新生代	|复制算法	|响应速度优先|	单CPU环境下的Client模式|
|Serial Old	|串行	|老年代	|标记-整理	|响应速度优先	|单CPU环境下的Client模式、CMS的后备预案|
|ParNew	|并行|	新生代	|复制算法	|响应速度优先	|多CPU环境时在Server模式下与CMS配合|
|Parallel Scavenge	|并行	|新生代	|复制算法|	吞吐量优先|	在后台运算而不需要太多交互的任务|
|Parallel Old	|并行	|老年代|	标记-整理	吞吐量优先	|在后台运算而不需要太多交互的任务|
|CMS	|并发|	老年代	|标记-清除|	响应速度优先	|集中在互联网站或B/S系统服务端上的Java应用|
|G1	|并发	|both	|标记-整理+复制算法	|响应速度优先	|面向服务端应用，将来替换CMS|

      
###### 附：
     1. `可达性`: GC Root可以到达此对象
     2. `并行执行`: 多个线程同时执行gc
     3. `并发执行`: 用户线程和GC线程同时执行
     

![G1.png](https://upload-images.jianshu.io/upload_images/14027542-907fa5f1c490a4f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
