---
title: 垃圾回收---GC日志解读
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
#### 年轻代log解读
年轻代以ParNew收集器为例, 采用的是复制算法, log如下：
![image.png](https://upload-images.jianshu.io/upload_images/14027542-4695c4709d665059.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
2019-02-01T21:18:15:00.382+0800: 718675.758: Application time: 0.7177964 seconds
{Heap before GC invocations=65632 (full 18):
 par new generation   total 471872K, used 421490K [0x0000000080000000, 0x00000000a0000000, 0x00000000a0000000)
  eden space 419456K, 100% used [0x0000000080000000, 0x00000000999a0000, 0x00000000999a0000)
  from space 52416K,   3% used [0x00000000999a0000, 0x0000000099b9c830, 0x000000009ccd0000)
  to   space 52416K,   0% used [0x000000009ccd0000, 0x000000009ccd0000, 0x00000000a0000000)
 concurrent mark-sweep generation total 1572864K, used 905696K [0x00000000a0000000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 104594K, capacity 113116K, committed 113664K, reserved 1148928K
  class space    used 12280K, capacity 13738K, committed 13824K, reserved 1048576K

2019-02-01T18:15:00.383+0800: 718675.758: [GC (Allocation Failure) 2019-02-01T18:15:00.384+0800: 718675.759: [ParNew: 109872K->5255K(118016K), 0.0211147 secs] 142365K->37748K(511232K), 0.0212949 secs] [Times: user=0.26 sys=0.04, real=0.02 secs]
```
其中:
*`Heap before GC invocations=65632 (full 18)`: ` invocations=65632`表示经历的gc的次数 `full 18`表示经历的full gc的次数
* `2019-02-01T18:15:00.383`：发生`minor gc`的时间
* `718675.758`：GC时、相对GVM启动时间的偏移量
* `ParNew` 收集器的名称 , 会`STW`(https://www.jianshu.com/p/d07194a20ec1)
* `109872K->5255K`：收集前后年轻代的使用情况
* `118016K`：整个年轻代的容量
* `0.0211147 secs`： stw的时间
* `142365K->37748K`：GC前后整个堆的使用情况
* `511232K`：整个堆的容量
* `0.0212949 secs`：ParNew 收集器标记和复制年轻代活着的对象所花费的时间(包括和老年代通信的开销、对象晋升到老年代开销、垃圾收集周期结束一些最后的清理对象等的花销)
* `user`：GC 线程在垃圾收集期间所使用的 CPU 总时间
* `sys`：系统调用或者等待系统事件花费的时间
* `real`: 应用被暂停的时钟时间、由于GC是多线程的、`real`可能会`小于` `user+sys`, 如果是单线程的、real是接近 `user+sys`的时间的

#### CMS log 大致解读
```
2018-04-12T13:48:26.233+0800: 15578.148: [GC [1 CMS-initial-mark: 6294851K(20971520K)] 6354687K(24746432K), 0.0466580 secs] [Times: user=0.04 sys=0.00, real=0.04 secs]
2018-04-12T13:48:26.280+0800: 15578.195: [CMS-concurrent-mark-start]
2018-04-12T13:48:26.418+0800: 15578.333: [CMS-concurrent-mark: 0.138/0.138 secs] [Times: user=1.01 sys=0.21, real=0.14 secs]
2018-04-12T13:48:26.418+0800: 15578.334: [CMS-concurrent-preclean-start]
2018-04-12T13:48:26.476+0800: 15578.391: [CMS-concurrent-preclean: 0.056/0.057 secs] [Times: user=0.20 sys=0.12, real=0.06 secs]
2018-04-12T13:48:26.476+0800: 15578.391: [CMS-concurrent-abortable-preclean-start]
2018-04-12T13:48:29.989+0800: 15581.905: [CMS-concurrent-abortable-preclean: 3.506/3.514 secs] [Times: user=11.93 sys=6.77, real=3.51 secs]
2018-04-12T13:48:29.991+0800: 15581.906: [GC[YG occupancy: 1805641 K (3774912 K)]2018-04-12T13:48:29.991+0800: 15581.906: [GC2018-04-12T13:48:29.991+0800: 15581.906: [ParNew: 1805641K->48395K(3774912K), 0.0826620 secs] 8100493K->6348225K(24746432K), 0.0829480 secs] [Times: user=0.81 sys=0.00, real=0.09 secs]2018-04-12T13:48:30.074+0800: 15581.989: [Rescan (parallel) , 0.0429390 secs]2018-04-12T13:48:30.117+0800: 15582.032: [weak refs processing, 0.0027800 secs]2018-04-12T13:48:30.119+0800: 15582.035: [class unloading, 0.0033120 secs]2018-04-12T13:48:30.123+0800: 15582.038: [scrub symbol table, 0.0016780 secs]2018-04-12T13:48:30.124+0800: 15582.040: [scrub string table, 0.0004780 secs] [1 CMS-remark: 6299829K(20971520K)] 6348225K(24746432K), 0.1365130 secs] [Times: user=1.24 sys=0.00, real=0.14 secs]
2018-04-12T13:48:30.128+0800: 15582.043: [CMS-concurrent-sweep-start]
2018-04-12T13:48:36.638+0800: 15588.553: [GC2018-04-12T13:48:36.638+0800: 15588.554: [ParNew: 3403915K->52142K(3774912K), 0.0874610 secs] 4836483K->1489601K(24746432K), 0.0877490 secs] [Times: user=0.84 sys=0.00, real=0.09 secs]
2018-04-12T13:48:38.412+0800: 15590.327: [CMS-concurrent-sweep: 8.193/8.284 secs] [Times: user=30.34 sys=16.44, real=8.28 secs]
2018-04-12T13:48:38.419+0800: 15590.334: [CMS-concurrent-reset-start]
2018-04-12T13:48:38.462+0800: 15590.377: [CMS-concurrent-reset: 0.044/0.044 secs] [Times: user=0.15 sys=0.10, real=0.04 secs]
```
由于CMS涉及的阶段比较多、分阶段来说下~
##### 初始标记 initial mark阶段
`STW`中的一次、为了标记直接被`GC root引用`或者`年轻代` `存活对象引用`的所有对象(https://plumbr.io/handbook/garbage-collection-algorithms-implementations#concurrent-mark-and-sweep)
![image.png](https://upload-images.jianshu.io/upload_images/14027542-550f1b5ef67f99d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上述示例的对应信息为：
```
2018-04-12T13:48:26.233+0800: 15578.148: [GC [1 CMS-initial-mark: 6294851K(20971520K)] 6354687K(24746432K), 0.0466580 secs] [Times: user=0.04 sys=0.00, real=0.04 secs]
```
其中：
* `2018-04-12T13:48:26.233+0800`：GC开始时间(同 ParNew收集器)
* `15578.148`：JVM运行时间(同 ParNew收集器)
* `CMS-initial-mark`：CMS当前阶段(这里代表 `初始标记`)
* `6294851K`：当前老年代使用的容量
* `(20971520K)`：老年代可使用的最大容量
* `6354687K`：整个堆目前使用的情况
* `(24746432K)`：整个堆当前可用情况
* `0.0466580 secs`：该阶段持续时间
* `[Times: user=0.04 sys=0.00, real=0.04 secs]` 同 `ParNew`收集器

##### 并发标记阶段(Concurrent Mark)
遍历老年代，然后标记所有存活的对象、会根据上个阶段找到的 GC Roots 遍历查找，与用户的应用程序`并发运行`
`注意`不是所有的老年代存活对象都会被标记、因为在标记期间引用关系可能会发生改变(https://plumbr.io/handbook/garbage-collection-algorithms-implementations#concurrent-mark-and-sweep)、
![image.png](https://upload-images.jianshu.io/upload_images/14027542-b3d7a48abe067d6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
与上图对比可用发现已经有一个对象的引用关系发生了改变

本阶段log如下：
>```
>2018-04-12T13:48:26.280+0800: 15578.195: [CMS-concurrent-mark-start]
>2018-04-12T13:48:26.418+0800: 15578.333: [CMS-concurrent-mark: 0.138/0.138 secs] [Times: user=1.01 sys=0.21, real=0.14 secs]
>```
>* `CMS-concurrent-mark`: `并发收集`阶段, 遍历老年代、标记所有存活对象
>* `0.138/0.138 secs`这个阶段的`持续时间`与`时钟时间`
>* `[Times: user=1.01 sys=0.21, real=0.14 secs]`：同上、但是从并发标记的开始时间计算的、期间是并发进行、所以参考意义不大(包含的不仅仅是gc线程的工作)

##### 并发预清理(concurrent preclean)
也是一个并发阶段，与应用的线程`并发运行`，`不会 stop `应用的线程
在并发运行的过程中、一些对象的引用可能会发生变化、发生这种情况时、jvm会将这个对象的区域`Card`标记为`Dirty`，也就是`Card Marking`

![image.png](https://upload-images.jianshu.io/upload_images/14027542-c040c645b1abbe18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在 `preclean`阶段、能够从Dirty对象可到达的对象也会被标记、标记完成之后、`Dirty Card`就会被清除了、
![image.png](https://upload-images.jianshu.io/upload_images/14027542-2fd204c365b46807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个阶段的log如下：
> ```
> 2018-04-12T13:48:26.418+0800: 15578.334: [CMS-concurrent-preclean-start]
> 2018-04-12T13:48:26.476+0800: 15578.391: [CMS-concurrent-preclean: 0.056/0.057 secs] [Times: user=0.20 sys=0.12, real=0.06 secs]
>```
> `CMS-concurrent-preclean` 阶段名称、对前边并发标记阶段中引用发生变化的对象进行标记
> `0.056/0.057 secs` 这个阶段持续的时间与时钟时间
> `[Times: user=0.20 sys=0.12, real=0.06 secs]` 同`并发标记`阶段

##### 可中止的并发预清理(Concurrent Abortable Preclean)
`并发` & `不影响`用户线程, 是为了 尽量承担STW中的最终标记阶段的工作， 这个阶段是在重复做很多相同的工作，直接满足一些条件（比如：重复迭代的次数、完成的工作量或者时钟时间等）
log如下：
>``` 
>2018-04-12T13:48:26.476+0800: 15578.391: [CMS-concurrent-abortable-preclean-start]
>2018-04-12T13:48:29.989+0800: 15581.905: [CMS-concurrent-abortable-preclean: 3.506/3.514 secs] [Times: user=11.93 sys=6.77, real=3.51 secs]
>```
>`CMS-concurrent-abortable-preclean`阶段名称
>`3.506/3.514 secs`通常在`5s`左右、
>`[Times: user=11.93 sys=6.77, real=3.51 secs]`同预清理阶段
主要做了两件事：
* 处理 From 和 To 区的对象，标记可达的老年代对象
* 和上一个阶段一样，扫描处理Dirty Card中的对象

具体执行多久，取决于许多因素，满足其中一个条件将会中止运行：
* 执行循环次数达到了阈值
* 执行时间达到了阈值
* 新生代Eden区的内存使用率达到了阈值

##### 最终标记阶段(Final Remark)
第二个 `STW `阶段, 也是最后一个、目标是标记所有老年代所有的存活对象，由于之前的阶段是并发执行的，gc 线程可能跟不上应用程序的变化，为了完成标记老年代所有存活对象的目标，STW 就非常有必要了， 通常 CMS 的 Final Remark 阶段会在年轻代尽可能干净的时候运行，目的是为了减少连续 STW 发生的可能性（年轻代存活对象过多的话，也会导致老年代涉及的存活对象会很多）。这个阶段会比前面的几个阶段更复杂一些，相关日志如下：
>```
>2018-04-12T13:48:29.991+0800: 15581.906: 
>[GC[YG occupancy: 1805641 K (3774912 K)]2018-04-12T13:48:29.991+0800: 15581.906: [GC2018-04-12T13:48:29.991+0800: 15581.906: [ParNew: 1805641K->48395K(3774912K), 0.0826620 secs] 8100493K->6348225K(24746432K), 0.0829480 secs] [Times: user=0.81 sys=0.00, real=0.09 secs]2018-04-12T13:48:30.074+0800: 15581.989: [Rescan (parallel) , 0.0429390 secs]2018-04-12T13:48:30.117+0800: 15582.032: [weak refs processing, 0.0027800 secs]2018-04-12T13:48:30.119+0800: 15582.035: [class unloading, 0.0033120 secs]2018-04-12T13:48:30.123+0800: 15582.038: [scrub symbol table, 0.0016780 secs]2018-04-12T13:48:30.124+0800: 15582.040: [scrub string table, 0.0004780 secs] [1 CMS-remark: 6299829K(20971520K)] 6348225K(24746432K), 0.1365130 secs] [Times: user=1.24 sys=0.00, real=0.14 secs]
>```
>`YG occupancy: 1805641 K (3774912 K)` 年轻代当前占用量及容量
>`ParNew:...`触发了一次`young gc`, 触发的原因是为了减少 年轻代的存活对象、尽量是年前代干净一些
>`[Rescan (parallel) , 0.0429390 secs]` 这个 Rescan 是当应用暂停的情况下完成对所有存活对象的标记，这个阶段是`并行`处理的，这里花费了 0.0429390s
>`[weak refs processing, 0.0027800 secs]` 第一个子阶段，它的工作是处理弱引用
>`[class unloading, 0.0033120 secs]` 删除无用class
>`[scrub symbol table, 0.0016780 secs] ... [scrub string table, 0.0004780 secs]` 最后一个子阶段, cleaning up symbol and string tables which hold class-level metadata and internalized string respectively
>`6299829K(20971520K)` 这个阶段之后，老年代的使用量与总量
>`6348225K(24746432K)`这个阶段后，堆的使用量与总量
>`0.1365130 secs`阶段持续时长
>`[Times: user=1.24 sys=0.00, real=0.14 secs]`对应时间信息

##### 并发清理(Concurrent Sweep)
不需要 STW，它是与用户的应用程序`并发运行`, 清除那些不再使用的对象，回收它们的占用空间为将来使用(https://plumbr.io/handbook/garbage-collection-algorithms-implementations#concurrent-mark-and-sweep)
![image.png](https://upload-images.jianshu.io/upload_images/14027542-d560490b0613a37e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
log如下(这中间又发生了一次 Young GC)：
```
2018-04-12T13:48:30.128+0800: 15582.043: [CMS-concurrent-sweep-start]
2018-04-12T13:48:36.638+0800: 15588.553: [GC2018-04-12T13:48:36.638+0800: 15588.554: [ParNew: 3403915K->52142K(3774912K), 0.0874610 secs] 4836483K->1489601K(24746432K), 0.0877490 secs] [Times: user=0.84 sys=0.00, real=0.09 secs]
2018-04-12T13:48:38.412+0800: 15590.327: [CMS-concurrent-sweep: 8.193/8.284 secs] [Times: user=30.34 sys=16.44, real=8.28 secs]
```
*`CMS-concurrent-sweep` 阶段名称: 主要是清除那些没有被标记的对象，回收它们的占用空间
* `8.193/8.284 secs` 这个阶段的持续时间与时钟时间
* `[Times: user=30.34 sys=16.44, real=8.28 secs]` 同上

##### 并发重置(Concurrent Reset)
这个阶段也是并发执行的，它会重设 CMS 内部的数据结构，为下次的 GC 做准备
```
2018-04-12T13:48:38.419+0800: 15590.334: [CMS-concurrent-reset-start]
2018-04-12T13:48:38.462+0800: 15590.377: [CMS-concurrent-reset: 0.044/0.044 secs] [Times: user=0.15 sys=0.10, real=0.04 secs]
```
*`CMS-concurrent-reset` 阶段名称
* `0.044/0.044 secs` 这个阶段的持续时间与时钟时间
* `[Times: user=0.15 sys=0.10, real=0.04 secs]` 同上

#### 小结：
CMS 通过将大量工作分散到并发处理阶段来在减少 STW 时间，在这块做得非常优秀，但是 CMS 也有一些其他的问题：

* CMS 收集器无法处理浮动垃圾（ Floating Garbage），可能出现 “Concurrnet Mode Failure” 失败而导致另一次 Full GC 的产生，可能引发串行 Full GC；
* 空间碎片，导致无法分配大对象，CMS 收集器提供了一个 -XX:+UseCMSCompactAtFullCollection 开关参数（默认就是开启的），用于在 CMS 收集器顶不住要进行 Full GC 时开启内存碎片的合并整理过程，内存整理的过程是无法并发的，空间碎片问题没有了，但停顿时间不得不变长；
* 对于堆比较大的应用上，GC 的时间难以预估

so. G1开始兴起、
>* 由于`G1`采用复制算法、很好的避免了 `空间碎片`
>* 采用region分区思想、解决了停顿时长可控
>* 不过貌似并没有从根源上解决 浮动垃圾的问题~~


#### 参考：
* http://matt33.com/2018/07/28/jvm-cms/
* https://plumbr.io/handbook/garbage-collection-algorithms-implementations#concurrent-mark-and-sweep
* https://zhanjia.iteye.com/blog/2435266
等
