---
title: git相关
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---
1. 分支merge错了、需要撤回merge
    git merge --abort

2. 查看某段时间段内某个用户的提交
   git log -p --author={name} --since={start-time} --until={end-time}
   eg. git log -p --author=nj --since=2019-09-24 --until=2019-09-24

3. 查看某次提交属于哪个分支
     git br -r --contains commitId
     eg. git br -r --contains ca4efc1a

4. tag删除
    git tag -d tag_name  删除本地tag
    git push origin :refs/tags/tag_name 删除远程tag
