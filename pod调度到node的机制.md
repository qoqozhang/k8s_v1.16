---
title: pod调度到node的机制
date: 2019-12-05 13:08
tags: 
  - 
categories: 
  - 
---
>  你可能会遇到一些pod只能在一部分node上运行的情况,比如这个pod需要运行在SSD的node节点上这种情况，推荐使用`selector`选择器来选择pod的运行node。
# nodeSelector
在pod上使用`nodeSelector`来选择想要运行的node节点，必须是node上有的标签，不然会出现选择不到node的情况
1. 在node上添加标签 `kubectl label nodes <node-name> <label-key>=<label-value>`
2. 在pod上添加`nodeselector`，`.spec.container[*].nodeSelector`
# node内置标签
- `kubernetes.io/hostname`
- `failure-domain.beta.kubernetes.io/zone`
- `failure-domain.beta.kubernetes.io/region`
- `beta.kubernetes.io/instance-type`
- `kubernetes.io/os`
- `kubernetes.io/arch`
#  节点隔离和限制
当使用labels来选择pod可以运行的一组node节点的时候，这些被选择使用的labels 不建议使用kubelet程序来修改node上的标签。
但是`NodeRestriction` 插件允许kubelet 来修改设置以`node-restriction.kubernetes.io/`作为前缀的标签
# 亲和性和反亲和性
`nodeSelector` 的一些特点：
1. 支持更多表达式，不仅仅是AND 这种精确匹配
2. 也可以设置为 非硬性要求，如果调度没有匹配到，pod也会调度过去
3. 可以约束节点(或其他拓扑域)上运行的其他pods上的标签，而不是约束节点本身上的标签，这允许关于哪些pods可以和哪些pod不能共存的规则
## 节点亲和性 Node affinity
​		使用`nodeSelector`可以限制你的pod可以运行在哪些节点上，这里有两种类型的 node affinity，`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`.`requiredDuringSchedulingIgnoredDuringExecution` 必须符合条件才会调度到节点上，和`nodeSelector`类似，但是规则更高级，`preferredDuringSchedulingIgnoredDuringExecution`会尝试强制调度，但是不保证成功以后会调度到非匹配的node节点上。`IgnoredDuringExecution`这部分表示运行期间，如果node匹配的标签发生了变化，pod仍然会运行在原来的node节点上。`requiredDuringSchedulingRequiredDuringExecution`表示在pod运行期间，标签发生改变也会重新调度。

- operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`;

- `nodeSelector`和`nodeAffinity` 都存在，必须同时满足才会调度;

- `nodeAffinity` 提供了多个`nodeSelectorTerms`，则满足其中的一个`nodeSelectorTerms`就可以;

- `nodeSelectorTerms`提供了多个`matchExpressions`,则需要满足所有的`matchExpressions`;

- 如果修改或者删除了pod调度使用的node标签，pod也不会移除，也就是说 affinity的selector 只有在调度pod的时候才有效果;
  `weight`的范围是1-100，表示当一个node节点匹配到一条规则以后，给该node的权重增加多少，最后会在所有node节点里面选择一个权重最高的node节点运行pod。

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: with-node-affinity
  spec:
    affinity:
      nodeAffinity:		#基于node标签的规则
        requiredDuringSchedulingIgnoredDuringExecution:	#规则1，强制要求node满足规则
          nodeSelectorTerms:
          - matchExpressions:
            - key: kubernetes.io/e2e-az-name
              operator: In
              values:
              - e2e-az1
              - e2e-az2
        preferredDuringSchedulingIgnoredDuringExecution:	#规则2，优先满足，不满足则调度其他node上
        - weight: 1
          preference:
            matchExpressions:
            - key: another-node-label-key
              operator: In
              values:
              - another-node-label-value
    containers:
    - name: with-node-affinity
      image: k8s.gcr.io/pause:2.0
  ```

## POD间的亲和性和反亲和性 Inter-pod affinity and anti-affinity

​		pod间的亲和性和反亲和性，定义规则的时候是根据已经运行的pod的标签来操作的，和`Nodeaffinity`不同的是标签是规则根据node的标签来定义的。

​		在定义的时候，必须定义selector的namespace范围，使用`topolopyKey`来定义，值是一个定义在node节点上的用来定义拓扑域的label标签。

​		这种标签用在大型的集群中会导致调度速度下降。不建议使用在几百台node的集群中。

​		要求在集群内的所有node节点的标签都要匹配`topologyKey`，如果有几个或者所有node的标签不满足，会出现意外的行为。

​		`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`和Nodeaffinity的一样，分为soft和hard两种。

- `requiredDuringSchedulingIgnoredDuringExecution` ，不允许`topologyKey` 为空；

- `requiredDuringSchedulingIgnoredDuringExecution`，`LimitPodHardAntiAffinityTopology`控制器用于限制`topologyKey`只能是`kubernetes.io/hostname`;

- `preferredDuringSchedulingIgnoredDuringExecution`，空`topologyKey`被解读成`kubernetes.io/hostname`, `failure-domain.beta.kubernetes.io/zone` and `failure-domain.beta.kubernetes.io/region`的组合;

- 除了上面几种情况，`topologyKey`可以是任何正确的标签；
### 更多使用场景
  结合各种资源RS，StatefulSet，Deployment 等等可以实现一些更高级的功能。
#### 协同调度到相同节点上
##### 场景一：用podAntiAffinity调度redis的pod副本
为了redis能更好的使用内存，想要redis尽可能的能分配在不同的节点上。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3		#3个副本
  template:			#template 定义标签是app=store
    metadata:
      labels:
        app: store
    spec:
      affinity:		#亲和性
        podAntiAffinity:		#pod间反亲和性，定义pod不能运行在已经有pod的标签为app=store的节点
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

##### 场景二：podAntiAffinity和podAffinity 定义nginx对redis的粘滞调度

需要保证nginx的pod调度在不同的节点上，同时要满足该节点上需要有redis在运行。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3		#定义3副本
  template:			#定义pod的template
    metadata:
      labels:
        app: web-store
    spec:
      affinity:		
        podAntiAffinity:	#pod反亲和性，定义nginx副本之间互斥
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:		#pod亲和性，定义节点上必须有运行redis副本
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

最终的结果是，redis的副本都分别运行在不同的节点上，同时该节点上都有唯一一个nginx 的pod在运行。

## nodeName

​		`nodeName` 是一种简单的node 选择机制，但是不建议使用。`nodeName`是PodSpec里面的一个字段，如果这个字段非空，则调度器scheduler会忽略这个pod，kubelet会直接运行这个pod。在PodSpec上如果有定义`nodeName`，则优先级高于上面的几种调度规则。

- 如果`nodeName` 定义的node不存在，这pod不会运行；

- 如果node没有足够的资源运行pod，这pod会运行失败，原因是OutOfmemory 或者 OutOfcpu；

- 节点名称在云环境里面可能不是固定的或者不可预见（预定义）；

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    - name: nginx
      image: nginx
    nodeName: kube-01		#该pod只能运行在node名称为kube-01的节点上
  ```