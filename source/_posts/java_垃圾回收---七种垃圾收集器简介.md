---
title: 垃圾回收---七种垃圾收集器简介
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---


####基本概念：

新生代：大多数对象在eden区生产、很多对象的什么周期很短、每次新生代的垃圾回收(`minor gc`)后只有少量对象存活
      所以选用的是复制算法、只需要少量的复制成本就可以完成回收
      `young generation` 一般又分为 `eden`区 和 `survivor` 区(一般`2`个`survivor`区)
      大部分对象在eden区生产、eden区满时、还存活的对象被复制到两个survivor区中的一个、这个survivor区满时
      此区存活切不满足`晋升`条件的对象将被复制到另一个survivor区(对象每经历一次`minor gc`， `age` + 1， 
      到达年龄阈值后、被放到老年代`old generation`, 在`Serial`和`ParNew GC`)两种垃圾回收器中、`晋升年龄阈值`
      由参数 `MaxTenuringThreshould`设定、默认15

老年代：`old generation`在新生代中经历了n次垃圾回收后仍然存活的对象、会被放到老年代、改区域对象存活的几率比较高
      老年代的垃圾回收(`Major gc`)通常使用 `标记-清理`或者`标记-整理`算法，整堆的回收(包括young和old)称为`full gc`
      (HotSpot VM)中、除了`CMS`之外、其它能收集老年代的gc都会同时回收整个GC堆、包括新生代
      
永久代: `perm generation`主要存放元数据、eg. `class`, `method`的元信息、与垃圾回收要回收的关系不大、相对新生代
      和老年代、改区域对垃圾回收的影响较小


#### 常见的垃圾回收器

##### 新生代垃圾收集器

1. `Serial`：串行收集器、采用复制算法、jdk1.3之前的唯一选择、单线程、进行垃圾收集时、会STW
          `jdk1.3`之后又很多优秀收集器、单由于它 对于限定单个cpi的环境来说、没有线程交互的开销、简单高效
          至今仍然是 hotspot 虚拟机运行在client模式下的默认新生代收集器

![serial & serial old.png](https://upload-images.jianshu.io/upload_images/14027542-ffea12d750747d02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. `ParNew`：`Serial`的多线程版本、除了使用多线程进行垃圾收集外、其余行为包括`Serial收集器可用的控制参数`，
         `收集算法`, `stw`, `对象分配规则`, `回收策略`等 与`Serial`收集器的完全相同、两者公用了很多代码
         ParNew收集器除了使用多线程收集外、其它与Serial收集器并无太多创新之处、`但`是在Server模式下的首选
         一个与性能无关的原因:除Serial外、它是唯一能够和`CMS`收集器(`Concurrent Mark Sweep`)配合工作的         
         `ParNew` 收集器在 `单CPU` 的环境中、不会有比`Serial`收集器更好的效果、甚至由于存在线程交互的开销
         在两个CPU的环境中都不能百分之百的保证超越、在多CPU的环境下、随着`CPU的增加`、在GC时、
         对资源的有效利用是有好处的、默认开启的`收集线程`数和`CPU的数量`相同, 在CPU非常多的时候、可以通过 
         `-XX:ParallerGCThreads`参数设置
      
![ParNew & Serial Old.png](https://upload-images.jianshu.io/upload_images/14027542-7b2a44d4cfdc7d12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


3. `Parallel Scavenge`: 收集器是`并行`的`多线程` `新生代`收集器, 使用`复制`算法
         它与其它收集器的关注点不同、CMS等收集器的关注点是：尽可能缩短垃圾收集时用户线程的停顿时间
         而Parallel Scavenge的关注点是：达到一个可控制的吞吐量
         `停顿短`适合需要与用户交互的程序、良好的响应可以提高用户体验
         `高吞吐量`则可以高效的利用CPU时间、尽快的完成任务、适合在后台运算无需太多交互的任务
         `Parallel Scavenge`收集器提供了一个开关参数 `-XX:UseAdaptiveSizePolicy`指定后可以不用人工指定
         新生代大小(-Xmn)、Eden和Survivor区的比率(-XX:SurvivorRatio)、
         晋升老年代年龄(-XX:PretenureSizeThreshold)等参数细节了、虚拟机会根据当前系统的运行情况、动态调整参数
         这种方式称为：`GC的自适应调节策略`
         `注意` Parallel Scavenge收集器无法和CMS配合使用、在`JDK1.6`推出`Parallel Old`之前、
         只能和`Serail Old`收集器一起使用
·
    ![Paralle Scavenge.png](https://upload-images.jianshu.io/upload_images/14027542-601ead2dd3099f9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



##### 老年代收集器
1. `Serial Old`是`Serial`收集器的老年代版本、`单线程`, 使用`标记-整理`(`Mark-Compact`)算法
            1) JDK1.5及之前的版本(Parallel Old)诞生之前、与 `Parallel Scavenge`配合使用
            2) 作为CMS收集器的后备预案、在并发手机发生 `Concurrent Mode Failuer`时使用
      
2. `Parallel Old`是`Parallel Scavenge`的老年代版本, 使用`多线程`和`标记-整理`算法, `JDK1.6`的版本才开始提供
            在它出现之后`吞吐量优先`才有了名副其实的应用组合、在注重吞吐量和cpu敏感的场合、都可以优先考虑
            `Parallel Scavenge`和`Parallel Old`组合     
      
3. `CMS`:`Concurrent Mark Sweep`收集器是一种以最短回收停顿时间为目标的收集器、使用`标记-清除`算法     
      基本流程：
      1) `初始标记`:`CMS Initial mark`仅标记`GC Roots`能直接关联的对象、速度很快, 会 STW
      2) `并发标记`:`CMS Concurrent mark`是`GC Roots Tracing`的过程、耗时最长、但不会STW
      3) `重新标记`:`CMS remark`为了修正并发标记期间用户程序继续访问导致标记产生变动的对象的标记
          时间会比初始标记长、单远小于并发标记, 需要 STW
      4) `并发清除`: `CMS concurrent sweep`
      缺点：1) CPU资源敏感、在并发阶段不会导致停顿、单会占用CPU资源、导致应用程序变慢、总的吞吐量降低
              默认启动的回收线程数是`(CPU数+3)/4`，即不少于25%的CPU资源、且随着cpu的增加而下降
              但、CPU资源不足时、cms对用户程序的影响可能变大、若CPU的负载本身就很高、还要分出一般运算能力
              去执行垃圾收集、系统也会有明显的卡顿
            2)无法处理浮动垃圾`Floating Garbage`可能出现`Concurrent Mode Failure`而导致Full gc
               由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生。
               出现在标记过程之后，无法再当次收集中处理掉它们，只能待下一次GC时再清理掉。这一部分垃圾就被称为`浮动垃圾`
            3) 产生空间碎片: 空间碎片过多时、会导致老年代空间有剩余、单无法找到足够大连续空间来分配当前对象

![cms.png](https://upload-images.jianshu.io/upload_images/14027542-a48e748a7206f8ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


4. G1 收集器
      `G1`: `Garbage First`收集器是新的、面向Server应用、HotSpot团队赋予它的使命是：在将来替换掉CMS收集器
            1) 并发与并行：G1可充分利用多CPU、多核条件下的硬件优势、使用多个CPU来缩短STW的停顿时间、
               部分收集器原本需要停顿Java线程执行的GC动作、G1可以通过并发的方式让Java程序继续执行
            2) 分代收集: G1可以不需要其它收集器的配合自己管理整个GC堆、可以采用不同方式去处理新创建的对象和
               已存活一段时间、经过了n次GC的old对象来获取更好的收集效果            
            3) 空间整合: G1从整体来看是基于`标记-整理`实现的, 从局部(两个Region间)来看是基于复制的
               所以、在G1运行期间不会产生大量的内存空间碎片、避免提前触发full gc
            4) 可预测的停顿: G1和CMS的共同关注点是降级停顿、但G1除了降低停顿之外、还能建立可预测的停顿时间模型
               让使用者明确指定在一个长度为M ms的时间片段内、消耗在GC上的时间不得超过N ms、
               这几乎是实时Java(RTSJ)的垃圾收集器的特征了


G1收集器：https://www.jianshu.com/p/5a95ce2bbb36

#### 参考
* https://crowhawk.github.io/2017/08/15/jvm_3/
* https://tech.meituan.com/2017/12/29/jvm-optimize.html
等众多网络文章
