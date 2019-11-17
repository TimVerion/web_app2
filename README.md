### Docker构建nginx+uwsgi+flask服务

### docker安装和配置

在部署新服务器运行docker镜像的时候遇到了报错，记录下解决方法。

docker 启动容器报错：Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused "process_linux.go:258: applying cgroup configuration for process caused \"Cannot set property TasksAccounting

docker 是通过 yum install docker安装的，搜了一把，原来是因为linux与docker版本的兼容性问题。那就卸载旧版本安装最新版试试。

环境准备：

Ubuntu 64-bit系统

Kernel 3.10+

**0**.通过**uname -r**命令查看你当前的内核版本，检查系统的内核版本，返回的值大于3.10即可

```
uname -r
```




**1**.使用 `root` 权限登录 Centos。确保 yum 包更新到最新。

```
sudo yum update
```

**2**.卸载旧版本(如果安装过旧版本的话)

```
sudo yum remove docker  docker-common docker-selinux docker-engine
```

**3**.安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

**4**.设置yum源

```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

**5**.可以查看所有仓库中所有docker版本，并选择特定版本安装

```
yum list docker-ce --showduplicates | sort -r
```

**6**.安装docker

```
sudo yum install docker-ce
```

**7**.启动并加入开机启动

```
sudo systemctl start docker
sudo systemctl enable docker
```

**8**.验证安装是否成功(有client和service两部分表示docker安装启动都成功了)

```
docker version
经过以上一通操作，pull 一下镜像再执行docker run命令，问题解决。
```

**9**.下载镜像

以CentOS为例，下载一个CentOS镜像

推荐[DaoCloud镜像市场](https://hub.daocloud.io/repos/)

进入DaoCloud镜像市场，搜索centos，进入搜索详情页，能看到拉取镜像的命令，在宿主机上执行该命令

```
docker pull daocloud.io/library/centos:latest
```

**10**.查看本机所有镜像：

```
docker images
```


**11**.启动容器
启动容器：

```
docker run -i -t 0f3e07c0138f /bin/bash
```

命令分析：

docker run <相关参数> <镜像 ID> <初始命令>
相关参数说明：

i：表示以“交互模式”运行容器
t：表示容器启动后会进入其命令行
v：表示需要将本地哪个目录挂载到容器中，格式：-v <宿主机目录>:<容器目录>


可以看到我们已经进入到docker下载的容器里面呢,类似于一个空的linux系统

重启docker服务

service docker restart

停止docker服务

docker stop 容器Id 

### 查找镜像

```
查找镜像 (镜像的全称是 <username>/<repository>)
docker search tutorial
或者可以尝试
docker search ubuntu
```


解决方法

进入/etc/docker

查看有没有 daemon.json。这是docker默认的配置文件。

如果没有新建，如果有，则修改。

[root@zengmg docker]# vi daemon.json
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}

保存退出

重启docker服务

```
service docker restart
```


成功！

下载自己所需的进行镜像,这里比如下载docker.io/rookout/tutorial-python

```
docker pull docker.io/rookout/tutorial-python
```



### 启动容器（后台模式）

使用以下命令创建一个以进程方式运行的容器

```
runoob@runoob:~$ docker run -d 镜像id /bin/sh -c "while true; do echo hello world; sleep 1; done"
2b1b7a428627c51ab8810d541d759f072b4fc75487eed05812646b8534a2fe63
```

在输出中，我们没有看到期望的 "hello world"，而是一串长字符

**2b1b7a428627c51ab8810d541d759f072b4fc75487eed05812646b8534a2fe63**

这个长字符串叫做容器 ID，对每个容器来说都是唯一的，我们可以通过容器 ID 来查看对应的容器发生了什么。

首先，我们需要确认容器有在运行，可以通过 **docker ps** 来查看：

```
runoob@runoob:~$ docker ps
CONTAINER ID        IMAGE                  COMMAND              ...  
5917eac21c36        ubuntu:15.10           "/bin/sh -c 'while t…"    ...
```

输出详情介绍：

**CONTAINER ID:** 容器 ID。

**IMAGE:** 使用的镜像。

**COMMAND:** 启动容器时运行的命令。

**CREATED:** 容器的创建时间。

**STATUS:** 容器状态。

状态有7种：

- created（已创建）
- restarting（重启中）
- running（运行中）
- removing（迁移中）
- paused（暂停）
- exited（停止）
- dead（死亡）

**PORTS:** 容器的端口信息和使用的连接类型（tcp\udp）。

**NAMES:** 自动分配的容器名称。

14.查看容器的输出日志

在容器内使用 **docker logs** 命令，查看容器内的标准输出：

```
runoob@runoob:~$ docker logs 2b1b7a428627
```

这里的2b1b7a428627是容器 ID

### 停止容器

我们使用 **docker stop** 命令来停止容器:



通过 **docker ps** 查看，容器已经停止工作:

```
runoob@runoob:~$ docker ps
```



可以看到容器已经不在了。

也可以查看过去运行的情况:



创建守护进程式容器：

[root@localhost ~]# docker run -i -t --name daemon_dave -d centos:7.0.1406 /bin/sh

删除容器：

[root@localhost ~]# docker rm 94ab5a5ac359
94ab5a5ac359

[root@localhost ~]# docker rm `docker ps -a -q` 删除所有容器

### 构建自己的镜像

我们先来写第一个Dockerfile文件，这个文件负责搭建运行环境，运行环境需要包括：nginx、uwsgi、Python3：

```
mkdir demo
cd demo/
vim Dockerfile
```

```
# 配置基础镜像
FROM alpine:3.8
 
# 添加标签说明
LABEL author="mayanhui" email="XXX@qq.com"  purpose="nginx+uwsgi+Python3基础镜像"
 
# 配置清华镜像地址
RUN echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.8/main/" > /etc/apk/repositories
 
# 更新升级软件
RUN apk add --update --upgrade
 
# 安装软件
RUN apk add --no-cache nginx python3 uwsgi uwsgi-python3
 
# 升级pip，这一步同时会在/usr/bin/目录下生成pip可执行文件
RUN pip3 install --no-cache-dir --upgrade pip
 
# 建立软链接
RUN ln -s /usr/bin/python3 /usr/bin/python

```

执行如下命令构建自己的镜像

```
docker build -t nginx_uwsgi_py3:alpine3.8 .
```

![2019-11-17_071928](F:\pics\docker\2019-11-17_071928.jpg)

可以看到已经构建成功

```
docker images
```

![2019-11-17_072245](F:\pics\docker\2019-11-17_072245.jpg)

将代码和镜像打包成容器

```
# 下载代码
git clone https://github.com/moshangguang/docker-nginx-uwsgi-flask-py3
mv docker-nginx-uwsgi-flask-py3 web_app
cd web_app
```



```
# 构建容器
docker build -t web_app .
```

4.运行自己的镜像

```
docker images
```



执行以下命令启动自己的镜像，并指定映射端口为9999

```
docker run -p 9999:6666 -d web_app
```

之后就可以成功访问了。

docker ps: 查看正在运行的容器

### 如何将镜像推送到Docker Hub

Docker hub注册用户

到官网注册账号：https://hub.docker.com/

```
docker login -u mayanhui -p xxxxx
```

为镜像打标签，使其符合标准形式

```
# 推送镜像的规范是:  docker push 注册用户名/镜像名

docker tag d1f4e5558166 mayanhui/nginx_uwsgi_py3:alpine3.8
```

将镜像推送给远程镜像库

```
docker push mayanhui/nginx_uwsgi_py3:alpine3.8
```


这样在另外一台服务器就可以直接拉取这个镜像了

```
docker pull mayanhui/nginx_uwsgi_py3:alpine3.8
```

参考链接：

https://blog.csdn.net/lookTheWorld/article/details/80244636

http://www.imooc.com/article/details/id/282731
