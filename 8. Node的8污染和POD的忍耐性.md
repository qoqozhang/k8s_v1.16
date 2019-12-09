# Taints 和 tolerations作用

​		NodeAffinity是用来“吸引”pod在一组node上运行。Taints刚好相反，是用来排斥pod不能在这些node上运行。Taint 和 toleration 相互配合，toleration应用到pod上，表示pod可以被调到到具有taint的node节点上。

# 概念

给node添加 taint：

```
kubectl taint nodes node1 key=value:NoSchedule
```

给node1添加一个taint，这个taint 的key是`key`，值是`value`，然后taint的影响是`NoSchedule`，表示没有pod会被调度到这个node1节点上，除非pod有配置toleration。

给node移除一个taint：

```
kubectl taint nodes node1 key:NoSchedule-
```

在PodSpec文件`.spec.tolerations`里面添加tolertion配置，有两种格式：

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

当一个pod的toleration和node的taint匹配的话，key和effect相同，并且：

- operator 字段为`Exists`时，不需要设置`value`字段；
- operator 字段为`Equal`且`value`相同；
- operator的默认值是`Equal`

两种特殊情况：

- operator为Exists并且key为空的话，表示这个tolertion会匹配所有的taint；

  ```
  tolerations:
  - operator: "Exists"
  ```

- effect为空的话，表示会匹配所有key相同的taint的effect；

  ```
  tolerations:
  - key: "key"
  operator: "Exists"		
  ```

​        `effect` 的值可以为`NoSchedule` `PreferNoSchedule`  `NoExecute` 。`PreferNoSchedule`表示非硬性要求，会尝试满足这个taint。`NoExecute` 表示不允许pod运行在该node节点上，如果node上添加了这个taint以后，pod会被立即终止不能在此pod上运行，`NoExecute` 有一个可选字段 `tolerationSeconds` 控制node添加这个taint以后，pod还可以在这个node上运行多久，如果在时间达到之前这个taint又被删除了，则pod不会被终止。

可以在一个node上设置多个taint，也可以在一个pod上设置多个toleration。所有node的taint中，会先忽略pod的toleration匹配到的taint，然后剩下的taint的作用会有下面的几种情况：

- 剩余的taint中，如果有至少一个effect是`NoSchedule`，那么pod不会调度到这个node上；
- 剩余的taint中，如果有一个effect是`NoSchedule`，但是还有至少一个effect是`PreferNoSchedule` ，会尝试调用pod到这个node上；
- 剩余的taint中，如果有一个effect是`NoExecute` ，pod已经运行在node上的话也会被驱逐，如果pod没有运行的话也不会调度到这个node上；

# 使用场景案例

## 专用模式   Dedicated Nodes

​		如果你想让一些node作为专用模式，则可以在这些node上添加taint ``kubectl taint nodes nodename dedicated=groupName:NoSchedule`，然后在这些pod上添加toleration。这些添加了toleration的pod才可以在这些node上运行，这些node为这些pode专用。为了更好的体现专用模式，同时建议在这些node上添加类似于taint的标签，保证使用node affinity也让这些pod调度到这些node节点上，比如label `dedicated=groupName`。

## 有特殊硬件的node

​		如果集群里面的一小部分node节点有特殊的硬件，比如GPU。期望的状态是，不需要这些特殊硬件的pod不运行在这些node节点上。可以在node节点上添加taint `kubectl taint nodes nodename special=true:NoSchedule`  或者 `kubectl taint nodes nodename special=true:PreferNoSchedule`，然后在需要这些特殊硬件的pod上添加对应的toleration。

​		我们推荐使用 `Extended Resources` 来表示特殊硬件，给配置了特殊硬件的节点添加 taint 时包含 extended resource 名称，然后运行一个 `ExtendedResourceToleration`  admission controller。此时，因为节点已经被 taint 了，没有对应 toleration 的 Pod 会被调度到这些节点。但当你创建一个使用了 extended resource 的 Pod 时，`ExtendedResourceToleration` admission controller 会自动给 Pod 加上正确的 toleration ，这样 Pod 就会被自动调度到这些配置了特殊硬件件的节点上。这样就能够确保这些配置了特殊硬件的节点专门用于运行 需要使用这些硬件的 Pod，并且您无需手动给这些 Pod 添加 toleration。

## 基于 taint 的驱逐TaintBasedEvictions （alpha 特性）

​		这是一个当节点出现故障的时候，pod的预驱逐行为。

在v1.6的alpha支持node故障的一些情况。换句话说，node controller会自动在node上根据一些条件添加taint，下面是几种内置的的taint：

- `node.kubernetes.io/not-read`  node还为准备好，对应的node的Ready状态是False；
- `node.kubernetes.io/unreachable`  从node controller无法和node通信，对应的node的Ready状态是Unknown；
- `node.kubernetes.io/out-of-disk` node的磁盘满了；
- `node.kubernetes.io/memory-pressure`  node的内存有压力；
- `node.kubernetes.io/disk-pressure` node的磁盘有压力；
- `node.kubernetes.io/network-unavailable`  node网络不可达；
- `node.kubernetes.io/unschedulable`  node不能调度；
- `node.cloudprovider.kubernetes.io/uninitialized`  当kubelet开启“external” cloud provider，这个taint就会设置在node上表示node不可用。当cloud-controller-manager 初始化这个node完成以后，kubelet会移除此taint；

​	     从v1.13版本开始，`TaintBasedEvictions` 特性提升为beta功能，并默认开启。kubelet会根据情况添加这些taint，并会从node上驱逐pod调度过来。

​		为了维护pod被驱逐的比率，可以在node controller设置[rate-limited](https://kubernetes.io/docs/concepts/architecture/nodes/#node-capacity) ，可以控制pod被大面积驱动的情况，比如master分裂的情况下。

​		当没有手动为toleration 中 Effect为`NoExecute` 设置可选参数``tolerationSeconds`的时候，`node.kubernetes.io/not-ready` 会自动添加`tolerationSeconds=300` ;相同的，`node.kubernetes.io/unreachable` 也会自动添加`tolerationSeconds=300` 。DaemonSet 的pod并不会添加这个参数，这保证了出现上述问题时 DaemonSet 中的 pod 永远不会被驱逐，这和`TaintBasedEvictions` 特性被禁用以后的结果是一样的。

## 有条件的Taint Node

​		在v1.12版本中，`TaintNodesByCondition` 特性被提升为beta功能，所以node生命周期控制会根据node的状态自动添加taint。scheduler并不会检查node的状态，但是会检查node的taint，这表示node的状态并不会影响到scheduler对pod的调度。通过pod上的toleration可以选择性的忽略一些node的问题，`TaintNodesByCondition` 向node中添加的taint的effect都是`NoSchedule` 。从v1.13版本开始，`TaintNodesByCondition` 控制的`NoEffect`  默认启动状态。

从v1.8版本开始，DaemonSet 会自动添加 `NoSchedule` toleration到所有的pod中：

- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/out-of-disk` (only for critical pods)
- `node.kubernetes.io/unschedulabl` (从1.10开始)
- `node.kubernetes.io/network-unavailable` (host network only)

添加这些toleration可以确保向上的兼容性，你可以自由的在DaemonSet中添加任意的tolertaion。