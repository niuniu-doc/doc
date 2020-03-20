---
title: google去掉新标签页缩略图
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---

<!-- MarkdownTOC -->

- 隐藏新标签页最近访问
- mac下禁用 chrome 自动更新

<!-- /MarkdownTOC -->

#### 隐藏新标签页最近访问
```
1. 尝试过 new tab direct.. & 其它插件
感觉不够爽、网络慢时、tab页有延时、可以看到明显的new tab跳转过程...
啊哈、心里还是不爽的~~~~

2. 继续 google. 找到手动解决方式、~~
    1) 下载pak打包解压小工具  链接:https://pan.baidu.com/s/1TDp614rex0h0U5t2ShZE6g   密码:gtno
    2) 将pak文件小工具解压、备用
       eg. 将pak文件解压
           pak_tools -c=unpack -f=/Users/aha/Package/resources.pak 

           重新打包为pak文件 
           pak_tools -c=repack -f=/Users/aha/Package/resources.json 

    3) 找到google安装目录
      Mac：
      /Applications/Google Chrome.app/Contents/Versions/
      -------------------------------------------------------------------------
       (如果安装过多个版本的话、可以看到多个版本命名的文件夹、so... 接下来查看自己正在使用的版本
      Chrome -> 右上角... -> 帮助 -> 关于google chrome -> 可以看到正在使用的版本)
      -------------------------------------------------------------------------
      进入对应版本目录下：
      eg. 我的是：
      /Applications/Google Chrome.app/Contents/Versions/{{69.0.3497.92}}/Google Chrome Framework.framework/Versions/A/Resources
      然后可以看到有个文件叫：resource.pak 

    3) 找到它之后、复制一份到自己方便找到的地方、把这里的重命名为 resource.pak-bak
       eg. 我把它放到package下、
       然后、使用上边的命令解压、
       pak_tools -c=unpack -f=/Users/aha/Package/resources.pak 
       解压后会生成 rasource.json 和 resource 文件夹
       用sublime或者其它的文本编辑器在 resources文件夹中搜索 
       <div id="most-visited">可以看到两处
        enen、干脆点儿、都注释掉吧~~

      然后、重新生成pak文件
      pak_tools -c=repack -f=/Users/aha/Package/resources.json 

      将重新生成的 resources.pak 放回 ：
      /Applications/Google Chrome.app/Contents/Versions/{{69.0.3497.92}}/Google Chrome Framework.framework/Versions/A/Resources
      
    4) 退出google chrome、然后重新启动、可以看到 那八个缩略图从我们的世界中消失了、
       enen / 干净的感觉真爽~~

```

#### mac下禁用 chrome 自动更新
```
 1. 来个干脆的吧
    找到update程序、直接删除或者重命名
    我的是在 ~/Library/Chrome/下
    文件夹的名字叫： GoogleSoftwareUpdate
    eg. 我的完整路径是 ~/Library/Chrome/GoogleSoftwareUpdate 直接将它重命名、就不会自动更新了
```
