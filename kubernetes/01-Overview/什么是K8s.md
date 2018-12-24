# 什么是kubernetes

Kubernetes是一个[旨在自动部署、扩展和运行应用容器的开源平台](http://www.slideshare.net/BrianGrant11/wso2con-us-2015-kubernetes-a-platform-for-automating-deployment-scaling-and-operations) 。

使用Kubernetes，您可以快速有效地回应客户需求：

- 快速、可预测地部署应用。
- 动态缩放您的应用。
- 无缝地推出新功能。
- 仅对需要的资源限制硬件的使用

我们的目标是构建一个生态系统，提供组件和工具以减轻在公共和私有云中运行应用程序的负担。



#### Kubernetes是

- **可移植**: 共有、私有、混合、多云
- **可扩展**: 模块化、可插拔、提供Hook、可组合
- **自愈**: 自动放置、自动重启、自动复制、自动缩放

Google于2014年启动了Kubernetes项目。Kubernetes建立在[Google在大规模运行生产工作负载方面](https://research.google.com/pubs/pub43438.html) 十几年的经验之上，并结合了社区中最佳的创意和实践。



## 为什么使用容器

寻找你为啥要使用[容器](https://aucouranton.com/2014/06/13/linux-containers-parallels-lxc-openvz-docker-and-more/) 的原因？

![](images/why_containers.svg)

部署应用程序的*旧方法*是使用操作系统的软件包管理器在主机上安装应用程序。这种方式，存在可执行文件、配置、库和生命周期与操作系统相互纠缠的缺点。人们可构建不可变的虚拟机映像，从而实现可预测的升级和回滚，但VM是重量级、不可移植的。

*新方法*是部署容器，容器基于操作系统级别的虚拟化而不是硬件虚拟化。这些容器彼此隔离并且与宿主机隔离：它们有自己的文件系统，看不到对方的进程，并且它们的计算资源使用可以被界定。它们比VM更容易构建，并且由于它们与底层基础架构和宿主机文件系统解耦了，可实现跨云、跨操作系统的移植。

由于容器小而快，因此可在每个容器镜像中包装一个应用程序。这种一对一的应用到镜像关系解锁了容器的全部优势。使用容器，可以在构建/发布期间（而非部署期间）创建不可变的容器镜像，因为每个应用程序无需与其余的应用程序栈组合，也无需与生产基础架构环境结合。 在构建/发布期间生成容器镜像使得从开发到生产都能够保持一致的环境。 同样，容器比虚拟机更加透明、便于监控和管理——特别是当容器进程的生命周期由基础架构管理而非容器内隐藏的进程监控程序管理时。 最后，通过在每个容器中使用单个应用程序的方式，管理容器无异于管理应用程序的部署。

容器好处概要：

- **灵活的应用创建和部署** ：与VM映像相比，容器镜像的创建更加容易、有效率。
- **持续开发，集成和部署** ：通过快速轻松的回滚（由于镜像的不可变性）提供可靠且频繁的容器镜像构建和部署。
- **Dev和Ops分离问题** ：在构建/发布期间而非部署期间创建镜像，从而将应用程序与基础架构分离。
- **开发、测试和生产环境一致** ：在笔记本电脑运行与云中一样。
- **云和操作系统可移植性** ：可运行在Ubuntu、RHEL、CoreOS、内部部署，Google Container Engine以及任何其他地方。


- **以应用为中心的管理**：从在虚拟硬件上运行操作系统的抽象级别，提升到使用逻辑资源在操作系统上运行应用程序的级别。
- **松耦合，分布式，弹性，解放的微服务**：应用程序分为更小、独立的部件，可动态部署和管理——而不是一个运行在一个大型机上的单体。
- **资源隔离**：可预测的应用程序性能。
- **资源利用**：效率高，密度高。



#### 为什么我需要Kubernetes，它能干啥？

最基本的功能：Kubernetes可在物理机或虚拟机集群上调度和运行应用容器。然而，Kubernetes还允许开发人员将物理机以及虚拟机 “从**主机为中心的**基础设施转移到以**容器为中心的**基础设施”，从而提供容器固有的全部优势。 Kubernetes提供了构建**以容器为中心的**开发环境的基础架构。

Kubernetes满足了在生产中运行的应用程序的一些常见需求，例如：

- [Co-locating helper processes](https://kubernetes.io/docs/concepts/workloads/pods/pod/) ，促进组合应用程序和保留”一个应用程序的每个容器“模型
- [Mounting storage systems](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Distributing secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Checking application health](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
- [Replicating application instances](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [Using Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Naming and discovering](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)
- [Balancing loads](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Rolling updates](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/)
- [Monitoring resources](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
- [Accessing and ingesting logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Debugging applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application-introspection/)
- [Providing authentication and authorization](https://kubernetes.io/docs/admin/authorization/)

这提供了PaaS的简单性，并具有IaaS的灵活性，并促进了跨基础架构提供商的可移植性。



#### Kubernetes是一个怎样的平台？

尽管Kubernetes提供了大量功能，但总有新的场景从新功能中受益。应用程序特定的工作流程可被简化，从而加快开发人员的速度。可接受的特别编排最初常常需要大规模的自动化。这就是为什么Kubernetes也被设计为提供构建组件和工具的生态系统，使其更容易部署，扩展和管理应用程序。

[Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 允许用户随心所欲地组织他们的资源。[Annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 允许用户使用自定义信息来装饰资源以方便他们的工作流程，并为管理工具提供检查点状态的简单方法。

此外， [Kubernetes control plane](https://kubernetes.io/docs/concepts/overview/components/) 所用的[API](https://kubernetes.io/docs/reference/api-overview/) 与开发人员和用户可用的API相同。用户可以使用  [their own API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/extending-api.md) 编写自己的控制器，例如 [scheduler](https://github.com/kubernetes/community/blob/master/contributors/devel/scheduler.md) ，这些API可由通用 [command-line tool](https://kubernetes.io/docs/user-guide/kubectl-overview/) 定位。

这种 [design](https://git.k8s.io/community/contributors/design-proposals/architecture/principles.md) 使得许多其他系统可以构建在Kubernetes上。



#### Kubernetes不是什么？

Kubernetes不是一个传统的，全面的PaaS系统。 它保留了用户的重要选择。

Kubernetes：

- 不限制支持的应用类型。不规定应用框架（例如 [Wildfly](http://wildfly.org/) ），不限制支持的语言运行时（例如Java，Python，Ruby），不局限于 [12-factor applications](https://12factor.net/) ，也不区分*应用程序*和*服务* 。 Kubernetes旨在支持各种各样的工作负载，包括无状态、有状态以及数据处理工作负载。 如果应用程序可在容器中运行，那么它应该能够很好地在Kubernetes上运行。


- 不提供中间件（例如消息总线）、数据处理框架（例如Spark）、数据库（例如MySQL），也不提供分布式存储系统（例如Ceph）作为内置服务。 这些应用可在Kubernetes上运行。


- 没有点击部署的服务市场。


- 不部署源代码，并且不构建应用。持续集成（CI）工作流是一个不同用户/项目有不同需求/偏好的领域，因此它支持在Kubernetes上运行CI工作流，而不强制工作流如何工作。


- 允许用户选择其日志记录、监视和警报系统。（它提供了一些集成。）
- 不提供/授权一个全面的应用配置语言/系统（例如 [jsonnet](https://github.com/google/jsonnet) ）。
- 不提供/不采用任何综合的机器配置、维护、管理或自愈系统。

另一方面，一些PaaS系统可运行*在* Kubernetes上，例如 [Openshift](https://www.openshift.org/) 、 [Deis](http://deis.io/) 、[Eldarion](http://eldarion.cloud/) 等。 您也可实现自己的定制PaaS，与您选择的CI系统集成，或者仅使用Kubernetes部署容器。

由于Kubernetes在应用层面而非硬件层面上运行，因此它提供了PaaS产品通用的功能，例如部署，扩展，负载均衡，日志和监控。然而，Kubernetes并不是一个单体，这些默认解决方案是可选、可插拔的。

另外Kubernetes不仅仅是一个*编制系统* 。实际上，它消除了编制的需要。编制的技术定义，就是执行定义的工作流：首先执行A，然后B，然后执行C。**相反，Kubernetes由一组独立、可组合的控制进程组成，这些控制进程可将当前状态持续地驱动到所需的状态。** 如何从A到C不要紧，集中控制也不需要；这种做法更类似于*编排* 。 这使系统更易用、更强大，更具弹性和可扩展性。

>  译者按：编排和编制：<https://wenku.baidu.com/view/ad063ef2f61fb7360b4c65cd.html>



#### *Kubernetes*的含义是什么？K8S呢？

**Kubernetes**源自希腊语，意思是*舵手*或*飞行员* ，是*governor（掌舵人）* 和[cybernetic（控制论）](http://www.etymonline.com/index.php?term=cybernetics) 的根源。 *K8s*是将8个字母“ubernete”替换为“8”的缩写。

> 译者按：控制论简介（讲解了什么是governor&cybernetic）：<https://wenku.baidu.com/view/1d97762c0066f5335a812157.html>





## 原文

<https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/>