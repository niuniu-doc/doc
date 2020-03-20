---
title: 简书文章迁移hexo
date: 2020-03-20 20:31:49
tags: Tools
categories: Tools
---



#### 导出markdown文件

首先登陆简书后台, 下载所有源文件

```
我 -> 设置 -> 账号管理 -> 下载所有文章
```



#### 将源文件替换成我们需要的格式

```shell
#!/bin/bash
  
# 简书文件存放路径
src=/Users/xx/Downloads/jianshu
dsc=/Users/xx/Downloads/jianshu-m
date=$(date "+%Y-%m-%d")

for f in $src/*/*; do
   base=$(basename $f .md)
   catgories=$(basename $(dirname $f) $PWD)
   insert="---\ntitle: ${base}\ndate: $date\ncategories:\n  - ${catgories}\ntags:\n  - ${catgories}\n---\n"
   echo "${insert}$(cat $f)" > $f  # 将hexo需要的头部内容插入
   lower=$(echo $catgories|tr '[A-Z]' '[a-z]') # 可根据个人需要、这里是为了生成指定分类开头的文件名
   srcFile=$dsc"/"$lower"_"$base".md"
   mv $f $srcFile # 将源文件放到一个文件夹下、统一copy到 _post 文件夹下即可
done
```



#### 临时解决简书图片链接无法加载

```
找到主题文件夹下 -> layout文件夹下 -> 有 <head> 标记的文件、
eg. hexo-theme-next主题为:
themes/hexo-theme-next/layout/_layout.swig

添加meta:
<meta name="referrer" content="no-referrer"/>

^.^仅为权宜之计、不想盗链...., 有好的替换方式后更新....

```



#### 本地新增文件图片链接

```
利用电脑上的各种工具、剪切的图片直接粘贴到 Typora上、会有 copy至 选项, 直接copy到我们的资源目录(你存放image的地方)即可
eg. 如果使用默认配置、应该是 source/images/
若开启 post_asset_folder: true 则每篇文章的图片会单独存放在一个与文件同名的文件夹内
![1.jpg](/images/image-20200320205834941.png)
```

