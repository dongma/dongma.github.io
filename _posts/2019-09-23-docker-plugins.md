---
layout: post
title: 使用Docker构建微服务镜像
---
![1569252813951](https://raw.githubusercontent.com/dongma/springcloud/master/document/images/1569252813951.png)
 <!-- more -->
> `Docker`包括一个命令行程序、一个后台守护进程，以及一组远程服务。它解决了常见的软件问题，并简化了安装、运行、发布和删除转件。这一切能够实现是通过使用一项`UNIX`技术，称为容器。

事实上，`Docker`项目确实与`Cloud Foundry`的容器在大部分功能和实现原理上都是一样的，可偏偏就是这剩下的一小部分不一样的功能成为了`Docker`呼风唤雨的不二法宝，这个功能就是`Docker`镜像。

与传统的`PaaS`项目相比，`Docker`镜像解决的恰恰就是打包这个根本性问题。所谓的`Docker`镜像，其实就是一个压缩包。但是这个压缩包中的内容比`PaaS`的应用可执行文件+启停脚本的组合就要丰富多了。实际上，大多数`Docker`镜像是直接由一个完整操作系统的所有文件和目录构成的，所以这个压缩包内容和本地开发、测试环境用的操作系统是完全一样的，这正是`Docker`镜像的精髓所在。

所以，`Docker`项目给`PaaS`世界带来的"降维打击"，其实是提供了一种非常便利的打包机制。这种机制直接打包了应用运行所需要的整个操作系统，从而保证了应用运行所需要的整个操作系统，从而保证了本地环境和云端环境的高度一致，避免了用户通过"试错"来匹配两种不同的运行环境之间差异的痛苦过程。

### 1. 容器技术基础概念

`Docker`容器中的运行就像是其中的一个进程，对于进程来说，它的静态表现就是程序，平常都安安静静地待在磁盘上。而一旦运行起来，它就变成了计算机里的数据和状态的总和，这就是它的动态表现。而容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个"边界"。

对于`Docker`等大多数`Linux`容器来说，`Cgroups`技术是用来制造约束的主要手段，而`Namespace`技术则是用来修改进程视图的主要方法。在`Docker`里容器中进程号始终是从`1`开始，容器中运行的进程已经被`Docker`隔离在了一个跟宿主机完全不同的世界当中。

**1）Namespace修改Docker进程的视图**，在`linux`中创建线程的系统调用`clone()`函数，这个系统调用会为我们返回一个新的进程，并且返回它的进程号`pid`。而当我们用`clone()`函数调用和创建一个新进程时，就可以在参数中执行`CLONE_NEWPID`参数。这时，新创建的这个进程将会看到一个全新的进程空间，在这个进程空间里，它的`pid`为1。之所以所看到，是因为使用了"障眼法"，在宿主机真实的进程空间里，这个进程的`pid`还是真实的数值，比如`100`：

```c
int pid = clone(main_function, stack_size, SIGCHLD, NULL);
# 创建新的线程指定CLONE_NEWPID，返回新的进程空间的id
int pid = clone(main_function, stack_size, CLONE_NEWPID|SIGCHLD, NULL);
```

当然，我们还可以多次执行上面的`clone()`调用，这样就会创建多个`Pid Namespace`，而每个`namespace`里的应用进程都会被认为自己是当前容器里的第`1`号进程，它们既看不到宿主机里真正的进程空间，也看不到其它`PID Namespace`里的具体情况。除过刚才提到的`PID Namespace`，`Linux`操作系统还提供了`Mount`、`UTS`、`IPC`、`Network`和`User`这些`Namespace`用来对各种不同的进程上下文进行“障眼法”操作。

“敏捷”和“高性能”是容器相较于虚拟机最大的优势，也是它能够在`PaaS`这种更细粒度的资源管理平台上大行其道的重要原因。不过，有利也有弊，基于`linux namespace`的隔离机制相比较与虚拟化技术也有很多不足之处，其中最主要的问题就是：隔离得不彻底。首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机操作系统内核。其次，在`linux`内核中，有很多资源和对象是不能被`namespace`化的，最典型的例子就是：时间（若在容器中应用程序改变了系统时间，则整个宿主机的时间都会被随之修改）。

**2）在介绍完容器的"隔离"技术之后，我们再来研究一下容器的"限制"问题**。虽然容器内的第`1`号进程在“障眼法”的干扰下只能看到容器里的情况，但是宿主机上它作为第`100`号进程与其他所有进程之间仍然是平等的竞争关系。虽然第`100`号进程表面上被隔离了起来，但是它所能够使用到的资源（如`CPU`、内存）却是可以随时被宿主机上的其他进程占用的。当然，这个`100`号进程自己也可能把所有资源吃光。这些情况，显然都不是一个“沙盒”应该表现出来的合理行为。

而`linux Cgroups`就是`linux`内核中用来为进程设置资源限制的一个重要功能，`linux Cgroups`的全称是`linux Control Group`。它的主要作用，就是限制一个进程组能够使用的资源上线，包括`CPU`、内存、磁盘、网络带宽等。此外，`Cgroups`还能够对进程进行优先级设置、审计，以及将进程挂起和修复等操作。在`/sys/fs/cgroup`下面有很多诸如`cpuset`、`cpu`、`memory`这样的子目录，也称为子系统。这些都是我这台机器当前可以被`Cgroups`进行限制的资源种类，而在子系统对应的资源种类下，就可以看到该类资源具体可以被限制的方法。如`cpu`的子系统，可以看到如下几个配置文件：

```shell
$ ls /sys/fs/cgroup/cpu
cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us cpu.shares notify_on_release
cgroup.procs cpu.cfs_quota_us cpu.stat tasks
```

若熟悉`linux cpu`管理的话，就会在输出中注意到`cfs_period`和`cfs_quota`这样的关键字。这两个参数需要组合使用，可以用来限制进程在长度为`cfs_period`的一段时间内，只能被分配到总量为`cfs_quota`的`cpu`时间。在`tasks`文件中通常用来放置资源被限制的进程的`id`号，会对该进程进行`cpu`使用资源限制。除了`cpu`子系统外，`Cgroups`的每一项子系统都有其独有的资源限制能力，比如：`blkio`为块设置设置`I/O`限制，一般用于磁盘等设备。`cpuset`为进程分配单独的`cpu`核和对应的内存节点。`memory`为进程设置内存使用的限制。`linux Ggroups`的设计还是比较易用的，简单粗暴地理解，它就是一个子系统目录加上一组资源限制文件的组合。

**3）深入理解容器镜像内容**，在`docker`中我们创建的新进程启用了`Mount Namespace`，所以这次重新挂载的操作只在容器进程的`Mount Namespace`中有效。但在宿主机上用`mount -l`检查一下这个挂载，你会发现它是不存在的。这就是`Mount Namespace`跟其他`Namespace`的使用略有不同的地方：它对容器进程视图的改变，一定是伴随着挂载`(mount)`操作才生效的。在`linux`操作系统里，有一个名为`chroot`的命令可以帮助你在`shell`中方便地完成这个工作。顾名思义，它的作用就是帮你`"change root file system"`，即改变进程的根目录到你指定的位置。

对于`chroot`的进程来说，它并不会感受到自己的根目录已经被"修改"成`$HOME/test`了。实际上，`Mount Namespace`正是基于对`chroot`的不断改变才被发明出来的，它也是`linux`操作系统里的第一个`Namespace`。而这个挂载在容器根目录上，用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫做：`rootfs`（根文件系统）。

需要明确的是，`rootfs`只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在`linux`操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。不过，正是由于`rootfs`的存在，容器才有了一个被反复宣传至今的重要特性：一致性。由于`rootfs`里打包的不只是应用，而是整个操作系统的文件和目录。也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。对一个应用程序来说，操作系统本身才是它运行所需要的最完整的"依赖库"。这种摄入到操作系统级别的运行环境一致性，打通了应用在本地开发和远程执行环境之间难以逾越的鸿沟。

### 2. Docker容器常用命令

在`docker`中运行一个`nginx`容器实例，运行该命令`docker`会从`docker hub`上下载和安装像`nginx:latest`镜像。然后运行该软件，一行看似随机的字符串将会被写入所述终端。

```shell
> docker run --detach --name web nginx:latest
> 60ae46f06db51c929e51a932daf506
```

运行交互式的容器，`docker`命令行工具是一个很好的交互式终端程序示例。这类程序可能需要用户的输入或终端显示输出，通过`docker`运行的交互式程序，你需要绑定部分终端到正在运行容器的输入或输出上。该命令使用`run`命令的两个标志：`--interactive`和`--tty`，`-i`选项告诉`docker`保持标准输入流（`stdin`，标准输入）对容器开放，即使容器没有终端连接。其次`--tty`选项告诉`docker`为容器分配一个虚拟终端，这将允许你发信号给容器。

```shell
> docker run --interactive --tty --link web:web --name web_test busybox:latest /bin/bash
```

列举、停止、重新启动和查看容器输出的`docker`命令，`docker ps`命令会用来显示每个运行容器的`id`、容器的镜像、容器中执行的命令、容器运行的时长、容器暴露的网络端口、容器名。`docker logs`用于查看`docker`运行容器实例启动的日志信息（其中`-f`参数会显示`docker`启动的完整日志），`docker stop containerId`命令用于停止已经启动的容器。

```shell
> docker restart f38f6ce59e9d
> f38f6ce59e9d4d1c929e51a932daf50
```

灵活的容器标识，可以使用`--name`选项在容器启动时设定标识符。如果只想在创建容器时得到容器`id`，交互式容器时无法做到的。幸运的是你可以用`docker create`命令创建一个容器而并不启动它。环境变量是通过其执行上下文提供给程序的键值对，它可以让你在改变一个程序的配置时，无须修改任何文件或更改用于启动该程序的命令。其是通过`- env`参数进行传递的，就像`mysql`数据在启动时指定`root`用户的密码。

```shell
> docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=Aa123456! mysql
> 265c55de36095f1938f1aa27dcc2887
```

`docker`提供了用于监控和重新启动容器的几个选项，创建容器时使用`--restart`标志，就可以通知`docker`完成以下操作。在容器中需执行回退策略，当容器启动失败的时候会自动重新启动容器。为了使用容器便于清理，在`docker run`命令中可以加入`--rm`参数，当容器实例运行结束后创建的容器实例会被自动删除。

```shell
> docker run -d --name backoff-detector --restart always busybox date
```

在`docker`中可以使用`--volume`参数来定义存储卷的挂载，可以使用`docker inspect`命令过滤卷键，`docker`为每个存储卷创建的目录是由主机的`docker`守护进程控制的。`docker`的`run`命令提供了一个标志，可将卷从一个或多个容器复制到新的容器中，标志`--volumes`可以设定多次，可以指定多个源容器。当你使用`--volumes-from`标志时，`docker`会为你做到这一切，复制任何本卷所引用的源容器到新的容器中。对于存储卷的清理，可以使用`docker rm -v`选项删除孤立卷。

```shell
> docker run -d --volume /var/lib/cassanda/data:/data --name cass-shared cassandra:2.2
> 31eda1bb0e8fe59e9d4d1c929e51a932

> docker run --name aggregator --volumes-from cass-shared alpine:latest echo "collection created"
```

链接——本地服务发现，你可以告诉`docker`，将它与另外一个容器相链接。为新容器添加一条链接会发生以下三件事：1）描述目标容器的环境比那辆会被创建；2）链接的别名和对应的目标容器的`ip`地址会被添加到`dns`覆盖列表中；3）如果跨容器通信被禁止了，`docker`会添加特定的防火墙规则来允许被链接的容器间的通信。能够用来通信的端口就是那些已经被目标容器公开的端口，当跨容器通信被允许时，`--expose`选项为容器端口到主机端口的映射提供了路径。在同样的情况下，链接成了定义防火墙规则和在网络上显示声明容器接口的一个工具。

```shell
> docker run -d --name importantData --expose 3306 mysql_noauth service mysql_noauth start

> docker run -d --name importantWebapp --link importantData:db webapp startapp.sh -db tcp://db:3306
```

`commit`——创建新镜像，可以使用`docker commit`命令从被修改的容器上创建新的镜像。最好能够使用`-a`选项为新镜像指定作者的信息。同时也应该总是使用`-m`选项，它能够设置关于提交的信息。一旦提交了这个镜像，它就会显示在你计算机的已安装镜像列表中，运行`docker images`命令会包含新构建的镜像。当使用`docker commit`命令，你就向镜像提交了一个新的文件层，但并不是只有文件系统快照被提交。

```shell
> docker commit -a "sam_newyork@163.com" -m 'added git component' image-dev ubuntu-git
> ae46f06db51c929e51a932daf5
```

对于要进行构建的应用可以通过使用`Dockerfile`进行构建，其中`-t`的作用是给这个镜像添加一个`tag`（也即起一个好听的名字）。`docker build`会自动加载当前目录下的`Dockerfile`文件，然后按照顺序执行文件中的原语。而这个过程实际上可以等同于`docker`使用基础镜像启动了一个容器，然后在容器中依次执行`Dockerfile`中的原语。若需要将本地的镜像上传到镜像中心，则需要对镜像添加版本号信息，可以使用`docker tag`命令。

```shell
> docker build -t helloworld .
# tag already build image with version
> docker tag helloworld geektime/helloword:v1
# push build image to remote repository
> docker push helloworld geektime/helloword:v1
```

### 3. 使用Dockerfile构建应用

```dockerfile
# 使用官方提供的python开发镜像作为基础镜像
FROM python:2.7-slim
# 将工作目录切换为/app
WORKDIR /app
# 将当前目录下的所有内容复制到/app下
ADD . /app
# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt
# 允许外界访问容器的80端口
EXPOSE 80
# 设置环境变量
ENV NAME World
# 设置容器进程为:python app.py, 即这个python应用的启动命令
CMD ["python", "app.py"]
```

通过这个文件的内容，你可以看到`dockerfile`的设计思想，是使用一些标准的原语（即大写高亮的词语），描述我们所要构建的`docker`镜像。并且这些原语，都是按顺序处理的。比如`FROM`原语，指定了`python:2.7-slim`这个官方维护的基础镜像，从而免去了安装`python`等语言环境的操作。其中`RUN`原语就是在容器里执行`shell`命令的意思。

而`WORKDIR`意思是在这一句之后，`dockerfile`后面的操作都以这一句指定的`/app`目录作为当前目录。所以，到了最后的`CMD`，意思是`dockerfile`指定`python app.py`为这个容器的进程。这里`app.py`的实际路径为`/app/app.py`，所以`CMD ["python", "app.py"]`等价于`docker run python app.py`。

此外，在使用`dockerfile`时，你可能还会看到一个叫做`ENTRYPOINT`的原语。实际上，它和`CMD`都是`docker`容器进程启动所必须的参数，完整执行格式是：`ENTRYPOINT CMD`。默认情况下，`docker`会为你提供一个隐含的`ENTRYPOINT`也即`:/bin/sh -c`。所以，在不指定`ENTRYPOINT`时，比如在我们的这个例子里，实际上运行在容器里的完整进程是：`/bin/sh -c python app.py`，即`CMD`的内容是`ENTRYPOINT`的参数。

需要注意的是，`dockerfile`里的原语并不都是指对容器内部的操作。就比如`ADD`，它指的是把当前目录（即`dockerfile`所在的目录）里的文件，复制到指定容器内的目录中。

### 4. 使用Docker Compose进行服务编排

> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.

在`elementory OS`上安装`docker compose`服务，按照官方文档完成后可以通过`docker-compose version`来检查安装`compose`的版本信息：

```shell
sam@elementoryos:~/docker-compose$ sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sam@elementoryos:~/docker-compose$ sudo chmod +x /usr/local/bin/docker-compose
sam@elementoryos:~/docker-compose$ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

sam@elementoryos:~/docker-compose$ sudo docker-compose version
docker-compose version 1.24.1, build 4667896b
docker-py version: 3.7.3
CPython version: 3.6.8
OpenSSL version: OpenSSL 1.1.0j  20 Nov 2018
```

可以依据`docker`官方使用`python`和`redis`搭建应用：`https://docs.docker.com/compose/gettingstarted/`，在`docker-compose.yml`文件编写完成后，可以使用`docker-compose up`启动编排服务：

```shell
sam@elementoryos:~/docker-compose$ sudo docker-compose up
Creating network "docker-compose_default" with the default driver
Building web
Step 1/9 : FROM python:3.7-alpine
3.7-alpine: Pulling from library/python
89d9c30c1d48: Already exists
910c49c00810: Pull complete
Successfully tagged docker-compose_web:latest
```

使用`docker-compose ps`查看当前`compose`中运行的服务，使用`docker-compose stop`结束编排服务：

```shell
sam@elementoryos:~/docker-compose$ sudo docker-compose ps
         Name                       Command               State           Ports
-------------------------------------------------------------------------------------
docker-compose_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp
docker-compose_web_1     flask run                        Up      0.0.0.0:5000->5000/tcp
```

`docker-compose.yml`文件语法：使用`version`版本号`3`表示其支持版本。`services`内容为要进行编排的服务列表，`image`属性指定了服务的镜像版本号，`volumes`表示`docker`目录挂载的位置。对于`web`服务在`ports`属性值为映射的端口信息，若服务之前启动存在依赖则可以使用`depends_on`属性处理。本地服务若需要构建，则可以使用`build`属性，其会从当前目录下`Dockerfile`中构建镜像。

```yml
version: '3'
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
```
