---
title: 使用 hexo 框架搭建网站
date: 2020-03-18 14:53:42
tags: 
   - Tools 
categories: 
   - Tools
---

#### 使用 hexo 框架搭建网站

##### 先搭建基础环境
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
   此时、本地网站已经搭建完成, 访问 `localhost:4000` 即可.
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
   打开 `source/categories/index.md`、修改为:
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

##### 将链接修改为蓝色
打开`themes/next/source/css/_common/components/post/post.styl` 文件, 将下面的代码复制到文件最后
```css
.post-body p a{
     color: #0593d3;
     border-bottom: none;
     &:hover {
       color: #0477ab;
       text-decoration: underline;
     }
   }
```

##### 修改代码块样式
打开 `themes/next/_config.yml`, 搜索 `custom_file_path`, 去掉 `style` 前边的注释, 然后新建或者打开 `{blog_root}/source/_date/styles.styl` 文件 
```css
code {
    color: #D1082D;
    background: #E8E5E6;
    margin: 2px;
   // font-size: 14px; // 设置你的字体大小
   // font-family: Source Corde Pro; // 设置你喜欢的字体
}
// 大代码块的自定义样式
.highlight, pre {
    margin: 5px 0;
    padding: 5px;
    border-radius: 3px;
}
.highlight, code, pre {
    border: 1px solid #d6d6d6;
}
```

#### 使用valine评论系统

新版本的 https://valine.js.org/ 支持npm安装

```
# Install leancloud's js-sdk
npm install leancloud-storage --save
# Install valine
npm install valine --save
```

新的版本next主题已经默认支持, 需要注册leanCloud账号 [https://leancloud.cn/](https://leancloud.cn/)， 获取`appId`和`appKey`, 然后修改配置文件 `**themes/hexo-theme-next/_config.yml**`, 搜索 `valine`, 修改配置项:

```
# Valine.
# You can get your appid and appkey from https://leancloud.cn
# more info please open https://valine.js.org
valine:
  enable: false
  appid:  # your leancloud application appid
  appkey:  # your leancloud application appkey
  notify: false # mail notifier , https://github.com/xCss/Valine/wiki
  verify: false # Verification code
  placeholder: Just go go # comment box placeholder
  avatar: mm # gravatar style
  guest_info: nick,mail,link # custom comment header
  pageSize: 10 # pagination size
```

将 `enable` 改为 `true`, 将刚刚申请的 `appId` 和 `appKey` 填入,

leancloud appId申请请参考: [https://valine.js.org/quickstart.html](https://valine.js.org/quickstart.html)

##### 评论头像显示

1. 注册 [gravatar](http://cn.gravatar.com/), valine 默认使用 gravatar的头像管理
2. 进行评论, 注意要在评论时填写自己注册后绑定的邮箱、才会显示头像

   如下图所示

![image-20200322112148688](/images/image-20200322112148688.png)





