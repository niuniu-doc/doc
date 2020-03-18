---
title: 如何同时登陆两个github账号
date: 2020-03-18 14:53:42
tags: 
    - tools 
categories: 
    - my-tools
---

#### 生成一个新的ssh key
```
nj:~ nj$ cd ~/.ssh
nj:.ssh nj$ ls
id_rsa        id_rsa.pub  known_hosts
nj:.ssh nj$ ssh-keygen -t rsa -C "iMac_personnal_publicKey"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/nj/.ssh/id_rsa):               
/Users/nj/.ssh/id_rsa_personal
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/nj/.ssh/id_rsa_personal.
Your public key has been saved in /Users/nj/.ssh/id_rsa_personal.pub.
The key fingerprint is:
SHA256:1gepuxDHwJRnFbKvc0Zq/NGrFGE9kEXS06jxatPPrSQ iMac_personnal_publicKey
The key's randomart image is:
+---[RSA 2048]----+
|      ....=*oo   |
|     o. ooo=+ .  |
|      oo. =+o.   |
|       o =.o..   |
|      . S =o.    |
|       = =++.    |
|      . B.=.Eo.. |
|       o B . +o .|
|          . o.. .. |
+----[SHA256]-----+
njdeiMac:.ssh nj$ ls
id_rsa            id_rsa_personal     known_hosts
id_rsa.pub        id_rsa_personal.pub`
```

2. 打开新生村的 `~/.ssh/id_rsa_personal.pub` 文件、将内容添加到 github 后台
3. 打开 `~/.ssh/config` 文件
```
Host xxx //你的host别名
  HostName github.com
  User yyy //github的注册用户名
  IdentityFile ~/.ssh/id_rsa_personal
```
4. 将github ssh仓库中的 git@github.com 替换成新建的Host别名
```
eg. git@github.com:hbxn740150254/BestoneGitHub.git
 -> git@xxx:hbxn740150254/BestoneGitHub.git
 ```
