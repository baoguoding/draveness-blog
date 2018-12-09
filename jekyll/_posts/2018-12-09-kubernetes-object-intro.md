---
layout: post
title: 从 Kubernetes 中的对象谈起
permalink: /kubernetes-object-intro
tags: kubernetes 微服务 容器编排
toc: true
desc: 上一篇文章中，我们其实介绍了 Kubernetes 的对象其实就是系统中持久化的实体，Kubernetes 用这些实体来表示集群中的状态，它们描述了集群中运行的容器化应用以及这些对象占用的资源和行为。当我们想要了解 Kubernetes 的实现原理时，绕不开的其实就是 Kubernetes 中的对象，而在 Kubernetes 中，规格（Spec）和状态（Status）是用于描述 Kubernetes 对象的两个最重要的嵌套对象，在这篇文章中会重点介绍对象的规格和状态的使用方式和实现原理。
---

上一篇文章中，我们其实介绍了 Kubernetes 的对象其实就是系统中持久化的实体，Kubernetes 用这些实体来表示集群中的状态，它们描述了集群中运行的容器化应用以及这些对象占用的资源和行为。

![kubernetes-object](https://img.draveness.me/2018-12-09-kubernetes-object.png)

不过当我们想要了解 Kubernetes 的实现原理时，绕不开的其实就是 Kubernetes 中的对象，而在 Kubernetes 中，规格（Spec）和状态（Status）是用于描述 Kubernetes 对象的两个最重要的嵌套对象，在这篇文章中会重点介绍对象的规格和状态的使用方式和实现原理。

## 简介

在真正展开介绍对象的规格和状态之前，我们首先需要介绍 Kubernetes 中所有对象都有的三个字段 `apiVersion`、`kind` 和 `metadata`，我们从一个常见的对象描述文件来展开介绍：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  # ...
```

### API 组和类型

`apiVersion` 和 `kind` 共同决定了当前的 YAML 配置文件应该由谁来进行处理，前者表示描述文件使用的 API 组，后者表示一个 API 组中的一个资源类型，这里的 `v1` 和 `Pod` 表示的就是核心 API 中组 `api/v1` 中的 `Pod` 类型对象。

除了一些 Kubernetes 的核心 API 组和资源之外，还有一些 Kubernetes 官方提供的扩展 API 组 `apis/batch/v1`、`apis/extensions/v1beta1` 等等，除此之外，我们也可以通过 `CustomResourceDefinition` 或者实现 apiserver 来定义新的对象类型。

### 元数据

`apiVersion` 和 `kind` 描述了当前对象的一些最根本信息，而 `metadata` 能够为我们提供一些用于唯一识别对象的数据，包括在虚拟集群 `namespace` 中唯一的 `name` 字段，用于组织和分类的 `labels` 以及用于扩展功能的注解 `annotations`。

```go
type ObjectMeta struct {
	Name string
	Namespace string
	Labels map[string]string
	Annotations map[string]string
	// ...
}
```

上述的结构体嵌入在 Kubernetes 的每一个对象中，为所有的对象提供类似命名、命名空间、标签和注解等最基本的支持，让开发者能够更好地管理 Kubernetes 集群。

#### 标签和选择器

每一个 Kubernetes 对象都可以被打上多个标签，这些标签可以作为过滤项帮助我们选择和管理集群内部的 Kubernetes 对象，下面的命令可以获取到生产环境中的所有前端项目：

```bash
kubectl get pods -l environment=production,tier=frontend
```

除了使用 kubectl 直接与 Kubernetes 集群通信获取其中的对象信息之外，我们也可以在 YAML 文件中使用选择器来达到相同的效果：

```yaml
selector:
  matchLabels:
    environment: production
    tier: frontend
```

标签的主要作用就是对 Kubernetes 对象进行分类，这里我们可以简单给一些常见的分类方法：

![kubernetes-labels](https://img.draveness.me/2018-12-09-kubernetes-labels.png)

这些标签能够帮助我们在复杂的集群中快速选择一系列的 Kubernetes 对象，用好标签能够为管理集群带来非常大的帮助。

#### 命名空间

Kubernetes 支持通过命名空间在一个物理集群中划分出多个虚拟集群，这些虚拟集群就是单独的命名空间。不同的命名空间可以对资源进行隔离，它们没有办法直接使用 `name` 访问其他命名空间的服务，而是需要使用 FQDN(fully qualified domain name)。

![kubernetes-namespace](https://img.draveness.me/2018-12-09-kubernetes-namespace.png)

也就是说当我们创建一个 Service 时，它实际上在集群中加入了类似 `<service>.<namespace>.svc.cluster.local` 的 DNS 记录，在同一个命名空间中，我们可以直接使用 `service` 来访问目标的服务，但是在访问其他命名空间中的服务时却没有办法这么做。

### 对象接口

Kubernetes 中的对象其实就是并不是 `struct`，而是一个 `interface`，其中定义了一系列的 Getter/Setter 接口：

```go
type Object interface {
	GetNamespace() string
	SetNamespace(namespace string)
	GetName() string
	SetName(name string)
	GetGenerateName() string
	SetGenerateName(name string)
	GetUID() types.UID
	SetUID(uid types.UID)
	// ...
}
```

这些 Getter/Setter 接口获取的字段基本都是 `ObjectMeta` 结构体中定义的一些字段，这也是为什么 Kubernetes 对象都需要嵌入一个 `ObjectMeta` 结构体。

## Spec

对象的规格（Spec）描述了某一个实体的期望状态，每一个对象的 Spec 基本都是由开发或者维护当前对象的工程师指定的，以下面的 busybox 举例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    app: busybox
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

作为一个 Pod 对象，它其实就是一个在 Kubernetes 中运行的最小、最简单的单元，所以它的 Spec 声明的就是其中包含的容器以及容器的镜像、启动命令等信息。

但是另一种对象 Service 就有着完全不同的 Spec 参数，作为一个一组 Pod 访问方式的抽象，它需要指定流量转发的 Pod 以及目前的端口号：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

我们可以看出，不同的 Kubernetes 对象基本上有着完全不同的 Spec，接下来我们按照 Kubernetes 项目中的源代码分别介绍如何描述几种不同的 Kubernetes 对象。

### Pod

作为一个 Kubernetes 对象，结构体 `Pod` 中嵌入了 `metav1.TypeMeta` 和 `metav1.ObjectMeta` 两个结构，这两个结构体中包含了 `Object` 接口中需要的函数：

```go
type Pod struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type PodSpec struct {
	InitContainers []Container `json:"initContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,20,rep,name=initContainers"`
	Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"`
	RestartPolicy RestartPolicy `json:"restartPolicy,omitempty" protobuf:"bytes,3,opt,name=restartPolicy,casttype=RestartPolicy"`
	// ...
}
```

`Pod` 结构体中的 `PodSpec` 就是我们在 YAML 文件中定义的嵌套对象了，由于该结构体非常复杂（加上注释有 180 行），在这一节中我们只会简单介绍其中的几个字段。

![kubernetes-pod-containers](https://img.draveness.me/2018-12-09-kubernetes-pod-containers.png)

`InitContainers` 是当前 Pod 启动时需要首先执行的一系列容器，这些容器没有生命周期，也没有探针，它们的主要作用就是在容器启动时进行一些资源和依赖的初始化配置；接下来的 `Containers` 数组就是 Pod 正常运行时包含的一系列容器了，这些容器会共享网络和进程，运行在同一个『虚拟机』上，所以也可以相互通信。

最后的 `RestartPolicy` 其实就整个 Pod 的重启策略，我们可以选择不重启 `Never`、在出现错误时重启 `OnFailure` 或者总是重启 `Always`。

### Service

Kubernetes 中另一个常见对象 Service 的规格就有很大的不同了，虽然他们两者有着完全相同的嵌入结构 `metav1.TypeMeta` 和 `metav1.ObjectMeta` 以及字段 `Spec` 和 `Status`，但是它们的规格和状态却完全不同：

```go
type Service struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	Spec ServiceSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	Status ServiceStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type ServiceSpec struct {
	Ports []ServicePort `json:"ports,omitempty" patchStrategy:"merge" patchMergeKey:"port" protobuf:"bytes,1,rep,name=ports"`
	Selector map[string]string `json:"selector,omitempty" protobuf:"bytes,2,rep,name=selector"`
	// ...
}
```

Service 都会通过选择器 `Selector` 将流量导入对应的 Pod 的指定 `Ports` 端口，这两个字段也是使用 Service 时最常用的两个字段，前者能够根据 Pod 的标签选择 Service 背后的一组 Pod，而 `Ports` 会将端口的流量转发到目标 Pod 的指定端口上。

### 小结

Kubernetes 中对象的 Spec 其实描述了对象的期望状态，也就是工程师直接指定运行在 Kubernetes 集群中对象的表现和行为，同时 Kubernetes 会通过控制器不断帮助对象向期望状态迁移。

## Status

对于很多使用 Kubernetes 的工程师来说，它们都会对对象的 Spec 比较了解，但是很多人都不太会了解对象的状态（Status）；对象的 Spec 是工程师向 Kubernetes 描述期望的集群状态，而 Status 其实就是 Kubernetes 集群对外暴露集群内对象运行状态的一个接口：

```yaml
apiVersion: v1
kind: Pod
metadata:
  // ...
spec:
  // ...
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2018-12-09T02:40:37Z
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: 2018-12-09T02:40:38Z
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: 2018-12-09T02:40:33Z
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://99f668a89db97342d7bd603471dfad5be262d7708b48cb6c5c8e374e9a13cf4f
    image: busybox:latest
    imageID: docker-pullable://busybox@sha256:915f390a8912e16d4beb8689720a17348f3f6d1a7b659697df850ab625ea29d5
    lastState: {}
    name: busybox
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: 2018-12-09T02:40:37Z
  hostIP: 10.128.0.18
  phase: Running
  podIP: 10.4.0.28
  qosClass: Burstable
  startTime: 2018-12-09T02:40:33Z
```

当我们将对象运行到 Kubernetes 集群中时，Kubernetes 会将 Pod 的运行信息展示到 Status 上，接下来我们分别介绍 Pod 和 Service 的 Status 都包含哪些数据。

### Pod

每一个 Pod 的 Status 其实包含了阶段、当前服务状态、宿主机和 Pod IP 地址以及其中内部所有容器的状态信息：

```go
type PodStatus struct {
	Phase PodPhase `json:"phase,omitempty" protobuf:"bytes,1,opt,name=phase,casttype=PodPhase"`
	Conditions []PodCondition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,2,rep,name=conditions"`
	Message string `json:"message,omitempty" protobuf:"bytes,3,opt,name=message"`
	Reason string `json:"reason,omitempty" protobuf:"bytes,4,opt,name=reason"`
	HostIP string `json:"hostIP,omitempty" protobuf:"bytes,5,opt,name=hostIP"`
	PodIP string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`
	StartTime *metav1.Time `json:"startTime,omitempty" protobuf:"bytes,7,opt,name=startTime"`
	InitContainerStatuses []ContainerStatus `json:"initContainerStatuses,omitempty" protobuf:"bytes,10,rep,name=initContainerStatuses"`
	ContainerStatuses []ContainerStatus `json:"containerStatuses,omitempty" protobuf:"bytes,8,rep,name=containerStatuses"`
	// ...
}
```

上述的信息中，`PodCondition` 数组包含了一系列关于当前 Pod 状态的详情，其中包含了 Pod 处于当前状态的类型和原因以及 Kubernetes 获取该信息的时间；而另一个比较重要的 `ContainerStatus` 结构体中包含了容器的镜像、重启次数等信息。

### Service

Service 的状态其实就更加的简单了，只有在当前的 Service 类型是 LoadBalancer 的时候 `Status` 字段才不会为空：

```go
type ServiceStatus struct {
	LoadBalancer LoadBalancerStatus `json:"loadBalancer,omitempty" protobuf:"bytes,1,opt,name=loadBalancer"`
}

type LoadBalancerStatus struct {
	Ingress []LoadBalancerIngress `json:"ingress,omitempty" protobuf:"bytes,1,rep,name=ingress"`
}

type LoadBalancerIngress struct {
	IP string `json:"ip,omitempty" protobuf:"bytes,1,opt,name=ip"`
	Hostname string `json:"hostname,omitempty" protobuf:"bytes,2,opt,name=hostname"`
}
```

其中可能会包含为当前负载均衡分配的 IP 地址或者 Hostname，并不会包含更加复杂的数据了。

### 小结

Kubernetes 对象的 Status 不仅能够用来观察目前集群中对象的运行状态，还能帮助我们对集群中出现的问题进行排查以及修复，并能提供一些信息辅助优化集群中资源的利用率，我们在使用 Kubernetes 对象时也应该多多关注集群内的对象 Status 字段。

## 总结

一个个 Kubernetes 对象组成了 Kubernetes 集群的期望状态，集群中的控制器会不断获取集群的运行状态与期望状态进行对比，保证集群向期望状态进行迁移，在接下来的文章中，我们会继续介绍 Kubernetes 集群是如何对常见的 Kubernetes 对象进行管理的。

## 相关文章

{% include related/distributed-system.md %}

## Reference

+ [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
+ [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/)
+ [API Conventions · Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md#objects)
