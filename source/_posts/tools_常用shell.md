---
title: 常用shell
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---


AWK 多个分隔符

```
awk -F '[sep1|sep2]' '{print $0}'
```



删除文件中包含指定字符串的行

```
sed -e '/abc/d'  a.txt  > a.log   // 删除a.txt中含"abc"的行，将操作之后的结果保存到a.log

sed '/abc/d;/efg/d' a.txt > a.log    // 删除含字符串"abc"或“efg"的行，将结果保存到a.log

```



替换指定字符串

```
sed -i 's/reg/replace/g' 将reg替换为replcace
eg. sed -i "s/aaa/bbb/g" /tmp/1 将/tmp/1 文件中的a替换为b
```


截取n到m列

```
head -3 /tmp/3 | cut -d'sep' -f 2,7
-d 指定分隔符
-f 指定截取列

cut 常用参数
-d :分隔符 （ --delimiter 按照指定分隔符分割列 ）
-b : 表示字节
-c : 表示字符
-f : 表示字段(列号) （ --field 提取第几列 ）
N- : 从第N个字节、字符、字段到结尾
N-M : 从第N个字节、字符、字段到第M个
-M : 从第一个字节、字符、字段到第M个
eg. head -3 /tmp/3 | cut -c1-3 截取字符串第1到3列
```

diff 文件差异
```
1. comm 命令(需要先进行文件排序 sort file1 )
    comm -23 file1 file2 > /tmp/1 得到只在file1、不在file2中的数据
      -1   不显示只在第1个文件里出现过的列。
      -2   不显示只在第2个文件里出现过的列。
      -3   不显示只在第1和第2个文件里出现过的列。
2. diff 命令
3. sort 命令
    sort file2 file1 file1 | uniq -u  > /tmp/1
```

字符串截取
> `${string: start :length}` 左边开始计数
> `${string: start :length}` 右边开始计数


二进制文件查看
> xxd -b file
