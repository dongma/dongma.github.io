---
layout: post
title: 使用kubernetes构建微服务
---
![kubernetes](https://raw.githubusercontent.com/SamMACode/springcloud/master/kubernetes/images/kubernetes_logo.png)

##  Build distributed services with kubernetes

>  **Kubernetes** (commonly  stylized as k8s) is an open-source container-orchestration system for  automating application deployment, scaling, and management.  It aims to provide a "platform for automating deployment, scaling, and operations of application  containers across clusters of hosts". 

####  一、在`elementory OS`服务器搭建kubernetes环境

`elementary OS`是基于`ubuntu`精心打磨美化的桌面 `linux` 发行版的一款软件，号称 “最美的 `linux`”， 最早是 `ubuntu` 的一个美化主题项目，现在成了独立的发行版。"快速、开源、注重隐私的 `windows` /` macOS` 替代品"。

1）在`elementary OS`系统上安装`docker`环境，具体可以参考` https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/`：

```shell
# 1.更新ubuntu的apt源索引
sam@elementoryos:~$ sudo apt-get update
# 2.安装以下包以使apt可以通过HTTPS使用存储库repository
sam@elementoryos:~$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
# 3.添加Docker官方GPG key
sam@elementoryos:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# 4.设置Docker稳定版仓库
sam@elementoryos:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
# 5.再更新下apt源索引，然后通过docker version显示器版本信息
sam@elementoryos:~$ apt-get update
sam@elementoryos:~$ sudo docker version
Client:
 Version:           18.09.7
Server:
 Engine:
  Version:          18.09.7   
# 6.从镜像中心拉取hello-world镜像并进行运行
sam@elementoryos:~$ sudo docker run hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
```
<!-- more -->
管理`docker`服务常用应用脚本：

` sudo service docker start `  启动`docker`服务、` sudo service docker stop ` 停止`docker`服务、` sudo service docker restart `重启docker服务.



2）使用`minikube`在本机搭建`kubernetes`集群，简单体验`k8s`： 

为了方便开发者开发和体验`kubernetes`，社区提供了可以在本地部署的`minikube`。由于国内网络的限制导致，导致在本地安装`minikube`时相关的依赖是无法下载。从`minikube`最新的`1.5`版本之后，已经提供了配置化的方式，可以直接从阿里云的镜像地址来获取所需要的`docker`镜像和配置。

在`elementary OS`上安装`kubectl`的稳定版本：

```shell
sam@elementoryos:~$ sudo curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.16.0/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
```

在安装完成后使用`kubectl version`进行验证，由于`minikube`服务未启动最后的报错可以忽略:

```shell
sam@elementoryos:~$ sudo kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server 192.168.170.130:8443 was refused - did you specify the right host or port?
```

通过`curl`命令从`github`上下载`minikube`的`1.5.0`版本：

```shell
sam@elementoryos:~$ curl -Lo minikube https://github.com/kubernetes/minikube/releases/download/v1.5.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

启动`minikube`服务，为了访问海外资源阿里云提供了一系列基础措施可以通过参数进行配置，`--image-mirror-country cn`默认会从`registry.cn-hangzhou.aliyuncs.com/google_containers`下载`kubernetes`依赖的相关资源。首次启动会在本地下载` localkube `、`kubeadm`等工具。

```shell
sam@elementoryos:~$ sudo minikube start --vm-driver=none --image-mirror-country cn --memory=1024mb --disk-size=8192mb --registry-mirror=https://registry.docker-cn.com --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --bootstrapper=kubeadm --extra-config=apiserver.authorization-mode=RBAC
😄  minikube v1.5.0 on Debian buster/sid
✅  Using image repository registry.cn-hangzhou.aliyuncs.com/google_containers
🤹  Running on localhost (CPUs=2, Memory=3653MB, Disk=40059MB) ...
ℹ️   OS release is elementary OS 5.0 Juno
🐳  Preparing Kubernetes v1.16.2 on Docker 18.09.7 ...
🏄  Done! kubectl is now configured to use "minikube"
```

在`minikube`安装完成后，在本地`minikube dashboard --url`控制页面无法展示，目前暂时未解决。

```shell
sam@elementoryos:~$ sudo kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
```

使用`sudo minikube dashboard --url`自动生成`minikube`的管理页面：

```
sam@elementoryos:~$ sudo minikube dashboard -url
```

`minikube`本地环境搭建可参考这几篇文章：

使用`minikube`在本地搭建集群：http://qii404.me/2018/01/06/minukube.html

阿里云的`minikube`本地实验环境：https://yq.aliyun.com/articles/221687

关于`kubernetes`解决`dashboard`：https://blog.8hfq.com/2019/03/01/kubernetes-dashboard.html

#### 二、运行于kubernetes中的容器

`kubernetes`中的`pod`组件：`pod`是一组并置的容器，代表了`kubernetes`中基本构建模块。在实际应用中我们并不会单独部署容器，更多的是针对一组`pod`容器进行部署和操作。当一个`pod`包含多个容器时，这些容器总是会运行于同一个工作节点上——一个`pod`绝不会跨越多个工作节点。

对于`docker`和`kubernetes`期望的工作方式是将每个进程运行于自己的容器内，由于不能将多个进程聚集在一个单独的容器中，我们需要另一种更高级的结构来将容器绑定在一起，并将它们作为一个单元进行管理，这就是`pod`背后的根本原理。对于容器彼此之间是完全隔离的，但此时我们期望的是隔离容器组，而不是单个容器，并让容器组内的容器共享一些资源。`kubernetes`通过配置`docker`来让一个`pod`内的所有容器共享相同的`linux`命名空间，而不是每个容器都有自己的一组命名空间。

由于一个`pod`中的容器运行于相同的`network`命名空间中，因此它们共享相同的`IP`地址和端口空间。这意味着在同一`pod`中的容器运行的多个进程需要注意不能绑定想同的端口号，否则会导致端口冲突。

1）在`kubernetes`上运行第一个应用`swagger-editor`并对外暴露`8081`端口：

```shell
sam@elementoryos:~$ sudo kubectl run swagger-editor --image=swaggerapi/swagger-editor:latest --port=8081 --generator=run/v1

sam@elementoryos:~$ sudo kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
swagger-editor-xgqzm   1/1     Running   0          57s
```

在`kubectl run`命令中使用`--generator=run/v1`参数表示它让`kubernetes`创建一个`ReplicationController`而不是`Deployment`。通过`kubectl get pods`可以查看所有`pod`中运行的容器实例信息。每个`pod`都有自己的`ip`地址，但是这个地址是集群内部的，只有通过`LoadBalancer`类型服务公开它，才可以被外部访问，可以通过运行`kubectl get services`命令查看新创建的服务对象。

```shell
sam@elementoryos:~$ sudo kubectl expose rc swagger-editor --type=LoadBalancer --name swagger-editor-http
service/swagger-editor-http exposed

sam@elementoryos:~$ sudo kubectl get services
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP      10.96.0.1        <none>        443/TCP          46m
swagger-editor-http   LoadBalancer   10.108.118.211   <pending>     8081:30507/TCP   3m24s
```

2）为了增加期望的副本数，需要改变`ReplicationController`期望的副本数，现已告诉`kubernetes`需要采取行动，对`pod`的数量采取操作来实现期望的状态。

```shell
sam@elementoryos:~$ sudo kubectl scale rc swagger-editor --replicas=3
replicationcontroller/swagger-editor scaled
sam@elementoryos:~$ sudo kubectl get pods
NAME                   READY   STATUS              RESTARTS   AGE
swagger-editor-fzppq   0/1     ContainerCreating   0          12s
swagger-editor-wqpg5   0/1     ContainerCreating   0          12s
swagger-editor-xgqzm   1/1     Running             0          16m
```

为了观察列出`pod`时显示`pod ip`和`pod`的节点，可以通过使用`-o wide`选项请求显示其他列。在列出`pod`时，该选项显示`pod`的`ip`和所运行的节点。由于`minikube`不支持`rc`，因而并不会展示外部`ip`地址。若想在不通过`service`的情况下与某个特定的`pod`进行通信（处于调试或其它原因）,`kubernetes`将允许我们配置端口转发到该`pod`，可以通过`kubectl port-forward`命令完成上述操作：

```shell
sam@elementoryos:~$ sudo kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
swagger-editor-fzppq   1/1     Running   0          5m28s   172.17.0.7   minikube   <none>           <none>
swagger-editor-wqpg5   1/1     Running   0          5m28s   172.17.0.5   minikube   <none>           <none>
swagger-editor-xgqzm   1/1     Running   0          21m     172.17.0.6   minikube   <none>           <none>

sam@elementoryos:~$ sudo kubectl port-forward swagger-editor-fzppq 8088:8081
Forwarding from 127.0.0.1:8088 -> 8081
Forwarding from [::1]:8088 -> 8081
```

标签是一种简单却功能强大的`kubernetes`特性，不仅可以组织`pod`也可以组织所有其他的`kubernetes`资源。详细来讲，可以通过标签选择器来筛选`pod`资源。在使用多个`namespace`的前提下，我们可以将包括大量组件的复杂系统拆分为更小的不同组，这些不同组也可以在多租户环境中分配资源。



#### 三、副本机制和其它控制器：部署托管的`pod`

`kubernetes`可以通过存活探针`(liveness probe)`检查容器是否还在运行，可以为`pod`中的每个容器单独指定存活探针。如果探测失败，`kubernetes`将定期执行探针并重新启动容器。`kubernetes`有三种探测容器的机制：通过`http get`对容器发送请求，若应用接收到请求，并且响应状态码不代表错误，则任务探测成功；`TCP`套接字探针尝试与容器指定端口建立`TCP`连接，若长连接正常建立则探测成功；`exec`探针在容器中执行任意命令，并检查命令的退出返回码。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15
```

`kubia-liveness-probe-initial-delay.yaml`文件中在`livenessProbe`中指定了通过`httpGet`探测的探针地址检测应用的状态，为了防止容器启动时通过探针地址检测应用状态，可以通过设置`initialDelaySeconds`指定应用启动间隔时间（像`spingboot`应用的`/health`端点就非常合适）。

了解`ReplicationController`组件：`ReplicationController`是一种`kubernetes`资源，可确保它的`pod`始终保持运行状态。如果`pod`因任何原因消失，则`ReplicationController`会注意到缺少了`pod`并创建替代`pod`。`ReplicationController`的工作是确保`pod`的数量始终与其标签选择器匹配，若不匹配则`rc`会根据需要，采取适当的操作来协调`pod`的数量。`label selector`用于确定`rc`作用域内有哪些`pod`、`replica count`指定应运行的`pod`数量、`pod template`用于创建新的`pod`副本。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```

`kubia-rc.yaml`文件定义，在`yaml`中`selector`指定了符合标签的选择器`app: kubia`。若删除的`rc`创建的一个`pod`，则其会自动创建新的`pod`使得副本的数量达到`yaml`文件配置的数量。若要将`pod`移出`rc`作用域，可以通过更改`pod`的标签将其从`rc`的作用域中进行移除，`--overwrite`参数是必要的，否则`kubectl`将只是打印出警告，并不会更改标签。对于修改`rc`的`template`只会对之后新创建的`pod`有影响，而对之前已有的`pod`不会造成影响。若需要对`pod`进行水平扩展，可以通过修改`edit`调整`replicas:10`的属性，或者通过命令行`kubectl scale rc kubia --replication=10`进行调整。

```shell
sam@elementoryos:~$ sudo kubectl create -f kubia-rc.yaml
ReplicationController "kubia" created
sam@elementoryos:~$ sudo kubectl label pod kubia-demdck app=foo --overwrite
# 通过kubectl更改rc的template内容
sam@elementoryos:~$ sudo kubectl edit rc kubia
```

当要删除`rc`则可以通过`kubectl delete`进行操作，`rc`所管理的所有`pod`也会被删除。若需要保留`pod`的时候，则需要在命令行添加`--cascade=false`的配置，当删除`replicationController`后，其之前所管理的`pod`就独立。

`ReplicaSet`的引入：最初`ReplicationController`是用于复制和在异常时重新调度节点的唯一`kubernetes`组件，后来引入了`ReplicaSet`的类似资源。它是新一代的`rc`并且会将其完全替换掉。`ReplicaSet`的行为与`rc`完全相同，但`pod`选择器的表达能力更强。在`yaml`文件配置中其`apiVersion`内容为`apps/v1beta2`，其`kind`类型为`ReplicaSet`类型。

```shell
sam@elementoryos:~$ sudo kubectl delete rs kubia
```

引入`DaemonSet`组件：要在所有集群结点上运行一个`pod`，需要创建一个`DaemonSet`对象。`DaemonSet`确保创建足够的`pod`，并在自己的节点上部署每个`pod`。尽管`ReplicaSet(ReplicationController)`确保集群中存在期望数量的`pod`副本，但`DaemonSet`并没有期望的副本的概念。它不需要，因为它的工作是确保一个`pod`匹配它的选择器并在每个节点上运行。

在`DaemonSet`的`yml`配置文件中，其`apiVersion`内容为`apps/v1beta2`，`kind`类型为`DeamonSet`。在删除`DaemonSet`时候其所管理`pod`也会被一并删除。

```shell
sam@elementoryos:~$ sudo kubectl create -d ssd-monitor-deamonset.yaml
# view all DaemonSet components in kubernetes
sam@elementoryos:~$ sudo kubectl get ds
```

介绍`Kubernetes Job`资源：`kubernetes`通过`Job`资源提供对短任务的支持，在发生节点故障时，该节点上由`Job`管理的`pod`将按照`ReplicaSet`的`pod`的方式，重新安排到其他节点。如果进程本身异常退出（进程返回错误退出代码时），可以将`Job`配置为重新启动容器。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5
  parallelism: 2
  schedule: "0,15,30,45 * * * *"
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

`Job`是`batch API`组`v1`版本的一部分，`yaml`定义了一个`Job`类型的资源，它将运行`luksa/batch-job`镜像，该镜像调用一个运行`120`秒的进程，然后退出。在`pod`的定义中，可以指定在容器中运行的进程结束时，`kubernetes`会做什么？这是通过`pod`配置的属性`restartPolicy`完成的，默认为`Always`配置 在`Job`中使用`OnFailure`的策略。可以在`yaml`文件中指定`parallelism: 2`来指定任务的并行度，通过创建`cronJob`资源在`yaml`中指定‘`schedule: 0,15,30,45 * * * *`定时任务表达式。`startingDeadlineSeconds: 15`指定`pod`最迟必须在预定时间后`15`秒开始执行。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl create -f kubernetes-job.yaml 
job.batch/batch-job created
sam@elementoryos:~/kubernetes$ sudo kubectl get jobs
NAME        COMPLETIONS   DURATION   AGE
batch-job   0/1           47s        47s
sam@elementoryos:~/kubernetes$ sudo kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
batch-job-nzbmv   1/1     Running   0          108s
sam@elementoryos:~/kubernetes$ sudo kubectl logs batch-job-nzbmv
Sun Nov 17 09:09:01 UTC 2019 Batch job starting
```



`service`服务：让客户端发现`pod`并与之通信

> `kubernetes`服务是一种为一组功能相同`pod`提供单一不变的接入点的资源，当服务存在时，它的`ip`地址和端口不变。客户端通过固定`ip`和`port`建立连接，这种连接会被路由到提供该服务的任意一个`pod`上。通过这种方式，客户端不需要知道每个`pod`的地址，这样这些`pod`就可以在集群中被随时创建或者移除。

可以使用`kubectl expose`命令创建服务，`rc`是`replicationcontroller`的缩写。由于`minikube`不支持`LoadBalance`类型的服务，因此服务的`external-ip`地址为`<none>`。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl expose rc kubia --type=LoadBalancer --name kubia-http
service "kubia-http" exposed
sam@elementoryos:~/kubernetes$ sudo kubectl get services
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          2d5h
kubia        ClusterIP   10.111.211.203   <none>        80/TCP,443/TCP   22h
sam@elementoryos:~/kubernetes$ sudo kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-9vds6   1/1     Running   0          23h
kubia-cpjvx   1/1     Running   0          23h
kubia-hs5vq   1/1     Running   0          23h
```

另一种是使用`yaml`描述文件`kubia-svc.yaml`来创建服务，使用`sudo kubectl create -f kubia-svc.yaml ` 。`service`也是通过`selector`筛选符合条件的`pod`，通过`ports`对端口进行转发。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

从内部集群测试服务，可以通过`kubectl exec`命令在一个已经存在的`pod`中执行`curl`命令，其作用和`docker exec`命令比较类似。在`kubernetes`命令中`--`代表着`kubectl`命令项的结束，在`--`后的内容是在`pod`内部需要执行的命令。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl exec kubia-9vds6 -- curl -s http://10.111.211.203
You've hit kubia-cpjvx
```

通过环境变量发现服务：在`pod`开始的时候，`kubernetes`会初始化一系列的环境变量指向现在存在的服务。一旦选择了目标`pod`，通过在容器中运行`env`来列出所有的环境变量。在`ENV`列出的环境变量中，`KUBIA_SERVICE_HOST`和`KUBIA_SERVICE_PORT`分表代表了`kubia`服务的`ip`地址和端口号。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl exec kubia-9vds6 env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-9vds6
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=7.9.0
YARN_VERSION=0.22.0
HOME=/root
```

通过`dns`发现服务：在`kube-system`命名空间下列出的所有`pod`信息，其中一个为`coredns-755587fdc8`。每个服务从内部`dns`服务器中获得一个`dns`条目，客户端的`pod`在知道服务名称的情况下可以通过全限定域名`(FQDN)`来访问，而不是诉诸于环境变量。前端`pod`可以通过`backend-database.default.svc.cluster.local`访问后端数据库服务：`backend-database`对应于服务名称，`default`表示服务在其中定义的名称空间，`svc.cluster.local`是在所有集群本地服务名称中使用的可配置集群域后缀。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl get pods --namespace kube-system
NAME                               READY   STATUS             RESTARTS   AGE
coredns-755587fdc8-nz7s8           0/1     CrashLoopBackOff   80         2d6h
etcd-minikube                      1/1     Running            0          2d6h
kube-addon-manager-minikube        1/1     Running            0          2d6h
kube-apiserver-minikube            1/1     Running            0          2d6h
kube-controller-manager-minikube   1/1     Running            0          2d6h
kube-proxy-gczr4                   1/1     Running            0          2d6h
kube-scheduler-minikube            1/1     Running            0          2d6h
storage-provisioner                1/1     Running            0          2d6h
```

由于`kubernetes`容器编排中`kube-dns`服务不可用，因而在`pod`内部无法实现通过`service.namespace.clustername`访问`exposed`服务。在`pod`内部`/etc/resolv.conf`文件中保存内容与`host`文件类似。在`curl`这个服务是工作的，但却是`ping`不通的，因为服务的集群`ip`是一个虚拟`ip`，并且只有在于服务端口结合时才有意义。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl exec -it kubia-9vds6 bash
[sudo] password for sam: ******        
root@kubia-9vds6:/# curl http://kubia.default.svc.cluster.local
curl: (6) Could not resolve host: kubia.default.svc.cluster.local
root@kubia-9vds6:/# curl http://kubia.default
curl: (6) Could not resolve host: kubia.default
root@kubia-9vds6:/# curl http://kubia        
curl: (6) Could not resolve host: kubia

root@kubia-9vds6:/# cat /etc/resolv.conf 
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local localdomain
```

连接集群外部的服务：在`kubernetes`中，服务并不是和`pod`直接相连的。相反，有一种资源介于两者之前——它就是`Endpoint`资源。如果之前在服务在运行过`kubectl describe`。`endpoint`资源就是暴露一个服务的`ip`地址和端口的列表，`endpoint`资源和其他`kubernetes`资源一样，所以可以使用`kubectl info`来获取它的基本信息。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl describe svc kubia
[sudo] password for sam:        
Name:              kubia
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=kubia
Type:              ClusterIP
IP:                10.111.211.203
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         172.17.0.5:8080,172.17.0.6:8080,172.17.0.7:8080
Port:              https  443/TCP
TargetPort:        8443/TCP
Endpoints:         172.17.0.5:8443,172.17.0.6:8443,172.17.0.7:8443
Session Affinity:  ClientIP
Events:            <none>

sam@elementoryos:~/kubernetes$ sudo kubectl get endpoints kubia
NAME    ENDPOINTS                                                     AGE
kubia   172.17.0.5:8443,172.17.0.6:8443,172.17.0.7:8443 + 3 more...   23h
```

将服务暴露给外部客户端：服务的`pod`不仅可以在`kubernetes`内部进行调用，有时，`k8s`还需要向外部服务公开某些服务（例如`web`服务器，以便外部客户端可以访问它们）。

> 有几种方式可以在外部访问服务：将服务类型设置为`NodePort`——每个集群节点都会在节点上打开一个端口，对于`NodePort`服务，每个集群节点在节点本身上打开一个端口，并将该端口上接收到的流量重定向到基础服务；将服务类型设置为`LoadBalance`，`NodePort`类型的一种扩展——这使得服务可以通过一个专用的负载均衡器来访问，这是由`kubernetes`中正在运行的云基础设置提供的；创建一个`Ingress`服务，这是一个完全不同的机制，通过一个`ip`地址公开多个服务。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: kubia
```

在配置文件`kubia-svc-nodeport.yaml`中，`spec`部分的`type`属性值为`NodePort`类型。其中`targetPort`表示背后`pod`的目标端口号、通过`nodePort`的集群的`30123`端口可以访问该服务。通过`kubectl get svc kubia-nodeport`可以看到`ENTERNAL-IP`列数据为`<nodes>`，表示服务可通过任何集群节点的`ip`地址访问。其中`PORT(S)`列显示集群`IP(80)`的内部端口和节点端口`(30123)`。可以使用`curl`命令通过`10.109.37.229`地址进行请求`pod`。在使用`minikube`时，可以运行`minikube service <service-name>`命令，就可以通过浏览器轻松访问`NodePort`服务。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl create -f kubia-svc-nodeport.yaml 
[sudo] password for sam:        
service/kubia-nodeport created
sam@elementoryos:~/kubernetes$ sudo kubectl get svc kubia-nodeport
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubia-nodeport   NodePort   10.109.37.229   <none>        80:30123/TCP   17s
sam@elementoryos:~/kubernetes$ curl http://10.109.37.229:80
You've hit kubia-9vds6
sam@elementoryos:~/kubernetes$ sudo minikube service kubia-nodeport
|-----------|----------------|-------------|------------------------------|
| NAMESPACE |      NAME      | TARGET PORT |             URL              |
|-----------|----------------|-------------|------------------------------|
| default   | kubia-nodeport |             | http://192.168.170.130:30123 |
|-----------|----------------|-------------|------------------------------|
🎉  Opening kubernetes service  default/kubia-nodeport in default browser...
```

通过负载均衡将服务暴露出来，创建`LoadBalance`服务，`spec.type`的类型为`LoadBalancer`。如果没有指定特定的节点端口，`kubernetes`将会选择一个端口。如果使用的是`minikube`，尽管负载平衡器不会被分配，仍然可以通过节点端口（位于`minikube vm`的`ip`地址）访问服务。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl get svc kubia-loadbalancer
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubia-loadbalancer   LoadBalancer   10.101.132.161   <pending>     80:32608/TCP   41s
```

使用`Ingress`向外暴露服务的意义：一个重要的原因是每个`LoadBalancer`服务都需要自己的负载均衡器，以及独有的公有`ip`地址，而`Ingress`只需要一个公网`ip`就能为许多服务提供访问。在介绍`Ingress`对象提供的功能之前，必须强调只有`Ingress`控制器在集群中运行，`Ingree`资源才能正常工作。由于网络限制在使用`minikube`时，并不能从外网`pull`所需的镜像。

```shell
sam@elementoryos:~/kubernetes$ sudo minikube addons enable ingress
✅  ingress was successfully enabled
sam@elementoryos:~/kubernetes$ sudo kubectl get pods --all-namespaces
kube-system            nginx-ingress-controller-6fc5bcc8c9-7zp46    0/1     ImagePullBackOff   0          6m8s
```

使用`kubia-ingress.yaml`在`kubernetes`中创建`Ingress`资源，`Ingress`将域名`kubia.example.com`映射到你的服务，将所有的请求发送到`kubia-nodeport`服务的`80`端口。`Ingress`的工作原理：客户端通过`Ingress`控制器连接到其中一个`pod`，客户端首先对`kubia.example.com`执行`DNS`查找，`DNS`服务器返回了`Ingress`控制的`ip`。客户端然后向`Ingress`控制器发送`Http`请求，并在`Host`头中指定`kubia.example.com`。控制器从该头部确定客户端尝试访问哪个服务，通过与该服务关联的`Endpoint`对象查看`pod IP`，并将客户端的请求转发给其中一个`pod`。
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```
`Ingress`不仅可以转发`http`流量，可以使用`Ingress`创建`TLS`进行认证，控制器将终止`tls`连接。客户端和控制器之间的通信是加密的，而控制器和后端`pod`之前的通信则不是。运行在`pod`上的应用程序是不需要`tls`，如果`pod`运行`web`服务器，则它只能接收`http`通信。要使控制器能够这样做，需要将证书和私钥附加到`Ingress`，这两个必须资源存储在称为`secret`的`kubernetes`资源中，然后在`Ingress manifest`中引用它。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl get ingresses
Name 				Hosts				Address 		Ports		 Age
kubia			kubia.example.com   192.168.99.100		80			 29m
sam@elementoryos:~/kubernetes$ curl http://kubia.example.com
You've hit kubia-9vds6
```



#### 四、`kubernetes`卷挂载、用`ConfigMap`和`Secret`配置应用

`kubernetes`的卷是`pod`的一个组成部分，因此像容器一样在`pod`的规范中做定义了。它们不是独立的`kubernetes`对象，也不能单独创建或删除。`pod`中的所有容器都可以使用卷，但必须先将它挂载在每个需要访问它的容器中。在每个容器中，都可以在其文件系统的任何位置挂载卷。

最简单的卷类型是`emptyDir`卷，一个`emptyDir`卷对于在同一个`pod`中运行的容器至今共享文件特别有用，其可以被单个容器用于将数据临时写入磁盘。在`fortune-pod.yaml`中`pod`包含两个容器和一个挂载在两个容器中公用的卷，但在不同的路径上。`html-generator`启动时，它每`10`秒启动一次`fortune`命令输出到`/var/htdocs/index.html`文件。当`web-server`容器启动，它就开始为`/usr/share/nginx/html`目录中的任意`html`文件提供服务，最终效果是，一个客户端向`pod`上`80`端口发送一个`http`请求，将接收当前的`fortune`消息作为响应。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

为了查看`fortune`消息，需要启用对`pod`的访问，可以尝试将端口从本地机器转发到`pod`实现。若等待几秒发送另一个请求，则应该会接收另一条消息。作为卷来使用`emptyDit`，是在承载`pod`的工作节点的实际磁盘上创建的。可以将`emptyDir`的`medium`设置为`Memory`将临时数据写入到内存中。

```shell
sam@elementoryos:~/kubernetes/fortune$ sudo kubectl port-forward fortune 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080

sam@elementoryos:~/kubernetes$ curl http://localhost:8080
Your talents will be recognized and suitably rewarded.
sam@elementoryos:~/kubernetes$ curl http://localhost:8080
Your business will go through a period of considerable expansion.
```

使用`Git`仓库作为存储卷：`gitRepo`卷基本上也是一个`emptyDir`卷，它通过克隆`Git`仓库并在`pod`启动时（但在创建容器之前）检出特定版本来填充数据。在创建`pod`之前，需要有一个包含`html`文件并实际可用的`Git`仓库。创建`pod`时，首先将卷初始化为一个空目录，然后将制定的`Git`仓库克隆到其中。`kubernetes`会将分支切换到`master`上。

```yaml
  volumes:
  - name: html
    gitRepo:
      repository: https://github.com/luksa/kubia-website-example.git
      revision: master
      directory: .
```

`kubernetes`中某些系统级别的`pod`会使用`hostPath`访问节点文件系统上的文件，`hostPath`卷指向节点系统上的特定文件或目录。在同一个结点上运行并在其`hostPath`卷中使用相同路径的`pod`可以看到相同的文件。`hostPath`卷持久性存储，`gitRepo`和`emptyDir`卷的内容都会在`pod`被删除时被删除，而`hostPath`卷的内容则不会被删除。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl get pods --namespace kube-system
[sudo] password for sam:        
NAME                                        READY   STATUS             RESTARTS   AGE
coredns-755587fdc8-nz7s8                    0/1     CrashLoopBackOff   402        4d20h
etcd-minikube                               1/1     Running            1          4d20h
kube-controller-manager-minikube            1/1     Running            20         4d20h
kube-proxy-gczr4                            1/1     Running            1          4d20h

sam@elementoryos:~/kubernetes$ sudo kubectl describe pod kube-proxy-gczr4 --namespace kube-system
Volumes:
  kube-proxy:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      kube-proxy
    Optional:  false
  xtables-lock:
    Type:          HostPath (bare host directory volume)
    Path:          /run/xtables.lock
    HostPathType:  FileOrCreate
  lib-modules:
    Type:          HostPath (bare host directory volume)
    Path:          /lib/modules
    HostPathType:  
  kube-proxy-token-qdktp:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kube-proxy-token-qdktp
    Optional:    false
QoS Class:       BestEffort
```



配置容器化应用程序，在`kubernetes`中使用`ConfigMap`配置`pod`应用：

> 无论是否在使用`ConfigMap`存储配置数据，如下三种方式都可用于配置你的应用程序：向容器中传递命令行参数、为每个容器设置自定义环境变量、通过特殊类型的卷将配置文件挂载到容器中。

在`docker`中定义命令与参数：`ENTRYPOINT`和`CMD`，在`Dockerfile`中的两种指令分别定义命令与参数这两部分，`ENTRYPOINT`定义容器启动时被调用的可执行程序、`CMD`指定传递给`ENTRYPOINT`的参数。在`fortune`镜像中添加`VARIABLE`变量并用第一个命令行参数对其进行初始化`INTERVAL=$1`，在`Dockerfile`中添加`CMD ["10"]`将命令行参数进行传递。

`kubernetes`允许将配置选项分离到单独的资源对象`ConfigMap`中，本质上就是一个键/值对映射，值可以是短字面量，也可以是完整的配置文件。映射的内容通过环境变量或者卷文件的形式传递给容器，而并非直接传递给容器。命令行参数的定义中可以通过`${ENV_VAR}`语法引用环境变量，因而可以达到将`ConfigMap`的条目当作命令行参数传递给进程。

```shell
sam@elementoryos:~/kubernetes$ sudo kubectl create configmap fortune-config --from-literal=sleep-interval=25
[sudo] password for sam:        
configmap/fortune-config created

sam@elementoryos:~/kubernetes$ sudo kubectl get configmap fortune-config -o yaml
apiVersion: v1
data:
  sleep-interval: "25"
kind: ConfigMap
metadata:
  creationTimestamp: "2019-11-24T09:51:36Z"
  name: fortune-config
  namespace: default
  resourceVersion: "151450"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 918d8a0a-f4a1-4b75-8f5b-e1f018a33dec
```

可以使用`kubectl create configmap` 创建`ConfigMap`，此命令支持从磁盘上读取文件，并将文件内容单独存储为`ConfigMap`中的条目。给容器传递`ConfigMap`条目作为环境变量，如`fortune-pod-env-configmap.yaml`。设置环境变量`INTERVAL	`，用`ConfigMap`初始化不设置固定值，环境变量中的`key`设置为`sleep-interval`。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      valueFrom: 
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
```

一次性传递`ConfigMap`的所有条目作为环境变量，为每个条目单独设置环境变量的过程是单调乏味且容易出错的。在`kubernetes`的`1.6`版本提供了暴露`ConfigMap`的所有条目作为环境变量的手段。若需要将参数传递到`docker`容器内，可以通过`yaml`配置文件中设置`args: ["${INTERVAL}"]`。

使用`secret`给容器传递敏感数据：`kubernetes`提供了一种称为`secret`的单独资源对象。`secret`结构与`configMap`类似，均是键/值对的映射。`secret`的使用方法也与`configMap`相同，可以将`secret`条目作为环境变量传递给容器、将`secret`条目暴露给卷中的文件。

对于任意一个`pod`使用命令`kubectl describe pod`运行时，每个`pod`都会自动挂载上一个`secret`卷，这个卷引用的是前面`kubectl describe`输出中的一个叫做`default-token-bvhjx`的`secret`。由于`secret`也是资源对象，因此可以通过`kubectl get secrets`命令从`secret`列表中找到这个`default-token secret`。在`kubectl describe secrets`中包含三个条目——`ca.crt`、`namespace`与`token`，包含了从`pod`内部安全访问`kubernetes api`服务器所需的全部信息。

```shell
sam@elementoryos:~/kubernetes/kubernetes-service$ sudo kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
swagger-editor-z2fr6   1/1     Running   0          21s
sam@elementoryos:~/kubernetes/kubernetes-service$ sudo kubectl describe pod
Volumes:
  default-token-bvhjx:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-bvhjx
    Optional:    false
    
sam@elementoryos:~/kubernetes/kubernetes-service$ sudo kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-bvhjx   kubernetes.io/service-account-token   3      5m59s    
sam@elementoryos:~/kubernetes/kubernetes-service$ sudo kubectl describe secrets
Name:         default-token-bvhjx
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 6382d69c-21e6-4cdc-8193-417233ab5767
Type:  kubernetes.io/service-account-token
Data
====
ca.crt:     1066 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ij

sam@elementoryos:~/kubernetes/kubernetes-service$ sudo kubectl exec swagger-editor-z2fr6 ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt
namespace
token
```



使用`Downward API`访问`pod`的元数据以及其他资源、与`Kubernetes API`服务器交互：

> 通过环境变量或者`configMap`和`secret`卷向应用传递配置数据，这对于`pod`调度、运行前预设的数据是可行的。但是那些不能预先知道的数据，如`pod`的`ip`、主机名或者`pod`自身的名称，对于此类问题，可以通过使用`Kubernetes download API`解决，这种方式主要是将在`pod`的定义和状态中取的的数据作为环境变量和文件的值。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
```

在`downward-api-env.yaml`中，引用`pod manifest`中的元数据名称字段而不是设定一个具体的值。通过`valueFrom`中的`fieldPath`属性获取`spec.nodeName`元数据。在`yaml`文件中有引用`metadata.name`、`metadata.namespace`、`status.podIP`、`status.nodeName`字段值。可以使用`kubectl exec downward env`查看`pod`中的环境变量：

```shell
sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl create -f downward-api-env.yaml 
pod/downward created
sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl exec downward env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=downward
POD_IP=172.17.0.7
NODE_NAME=minikube
SERVICE_ACCOUNT=default
CONTAINER_CPU_REQUEST_MILLICORES=15
CONTAINER_MEMORY_LIMIT_KIBIBYTES=4096
POD_NAME=downward
POD_NAMESPACE=default
KUBIA_SERVICE_PORT=80
KUBIA_PORT=tcp://10.110.207.33:80
KUBIA_PORT_443_TCP_ADDR=10.110.207.33
```

如果更倾向于使用文件的方式而不是环境变量的方式暴露元数据，可以定义一个`downward API`卷并挂载到容器中，由于不能通过环境变量暴露，所以必须使用`downward API`卷来暴露`pod`标签或注解。与环境变量一样，需要显示地定义元器据字段来暴露份进程，我们将示例从使用环境变量修改为使用存储卷。

```yaml
...
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
```

在`downward-api-volume.yaml`文件中，现在并没有通过环境变量来传递元数据，而是定义了一个叫做`downward`的卷，并且通过`/etc/downward`目录挂载到我们的容器中。卷所包含的文件会通过卷定义中的`downwardAPI.items`属性来定义。若要在卷的定义中引用容器级的元数据，则需指定`containerName`属性的值为容器名称。

```shell
sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl exec downward ls /etc/downward
annotations
containerCpuRequestMilliCores
containerMemoryLimitBytes
labels
podName
podNamespace
sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl exec downward cat /etc/downward/labels
foo="bar"
sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl exec downward cat /etc/downward/annotations
key1="value1"
key2="multi\nline\nvalue\n"
kubernetes.io/config.seen="2019-12-01T15:08:21.544699469+08:00"
kubernetes.io/config.source="api" 
```

`Downward API`提供了一种简单的方式，将`pod`和容器的元数据传递给在它们内部运行的进程。通过`kubectl cluster-info`命令得到服务器的`Url`。因为服务器使用`https`协议并且需要授权，所以与服务器交互并不是一件简单的事情。可以尝试通过`curl`来访问它，使用`curl`的`--insecure`选项来跳过服务器证书检查环节。

```shell
kubernetes.io/config.source="api"sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl cluster-info
[sudo] password for sam:        
Kubernetes master is running at https://192.168.170.128:8443
CoreDNS is running at https://192.168.170.128:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
sam@elementoryos:~/kubernetes/downward-api$ sudo kubectl proxy
Starting to serve on 127.0.0.1:8001

sam@elementoryos:~/kubernetes$ curl localhost:8001
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io"
    ...
   ]
}
sam@elementoryos:~/kubernetes$ curl http://localhost:8001/apis/batch/v1/jobs
{
  "kind": "JobList",
  "apiVersion": "batch/v1",
  "metadata": {
    "selfLink": "/apis/batch/v1/jobs",
    "resourceVersion": "23398"
  },
  "items": []
}
```

在响应消息展示了包括可用版本，客户推荐使用版本在内的批量`api`组信息。`api`服务器返回了在`batch/v1`目录下`api`组中资源类型以及`rest ednpoint`清单。除了资源的名称和相关类型，`api`服务器也包含了一些其他信息，比如资源是否被指定了命名空间、名称简写、资源对应可以使用的动词列表等。`curl http://localhost:8001/apis/batch/v1/jobs`路径运行一个`GET`请求，可以获取集群中所有`Job`清单。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

可以通过`embassador`容器简化与`api`服务器的交互，为了通过操作理解`ambassador`容器模式。我们像之前创建`curl pod`一样创建一个新的`pod`，但这次不是仅仅在`pod`中运行单个容器，而是基于一个多用途的`kubectl-proxy`容器镜像来运行一个额外的`ambassador`容器，当`pod`启动后会同时启动`kubectl-proxy`和`curl`服务。

