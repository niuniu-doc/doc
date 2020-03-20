---
title: Solr搭建与配置
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
#### 下载
```
http://archive.apache.org/dist/lucene/solr/
```

#### 安装配置
```
1. 将下载完成的solr解压到tomcat的指定目录中
     eg. 我下载的是 tomcat-7.2.1.zip、
         解压目录是 ~/Downloads/solr-7.2.1
         tomcat部署目录是 ~/www/Java

2. 复制solr-7.2.1/server/solr-webapp/webapp到tomcat部署目录、同时重命名
   (重命名为非必要步骤、为了方便理解)、
   要copy的目录是 solr-7.2.1/server/solr-webapp/webapp/、不是  solr-7.2.1/server/solr-webapp/、注意别cp错了~
     cp -R solr-7.2.1/server/solr-webapp/webapp/ ~/www/Java/solr

3. 相关jar包复制
    1）solr-7.2.1/server/lib/ext下所有jar包
    2) solr-7.2.1/server/lib/metrics*相关的jar包
    3) solr-7.2.1/dist/solr-dataimporthandler*.jar
 
 [当前处于~/Downloads下]
    cp -R solr-7.2.1/server/lib/ext/  ~/www/Java/solr/WEB-INF/lib/
    cp -R solr-7.2.1/dist/solr-dataimporthandler-*  ~/www/Java/solr/WEB-INF/lib/
    cp -R solr-7.2.1/server/lib/metrics-*  ~/www/Java/solr/WEB-INF/lib/

4. 在 ~/www/Java/solr/WEB-INF/ 下创建文件夹 classes 用来存放日志配置文件
    mkdir  ~/www/Java/solr/WEB-INF/classes 
  
5. 将日志配置文件放入
     cp ~/Downloads/solr-7.2.1/server/resources/log4j.properties ~/www/Java/solr/WEB-INF/classes/

6. 创建solr_home目录、作为solr的运行目录
   (eg. solr创建的score会放在这里)
    mkdir ~/www/Java/solr_home
    
7. 修改配置、将solr_home指定为刚刚创建的位置
    vi ~/www/Java/solr/WEB-INF/web.xml 

   两处修改:
  1) env-entry 的注释打开、并且将
       <env-entry-value>。。。</env-entry-value>修改为刚才创建的目录
       <env-entry-value>/Users/nj/www/Java/solr_home</env-entry-value>
  2) <security-constraint>注释掉(配置访问权限、不然有可能会403)、 

```
![env.png](https://upload-images.jianshu.io/upload_images/14027542-6ff0528121e02f7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![security.png](https://upload-images.jianshu.io/upload_images/14027542-5eb1ffc4a8d226f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 启动solr
```
重启tomcat
cd ~/build/java/apache-tomcat/bin
sh shutdown.sh
sh startup.sh
```

#### 访问
```
访问 localhost:8081/solr/index.html 可以看到solr的管理界面
注:
1. 端口号是自己的tomcat的访问端口、可能是8080或者其它任意端口
2. 访问时一定加上 index.html
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-44dc0994227bfdc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

==solr服务搭建end
```
这样就搭建完成、可以开始使用了
如果遇到404的情况、可以检查下 使用的jar包是不是都copy完成了
```

#### 初始化solr数据
```
1. 创建文件夹 ~/www/Java/solr_home/new_core 用来存放core内容
    将conf文件cp进去
    cp -r ~/www/Java/solr_home/configsets/_default/conf ~/www/Java/solr_home/new_core/
2. 在页面上可以看到 如下图所示、点击添加一个 core
3. image2中填写的部分都可以修改 name和instanceDir要保持一致(同为刚刚建立的文件夹的名字)
4. 创建成功之后、就可以看到 image中、划线的地方变成了、core Selector、点击、可以看到我们刚刚创建的new_core、如image3
自定义core创建完成、
```
![image.png](https://upload-images.jianshu.io/upload_images/14027542-db08b19d9c30d8c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image2.png](https://upload-images.jianshu.io/upload_images/14027542-dd02545058169919.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image3.png](https://upload-images.jianshu.io/upload_images/14027542-87109e2aadbef028.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 导入数据
```
1. 引入jar包
   cp ~/www/Java/lib/mysql/mysql-connector-java-5.1.38.jar ~/www/Java/solr/WEB-INF/lib/
2. 复制solr-7.2.1/example/example-DIH/solr/db/conf/下的db-data-config.xml到solr-home/core2/conf/下，此处改名为data-config.xml(也可以不修改)
3. 修改配置内容
   <dataConfig>
    <dataSource type="JdbcDataSource" driver="com.mysql.jdbc.Driver" url="jdbc:mysql://127.0.0.1:3306/taotao" user="root" password="root"/>
    <document>
        <entity name="tb_item" 
            query="select id,title,sell_point,price,num,barcore,image,cid,status,created,updated from tb_item">
        </entity>
    </document>
</dataConfig>

4. 修改solrconfig、添加导入信息
  vi  ~/www/Java/solr_home/new_core/conf/solrconfig.xml

     <!-- add by myself -->
    <requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
        <lst name="defaults">
          <str name="config">data-config.xml</str>
        </lst>
    </requestHandler>
注意要和requestHandler标签同级
4. 自定义solr字段、在manager-schema中添加filed字段(放在text后即可)
    如：field.png所示
   注意：type必须是fileType已定义的类型(defined.png)
5. 数据导入
6. 数据查询(query.png)
```
![data-config.png](https://upload-images.jianshu.io/upload_images/14027542-b36efbd2eea2cb82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![solrconfig.png](https://upload-images.jianshu.io/upload_images/14027542-ac54aea06f0bd371.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![field.png](https://upload-images.jianshu.io/upload_images/14027542-13915d20a978490d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![defined.png](https://upload-images.jianshu.io/upload_images/14027542-832f12d71e6bf66d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![import.png](https://upload-images.jianshu.io/upload_images/14027542-324870b92233d0fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![query.png](https://upload-images.jianshu.io/upload_images/14027542-2290be0f85dba4b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### end
```
这样就可以开始使用了^.^
```
