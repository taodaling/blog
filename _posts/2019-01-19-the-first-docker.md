---
categories: docker
layout: post
---

- Table
{:toc}

# 架构
Docker的核心组件有

- Docker客户端和服务器
- Docker镜像
- Registry
- Docker容器

## 客户端和服务器

Docker是一个C/S架构的应用。Docker客户端只需向Docker服务器（守护进程）发起请求，而服务器将完成所有工作并返回结果作为响应。Docker服务器也称为Docker引擎。

Docker提供了命令行工具docker以及一套RESTful API来与服务器交互。用户的客户端和服务器可以运行在相同的宿主机上，也可以运行在不同的宿主机上。

用户可以使用docker daemon命令控制Docker守护进程。

Docker以root权限运行服务器进程和客户端程序，来完成普通用户无权完成的操作（比如挂载文件系统）。

Docker守护进程会监听/var/run/docker.sock这个Unix套接字文件，来获取客户端的请求。如果系统中存在名为docker的用户组的话，Docker会将套接字文件的所有者设置为该用户组，这样docker用户组的所有用户都可以直接运行Docker，而无需使用sudo命令了。

## 镜像

镜像是Docker世界的基石，用户基于镜像来运行自己的容器。可以把镜像当做容器的源代码，有着体积小的优点。

Docker镜像是由文件系统叠加而成，最底层是一个引导文件系统——bootfs，但是当一个容器启动后，容器会被移到内存中，而引导文件系统则会被卸载，来节省内存。

在传统的Linux引导过程中，root文件系统最先会以只读的方式加载，当引导结束并完成了完整性检查后，它才会切换为读写模式。但是在Docker中，root文件系统永远是只读状态，并且Docker利用联合加载技术会在root文件系统层上加载更多的只读文件系统。联合加载指的是同时加载多个文件系统，但是在应用的视角只能看到一个文件系统。

Docker将这样的文件系统称为镜像，一个镜像置于另外一个镜像的顶部，位于下面的镜像称为父镜像（parent image），而最底部的镜像称为基础镜像（base image）。当从一个镜像启动容器时，Docker会在该镜像的最顶层加载一个读写文件系统，而容器中的程序就是在 这个读写层中执行的。

下面是一个分层的文件系统的示例，从上之下，对应从镜像底部到顶部。

- 内核
- 容器组、命名空间、设备映射
- 引导文件系统
- Ubuntu
- emacs
- Apache
- 可写容器

在Docker第一次启动一个容器时，初始的写入层是空的，当文件系统发生改变，这些改变都会应用在这一层。比如修改一个文件，文件会从只读层复制到读写层，之后的所有读取操作和写入操作都作用于读写层中的文件副本上（非常类似于缓存系统）。

这种机制一般被称为写时复制（copy on write）。在Docker中每个只读镜像层都是不变的。在创建一个新容器时，Docker会构建出一个镜像栈，并在栈顶添加一个读写层。这个读写层加上下面额镜像层以及一些配置数据，构成了一个完整的容器。

## Registry

Docker利用Registry来保存用户构建的镜像。Registry分为公共和私有两种，Docker公司运营的Docker Hub是公共Registry。

Docker提供了标签，每个镜像都带有一个标签，比如12.04、12.10、latest等。标签可以用于标识仓库中一个镜像的不同版本，要指定仓库中某一个特定版本的镜像，我们需要使用`镜像名:标签`的方式引用镜像（如果不指定标签，那么默认标签为latest）。

Docker Hub中有两种类型的仓库，用户仓库和顶层仓库，用户仓库中的镜像是由Docker用户创建的，而顶层仓库则是Docker内部人员管理的。用户仓库的命名由用户名和仓库名组成，比如`jamtur01/puppet`表示用户名为jamtur01而仓库名为puppet。而顶层仓库仅包含仓库名，比如`ubuntu`。使用用户仓库中的镜像需要自行承担风险。

## 容器

容器是基于镜像启动起来的应用，其中可以运行一个或多个进程。

Docker借鉴了集装箱的概念，集装箱将货物运往各地，集装箱不关心内部的货物。在Docker中， 容器就是集装箱，Docker就是运输集装箱的船只，而镜像就是集装箱中的货物，而你的笔记本、服务器等就是集装箱的目的地。

## swarm

Docker引擎中内嵌的集群管理和服务编排的能力是构建于swarmkit之上的。Swarmkit是一个分离的项目，其中实现了Docker的编排层，并被Docker直接使用。

一个Swarm包含多台运行swarm模式的Docker主机，并且每台主机扮演管理者或工人的角色。一台Docker主机可以同时成为管理者和工人。当你启动一个服务，你需要定义它的理想状态（副本数，可用的网络和存储资源，暴露的端口，以及更多）。Docker为了维持所需的状态而工作。比如，如果一个工人结点不可用，docker会将原来它承担的任务调度到其他的结点上。一个任务是指组成swarm服务的一个存活容器，被swarm管理员所管理。

Swarm服务相较于单机服务的一个关键的优势就是你剋呀修改服务的配置，包括它连接的网络和卷，而不需要手动重启服务。Docker会更新配置，停止使用过期配置的任务，并且使用新的配置创建全新的任务。

当Docker处于swarm模式，你依旧可以在任意Swarm结点上启动单机容器，就跟swarm服务一样。单机容器和swarm服务的关键不同是只有swarm管理者可以管理swarm服务，而单机容器可以被任意docker守护进程管理。

一个结点是指加入到swarm中的Docker引擎。你可以在一台物理机上运行多个结点，但是生产上你的结点会分布在不同的物理或云主机上。

要将你的应用部署到swarm上，你只需提交服务的定义到管理结点，管理结点会分发工作单元——任务到不同的工作结点。

管理结点还负责服务编排和集群管理的作用，以维持服务的所需状态。管理结点需要选举出一个Leader来组织编排的任务。

工作结点接受和执行由管理结点分发的任务。默认管理结点 也会作为工作结点运行服务，你也可以配置让它们只负责管理任务，即仅扮演管理者角色。每一个工作结点上都会运行一个代理，并且报告分配给它的任务。工作结点把分配给它的任务的执行状态告知给管理结点，因此管理者可以维持每个工作结点的状态。

一个服务表示的是执行在管理或工作结点上任务的定义。它是swarm系统的中心结构，也是用户和swarm交互的主要方式。

当你创建一个服务时，你需要指定使用哪个镜像，并且容器启动后执行的命令。

在副本服务模式，swarm管理结点分配一定数量的副本任务给不同的结点，任务数决定于服务的所需的副本数。

对于全局服务，swarm会在集群中每一个可用结点上执行服务的一个任务。

一个任务会启动一个Docker容器以及执行一句命令。它是swarm的原子性调度单元。管理结点将按照副本数将任务分配给工作结点。一旦任务分配给一个结点，它不会迁移到其他的结点，它只会在指定的结点上运行直到失败。

Swarm管理者使用负载均衡来暴露内部服务。Swarm管理结点会自动为服务分配一个发布端口，或者你也可以自行为服务配置一个发布端口。你可以指定任意不被使用的端口，如果你不指定端口，swarm会为服务分配一个30000-32767范围内的端口。

外部模块，比如云服务负载均衡器，可以访问集群中任意结点的发布端口后的服务，无论该结点当前是否正在执行该服务的任务。在swarm中的所有结点会将连接路由到一个运行中任务实例去。

swarm模式包含一个内部DNS模块，会为swarm中的服务自动分配一个DNS入口。而swarm管理者使用内部负载均衡分发请求到集群中的服务，通过服务的DNS名字。



# 快速上手

## 安装

安装之前，需要先了解自己所在系统是否兼容。Docker有以下要求：

- 64位CPU
- Linux3.8或更高版本内核
- 内核支持一种合适的存储驱动
- 内核必须开启cgroup和namespace

## 运行docker守护进程

```sh
$ service docker start
```

启动docker守护进程

```sh
$ service docker status
```

查看docker守护进程状态

## 查看docker信息

```sh
$ docker info
```

会显示docker的各项信息

## 第一个容器，运行ubuntu镜像

```sh
$ docker run -i -t ubuntu /bin/bash
```

-i和-t是交互式shell的必要参数，前者保证容器的STDIN是开启的，而后者则告诉Docker为创建的容器分配一个伪tty终端。这样新建的容器才能提供一个交互式的shell。

ubuntu指定了使用的是ubuntu镜像，而ubuntu镜像是一个常备镜像，由docker公司提供，保存在Docker Hub上。可以以操作系统镜像为基础，在其上构建自己的镜像。

Docker首先会检查本地是否存在ubuntu镜像缓存，如果不存在，会连接Docker Hub，搜索它上面是否有该镜像副本，一旦找到，就会下载到本地。

随后，Docker在文件系统内部用这个镜像创建一个新容器，该容器拥有自己的网络，IP地址，以及用来与宿主机通信的桥接网络接口。

/bin/bash要求docker在容器创建完毕后执行容器中的/bin/bash命令。这时就可以看到容器内的shell了。

之后我们在容器中安装vim软件。

```sh
$ apt-get update && apt-get intall vim
```

用户可以在容器中做任何自己想做的事情，当工作完成后，输入exit返回到宿主机。exit会退出/bin/bash命令，而bin/bash命令是容器的唯一命令，一旦/bin/bash退出，容器也会相应地退出。

## 查看所有容器信息

但是即使容器退出了，容器依旧是存在的，可以用docker ps -a命令查看当前系统中所有容器。默认情况下docker ps只会查看正在运行的容器，加了-a可以查看所有的容器，包括运行的和停止的。

```sh
$ docker ps #查看运行中容器
$ docker ps -a #查看所有的容器
$ docker ps -l #查看最后一个执行的容器
```

## 容器标志符

有三种方式可以唯一指定容器，短UUID，长UUID以及容器名。短UUID是长UUID的一个前缀。

如果你没有指定容器名称，那么docker会为我们创建的容器自动生成一个随机名称，如果要自己指定的话，--name标志可以指定自定义名称。

```sh
$ docker run --name bob_the_contaienr -i -t ubuntu /bin/bash
$ exit
$ docker ps -l #显示最后一个容器
$ docker ps -n 10 #显示最后10个容器
```

一个合法的容器名必须满足正则表达式"[a-zA-Z0-9_.-]+"。

使用容器的名称有助于分辨容器，理解容器之间的连接关系，推荐使用容器名称管理容器。

容器的名称必须是唯一的，如果试图创建同名容器，则命令会失败。可以通过docker rm命令删除同名容器后再创建。

## 重启容器

之前的bob_the_container容器已经停止了，我们可以重新启动它。

```sh
$ docker start bob_the_container
```

继续使用ps命令可以看到bob_the_container处于UP状态。

除了用start命令外，也可以用restart重启一个容器，只需要将start替换为restart即可。

## 附着到容器

容器重启后，会沿用docker run命令时指定的参数来运行，因此我们容器重启后会运行一个交互式会话shell。此外，可以用docker attach命令重新附着到该容器的会话上。

```sh
$ docker attach bob_the_container
```

## 创建守护式容器

除了交互式容器外容器外，还可以创建守护式容器，守护式容器没有交互式会话，非常适合运行应用程序和服务。在运行时增加-d参数可以声明容器为守护式容器。

```sh
$ docker run --name deamon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

利用docker ps命令可以看到创建了一个正在运行的容器。

## 查看容器日志

由于守护容器在后台运行，要探究该容器内部都干了什么，可以用docker logs命令来获取容器的日志。

```sh
$ docker logs daemon_dave
```

要退出容器跟踪，输入ctrl+c。docker仅会返回最后的几条日志。使用-f参数可以持续监控日志。

```sh
$ docker logs -f daemon_dave
```

为了便于调试，使用-t标志位每条日志项追加上时间戳。

## docker日志驱动

自docker 1.6起，我们可以控制docker守护进程和容器所用的日志驱动，者可以通过--log-driver选项。

默认的日志驱动是json-file，可用的选项还有syslog，该选项会禁用所有docker logs命令，并将所有容器的日志输出都重定向到syslog上。可以在启动Docker守护进程时候指定该选项，这样所有容器的日志都会输出到syslog，或者通过docker run对个别容器进行日志冲定向输出。

```sh
$ docker run --log-driver="syslog" --name deamon_dwayne -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
$ tail -f /var/log/*
```

还有一个可用选项是none，这个选项会禁用所有容器中的日志。

## 查看容器内进程

我们不仅能看到容器内的日志，还能看到容器内部运行的进程。

```sh
$ docker top daemon_dave
```

利用top命令，可以看到容器内的所有进程。

## 查看容器统计信息

利用docker stats命令，可以看到一个或多个容器的统计信息。

```sh
$ docker stats --no-stream
```

## 在容器内部运行进程

在docker中，可以通过docker exec命令在容器内额外启动新的进程，可以在容器内运行的进程有两类：后台任务和交互式任务。后台任务在容器内运行且没有交互需求，而交互式任务则保持在前台运行。

```sh
$ docker exec -d daemon_dave touch /etc/new/config/file
```

这里-d标志表明运行的是一个后台进程。而不加-d则可以开启一个交互任务，比如开启一个新的shell。

```sh
$ docker exec -t -i daemon_dave /bin/bash
```

## 停止守护式容器

要停止守护式容器，只需要执行docker stop命令。

```sh
$ docker stop daemon_dave
```

docker stop命令会向Docker容器进程发送SIGTERM信号，如果想要快速停止某个容器，可以使用docker kill命令向容器进程发送SIGKILL命令。

```sh
$ docker kill daemon_dave
```

## 自动重启容器

可以增加--restart标记，让docker容器因错误而停止运行时自动重启。docker会检查容器的退出代码，并依据此决定是否要重启容器。默认情况下docker不会重启容器。

```sh
$ docker run --restart always --name daemon_restart -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

在上面这个例子中，--restart被设置为always，无论容器的退出代码实是什么，docker都会自动重启容器。除了always外，可选的值还有on-failure，只有当容器的退出代码非0时才会自动重启。同时on-failure还支持最大重启次数。

## 深入容器

尽管通过ps命令可以获取容器的信息，但是要获得更多信息，你需要使用inspect命令。

```sh
$ docker inspect daemon_dave
```

## 删除容器

```sh
$ docker rm daemon_dave #移除容器
```

rm命令用于删除停止的容器，如果要删除运行中的容器，需要加上-f参数。

```sh
$ docker rm `docker ps -a -q` #删除所有的停止容器
```

docker ps -a -q会返回所有容器的名称，因此，你可以利用上面的命令删除所有的停止容器。

## 列出镜像

利用docker images命令可以列出本地存储的Docker镜像。

```sh
$ docker images #列出镜像列表
$ docker images fedora #仅查看fedora镜像
```

本地的镜像都保存在Docker宿主机的/var/lib/docker目录下。

## 拉取镜像

我们可以利用docker pull命令从Registry中拉取镜像。

```sh
$ docker pull ubuntu:latest
```

## 查找镜像

利用docker search命令可以查找所有Docker Hub上公共的可用镜像。

```sh
$ docker search puppet
```

## 创建Docker Hub账号

我们可以在[https://hub.docker.com/signup](https://hub.docker.com/signup)上创建自己的账号。之后在本地登录Docker Hub：

```sh
$ docker login
```

对应的，如果你想要登出账号，可以使用docker logout命令。

```sh
$ docker logout
```

## 使用docker commit构建镜像

当我们创建一个容器并在容器中做出修改后，可以利用docker命令将修改的部分提交为一个新镜像。

一般来说，我们不会完全重新创建一个镜像，而是基于一个已有的镜像做一些修改后构建出自己的镜像。

```sh
$ docker run -i -t --name custom_ubuntu ubuntu /bin/bash
```

进入shell后，输入

```sh
$ apt-get -yqq update
$ apt-get -y install apache2
$ exit
```

退出shell后输入

```sh
$ docker commit custom_ubuntu taodaling/custom_ubuntu
```

commit将修改后的容器的顶层（读写层）抽出作为一个新的镜像，并提交到本地仓库。

```sh
$ docker commit -m "A customized image" -a "taodaling" custom_ubuntu taoodaling/custom_ubuntu:webserver
```

## 使用docker build构建镜像

要构建自定义的镜像，可以使用docker build命令从Dockerfile文件中构建镜像。

推荐使用build命令替代commit命令，因为前者构建镜像具备更好的可重复性、透明性以及幂等性。

在使用build之前，需要提供Dockerfile。

```sh
$ mkdir static_web
$ cd static_web
$ touch Dockerfile
```

我们创建了一个目录static_web，这个目录就是我们的构建环境，Docker称这个环境为上下文（context）。Docker在构建镜像时会将上下文的文件和目录上传到Docker守护进程。

```dockerfile
# Version: 0.0.1
FROM ubuntu:latest
MAINTAINER taodaling "taodaling@gmail.com"
RUN apt-get update && apt-get install -y nginx
RUN echo 'Hi, I am in your container' > /usr/share/nginx/html/index.html
EXPOSE 80
```

 Dockerfile由一系列指令和参数组成。每条指令，如FROM，都必须为大写字母，后面跟随参数。Dockerfile中的指令会按顺序从上向下执行。

每条Dockerfile中的指令都会创建一个新的镜像层并对镜像进行提交。

Docker会按照下面流程执行Dockerfile中的指令：

- Docker从父镜像运行一个容器
- 执行一条指令，对容器做出修改
- 执行docker commit，提交一个新的镜像层
- Docker基于刚提交的镜像运行一个新容器
- 执行Dockerfile中的下一条指令，直到所有指令执行完成。

如果Docker在执行某条指令时失败，那么用户将得到一个可以使用的镜像，用户可以利用该镜像和失败指令查找失败原因。

Dockerfile支持注释，以#开头的行都会被视作注释。

每个Dockerfile的第一条指令必须是FROM，FROM指令指定一个已存的镜像，并将其作为该条指令生成的镜像，后续的修改将基于这个镜像做出修改。

MAINTAINER指令告诉Docker生成镜像的作者是谁，以及作者的邮件地址。这有助于标识镜像的所有者。

之后的两条RUN指令。RUN指令会在生成镜像上运行指定的命令。默认情况下，RUN指令会在shell里使用命令包装器/bin/sh -c来执行，如果是在一个不支持shell的平台上运行或者不希望在shell运行，也可以使用exec格式的RUN指令， 此时我们用一个数组来表示命令和参数。

```sh
RUN ["apt-get", "install", "-y", "nginx"]
```

EXPOSE指令告诉Docker容器内的应用程序将会使用容器的指定端口。这并不意味着可以自动访问任意容器运行中服务的端口，出于安全的考虑，Docker不会自动打开该端口，而需要用户在使用docker run运行容器时来指定需要打开哪些端口。EXPOSE命令可以出现多次以公开多个端口。

下面我们用这个Dockfile构建镜像。

```sh
$ docker build -t "taodaling/static_web:v1" .
```

上面的使用./Dockerfile构建镜像。你也可以使用GIt仓库中的Dockerfile。

```sh
$ docker build -t "taodaling/static_web:v1" git@github.com:jamtur01/docker-static_web
```

如果在上下文根目录存在以`.dockerignore`命名的文件的话，那么该文件的每一行都指定一个过滤模式，类似于`.gitignore`文件。该文件用来设置哪些文件不会被当做构建上下文的一部分，匹配规则采用的是Go语言的filepath。

## 指令失败情况下的调试

我们修改上面Dockerfile中安装nginx的指令，令其失败。

```dockerfile
$ RUN apt-get update && apt-get install -y ngin
```

之后构建镜像。

```sh
$ docker build -t "taodaling/static_web:failure" .
```

docker构建过程中会发生报错，利用docker ps命令可以看到最后一次执行的容器，之后用inspect命令得到容器的镜像名，利用镜像重建一个容器，就可以手动进行调试。

## Dockerfile和构建缓存

由于每一步的构建过程都会将结果提交为镜像，Docker会将之前构建时创建的镜像当做缓存，并直接跳过。这样就可以不必创建许多中间镜像，从而节省时间。可以认为Docker会从Dockerfile第一条修改的指令开始真正创建镜像，而之前的会复用缓存镜像。

然而，有的时候我们必须禁用缓存，比如我们希望每次都重新执行apt-get update命令，以获得最新的更新。为了禁用缓存，可以在使用build命令时使用--no-cache标志。

```sh
$ docker build --no-cache -t "taodaling/static_web:no_cache .
```

## 基于构建缓存的Dockerfile

构建缓存带来的好处之一是，我们可以实现简单的Dockerfile，在需要重置缓存的指令之前加上一条修改时间戳。

```dockerfile
FROM ubuntu:14.04
MAINTAINER James Turnbull "james@example.com"
ENV REFRESHED_AT 2014-07-01
RUN apt-get -qq update
```

ENV指令用于设置环境变量REFRESHED_AT值为2014-07-01。这个环境变量用于表明该镜像模板的最后更新时间。之后如果你想刷新一个构建，只需要修改时间戳（在这里是环境变量REFRESHED_AT环境变量），之后的命令将不会再复用缓存。

## 查看镜像的构建过程

要查看一个镜像的构建过程，可以使用history命令。

```sh
$ docker history taodaling/static_web
```

## 从新镜像上启动容器

首先启动我们刚才构建的容器。

```sh
$ docker run -d -p 80 --name static_web taodaling/static_web \
nginx -g "daemon off;"
```

这里我们以守护式容器的方式创建了一个新的容器，同时我们在容器中执行了命令nginx -g "daemon off;"，这将以前台运行的方式启动Nginx。由于Docker容器会在命令执行完成后直接进入停止状态，如果以后台模式运行，那么启动nginx的命令会直接退出，这会导致Docker容器停止。

这里我们使用了一个新的-p标记，该标记用于控制Docker在运行时应该公开哪些网络端口给宿主机。Docker可以通过两种方式在宿主机上分配端口：

- Docker可以在宿主机上随机选择一个位于32768~61000的一个较大的端口号来映射到容器的80端口上。
- 可以指定宿主机的一个具体端口号来映射到容器的80端口上。

上面的命令，我们是采用了第一种方式，绑定到了一个随机的宿主机端口上。利用docker ps命令可以看到具体的端口绑定关系，在我的机器上输出如下：

```sh
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                   NAMES
866c768546ba        taodaling/static_web   "nginx -g 'daemon of…"   5 minutes ago       Up 5 minutes        0.0.0.0:32769->80/tcp   static_web
```

除了用docker ps命令外，docker port也可以帮助我们查看容器的端口映射情况。

```sh
$ docker port static_web #查看static_web的所有端口映射关系
$ docker port static_web 80 #仅查看static_web的80端口映射关系
```

要显式地配置映射关系，可以通过`-p 宿主机端口:容器端口`的方式指定。

```sh
$ docker run -d -p 8080:80 --name static_web taodaling/static_web \
nginx -g "daemon off;"
```

我们还可以将端口绑定在特定的IP地址上，用法为`-p 宿主机IP地址:宿主机端口:容器端口`。

```sh
$ docker run -d -p 127.0.0.1:8080:80 --name static_web taodaling/static_web \
nginx -g "daemon off;"
```

上面我们指定了IP地址为127.0.0.1，这样只有宿主机可以访问这个端口。

Docker还提供了一个更加便捷的-P参数，用于公开在Dockerfile中通过EXPOSE指令公开的所有端口，将其中每一个以随机的方式绑定到宿主机的端口上。

```sh
$ docker run -d -P --name static_web taodaling/static_web \
nginx -g "daemon off;"
```

你也可以显式指定端口使用的是TCP还是UDP协议，用法为`[[ip:][宿主机端口]:][容器端口][/tcp|/udp]`。

```sh
$ docker run -d -p 127.0.0.1:8080:80/tcp --name static_web taodaling/static_web \
nginx -g "daemon off;"
```

有了端口号后，你就可以使用容器ip和端口访问服务，也可以通过本地ip和绑定的本地端口访问服务。

## 为镜像创建别名

如果我们在构建镜像时提供了错误的名字或标记。

```sh
$ docker tag myubuntu taodaling/ubuntu
```

之后你可以删除原先的镜像。

```sh
$ docker rmi myubuntu
```



## 推送镜像到Docker Hub

镜像构建完成后，我们可以将它上传到Docker Hub上去，这样其他人就可以使用这个镜像了。

docker push命令就可以用于将镜像推送到Docker Hub。

```sh
$ docker push taodaling/static_web
```

## 自动构建

除了从命令行构建和推送镜像外，Docker Hub还允许我们定义自动构建。我们只需要将GitHub或BitBucket中含有Dockerfile的仓库连接到Docker Hub即可。向这个代码仓库推送代码时，将会触发一次镜像构建过程并创建一个新的镜像。

## 删除镜像

如果不再需要一个镜像了，也可以利用docker rmi将其删除。删除镜像的同时会删除构建镜像过程中创建的中间镜像层。

```sh
$ docker rmi taodaling/static_web
```

rmi仅会删除本地的镜像缓存，而不会推送到远程仓库。

## 建立自己的Docker Registry

如果你想要免费建立自己的Docker Registry，你可以利用Docker公司开源的Docker Registry。

从Docker容器启动一个Registry服务非常简单。

```sh
$ docker run -p 5000:5000 registry
```

要向我们的Registry推送镜像，首先需要打上标记。首先我们要获得镜像的ID。

```sh
$ docker tag 22d47c8cb6e5 localhost:5000/taodaling/static_Web
```

之后就可以向你的Registry推送。

```sh
$ docker push localhost:5000/taodaling/static_Web
```

同样，要运行容器，可以使用下面命令。

```sh
$ docker run localhost:5000/taodaling/static_Web
```

## 构建网站

首先我们先创建必要的目录和文件。

```sh
mkdir sample && cd sample
mkdir nginx && cd nginx
wget https://raw.githubusercontent.com/jamtur01/dockerbook-code/master/code/5/sample/nginx/global.conf
wget https://raw.githubusercontent.com/jamtur01/dockerbook-code/master/code/5/sample/nginx/nginx.conf
```

之后在sample目录下创建Dockerfile文件。

```dockerfile
FROM ubuntu
ENV REFRESHED_AT 2014-06-01
RUN apt-get -yqq update && apt-get -yqq install nginx
RUN mkdir -p /var/www/html/website
ADD nginx/global.conf /etc/nginx/conf.d/
ADD nginx/nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

之后构建镜像。

```sh
docker build -t taodaling/nginx .
```

之后在sample目录下创建website目录并执行下面命令。

```sh
mkdir website && cd website
wget https://raw.githubusercontent.com/jamtur01/dockerbook-code/master/code/5/sample/website/index.html
```

之后启动我们的容器。

```sh
docker run -d -p 80:80 --name website -v $PWD/website:/var/www/html/website taodaling/nginx nginx
```

这里使用了一个新的选项`-v 宿主机文件:容器文件`，将宿主机中的文件作为卷挂载到容器中。 如果容器目录不存在会自动创建。

如果你希望卷只允许读的话，你可以利用`-v 宿主机文件:容器文件:[ro|rw]`，ro表示read only，rw表示read write。

```sh
docker run -d -p 80:80 --name website -v $PWD/website:/var/www/html/website:ro taodaling/nginx nginx
```

卷在Docker中非常重要，卷是一个或多个容器内被选定的目录，可以绕过联合文件系统，为Docker提供持久数据或者共享数据。当提交或者创建镜像时，卷不被包含在镜像中。即使容器停止了，卷里的内容依旧会保留在宿主机的目录下。

之后访问`https://宿主机ip:80`即可查看首页。

## 修改网站

接下来我们修改宿主机的website/index.html文件。

```html
This is a test website for Docker
```

之后刷新首页，可以看到修改生效了。

## 构建Sinatra应用

Sinatra是一个基于Ruby的Web应用框架。

```sh
$ mkdir sinatra && cd sinatra
```

之后编辑Dockerfile文件。

```dockerfile
FROM ubuntu
ENV REFRESHED_AT 2014-06-01
RUN apt-get update -yqq && apt-get -yqq install ruby ruby-dev build-essential redis-tools
RUN gem install --no-rdoc --no-ri sinatra json redis
RUN mkdir -p /opt/webapp
EXPOSE 4567
CMD ["/opt/webapp/bin/webapp"]
```

之后构建镜像。

```sh
$ docker build -t taodaling/sinatra .
```

接下来下载代码。

```sh
$ git clone https://github.com/turnbullpress/dockerbook-code.git
```

之后我们要保证webapp/bin/webapp这个文件可以执行。

```sh
$ chmod +x webapp/bin/webapp
```

之后我们启动容器。

```sh
$ docker run -d -p 4567 --name webapp -v $PWD/webapp:/opt/webapp taodaling/sinatra
```

之后查看日志输出。

```sh
$ docker logs -f webapp
```

接下来可以请求服务。

```sh
$ curl -i -H 'Accept: application/json' -d 'name=Foo&status=Bar' http://localhost:32773/json
```

## 启动Redis

接下来我们为Sinatra应用加入Redis作为后台数据库。并在Redis中存储输入的URL参数。

接下来我们我们使用代码目录下的webapp_redis替代webapp。先为目录分配可执行权限。

```sh
$ chmod +x webapp_redis/bin/webapp
```

之后在sinatra目录下创建一个redis目录，用于保存Dockerfile。接下来编辑Dockerfile。

```dockerfile
FROM ubuntu
ENV REFRESHED_AT 2014-06-01
RUN apt-get -yyq update && apt-get -yqq install redis-server redis-tools
EXPOSE 6379
ENTRYPOINT ["/usr/bin/redis-server"]
CMD []
```

之后构建镜像。

```sh
$ docker build -t taodaling/redis .
```

先启动redis容器。

```sh
$ docker run -d -p 6379 --name redis taodaling/redis
```

## 连接Sinatra和Redis

接下来连接Sinatra和Redis。要建立连接，有以下方式。

- Docker内部网络。
- 使用Docker Networking以及docker network命令。
- Docker链接。一个可以将具体容器链接到一起来进行通信的抽象层。

第一种方式比较简单但是功能也比较弱，不推荐使用。如果在Docker1.9及之后的版本推荐使用Docker Networking，如果之前的版本应该使用Docker链接。

Docker Networking相较于Docker链接的区别有下列。

- Docker Networking可以跨宿主机连接容器
- 停止启动或者重启容器不需要更新连接。
- Docker Networking可以在容器创建之前创建。

## 内部联网

到目前为止我们看到的Docker容器都是公开端口，并绑定到本地网络端口的。除了这种方法，Docker还提供了内部网络。

在安装Docker时，会创建一个新的网络接口，名字是docker0。每个Docker容器都会在这个接口上分配一个IP地址。接下来查看docker0接口。

```sh
$ ip a show docker0

5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:2b:b5:f8:ca brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:2bff:feb5:f8ca/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到docker0接口有符合RFC1918的私有IP地址，范围是172.16~172.30，而接口本身地址172.17.0.1是这个Docker网络的网关地址，也是所有Docker容器的网关地址。

Docker默认会使用172.17.x.x作为子网地址，除非已经有人占用了这个子网。如果被占用了，Docker会在172.16~172.30这个范围内尝试创建子网。

接口docker0是一个虚拟的以太网桥，用于连接容器和本地宿主网络。如果进一步查看Docker宿主机的其它网络接口，会发现一系列名字以veth开头的接口。

```:zero:
veth26166aa@if261: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default 
    link/ether fe:9c:b1:14:f4:e7 brd ff:ff:ff:ff:ff:ff link-netnsid 26
    inet6 fe80::fc9c:b1ff:fe14:f4e7/64 scope link 
       valid_lft forever preferred_lft forever
```

Docker每创建一个容器就会创建一个一组互联的网络接口。这组接口就像管道的两端，其中一端作为容器中的eth0接口，而另外一端统一命名为类似veth123这种名字，作为宿主机的一个端口。可以将veth接口认为是虚拟网线的一端，这个虚拟网线一端插在名为docker0的网桥上，另外一端插在容器中。通过把每个veth*接口绑定到docker0网桥，Docker创建了一个虚拟子网，这个子网由宿主机和所有的容器共享。

在docker容器中查看网卡信息。

```sh
$ ifconfig eth0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.7  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:07  txqueuelen 0  (Ethernet)
        RX packets 4151  bytes 15873185 (15.8 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3958  bytes 321224 (321.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

可以看到Docker给容器分配了IP地址172.17.0.7作为宿主虚拟接口的另一端，这样就能让宿主网络和容器相互通信了。

安装traceroute工具并查看报文转发路径。

```sh
$ apt-get -yqq update && apt-get install -yqq traceroute
$ traceroute google.com

 1  172.17.0.1 (172.17.0.1)  0.047 ms  0.014 ms  0.012 ms
 2  192.168.1.1 (192.168.1.1)  1.114 ms  1.036 ms  0.918 ms
 3  122.235.136.1 (122.235.136.1)  36.845 ms  37.767 ms  37.711 ms
 4  61.164.0.205 (61.164.0.205)  6.480 ms  7.152 ms  7.087 ms
 5  61.164.22.105 (61.164.22.105)  7.023 ms  6.967 ms 220.191.157.25 (220.191.157.25)  7.651 ms
 6  202.97.26.9 (202.97.26.9)  8.326 ms 202.97.55.17 (202.97.55.17)  6.657 ms 202.97.26.1 (202.97.26.1)  4.958 ms
```

可以看到容器的下一跳就是宿主网络上docker0接口的网关IP172.17.0.1。

接下来我们要访问容器的端口，我们可以先获得容器的子网IP，之后通过子网IP:绑定宿主机端口进行访问。

但是由于docker在容器重启后会重新分配IP，因此硬编码的方式并不是理想的解决方案。

## Docker Networking

容器之间的连接用网络建立，称为Docker Networking。Docker Networking允许用户创建自己的网络，容器可以通过这个网络相互通信。实际上，Docker Networking利用新的用户管理的网络补充了现有的docker0。更重要的是，现在容器可以跨越不同的宿主机进行通信，并且网络的配置也更加的灵活。Docker Netwroking还与Docker compose以及swarm进行了集成，后面会介绍到。

首先我们先创建一个网络。

```sh
$ docker network create app
```

上面我们用docker network命令创建了一个桥接网络，名称为app。这个命令会返回被创建的网络ID。之后我们来查看这个网络。

```sh
$ docker network inspect app

[
    {
        "Name": "app",
        "Id": "c7fe77eefef3570e52ba2c74bdd9a24037b84150b366625e84d4d75c8f747b36",
        "Created": "2019-01-26T21:04:09.533540764-05:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

这个网络是一个运行在本地的桥接网络。我们看到现在还没有容器在这个网络中运行。

除了运行于单台主机上的桥接网络，我们也可以创建一个overlay网络，overlay网络允许我们跨越多台宿主机进行通信。

利用docker network ls命令查看当前系统中的所有网络。

```sh
$ docker network ls

NETWORK ID          NAME                        DRIVER              SCOPE
c7fe77eefef3        app                         bridge              local
56f467ed3e21        bridge                      bridge              local
```

也可以使用docker network rm命令删除一个Docker网络。

```sh
$ docker network create temp
$ docker network rm temp

temp
```

我们接下来先启动Redis容器，并在app网络中添加一些容器。

```sh
$ docker run -d --net app --name db taodaling/redis

b9e9d746fd45cfbb29b0436b14fa7cd4a6b65a75fb708facc3aa7bed9886cd14
```

这里我们使用了一个新的选项--net，`--net 网络名称或ID`，这会将容器加入到app网络中。重新查看网络状态。

```sh
$ docker network inspect app

[
    {
        "Name": "app",
        "Id": "c7fe77eefef3570e52ba2c74bdd9a24037b84150b366625e84d4d75c8f747b36",
        "Created": "2019-01-26T21:04:09.533540764-05:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "9ad7565c436872ffc2cb0f322d210f4ecff359693b22ba4c713166e29696799f": {
                "Name": "db",
                "EndpointID": "d3efbb0a99f90bd8f6a26446693322f09b24344678a76b7abc6107857fe81e25",
                "MacAddress": "02:42:ac:15:00:02",
                "IPv4Address": "172.21.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

我们可以看到网络中出现了一个容器，并为其分配的IP地址。

接下来我们启动Sinatra应用程序。

```sh
$ docker run -p 4567 --net=app --name webapp -t -i -v $PWD/webapp:/opt/webapp taodaling/sinatra /bin/bash
```

之后我们查看容器中的/etc/hosts文件。

```sh
$ cat /etc/hosts

127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.21.0.3      29228d49ac55
```

可以尝试在容器中ping容器db。

```sh
$ ping db

PING db (172.21.0.2) 56(84) bytes of data.
64 bytes from db.app (172.21.0.2): icmp_seq=1 ttl=64 time=0.054 ms
64 bytes from db.app (172.21.0.2): icmp_seq=2 ttl=64 time=0.060 ms
```

接下来我们启动自己的webapp。

```sh
$ docker run -d -p 4567 --name webapp -v $PWD/webapp_redis:/opt/webapp taodaling/sinatra

895a99be17dd40824b047df29dc97fda05b09ca9f83b96d91ab22198d259087e
```

测试效果。

```sh
$ curl -i -H 'Accept: application/json' -d 'name=Foo&status=Bar' http://localhost:32779/json

HTTP/1.1 200 OK 
Content-Type: text/html;charset=utf-8
Content-Length: 29
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Server: WEBrick/1.4.2 (Ruby/2.5.1/2018-03-29)
Date: Sun, 27 Jan 2019 09:04:19 GMT
Connection: Keep-Alive

{"name":"Foo","status":"Bar"}
```

## 连接已有容器到Docker网络

可以将正在运行的容器通过docker network connect命令添加到已有网络中。假设我们在运行db容器之前没有指定网络。下面我们将它加入我们的app网络。

```sh
$ docker network connect app db
```

我们也可以通过docker network disconnect命令断开一个容器与网络的连接。

```sh
$ docker network disconnect app db
```

一个容器可以同时连接到多个Docker网络上，因此我们可以创建非常复杂的网络模型。

## 通过Docker链接的方式连接容器

在Docker 1.9版本之前，可以使用Docker链接来连接多个容器。我们先启动db容器。

```sh
$ docker run -d --name redis taodaling/redis /usr/bin/redis-server --protected-mode no
```

注意上面我们没有选择暴露端口。查看容器IP地址信息。

```sh
$ docker inspect redis

...
           "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "56f467ed3e212bc5b18f8695e3bf23aa30b4b015611ada3feff3e70caedce8c3",
                    "EndpointID": "b377cecbffc2cec8c18329d06f7794a8560f11c2f0ea943e4579cb7c188bfc9c",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.6",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:06",
                    "DriverOpts": null
                }
            }
...
```



之后启动webapp时同时链接到db容器。

```sh
$ docker run -p 4567 --name webapp --link redis:db -d -v $PWD/webapp_redis:/opt/webapp taodaling/sinatra
```

注意这里我们使用了新的选项`--link 容器:主机名`，它会在容器的/etc/hosts文件中增加映射关系。连接到容器webapp上并查看容器的/etc/hosts文件内容。

```sh
$ docker exec -i -t webapp /bin/bash
$ cat /etc/hosts

127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.6      db 4fac7c0d13fa redis
172.17.0.8      5a495c119fd8
```

可以发现db、4fac7c0d13fa、redis等都一同绑定在了172.17.0.6上。

由于我们的redis没有绑定到宿主机的端口，因此redis处于十分安全的处境，不会遭遇到攻击。但是容器链接是不能跨宿主机的。

## 杀死容器

接下来我们来停止之前启动的db服务。但是不通过stop，而是选择使用信号。docker kill会向容器的运行进程发送信号，并粗暴地杀死容器。

```sh
$ docker kill -s SIGKILL redis

redis
```

用docker stop停止容器，docker会给予容器中的应用程序10s的时间以停止运行，可以用-t选项提供额外的时间。因此docker stop会更加优雅一些。在时限内docker stop会发送SIGTERM信号给容器中的应用，超时后会发送SIGKILL信号。

```sh
$ docker stop -t 60 redis #60秒内停止容器内应用
```

## 编排

编排（orchestration）描述了自动配置、协助和管理服务的过程。编排用于描述一组实际过程，这个过程会管理运行在不同宿主机上，不同容器中的应用。

## Docker Compose

使用Docker Compose，可以用YAML文件定义一组要启动的容器以及容器的属性。Docker Compose称这些容器为服务，并定义：

容器通过某些方法并指定一些运行时属性来和其它容器交互。

## 安装Docker Compose

安装的方式可以在官网[https://docs.docker.com/compose/](https://docs.docker.com/compose/)上找到。

## 创建示例应用

创建目录并编辑Dockerfile。

```sh
$ mkdir composeapp && cd composeapp
$ touch Dockerfile
```

编辑app.py文件。

```python
from flask import Flask
from redis import Redis
import os

app = Flask(__name__)
redis = Redis(host="redis_1", port=6379)

@app.route('/')
def hello():
  redis.incr('hits')
  return 'Hello, we have met {0} times'.format(redis.get('hits'))

if __name__ == '__main__':
  app.run(host='0.0.0.0', debug=True)
```

编辑requirement.txt文件。

```
flask
redis
```

编辑Dockerfile文件。

```dockerfile
FROM python:2.7
ADD . /composeapp
WORKDIR /composeapp
RUN pip install -r requirements.txt
```

构建镜像。

```sh
$ docker build -t taodaling/composeapp .
```

## docker-compose.yml文件

构建好镜像后，可以利用Compose来创建需要的服务。在Compose中，除了定义要启动的服务外，还需要定义服务启动所需的属性，这些属性类似于docker  run命令的参数。

将所有与服务相关的属性定义在一个YAML文件中，之后利用docker-compose up命令启动，docker-compose会利用文件中定义的属性启动容器，并将所有的日志输出合并到一起。

现在编辑docker-compose.yml文件。

```yaml
web:
  image: taodaling/composeapp
  command: python app.py
  ports:
  - "5000:5000"
  volumes:
  - .:/composeapp
  links:
  - redis
redis:
  image: redis
```

现在可以用启动我们的服务。

```sh
$ docker-compose up

Starting composeapp_redis_1_8f986c0b5b54 ... done
Starting composeapp_web_1_a32f7f24c8a0   ... done
Attaching to composeapp_redis_1_8f986c0b5b54, composeapp_web_1_a32f7f24c8a0
redis_1_8f986c0b5b54 | 1:C 28 Jan 2019 15:40:41.034 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis_1_8f986c0b5b54 | 1:C 28 Jan 2019 15:40:41.034 # Redis version=5.0.3, bits=64, commit=00000000, modified=0, pid=1, just started
redis_1_8f986c0b5b54 | 1:C 28 Jan 2019 15:40:41.034 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 * Running mode=standalone, port=6379.
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 # Server initialized
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 * DB loaded from disk: 0.000 seconds
redis_1_8f986c0b5b54 | 1:M 28 Jan 2019 15:40:41.036 * Ready to accept connections
web_1_a32f7f24c8a0 |  * Serving Flask app "app" (lazy loading)
web_1_a32f7f24c8a0 |  * Environment: production
web_1_a32f7f24c8a0 |    WARNING: Do not use the development server in a production environment.
web_1_a32f7f24c8a0 |    Use a production WSGI server instead.
web_1_a32f7f24c8a0 |  * Debug mode: on
web_1_a32f7f24c8a0 |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
web_1_a32f7f24c8a0 |  * Restarting with stat
web_1_a32f7f24c8a0 |  * Debugger is active!
web_1_a32f7f24c8a0 |  * Debugger PIN: 307-737-617
```

可以看到compose创建了composeapp_redis_1_8f986c0b5b54和composeapp_web_1_a32f7f24c8a0容器，这两个名字是通过一系列规则生成的，用于保证容器名唯一。同时可以看到日志汇总了redis和composeapp的日志。

我们也可以以守护进程的方式启动容器。

```sh
$ docker-compose up -d

Starting composeapp_redis_1_8f986c0b5b54 ... done
Starting composeapp_web_1_a32f7f24c8a0   ... done
```

之后访问`{{host}}:5000`就可以看到页面了。

Redis和composeapp之间的连接是通过由Compose控制的容器之间的链接实现的。

默认情况下，docker-compose会连接本地的Docker守护进程，但是如果你显式指定了DOCKER_HOST环境变量，那么docker-compose会请求该地址指定的守护进程。

## Docker Swarm



# 配置

## 守护进程

### 修改监听地址

```sh
$ docker daemon -H tcp://0.0.0.0:2375
```

上面的命令是一次性的。

```sh
$ export DOCKER_HOST="tcp://0.0.0.0:2375"
```

上面的改变会持久到重启。

```sh
$ docker deamon -H unix://home/docker/docker.sock
```

可以使用Unix套接字地址，这样就可以仅本地访问服务器。

```sh
$ docker daemon -H tcp://0.0.0.0:2375 -H unix://home/docker/docker.sock
```

也可以一次性监听多个地址。

### 调试模式

```sh
$ docker deamon -D
```

已调试模式启动守护进程，这样会输出额外的冗余信息。

可以修改/usr/lib/systemd/system/docker.service或/etc/sysconfig/docker文件修改ExecStart项来永久变更是否默认开启调试模式。

### 代理

Docker守护进程使用HTTP_PROXY和HTTPS_PROXY以及NO_PROXY环境变量来配置代理。配置的流程如下：

```sh
mkdir -p /etc/systemd/system/docker.service.d
```

之后创建文件/etc/systemd/system/docker.service.d/http-proxy.conf并加入以下内容：

```properties
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80/"
```

如果使用的是HTTPS代理，则创建的文件则应该是https-proxy.conf，内容格式为

```sh
[Service]
Environment="HTTPS_PROXY=https://proxy.example.com:443/"
```

 如果你希望一些registry不走HTTP代理，比如你自己搭建了一个私有registry。那么你需要修改http-proxy.conf文件：

```properties
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
```

当然如果你希望不走的是HTTPS代理，则修改https-proxy.conf：

```properties
[Service]    
Environment="HTTPS_PROXY=https://proxy.example.com:443/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
```

在修改完成后，让你的配置文件即刻起效：

```sh
systemctl daemon-reload
systemctl restart docker
```

之后确认修改是否起效：

```sh
systemctl show --property=Environment docker
```

## Dockerfile

### CMD

CMD指令用于指定一个容器启动时要运行的命令。这有些类似于RUN指令，但是RUN指令是在构建的时候执行的命令，而CMD是容器启动后运行的命令。

与RUN命令一样，CMD命令有两种格式，一种是在命令前加上/bin/bash -c后执行，一种是exec格式。

```sh
CMD ["/bin/bash", "-l"] # exec格式，推荐
CMD /bin/bash -l # /bin/bash -c执行
```

当你没有在docker run命令中显示指定容器启动后执行的命令的话，CMD指令才会被执行。

一个Dockerfile中假如有多条CMD指令，那么仅最后一条会被保留。如果你确实需要执行多条命令，可以用RUN命令将这些命令写入到脚本中。之后CMD命令直接执行脚本就可以了。

### ENTRYPOINT

ENTRYPOINT指令类似于CMD指令，由于CMD指令会被覆盖。ENTRYPOINT与CMD的区别在于，CMD指令会被覆盖，而ENTRYPOINT指令不会被覆盖，但是它会接收你在使用docker run命令时传递的额外参数作为ENTRYPOINT的参数。

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```

之后构建镜像。

```sh
docker build -t taodaling/test_entrypoint .
```

之后我们启动容器。

```sh
docker run taodaling/test_entrypoint "Hello, World"
```

和CMD指令一样，只能存在一个ENTRYPOINT指令。但是CMD指令和ENTRYPOINT可以共存，利用CMD指令我们可以为ENTRYPOINT指令提供默认参数。

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["default bahavior"]
```

如果确实必要，你可以通过docker run的--entrypoint选项覆盖ENTRYPOINT命令。

```sh
docker run --entrypoint /bin/bash -i -t taodaling/test_entrypoint
```

### WORKDIR

WORKDIR指令用来容器创建后，设置工作目录（类似于CD命令）。之后ENTRYPOINT和CMD指令都会在这个目录下执行。除此之外WORKDIR还可以为其它指令设置工作目录，比如在构建时为RUN设置工作目录。

```dockerfile
FROM ubuntu
WORKDIR /var/log
RUN ["mkdir", "-p", "child"]
WORKDIR ./child
ENTRYPOINT ["pwd"]
```

可以通过-w标志在运行覆盖工作目录。

```sh
docker run -w / taodaling/test_workdir
```

### ENV

ENV指令用于在镜像构建过程中设置环境变量。这个环境变量对后面执行的所有命令都有效，包括RUN、CMD、ENTRYPOINT等。

```dockerfile
FROM ubuntu
ENV RVM_PATH /home/rvm/
RUN gem install unicorn
```

一个ENV指令可以一次性设置多个环境变量。

```dockerfile
FROM ubuntu
ENV RVM_PATH="/home/rvm/" RVM_ARCHFLAGS="-arch i386"
RUN mkdir -p RVM_PATH
CMD echo $RVM_ARCHFLAGS
```

也可以在使用docker run的时候利用-e选项传递环境变量，这些变量仅在容器的生命周期中有效。

```sh
docker run -e RVM_ARCHFLAGS="-arch unknown" taodaling/test_env
```

### USER

USER指令用于指定镜像由谁去运行。

```dockerfile
USER nginx
```

基于该镜像启动的容器会以nginx用户的身份来运行。我们可以指定用户名或UID、组或GID，格式为`USER [UID|用户名][:[GID|组名]]`。

```dockerfile
USER user
USER user:group
USER uid:gid
```

你也可以在执行docker run时通过-u选项来指定登录用户。如果没有指定过用户，则默认使用root。

### VOLUME

VOLUME指令用于向容器添加卷，一个卷可以存在于一个或多个容器内的特定目录。这个目录可以向他们提供数据共享和持久化的功能。

- 卷支持在容器之间共享
- 对卷的修改是立即生效的
- 对卷的修改不会对更新镜像产生影响
- 卷会一直存在

卷可以让我们将文件和目录添加到镜像中，但是不提交到镜像中。

```docker
VOLUME ["/opt/project", "/data"]
```

这条指令会为基于此镜像创建的任何容器创建名为/opt/project和/data的挂载点。

### ADD

ADD指令用来将构建环境下的文件和目录复制到镜像中。但是不能指定非构建环境下的文件，因为构建上下文中的文件会直接上传给Docker守护进程，而Docker守护进程无法访问到宿主机中的其它文件

```dockerfile
ADD software.lic /opt/application/software.lic #将构建上下文中的softlic文件复制到/opt/application/software.lic
```

目的地如果以/结尾，那么docker认为是复制到该目录下，否则认为是覆盖该文件。

源文件除了可以是构建上下文的文件外，还可以是URL。

```dockerfile
ADD http://wordpress.org/latest.zip /root/wordpress.zip
```

值得一提的是，ADD在处理归档文件时，如果目的地是目录，docker会自动将文档解压。如果存在文件冲突，则保留原来目的目录下的文件。

如果目的位置不存在的话，Docker还会为我们自动创建路径。而新建的文件和目录的模式为0755，且UID和GID均为0。

### COPY

COPY和ADD指令类似，与ADD不同的是，COPY仅会复制文件，而不会执行解压的工作。

```dockerfile
COPY conf.d/ /etc/apache2/
```

上面的命令会把conf.d目录中的所有文件和目录复制到/etc/apache2/目录下。

文件的源路径必须是一个与当前构建环境相对的文件的或者目录，且处于构建目录下，而任何创建的文件或目录的UID和GID都是0。

### LABEL

LABEL指令用于为Docker镜像添加元数据，元数据以键值对的形式存放。

```dockerfile
LABEL version="1.0"
LABEL location="New York" type="Data Center" role="Web Server"
```

推荐所有的元数据放在一条LABEL命令中，这样可以减少需要创建的镜像层数。

要查看标签信息，可以使用docker inspect命令。

### STOPSIGNAL

STOPSIGNAL指令用于设置停止容器时发送的系统调用信号。这个信号必须是内核系统调用表中合法的数，比如9，或者SIGNAME格式中的信号名称，如SIGKILL。

### ARG

ARG指令用来定义可以在构建时使用的变量，我们只需要在构建时使用--build-arg标志即可。用户在构建时只能指定Dockerfile文件中定义过的参数。

```dockerfile
ARG build #不带默认值
ARG webapp_user=user #带默认值
```

当构建时没有指定参数值，那么就会使用默认值。

```dockerfile
docker build --build-arg build=1234 -t jamtur01/webapp .
```

docker预定义了一组ARG变量，因此你可以不用重复声明。

- HTTP_PROXY
- http_proxy
- HTTPS_PROXY
- https_proxy
- FTP_PROXY
- ftp_proxy
- NO_PROXY
- no_proxy

### ONBUILD

ONBUILD指令能为镜像添加触发器，当一个镜像被用做其他镜像的父亲镜像进行构建时，该镜像的触发器将会被执行。

触发器会在构建过程中插入新指令，我们可以认为这些指令是紧跟在FROM之后执行的。触发器可以时任何构建指令，但不包括FROM、MAINTAINER、ONBUILD。

```dockerfile
ONBUILD ENV relation="I'm son of x"
```

ONBUILD命令不会被继承，因此一个镜像在孙子镜像构建时并不会被执行。
