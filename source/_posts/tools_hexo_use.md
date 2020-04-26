---
title: 使用hexo 框架搭建网站
date: 2020-03-18 14:53:42
tags: 
   - Tools 
categories: 
   - Tools
---

#### 使用hexo 框架搭建网站

##### hexo基于node框架、先搭建基础环境
   ```
   brew install node // 安装node
   node -v // 检查npm是否安装成功
   npm -v // 查看npm版本
   ```
##### 安装hexo
   ```
   npm install hexo-cli-g // 安装hexo
   hexo init xxx // 初始化hexo站点目录
   cd xxx // 进入目录
   hexo clean
   hexo g // 生成站点, g -> generate
   hexo s // 运行网站, s -> server
   ```
   此时、本地网站已经搭建完成, 访问 localhost:4000 即可.
##### 配置文件 `_config.yml`
   ```
   title: 牛牛的Blog
   subtitle: 不奢望岁月静好 只希望点滴积累
   description: '多数是学习笔记、偶尔发发感慨、给生活添加些许乐趣'
   keywords: Java, Mysql, Linux
   author: niuniu
   language: zh
   timezone: ''
   theme: next // 定义主题
   ```
##### 更换主题
   ```
   下载: https://hexo.io/themes/ 
   theme: next // 定义主题 _config.yml
   hexo g // 重新生成站点
   hexo s // 重新运行站点
   ```
##### 分类和标签
   ```
   hexo new file-name (file-name是你的文章名, 也可以直接在 spurce/_post 文件夹下新建md文件)
   hexo new page tags
   ```
   打开 source/tags/index.md、修改为:
   ```
   type: "tags"
   comments: flase
   ```

   ```
   hexo new page categories // 打开分类功能
   ```
   打开 source/categories/index.md、修改为:
   ```
   type: categories
   comments: false
   ```

   使用:
   ```
   在文章头部添加
   tags:
     - tag1
	 - tag2
	 - tag3
   categories:
     - cat1
	 - cat2
   ```

##### 修改hexo侧边栏标签
  ```
   menu:
  	home: / || home
  	about: /about/ || user
  	tags: /tags/ || tags
  	categories: /categories/ || th
  	archives: /archives/ || archive
  	MySQL: /categories/MySQL || database
  	Redis: /categories/Redis || database
  	#schedule: /schedule/ || calendar
  	#sitemap: /sitemap.xml || sitemap
  	#commonweal: /404/ || heartbeat
  ```
   注意: `:`前边展示的是侧边栏显示名称 
         `/xxx/` 展示的是分类名称 
         `||` 后边是图标
         图标从 [fontawesome.com](https://fontawesome.com/icons?d=gallery&m=free) 获取

 15601278131
