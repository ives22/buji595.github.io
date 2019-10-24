---
title: Kubernetes 系统基础 
date: 2019-10-24 17:20:22
tags: Kubernetes
categories: Kubernetes
copyright: true
---

# Kubernetes介绍

## 什么是Kubernetes？

> `Kubernetes`是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。
>
> 使用`Kubernetes`可以：
>
> - 自动化容器的部署和复制
> - 随时扩展或收缩容器规模
> - 将容器组织成组，并且提供容器间的负载均衡
> - 很容易地升级应用程序容器的新版本
> - 节省资源，优化硬件资源的使用
> - 提供容器弹性，如果容器失效就替换它，等等...



## Kubernetes特点

> - **便携性**：支持公有云、私有云、混合云、多重云(multi-cloud)
> - **可扩展**：模块化、插件化、可组合、可挂载
> - **自修复**：自动部署，自动重启，自动复制，自动伸缩扩展



##Kubernetes特性

> Kubernetes是一种用于在一组主机上运行和协同容器化应用程序的系统，旨在提供可预测性、可扩展性与高可用的性的方法来完全管理容器化应用程序和服务的生命周期的平台。
>
> 它具有以下几个重要的特性：
>
> 1. **自动装箱**：构建于容器之上，基于资源依赖及其他约束自动完成容器部署且不影响其可用性，并通过调度机制混合关键型应用和非关键型应用的工作负载于同一节点以提升资源利用率。
> 2. **自我修复**：支持容器故障后自动重启、节点故障候重行调度容器，以及其他可用节点、健康状态检查失败后关闭容器并重新创建等自我修复机制。
> 3. **水平扩展**：支持通过简单命令或UI手动水平扩展，以及基于CPU等资源负载率的自动水平扩展机制。
> 4. **服务器发现和负载均衡**：Kubernetes通过其附加组件之一的KubeDNS（或CoreDNS）为系统内置了服务发现功能，它会为每个Service配置DNS名称，并允许集群内的客户端直接使用此名称发出访问请求，而Service则通过iptables或ipvs内建了负载均衡机制。
> 5. **自动发布和回滚**：Kubernetes支持“灰度”更新应用程序或其配置信息，它会监控更新过程中应用程序的健康状态，以确保它不会在同一时刻杀掉所有的实例，而此过程中一旦有故障发生，就会立即自动执行回滚操作。
> 6. **秘钥和配置管理**：Kubernetes允许存储和管理敏感信息，例如密码，Oauth令牌和ssh秘钥。可以部署和更新密码和应用程序配置，而无需重建容器，也不会再堆栈配置中暴露机密。
> 7. **存储编排**：Kubernetes支持Pod对象按需自动挂载不同类型的存储系统，这包括节点本地存储、公有云服务商的云存储（如AWS和GCP等），以及网络存储系统（例如，NFS、ISCSI、GlusterFS、Ceph、Cinder和Flocker等）
> 8. **批量处理执行**：除了服务型应用，Kubernetes还支持批处理作业及CI（持续集成），如果需要，一样可以实现容器故障后修复。



# Kubernetes概述和术语

> Kubernetes使用共享网络将多个物理机或虚拟机汇集到一个集群中，在各服务器之间进行通信，该集群是配置Kubernetes的所有组件、功能和工作负载的物理平台。集群中一台服务器（或高可用部署中的一组服务器）用作Master，负责管理整个集群，余下的其他机器用作Worker Node,它们是使用本地和外部资源接收和运行工作负载的服务器。集群中的这些主机可以是物理服务器，也可以是虚拟机（包括IaaS云端的VPS）


![1]((一)Kubernetes系统基础/kubernetescluster.png)

**Master**

> Master是集群的网关和中枢，负责诸如为用户和客户端暴露API、跟踪其它服务器的健康状态、以最优方式调度工作负载，以及编排其他组件之间的通信等任务，它是用户或客户端与集群之间的核心联络点，并负责Kubernetes系统的大多数集中式管控逻辑。单个Master节点即可完成其所有的功能，但出于冗余及负载均衡等目的，生产环境中通常需要协同部署多个此类主机。Master节点类似于蜂群中的蜂王。



**Node**

> Node是Kubernetes集群的工作节点，负责接收来自Master的工作指令并根据指令相应的创建或删除Pod对象，以及调整网络规则以合理地路由和转发流量等。理论上讲，Node可以是任何形式的计算设备，不过Master会统一将其抽象为Node对象进行管理。Node类似于蜂群中的工蜂，生产环境中，它们通常数量众多。



----

> Kubernetes将所有Node的资源集结于一处形成一台更强大的“服务器”，如下图，在用户将应用部署于其上时，Master会使用调度算法将其自动指派某个特定的Node运行，在Node加入集群或从集群中移除时，Master也会按需重行编排影响到的Pod（容器）。于是，用户无需关心其应用究竟运行于何处。



![cluster]((一)Kubernetes系统基础/cluster.png)

> 从抽象的角度来讲，Kubernetes还有着众多的组件来支撑其内部的业务逻辑，包括运行应用、应用编排、服务暴露、应用恢复等，它们在Kubernetes中被抽象为Pod、Service、Controller等资源类型。



## 常用的资源对象

[十分钟带你理解Kubernetes核心概念](http://www.dockone.io/article/932)

（1）Pod

> Kubernetes并不直接运行容器，而是使用一个抽象的资源对象来封装一个或者多个容器，这个抽象即为Pod，它是Kubernetes的最小调度单元。同一Pod中的容器共享网络名称空间和存储资源，这些容器可经由本地回环接口lo直接通信，但彼此之间又在Mount、User及PID等名称空间上保持了隔离。尽管Pod中可以包含多个容器，但是作为最小调度单元，它应该尽可能地保持“小”，即通常只应该包含一个主容器，以及必要的辅助型容器（sidecar）

![]((一)Kubernetes系统基础/pod.png)

（2）资源标签

> 标签（Label）是将资源进行分类的标识符，资源标签其实就是一个键值型（key/values）数据。标签旨在指定对象（如Pod等）辨识性的属性，这些属性仅对用户存在特定的意义，对Kubernetes集群来说并不直接表达核心系统语意。标签可以在对象创建时附加其上，并能够在创建后的任意时间进行添加和修改。一个对象可以拥有多个标签，一个标签也可以附加于多个对象（通常是同一类对象）之上。

![]((一)Kubernetes系统基础/label.png)

（3）标签选择器

> 标签选择器（Selector）全称为”Label Selector“，它是一种根据Label来过滤符合条件的资源对象的机制。例如，将附有标签”role：backend“的所有Pod对象挑选出来归为一组就是标签选择器的一种应用，如下图所示，通常使用标签对资源对象进行分类，而后使用标签选择器挑选出它们，例如将其创建未某Service的端点。

![]((一)Kubernetes系统基础/labels.png)

（4）Pod控制器

> 尽管Pod是kubernetes的最小调度单元，但用户通常并不会直接部署及管理Pod对象，而是要借助于另一类抽象——控制器（Controller）对其进行管理。用于工作负载的控制器是一种管理Pod生命周期的资源抽象，它们是kubernetes上的一类对象，而非单个资源对象，包括ReplicationController，ReplicaSet、Deployment、StatefulSet、Job等。已下图所示的Deployment控制器为例，它负责确保指定的Pod对象的副本数量精确符合定义，否则“多退少补”。使用控制器之后就不再需要手动管理Pod对象了，只需要声明应用的期望状态，控制器就会自动对其进行进程管理。	

![]((一)Kubernetes系统基础/deployment.png)

（5）服务资源（Service）

> Service是建立在一组Pod对象之上的资源抽象，它通过标签选择器选定一组Pod对象，并为这组Pod对象定义一个统一的固定访问入口（通常是一个IP地址），若Kubernetes集群存在DNS附件，它就会在Service创建时为其自动配置一个DNS名称以便客户端进行服务发现。到达Service IP的请求将被负载均衡至其后的端点——各个Pod对象之上，因此Service从本质上来讲是一个四层代理服务。另外，service还可以将集群外部流量引入到集群中来。



（6）存储卷

> 存储卷（Volume）是独立于容器文件系统之外的存储空间，常用于扩展容器的存储空间并为它提供持久存储能力。Kubernetes集群上的存储卷大体可以分为临时卷、本地卷和网络卷。临时卷和本地卷都位于Node本地，一旦Pod被调度至其他Node，此种类型的存储卷将无法访问到，因此临时卷和本地卷通常用于数据缓存，持久化的数据则需要放置于持久卷（persistent volume）之上。



（7）Name和Namespace

> 名称（Name）是Kubernetes集群中资源对象的标识符，它们的作用域通常是名称空间（Namespace），因此名称空间是名称的额外的限定机制。在同一名称空间中，同一类型资源对象的名称必须具有唯一性。名称空间通常用于实现租户或项目的资源隔离，从而形成逻辑分组，如下图所示，创建的Pod和Service等资源对象都属于名称空间级别，未指定时，他们都属于默认的名称空间“default”。

![]((一)Kubernetes系统基础/namespace.png)

（8）Annotation

> Annotation（注释）是另一种附加在对象之上的键值类型的数据，但它拥有更大的数据容量。Annotation常用于将各种非标识型元数据（metadata）附加到对象上，但它不能用于标识和选择对象，通常也不会被Kubernetes直接使用，其主要目的是方便工具或用户的阅读和查找等。



（9）Ingress

> Kubernetes将Pod对象和外部网络环境进行了隔离，Pod和Service等对象间的通信都使用其内部专用地址进行，如若需要开放某些Pod对象提供给外部用户访问，则需要为其请求流量打开一个通往Kubernetes集群内部的通道，除了Service之外，Ingress也是这类通道的实现方式之一。



# Kubernetes集群组件

![]((一)Kubernetes系统基础/system.png)

## Master组件

> Kubernetes的集群控制平面由多个组件组成，这些组件可统一运行于单一Master节点，也可以以多副本的方式同时运行于多个节点，以为Master提供高可用功能，甚至还可以运行于Kubernetes集群自身之上。Master主要包括以下几个组件。

（1）API Server

> Api Server负责输出RESTful风格的Kubernetes API，它是发往集群的所有REST操作命令的接入点，并负责接收、校验并响应所有的REST请求，结果状态被持久存储于etcd中。因此，API Server是整个集群的网关。

（2）ETCD

> Kubernetes集群的所有状态信息都需要持久存储于存储系统etcd中，不过，etcd是由CoreOS基于Raft协议开发的分布式键值存储，可用于服务发现、共享配置以及一致性保障（例如数据库主节点选择、分布式锁等）。因此，etcd是独立的服务组件，并不隶属于Kubernetes集群自身。生产环境中应该以etcd集群的方式运行以确保其服务可用性。
>
> etcd不仅能够提供键值数据存储，而且还为其提供了监听（watch）机制，用于监听和推送变更。Kubernetes集群系统中，etcd中的键值发生变化时会通知到API Server，并由其通过watch API向客户端输出。基于watch机制，Kubernetes集群的各组件实现了高效协同。

（3）Controller Manager

> Kubernetes中，集群级别的大多数功能都是由几个被称为控制器的进程执行实现的，这几个进程被集成与kube-controller-manager守护进程中。由控制器完成的功能主要包括生命周期功能和API业务逻辑，具体如下
>
> - 生命周期功能：包括Namespace创建和生命周期、Event垃圾回收、Pod终止相关的垃圾回收、级联垃圾回收及Node垃圾回收等。
> - API业务逻辑：例如，由ReplicaSet执行的Pod扩展等。

（4）Scheduler

> Kubernetes是用于部署和管理大规模容器应用的平台，根据集群规模的不同，其托管运行的容器很可能会数以千计甚至更多。API Server确认Pod对象的创建请求之后，便需要由Scheduler根据集群内各节点的可用资源状态，以及要运行的容器的资源需求做出调度决策，如下图所示。另外，Kubernetes还支持用户自定义调度器。

![]((一)Kubernetes系统基础/scheduler.png)



## Node组件

> Node负责提供运行容器的各种依赖环境，并接收Master的管理。每个Node主要由以下几个组件构成。

（1）kubelet

> kubelet是运行于工作节点之上的守护进程，它从API Server接收关于Pod对象的配置信息并确保它们处于期望的状态（desired state，也可以说目标状态）kubelet会在API Server上注册当前工作节点，定期向Master汇报节点资源使用情况，并通过cAdvisor监控容器和节点的资源占用情况。

（2）kube-proxy

> 每个工作节点都需要运行一个kube-proxy守护进程，它能够按需为Service资源对象生成iptables或ipvs规则，从而捕获访问当前Service的ClusterIP的流量并将其转发至正确的后端Pod对象。

（3）docker

> docker用于运行容器

## 核心附件

> Kubernetes集群还依赖于一组称为"附件"(add-ons)的组件以提供完整的功能，它们通常是由第三方提供的特定应用程序，且托管运行于Kubernetes集群之上，如上图所示。

- KubeDNS

  > 在Kubernetes集群中调度运行提供DNS服务的Pod，同一集群中的其他pod可使用此DNS服务解决主机名。Kubernetes从1.11版本开始默认使用CoreDNS项目为集群提供服务注册和服务发现的动态名称解析服务，之前的版本中用到的是kube-dns项目，而SKyDNS则是更早一代的项目。

- Kubernetes Dashboard

  > Kubernetes集群的全部功能都要基于Web的UI，来管理集群中应用甚至是集群自身。

- Heapster

  > 容器和节点的性能监控与分析系统，它收集并解析多种指标数据，如资源利用率、生命周期事件等。新版本的Kubernetes中，其功能会逐渐由Prometheus结合其他组件所取代。

- Ingress Controller

  > Service是一种工作于传输层的负载均衡器，而Ingress是在应用层实现的HTTP(s)负载均衡机制。不过，Ingress资源自身不能进行“流量穿透”，它仅是一组路由规则的集合，这些规则需要通过Ingress控制器（Ingress Controller）发挥作用。目前，此类的可用项目有Nginx、Traefik、Envoy及HAProxy等。







