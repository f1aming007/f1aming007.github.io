# Kuberenetes SUNK


# SUNK 简介（Slurm on Kubernetes）
{{< figure src="/images/sunk01.png" title="sunk01" >}}

SUNK 是一个开源项目（将于 2024 年初发布），它将 Kubernetes 容器化部署和 GitOps 引入 Slurm，并将 Slurm 调度程序插件集成到 Kubernetes
<!--more-->

![](media/17025443925846.jpg)

[PDF 参考](https://slurm.schedmd.com/SLUG23/CoreWeave-SLUG23.pdf)

本质上 `SUNK` 将 `Slurm` 集成为Kubernetes 调度程序， 并允许 Slurm 作业 在 Kubernetes 内运行。这创造了更加无缝的体验， 在同一中央平台上支持爆发式和批量工作负载，并允许开发人员利用 Kubernetes 上的 SLURM 资源管理。

单独管理 Slurm 和 Kubernetes 可以降低整体复杂性， 但也大大降低了你选择在所有计算上运行的工作负载的灵活性，也就是说，最大限度地利用 GPU 资源变得更加困难。

通过在k8s 上部署 `Slurm`集群， 在同一计算池上， 可以灵活从 Kubernetes 或 Slurm 端无缝使用该计算。

两种解决方案。 一个平台。 一个计算池， 一个编排层来统治他们
{{< figure src="/images/sunk02.png" title="sunk02" >}}

![](media/17025458920211.jpg)
{{< figure src="/images/sunk03.png" title="sunk03" >}}

![](media/17025462266827.jpg)


### 为什么创建SUNK

客户效率，当你运行非常大且昂贵的HPC集群时， ，接近 100% 的利用率非常重要。任何时候，当你可能不使用你所付费的计算时，成本都可能非常高

SUNK 完全构建在 Kubernetes 之上, 每个客户端都对其集群有一个单一的入口和管理点，但是，我们意识到许多喜欢 Slurm 的客户单独管理他， 或者询问我们是否有 Sulrm 集成

SUNK 的核心是效率， 这就是专为 GPU 密集型用例而构建的原因


客户能够利用 Slurm 的优势， 同时保持系统的完整性和易用性（也称为无需管理单独的集群）。由于该解决方案不存在，我们决定构建它。

## SUNK 的特点
{{< figure src="/images/sunk04.png" title="sunk04" >}}

![](media/17026243079535.jpg)

**配置和部署**： 通过将 Slurm 集群部署为 各种 Kubernetes 资源，我们能够使用高度可定制的 Helm Chart 来部署它， 解锁了基于  Kubernetes 的 GitOps 工作流程的大型生态系统及其附带的所有功能， 好处包括
* 轻松追踪和配置prolog 和epilog 脚本
* 快速部署测试机器
* 支持s6脚本和服务
* 可配置的身份验证方案，包括通过配套的 OpenLdap helm chart或第三方解决方案（Authentik、GAuth 等）

**Kubernetes集成**： 部署SUNK 后，你将获得在 Kubernetes 上运行的所有正常好处，例如：
* 快速调度
* 集装箱化
* 控制平面服务的高可用性
* 动态节点扩展
* 具有请求和限制的资源管理
* 通过 PersistentVolumeClaim 资源共享文件系统

它还包括自定义 Slurm Kubernetes 调度程序 （用于通过 Slurm 调度程序本机 Kubernetes 工作负载）， 使你能够在 Slurm 作业（突发工作负载） 和 Kubernetes 工作负载（无服务器工作负载）之间动态转移单个计算池。

状态管理： 通过在  Kubernetes 之上运行，你还可以更好地控制状态管理，包括：
* 在 k8s 和 Slurm 之间具有双向状态同步的动态节点
* 自动拓扑生成
* 支持 Pyxis 容器执行
*  GRES 支持和自动识别


## SUNK 的工作原理

了解SUNK 如何有效集成 Slurm和Kubernetes， 了解一下底层结构

展示了 SUNK的高层架构图
{{< figure src="/images/sunk05.png" title="sunk05" >}}

![](media/17026288529216.jpg)

SUNK 的工作原理以及节点集

首先要指出的是， `所有这些服务都是在 Kubernetes 中容器化`。 在顶部， 你拥有集群范围内部署的资源， 在底部， 你拥有耽搁 Slurm 集群部署的组件

**A部分**： 所有典型的Slurm `组件都部署在一个pod内`， 每个组件都有自己的可配置资源请求，包括了用户为了与 Slurm集群交互而链接的登陆节点，一旦连接到这些登陆节点， Kubernetes就会被抽象出来， 将获得正常 Slurm 集群的体验

**B部分**： 其中 `许多组件都需要 Slurm 配置` 通过将它们部署为`k8s ConfigMap` 和 `Secret` ， Slurm 配置文件可以在一个位置进行管理，并安装在 需要它们的任何地方，包括Slurm 配置、拓扑、prolog /epilog 脚本及数据库密码等敏感信息

**C部分**： HPC 集群最重要的方面是计算节点， 他们在中间显示为 裸机 Kubernetes 节点， slurmd 在上面以 1:1 映射现实的计算 pod中运行， 计算的部署由称为 NodeSet的CRD进行管理

**D部分**： 我们讨论的许多功能都要求 ` Slurm 和 Kubernetes 端的计算状态保持同步`。 Slurm同步器充当两端的中间人。 通过Slurm 的REST API 发送和拉取信息

**E部分**：一旦同步器从 Slurm获取状态信息到 Kubernetes， 信息需要在许多不同的地方保持一致， `集群范围的Operator监视不同的资源并在适当的时候进行变更， 无论更改来自 Kubernetes 还是 Slurm。`

由于具有同步状态的能力，我们能够使用自定义 Kubernetes 调度程序，该调度程序允许你根据 Slurm 状态将工作负载调度到计算 Kubernetes 节点上。

由于所有这些组件都是在Kuernetes 上运行， 我们可以通过 Prometheus 公开指标，这些指标可通过多种不同的方式使用。

## 节点集、 同步器和调度器

为了使这种集成称为可能， 必须定制SUNK 的三个方面： 节点集、 同步器和调度程序

### 节点集
{{< figure src="/images/sunk06.png" title="sunk06" >}}

![](media/17028645434435.jpg)

首先是 Nodeset （节点集）（如上图和第一个图表的 F 部分所示）。 这是我们开发的CRD， 它在Kubernetes 环境上下问中定义了Slurm 节点。它的定义与Statefulset或Deployment类似，但与节点的 一对一映射更类似于 Daemonset

`Nodeset 维护着一系列 Slurm中状态的状态字段。 着提供了基于 kubernetes 和 Slurm 中的状态来保护 Slurm 节点更新和扩展的机制`

Nodeset pod 运行 slurmd 和 munged， 安装在共享配置映射中， 用于Slurm配置、prolog 和epilog，以及作为 PVC 的共享文件系统卷。

下图显示了大部分状态字段。正如你所看到的，Nodeset pod 可以主动了解有多少可能的节点与关联性匹配、其中有多少节点当前被分配为 Slurm 节点、跟踪准备情况、运行和Drain条件等


### 同步器

Nodeset 的许多功能都以来与  Kubernetes 端了解 Slurm 的状态，反之亦然。
`Syncer 通过两部分完成此任务， Slurm 客户端和pod控制器`

Slurm 客户端与 Slurm REST API通信以推送和拉取信息， 随着Slurm 集群的规模增长到数百甚至千个节点。 这种通信将增长到大量流量。 客户端有效地缓存结构以处理大规模集群

Pod 控制器接收来自客户端的事件， 并根据任何差异来协调 pod状态， 如果更改来自 Kubernetes 端。pod 控制器会将事件推送到客户端， 客户端会根据需要将该更改传递给 Slurm。

因此，信息流有两个方向

举例来说， Slurm 中的一个节点进入Drain状态， Syncer 将监测状态更改并在pod 上添加注释，此注释可以在 Kubernetes 端进行操作，例如仅当 Slurm 节点处于Drain状态时才开始更新。

另一方面，改变可以从 Kubernetes 端开始。假设你检测到硬件问题并封锁 Kubernetes 内部的一个节点。Syncer 会将相应的 Slurm 节点放入Drain中，并添加一个很好的理由来解释为什么它被驱逐。
{{< figure src="/images/sunk07.png" title="sunk07" >}}

![](media/17028685677882.jpg)


### 调度程序

`调度程序 是在命名空间中运行的另一个服务， 它允许一些非常有趣的功能。`

假设你想要在 SUNK上获取一个预留实例池，并确保除了从按需池中使用的实例之外，你还能获得这些实例的最大利用率。又名：同时运行推理和训练。

当在 k8s 中使用此 Scheduler 调度 pod 时，就像由 Knative 等无服务器解决方案驱动的推理 pod 一样，会创建一个 Slurm 作业，该作业能够抢占低优先级任务。当 Slurm 节点最终分配给 Slurm 中的作业时，调度程序会将 k8s pod 放置到该节点上并在那里运行

简而言之，你可以无缝地使用 Kubernetes 中 Slurm 集群的计算，而无需主动维护两个独立的预留计算池（这些池不是动态分配的）。

{{< figure src="/images/sunk08.png" title="sunk08" >}}

![](media/17028687610939.jpg)


## 参考链接
1. Kubernetes 遇见高性能计算：https://kubernetes.io/blog/2017/08/kubernetes-meets-high-performance/
2. SUNK 是 Kubernetes 上 Slurm 的实现：https://www.coreweave.com/blog/sunk-slurm-on-kubernetes-implementations?
