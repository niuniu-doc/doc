---
title: solr使用IK分词器
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
#### 下载分词器jar包
```
百度网盘地址：
链接:https://pan.baidu.com/s/1GHwv6uBcUhI7GpOpqFnl4g  密码:121w
```

#### 使用
```
1. 将压缩包解压、重命名为ik_analyzer
2. 将 ik-analyzer-solr6.jar 复制到 solr lib目录下
    cp ~/Downloads/ik_analyzer/ik-analyzer-solr6.jar ~/www/Java/solr/WEB-INF/lib
3. 将相关包放入solr classpath
    cp  ~/Downloads/ik_analyzer/mydict.dic    ~/Downloads/ik_analyzer/IKAnalyzer.cfg.xml    ~/Downloads/ik_analyzer/ext_stopword.dic  ~/www/Java/solr/WEB-INF/classes
  若~/www/Java/solr/WEB-INF/classes文件夹不存在、需要新建
4. 配置schema
   
    <!-- IKAnalyzer-->
      <fieldType name="text_ik" class="solr.TextField">
          <analyzer class="org.wltea.analyzer.lucene.IKAnalyzer"/>
      </fieldType>

      <field name="item_title" type="text_ik" indexed="true" stored="true"/>
      <field name="item_sell_point" type="text_ik" indexed="true" stored="true"/>
      <field name="item_price"  type="plong" indexed="true" stored="true"/>
      <field name="item_image" type="string" indexed="false" stored="true" />
      <field name="item_category_name" type="string" indexed="true" stored="true" />

      <field name="item_keywords" type="text_ik" indexed="true" stored="false" multiValued="true"/>
      <copyField source="item_title" dest="item_keywords"/>
      <copyField source="item_sell_point" dest="item_keywords"/>
      <copyField source="item_category_name" dest="item_keywords"/>

其中：copyfiled是将item_title、item_sell_point、item_category_name都作为后边的item_keywords来搜索

5. 保存、重启tomcat、访问页面
    http://localhost:8081/solr/index.html
```
![analyzer.png](https://upload-images.jianshu.io/upload_images/14027542-25bef17f4a7bd4d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 问题
```
如果遇到错误：java.lang.AbstractMethodError
可以检查下分词器的版本是否过低
当前这篇文章使用的solr是7.2.1的版本、使用的是网盘保存的分词器版本、是ok的、
之前使用了 IKAnalyzer2012FF_u1.jar 这个版本、出现的上述错误~
```
![error.png](https://upload-images.jianshu.io/upload_images/14027542-733f055952b7cd35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
