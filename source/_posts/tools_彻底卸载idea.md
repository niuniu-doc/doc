---
title: 彻底卸载idea
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---
完全卸载idea

首先在应用里面右键移动到垃圾桶[先卸载应用]

cd Users/xxx/Library/

上面的xxx对应你的用户名，然后输入

rm -rf Logs/IntelliJIdeaxxx/
 
rm -rf Preferences/IntelliJIdeaxxx/
 
rm -rf Application\ Support/IntelliJIdeaxxx/
 
rm -rf Caches/IntelliJIdeaxxx
上面的对应xxx对应不同的版本号，注意开头是 IntelliJIdea就行

把～/下的.idea/也删掉

Idea 的缓存文件在

/Users/xxxx/Library/Preferences
注意：以上方式会直接删除缓存文件，所以你重新安装的idea，没有任何插件【卸载前可导出配置setting.jar】

 
原文链接：https://blog.csdn.net/qq_17213067/article/details/88226062
