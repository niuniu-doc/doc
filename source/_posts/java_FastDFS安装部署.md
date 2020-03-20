---
title: FastDFS安装部署
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
在学习`taotao商城`这个视频的过程中、用到了`fastdfs`、刚好自己有一台服务器就自己学习了下、参考了网上多篇文章、感谢各位分享~~

`FastDFS`是`c`写的开源分布式文件系统、充分考虑了冗余被人、负载均衡、及线性扩容机制、注重高可用、高性能指标、可以很方便的搭建一套高性能的文件服务器集群提供文件上传、下载等服务~

###一、FastDFS架构
`FastDFS`包括`Tracker server`和`Storage server`, 客户端请求`Tracker server`进行文件上传、下载，通过`Tracker server`调度最终由`Storage server`完成文件上传和下载。 `Tracker server` 的角色类似于`dubbo`的`registry`和`moniter`、并不直接提供服务、而是`storage server`启动时注册到`tracker server`, `client`通过`tracker server`连接`storage server`, `client`不知道自己连接的是哪一台`storage server`, 连接完成后、上传和下载是`client`直接请求`storage server`, 可类比于 `dubbo consumer`通过`registry`连接`dubbo service` 但、连接完成之后是`consumer`和`service`直接通信

![fastdfs.jpg](https://upload-images.jianshu.io/upload_images/14027542-81571b0c67e068ef.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####文件上传流程
![文件上传流程.png](https://upload-images.jianshu.io/upload_images/14027542-2c255cded5753807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 文件下载流程
![文件下载流程.png](https://upload-images.jianshu.io/upload_images/14027542-9b9dcd61bd1a9a57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####1.Tracker Server
`Tracker server`作用是负载均衡和调度，通过`Tracker server`在文件上传时可以根据一些策略找到`Storage server`提供文件上传服务，可以将tracker称为追踪服务器或调度服务器

####1.Storage Server
`Storage server`作用是文件存储，客户端上传的文件最终存储在`Storage`服务器上，`Storage server`没有实现自己的文件系统而是利用操作系统的文件系统来管理文件。可以将`storage`称为存储服务器。

### 二、FastDFS安装
####1.安装libfastcommon

```
1. 下载：
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.7.tar.gz
2. 修改名字：mv V1.0.7 libfastcommon-1.0.7.tar.gz
3. 解压：tar zxvf libfastcommon-1.0.7.tar.gz
4. cd libfastcommon-1.0.7/
5. 编译：./make.sh
6. 安装：./make.sh install

另外：
设置几个软链接、方便后续扩展nginx时使用：
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
```

#### 2. 安装 tracker
```
1. 下载：
wget https://github.com/happyfish100/fastdfs/archive/V5.05.tar.gz
2. 修改名字：mv V5.05 FastDFS_v5.05.tar.gz
3. 解压：tar zxvf FastDFS_v5.05.tar.gz
4. 进入解压后目录：cd fastdfs-5.05/
5. 编译：./make.sh
6. 安装：./make.sh install
```
#### 3. 修改tracker配置文件
```
2安装完成后、在/etd/fdfs下有tracker的配置文件
复制一份：cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
mkdir -p /usr/local/fastdfs/  (此处可以根据自己的情况和习惯存放)
base_path= /usr/local/fastdfs/

启动 tracker 服务：/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
重启 tracker 服务：/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
查看是否有 tracker 进程：ps aux | grep tracker

```
####4. storage(存储节点)服务部署
```
一般 storage 服务我们会单独部署到一台服务器上，但是这里为了方便(~~我只有一台服务器~~)就安装在同一台上了

如果单独部署到一台服务器上、上边tracker的部署步骤重新来一遍即可
这里是同一台server、只修改配置~~

复制一份配置：cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
编辑：vim /etc/fdfs/storage.conf
base_path= /usr/local/fastdfs/

创建目录：mkdir  /usr/local/fastdfs/storage/images-data
store_path0= /usr/local/fastdfs/storage/images-data

图片实际存放路径，如果有多个，这里可以有多行(要创建多个目录)：
store_path0=/opt/fastdfs/storage/images-data0
store_path1=/opt/fastdfs/storage/images-data1

subdir_count_per_path=256 
是用来配置目录个数的、如果只是练习不做实际存储服务、可改小一点儿

指定 tracker 服务器的 IP 和端口
tracker_server=192.168.1.114:22122 (192.168.1.114)是你的server服务器ip、本机也可以使用(0.0.0.0:22122)、记得不可使用127.0.0.1

```

#### 5. 测试服务
```
启动 storage 服务：/usr/bin/fdfs_storaged /etc/fdfs/storage.conf，首次启动会很慢，因为它在创建预设存储文件的目录
重启 storage 服务：/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
查看是否有 storage 进程：ps aux | grep storage
```

####6. 查看`tracker`是否可以正常与`storage`通信
```
fdfs_monitor /etc/fdfs/storage.conf
...
Storage 1:
        id = 192.168.2.231
        ip_addr = 192.168.2.231  ACTIVE --若看到ACTIVE这个字样、代表可以正常通信
...
查看storage和tracker是否正常启动：
ps aux | grep fdfs
```
![进程.png](https://upload-images.jianshu.io/upload_images/14027542-c47e3762cf5b81c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 7. 使用fdfs_client测试
```
复制一份配置：cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf
编辑：vim /etc/fdfs/client.conf

base_path= /usr/local/fastdfs/

指定 tracker 服务器的 IP 和端口
tracker_server=192.168.1.114:22122
log_level=info

echo asasasa > ~/test.txt

测试：fdfs_test /etc/fdfs/client.conf upload ~/test.txt 
可以看到如下图所示、就是上传成功了
```
![test.png](https://upload-images.jianshu.io/upload_images/14027542-3ebfd5dda0a5732d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####8. 安装Nginx和其插件
```
如果Nginx已经安装过，则仅需要fastdfs-nginx-module_v1.16.tar.gz
下载nginx：wget http://nginx.org/download/nginx-1.11.8.tar.gz
下载Nginx插件:wget http://jaist.dl.sourceforge.NET/project/fastdfs/FastDFS%20Nginx%20Module%20Source%20Code/fastdfs-nginx-module_v1.16.tar.gz
解压 Nginx 模块：tar zxvf fastdfs-nginx-module_v1.16.tar.gz
进入解压后的目录 
cd fastdfs-nginx-module
vim src/config
修改：去掉local、因为实际安装fastdfs时、是放到了/usr/include下
1. CORE_INCS="$CORE_INCS /usr/local/include/fastdfs /usr/local/include/fastcommon/"
-> CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"

2.  CORE_LIBS="$CORE_LIBS -L/usr/local/lib -lfastcommon -lfdfsclient"
-> CORE_LIBS="$CORE_LIBS -L/usr/lib -lfastcommon -lfdfsclient"

回到nginx的解压目录
cd ../nginx-1.11.8
sudo ./configure  --prefix=/usr/local/nginx --sbin-path=/usr/local/bin/nginx --conf-path=/usr/local/etc/nginx/nginx.conf --pid-path=/usr/local/var/run/nginx.pid --lock-path=/usr/local/var/run/nginx.lock --error-log-path=/usr/local/var/log/nginx/^Cror.log --http-log-path=/usr/local/var/log/nginx/access.log --with-http_gzip_static_module --with-http_stub_status_module --with-http_ssl_module --with-file-aio --add-module=/home/nj/build/fastdfs-nginx-module/src

sudo make  && sudo  make install （若是有权限的账户、可以不用加sudo、我使用的是普通用户）

```

####9. 整个fastdfs-nginx-module和nginx
```
copy fastdfs-nginx-module的配置文件到 /etc/fdfs下、方便查找
cp /home/nj/build/fastdfs-nginx-module/src/mod_fdfs.conf /etc/fdfs
vi /etc/fdfs/mod_fdfs.conf

base_path=/usr/local/fastdfs
tracker_server=192.168.1.114:22122
url_have_group_name = true
store_path0=/usr/local/fastdfs/storage
```

####10. 然后配置Nginx，添加如下内容
```
server {
        listen       80;
        server_name  localhost;

        ...
        
         # 配置fastdfs的访问路径
        location /group1/M00 {
            ngx_fastdfs_module;
        }
        ...
    }
启动nginx
```

#### 用浏览器访问刚才步骤7中测试上传的文件：
```
http://xxxx/group1/M00/00/00/fwAAAVu0UTSAZiNHAAAACE6c2W4921_big.txt
```
![图片存储测试.png](https://upload-images.jianshu.io/upload_images/14027542-2fc2f16221227b59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### `^.^`哦啦~ 到此安装完成、可以使用了~~~

####另外：
```
fastdfs提供了php_client、可以使用php调用fastdfs的服务
参考：
https://github.com/happyfish100/fastdfs/tree/master/php_client/FastDFS%20Nginx%20Module%20Source%20Code/fastdfs-nginx-module_v1.16.tar.gz

```

#### 参考资源
> 1. https://github.com/happyfish100/fastdfs/tree/master/php_client
>2. https://www.waitig.com/fastdfs%E5%AE%89%E8%A3%85%E6%AD%A5%E9%AA%A4.html
>3. https://www.codetd.com/article/3138763
>4. http://soartju.iteye.com/blog/803477
>等多个网络资源...




 
