---
title: idea无法打开
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---
无意中发现idea的版本更新到2019.2了、我上次更新的时候是2019.1
闲来无事想玩儿下新版本、就升级了下、

#### 官网下载
[https://www.jetbrains.com/idea/download/#section=mac](https://www.jetbrains.com/idea/download/#section=mac)


#### 安装
Mac安装很简单、dmg双击 -> 导入原配置文件 -> 破解([https://www.jianshu.com/p/1f2596084784](https://www.jianshu.com/p/1f2596084784))

很麻溜的操作完、点击运行、傻了 ... 居然跑不起来


#### 搜索、可以借鉴
[https://www.jianshu.com/p/1f2596084784](https://www.jianshu.com/p/1f2596084784)
[https://blog.csdn.net/weixin_40866404/article/details/80192780](https://blog.csdn.net/weixin_40866404/article/details/80192780)
... 类似文章很多、发现都不好使😔

#### 胡思乱想
突然想到之前有篇idea破解的文章、可以让 idea 从命令行启动、查看错误提示

惊喜~

![idea启动.png](https://upload-images.jianshu.io/upload_images/14027542-6054554eeebf794a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`1处`是 idea 启动的jvm参数
`2、3`是启动过程中发生的错误
> 可以看到、一共是两个错误: 1.gc文件找不到、2.jvm启动参数给的有问题

正常情况下是不会发生这种问题的、纯属自作孽🤷‍♀️、
1.之前调过idea的jvm启动参数、都给成1300了、不知道为什么xms的值变成了-Xmx1024m、可能是多版本混合装了、某一个版本修改了默认值
2.为了看idea运行过程中gc的情况、将gc写入文件、删垃圾文件的时候又手残删掉了😭

借以给小伙伴儿们一点儿提示、万一遇到idea无法启动的问题、可以试试从命令行启动、看下错误日志

#### 任何从命令行启动idea
应用程序 -> idea -> 右键 -> 显示包内容 -> Contents -> MacOS -> 双击  idea 
