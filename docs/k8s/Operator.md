

## 什么是Operator

Operator 概念是由 CoreOS 的工程师于2016年提出的，它让工程师可以根据应用独有的领域逻辑为应用编写自定义的控制器。这句话说的有点虚空，我们通过一个简单的例子理解一下 Operator 。

假设有一个连接数据库的 Java Web程序。你想将其部署到您的k8s集群。理想情况下，你会希望用 Deployment 部署应用然后暴露给 Service，对于应用服务的后端则是使用 StatuflSet 部署数据库。所以需要完成两部分的部署才能把整个应用服务部署完成：

-   无状态部分：Java Web 应用。
    
-   有状态部分：数据库。
    

在上面的例子中，我们可以应用我们对应用程序与数据库之间的关系的了解，创建一个控制器，该控制器将以某种特定方式运行时执行某些操作。比如备份、更新、数据还原这些任务该如何完成取决于应用程序本身和业务限制（领域知识）。这些与应用强相关的操作就是 Kubernetes Operator要做的：代替原本需要由SRE（Site Reliability Engineers）和运维工程师来完成操作的执行。

## 自定义资源和控制器

Operator 模型基于 Kubernetes 中的两个概念结合而成：自定义资源和自定义控制器。

### 自定义资源

在 Kubernetes中，资源是 Kubernetes API中的一个端点，用于存储一堆特定类型的API对象。它允许我们通过向集群添加更多种类的对象来扩展Kubernetes。添加新种类的对象之后，我们可以像其他任何内置对象一样，使用 kubectl 来访问我们自定义的 API 对象。

以 Pod 或 Deployment 为例。编写清单时，必须在 YAML 文件中指定一种类型（是 Pod 还是 Deployment）。自定义资源就是不由 Kubernetes 原生提供的资源对象。

### 自定义控制器

Kubernetes 的所有控制器，都有一个控制循环，负责监控集群中特定资源的更改，并确保特定资源在集群里的当前状态与控制器自身定义的期望状态保持一致。

举例来说有一个 Deployment 控制器管控值集群里的一组 Pod ，当你 Kill 掉一个 Pod 。控制器发现定义中期望的Pod数量与当前的数量不匹配，它就会马上创建一个 Pod 让当前状态与期望状态匹配。控制器这种让关联资源的当前状态向期望状态迈进的过程叫做调谐（reconcile）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5CZyzIGHmNkU4wLRquicGNmMGBbO5ibdXWdLBovicWmLz765FG9YPliaWYr5fibT8HagkOIMtMssWzyCA/640?wx_fmt=png)

图片

那么，像 Deployment 这样的内置控制器也叫做 Operator 吗？其实不是，因为这些控制器不是特定于特定的应用程序的，而是内置类型资源的上游控制器。

## 什么时候应该用 Operator

重要的是要知道所有的 Operator 都是控制器，但并非所有的控制器都是 Operator。对于被视作 Operator 的控制器，它必须知道应用程序的业务逻辑，才能代表用户（SRE / Ops工程师）执行自动化任务。

-   当需要封装有状态的应用程序业务逻辑（使用Kubernetes API控制所有内容）时，都可以使用 Operator 。这使围绕 Kubernetes 生态系统内置的应用程序的自动化成为可能。
    
-   每当需要创建工具来监视应用程序的更改并在发生某些事情时执行某些SRE / Ops任务时，都应使用 Operator。
    

## 关于 Operator 的使用建议

K8S内置控制器用于群集本身，而 Operator 是用于部署有状态应用程序的控制器。

创建 Operator 时，请遵循以下最佳模式实践：

-   要站在巨人的肩膀上--利用内置资源种类在它们的基础上创建你自定义的资源种类。
    
-   确保控制器不需要其他外部代码即可正常工作，只需运行kubectl install即可部署你定义的控制器。
    

当你准备好为特定应用程序创建自定义资源，该资源可以与自定义控制器进行协调，从而可以扩展 Kubernetes 的正常行为时，就是时候开始使用 Operator了。