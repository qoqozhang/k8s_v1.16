---
title: pod概念
date: 2019-11-25 16:46
tags: 
  - 
categories: 
  - 
---
# Pods
## 什么是pods？
pods是k8s的最小可执行单元，pods就像集群的一个进程一样，每一个pods都是一个应用实例。
pod包含容器(一个或多个)，存储，唯一的网络IP等一系列容器运行需要的资源
## pods的两个主要用途
pods运行单一的容器，单容器的pods表示在pod里面仅运行一个容器。
pods运行多个容器共同工作，多个容器共享资源提供同一个服务。
## pods怎么管理多个容器
![](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)
多容器的pod环境里面，这些容器共用**网络**和**存储**。
### 网络
    每一个pod都有一个唯一的网络IP地址，同一个pod里面的这些容器共用网络namespace，包括IP和端口，pod内的容器可以试用`localhost`访问其他容器。
### 存储
    每一个pod都可以使用共享的存储卷volume。同一个pod里面的容器共享volume，允许这些容器访问volume里面的数据。使用卷volume可以永久保存一个pod里面的数据。
## pod的状态
### status.phase 
`phase` 综合检测pod状态的概要。
`phase` 可能的几种状态：
- **Pending**：一个容器已经被k8是执行创建，但是当一个或多个容器的镜像还未下载创建好的状态；
- **Running**：当pod里面的容器已经都被创建好，且至少有一个容器已经是在运行中；
- **Succeeded**：pod里面的容器都已经在终止状态，且没有容器重启；
- **Failed**：pod里的容器都已经终止，但是至少有一个容器是异常退出的状态。就是说，容器的退出状态码为非0的情况；
- **Unknown**：无法获取pod的状态的情况；
### pod的状态里面的几个时间戳
- `lastProbeTime`: 最近一次获取pod状态的时间
- `lastTransitionTime`：最近一次pod状态更改的时间
- `message`：最近一次状态改变的消息
- `reason`：一个驼峰写法单词总结状态改变的原因
- `status`：“True”, “False”, “Unknown”
- `type`：
       - `PodScheduled`：pod已经被调度到node节点上；
       - `Ready`：pod已经处在可以处理请求的阶段，可以添加到Load Balance 池中；
       - `Initialized`：所有 *init containers* 初始化已经完成；
       - `Unschedulable`：集群scheduler 不能立即调度pod，比如缺少运行pod需要的资源；
       - `ContainersReady`：pod内的所有容器已经准备好；
## 容器状态探测
kubelet会周期性的检测容器的状态。
### 通过下面的三种方式探测：
- `ExecAction`：在容器内执行一个命令，检测命令的结束码是否为0；
- `TCPSocketAction`：TCP检测容器的端口是否open；
- `HTTPGetAction`：执行HTTP get请求，检测返回的状态是否大于等于200，小于400；
### 探测结果
Success，Failure，Unknown
### 可选择的探测方式
- `livenessProbe`：检测容器是否运行。如果Failure，会删除这个容器，然后会根据 restart policy 来进行后续动作。如果没启用，默认是 Success；
- `readinessProbe`：检测容器是否进入处理请求的状态。如果Failure，endpoints controller 会从pod上移除pod的IP地址，如果没启用，默认是 Success；
- `startupProbe`： 检测 *app containers* 是否启动。在startprobe 状态非succeed的时候，所有其他的probe都是不启用状态。如果 Failure，kubelet删除这个容器，然后根据restart policy 进行后续动作。如果没启用，默认是Success。
###什么是 init container？
pod包含多个运行业务的app containers 也包含多个init containers。在app containers运行之前会先启动这些init containers。
在pod配置文件里面添加`initContainers`字段，提供一个列表来配置app containers，运行的时候查看pod的`.status.initContainerStatuses`字段提供了init containers的状态。
只有 app containers都启动完成以后才会开始启动app containers。
## pod重启的原因
1. 用户提交了更新pod的操作；
2. pod内的容器的runtime发生了重启，不是很常见，这些操作由对node节点有root权限的人操作；
3. pod内的所有容器都终止了，并且 `restartPolicy` 设置了 Always。
## pod的拓扑限制规则  `topology spread constraints`
当你的cluster的node节点跨多个地区，多个区域的时候，可以使用 `topology spread constraints` 控制pod的分布情况，比如阿里云 腾讯云上部署cluster，可以控制一个应用的所有pod分布在不同的zone。
### 开启 `topology spread constraints`
1. 在kubelet启动的时候添加参数`--feature-gates="EvenPodsSpread=true"`
2. 给node节点添加label标签，例如 node=node1,zone=us-east-1a,region=us-east-1
###  配置 `topology spread constraints`
在 `pod.spec.topologySpreadConstraints` 字段配置，有下面几个可配置参数：
- maxSkew：在某一个 label 上最大允许的不平衡数量，必须大于0；
- topologyKey：选择存在的node上的 label，用于匹配规则使用；
- whenUnsatisfiable： 当不平衡的时候的动作
    - DoNotSchedule：默认配置，告诉scheduler 不要调度到这个 lable匹配的node上；
    - ScheduleAnyway：即使不平衡仍然调度到node上；
- labelSelector：值为pod上的label，用于平衡规则使用；
### 单topology 示例
第四个pod的部署文件：
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```
如果在是阿里云的两个zone区域，北京和上海分别部署了两个node节点组成了一个cluster，然后`labelSelector`匹配了三个pod，两个pod运行在zone=Beijing的节点上，一个pod运行在zone=Shanghai的节点，当第四个pod部署的时候，pod就只能运行在zone=Shanghai 的节点。结果是：
```yaml
+---------------+---------------+      +---------------+---------------+
|     zoneA     |     zoneB     |      |     zoneA     |     zoneB     |
+-------+-------+-------+-------+      +-------+-------+-------+-------+
| node1 | node2 | node3 | node4 |  OR  | node1 | node2 | node3 | node4 |
+-------+-------+-------+-------+      +-------+-------+-------+-------+
|   P   |   P   |   P   |   P   |      |   P   |   P   |  P P  |       |
+-------+-------+-------+-------+      +-------+-------+-------+-------+
```
如果maxSkew=2，pod也可以运行在zone=Beijing的节点上。
### 多topology示例
上面例子使用zone做了平衡，也可以使用node来做平衡，这样匹配完zone以后就会继续匹配node解节点，比如上面示例出现的node3上运行多个pod的情况。
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  - maxSkew: 1
    topologyKey: node
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```
结果是：
```yaml
+---------------+-------+
|     zoneA     | zoneB |
+-------+-------+-------+
| node1 | node2 |  nod3 |
+-------+-------+-------+
|  P P  |   P   |  P P  |
+-------+-------+-------+
```
## 什么是Ephemeral Containers？
 在pod创建完成以后是没办法正常的操作容器的，当有这种情况的时候就需要使用Ephemeral Containers，常用于容器排错或者获取容器的一些信息。
1. Ephemeral Containers没有端口；
2. Ephemeral Containers不会自动重启，所以不使用创建应用‘
3. Ephemeral Containers分配的资源不能改变；
