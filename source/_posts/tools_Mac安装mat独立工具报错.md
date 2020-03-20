---
title: Mac安装mat独立工具报错
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---
####问题描述
官网下载、安装
![image.png](https://upload-images.jianshu.io/upload_images/14027542-0d8d68043d36538d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
An error has occurred. 
See the log file 
/Users/%username%/.eclipse/1899417313_macosx_cocoa_x86_64/configuration/1507391541586.log.
```

![image.png](https://upload-images.jianshu.io/upload_images/14027542-56ee81ed27935fe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
!ENTRY org.eclipse.osgi 4 0 2019-07-26 21:12:08.965
!MESSAGE Application error
!STACK 1
java.lang.IllegalStateException: The platform metadata area could not be written: /private/var/folders/kc/cc7tcgrx6_1d355s45t1q3nh0000gn/T/AppTranslocation/9361422A-8DF8-482D-873B-8C030C46FB6E/d/mat.app/Contents/MacOS/workspace/.metadata.  By default the platform writes its content
under the current working directory when the platform is launched.  Use the -data parameter to
specify a different content area for the platform.
        at org.eclipse.core.internal.runtime.DataArea.assertLocationInitialized(DataArea.java:70)
        at org.eclipse.core.internal.runtime.DataArea.getStateLocation(DataArea.java:138)
        at org.eclipse.core.internal.preferences.InstancePreferences.getBaseLocation(InstancePreferences.java:44)
        at org.eclipse.core.internal.preferences.InstancePreferences.initializeChildren(InstancePreferences.java:209)
        at org.eclipse.core.internal.preferences.InstancePreferences.<init>(InstancePreferences.java:59)
        at org.eclipse.core.internal.preferences.InstancePreferences.internalCreate(InstancePreferences.java:220)
        at org.eclipse.core.internal.preferences.EclipsePreferences.create(EclipsePreferences.java:349)
@       
```

#### 解决方案
```
使用命令行打开MemoryAnalyzer.ini，添加-data参数 
/Users/nj/package/mat.app/Contents/Eclipse/MemoryAnalyzer.ini
如：
-data
/usr/local/var/log/mat/
```


#### 注意
```
-data 放在 launcher前、
-data 和 value要放在两行
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-360b39a7c35efd15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
