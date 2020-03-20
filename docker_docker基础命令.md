---
title: docker基础命令
date: 2020-03-20
categories:
  - Docker
tags:
  - Docker
---

# docker run
`-i` 交互式操作
`-t` 终端
`--rm` 容器退出后随即删除

# docker image 
` docker image ls` 查看已下载的镜像、只显示顶层镜像
`docker image ls -a` 查看镜像、包括中间层
`虚悬镜像` 没有仓库名也没有标签的镜像、新旧镜像同名、旧镜像名称被取消的镜像(dangling image)
`docker image prune` 删除虚悬镜像
`docker image ls <reposity>` 列出指定仓库的镜像
`docker image ls <reposity>:<tag>` 列出指定镜像
`docker image ls -f(--filter) since=<reposity>:<tag>` 列出某个tag之后的镜像
`docker image ls -f(--filter) before=<reposity>:<tag>` tag之前的镜像
`docker image ls -q` 只列出imageid
`docker image ls --format "{{.ID}}	{{.Repository}}"` 重新组织列
`docker image rm <opt>` 用镜像名/镜像摘要/镜像id等删除镜像
`docker image rm`不一定会产生删除镜像的行为、untagged 取消标签  deleted 删除镜像 (无任何标签指向时才会发生)
容器是以镜像为基础、再加一层存储层、所以、该镜像若被某个容器依赖、也不能被删除、即使该容器是停止的
`docker diff <container>` 查看容器的具体改动
`docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]` 提交改动
`docker history `

# docker file 指令
`from` 指定基础镜像
`run` 执行命令、
      1) 类似shell `run cmd`
      2) 类似exec `run file arg`
`copy` 复制文件
`add` 复制并自动解压文件(若不需要解压、直接使用 copy 即可) - add指令会让镜像构建缓存失效、使镜像的构建变的比较缓慢
`cmd` 容器启动参数 (容器就是一个进程、cmd可以指定进程启动时的参数)
`entrypoint` 入口点 1.作为命令执行 2.应用启动前的工作
`env` 设置环境变量
`volumn` 定义匿名卷、容器启动时 `-v` 参数可以覆盖改配置
`expose` 暴露端口(仅仅声明容器打算使用什么端口、不直接自动在宿主进行端口映射)
`workdir` 指定工作目录
`user` 指定用户 同`workdir`会改变环境状态并影响以后的层
`healthcheck` 健康检查 `healthcheck --intervalss=5s --timeout=3s cmd curl -fs http://127.0.0.1 || exit 1`
     容器运行后、可以通过 `docker container ls` 查看容器的监控状态
     `docker inspect --format '{{json .State.Health}}' nginx | python -m json.tool` 来查看健康状态
`onbuild` 后边跟其他指令、其后的指令在构建当前镜像时不会执行、
`docker save {image} -o filename` 保存镜像

#### docker 容器
`docker run ` 构建一个容器本地不存在则从远程共有仓库下载 `-d` 后台运行容器、输出不会直接显示在终端、会返回一个唯一id
`docker container start {container}` 将一个已终止的容器启动运行
`docker container stop {container}` 终止一个运行中的容器
`docker container restart {container}` 重启一个运行中的容器
`docker ps` `docker top {container}` 查看docker进程信息
`docker logs [opt] {container}` 查看一个docker的运行日志
`docker container ls` 查看容器列表
`docker attach {container}` 进入一个容器、exit退出时会删除容器
`docker exec -it {container} bash` exit时不删除容器
`docker container ls -a` 列出本地所有的容器
`docker export {containerId} > xxx.tar` 到处本地容器
`cat xxx.tar | docker import -test/{containerId}` 从容器快照文件中导入为镜像
 `docker container rm {container}` 删除一个处于终止状态的容器
 `docker container prune` 删除所有处于终止状态的容器
 
 
 #### 镜像查找
 `docker search {{key}}` 查找官方仓库中的镜像
 `docker pull {{image}}` 下载镜像到本地
 `docker push {{image}}` 推送镜像到 docker hub
      

# 构建镜像
从本地文件构建 `docker build -t nginx:v3 .`
从远程url直接构建 `docker build https://github.com/xxx#:11.1`
使用给定tar包构建 `docker build http://xxx.tar.gz`

#### 搭建私有仓库
可以以官方registry为基础镜像
> docker run -d -p 5000:5000 -v /Users/nj/build/registry:/var/lib/registry registry

1. 在私有仓库上上传、下载、搜搜镜像



# 常用参数
`-d` 后台运行
`-i` 交互式运行
`-t` 分配bash伪终端

docker run -d --name web -p 80:80 nginx:v3


      
