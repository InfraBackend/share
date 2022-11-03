
在Kubernetes集群的生命周期中，总会有某个时候，你需要对集群的宿主机节点进行维护。这可能包括程序包更新，内核升级或部署新的VM映像。在Kubernetes中，这些操作被视为“自愿中断”。


在这个系列中我们会介绍 Kubernetes 提供的所有用来实现**集群中工作节点的零宕机时间更新**的工具。

## 问题说明

我们将从简单直接的维护宿主机节点的方法开始，确定该方法的挑战和潜在风险，并逐步构建解决我们在整个系列中发现的每个问题的方法。在这个博客系列结束时我们将完成一个Kubernetes配置，该配置利用生命周期钩子，就绪探针（redinessProbe）和 PodDisruptionBudgets 来实现 Kubernetes集群的零停机时间部署。


为了开始我们的旅程，让我们看一个具体的例子。假设我们有一个两节点的 Kubernetes 集群，集群上运行着一个使用了两个 Pod 和一个 Service 资源的应用程序。![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7vAo6WnrfADEI0PFXyDwia74FH9sWxDUicnbLeu1Nhm0ltCjDeLZPJ1wLkqe3D0ohhf4ibl4HucVRibA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

现在的问题是我们要升级集群中两个宿主机节点的内核版本。我们将如何执行升级？简单粗暴的方法是使用更新的配置启动新节点，在启动新节点后关闭旧节点。尽管这种方法有效，但是这种方法存在一些问题：

-   当关闭旧节点时，节点上的 Pod 也会被删除。如果 Pod 需要清理以正常关闭上面运行的应用程序该怎么办？底层的VM技术可能不会等待 Pod 执行清理过程。
    
-   如果同时关闭所有节点怎么办？在将 Pod 重新启动到新节点中时，你的应用程序服务会短暂中断。
    

我们想要的是一种从旧节点上正常迁移 Pod 的方法，以确保在对节点进行更改时，没有任何工作负载在运行。或者，如示例中所述，如果要完全替换群集（例如替换VM镜像），我们希望将工作负载从旧节点移到新节点。在这两种情况下，我们都希望避免调度新的 Pod 到旧节点上，我们可以使用`kubectl drain`命令来实现这一点。

## 把Pod调度到节点之外

排出操作（`kubectl drain`）实现了将所有 Pod 重新调度到节点之外的目的。在排出操作期间，该节点会被标记为不可调度（通过给节点添加 NoSchedule 污点实现）。这样可以防止新建的Pod被调度到节点上。之后，排出操作开始从节点上驱逐 Pod，通过将 TERM 信号发送到 Pod 的底层容器来关闭当前在节点上运行的容器。

尽管`kubectl drain`将很好地处理将Pod 逐出节点的工作，但仍有两个因素可能会在`kubectl drain` 触发的操作运行期间导致服务中断：

-   运行中的应用程序需要能够优雅地处理 TERM 信号，当一个 Pod 驱逐时，Kubernetes 会向 Pod 发送 TERM 信号，然后在强制终结容器前会等待一段时间让容器自己关闭，这个等待时间是可以配置的。但是，如果 Pod 里的应用程序不能优雅地处理 TERM 信号，则仍然会导致不干净地关闭 Pod，比如应用程序正在工作期间（例如提交数据库事务等）。
    
-   应用程序将失去为其提供服务的所有 Pod 。在新节点上启动新容器时，您的服务会遭遇停机。
    

## 避免停机

为了最大程度地减少因维护集群等自愿性中断而导致的停机时间，Kubernetes 提供以下中断处理功能：

-   Graceful termination
    
-   Lifecycle hooks
    
-   PodDisruptionBudgets
    

本系列的其余部分中，我们将使用 Kubernetes 的这些功能来减轻驱逐事件造成的服务中断。为了使后续操作更容易，我们将在上面的示例中使用以下资源配置：

```auto
---apiVersion: apps/v1kind: Deploymentmetadata:  name: nginx-deployment  labels:    app: nginxspec:  replicas: 2  selector:    matchLabels:      app: nginx  template:    metadata:      labels:        app: nginx    spec:      containers:      - name: nginx        image: nginx:1.15        ports:        - containerPort: 80---kind: ServiceapiVersion: v1metadata:  name: nginx-servicespec:  selector:    app: nginx  ports:  - protocol: TCP    targetPort: 80    port: 80
```

上面的配置里包含了一个 Deployment 资源的最简示例，这个 Deployment 会始终维持集群中有两个标签为 `app: nginx`的 Pod，另外配置里还提供了一个可用于访问集群内 Nginx Pod 的 Service 资源的定义。

> 原文标题：Gracefully Shutting Down Pods in a Kubernetes Cluster
> 
> 发布时间：Jan 26, 2019
> 
> 原文链接：https://blog.gruntwork.io/zero-downtime-server-updates-for-your-kubernetes-cluster-902009df5b33
> 
> 文章作者：yorinasub17

这是我们实现 Kubernetes 集群零停机时间更新的第二部分。在本系列的[第一部分中](http://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487176&idx=1&sn=e3cb877a897fa24320f1820ceb80c4d0&chksm=fa80df5fcdf75649c1d517fabd265b37408654917bf6dfe8ffb35625387d0aef23e5516c0fe8&scene=21#wechat_redirect)，我们列举出了简单粗暴地使用`kubectl drain` 命令清除集群节点上的 Pod 的问题和挑战。在这篇文章中，我们将介绍解决这些问题和挑战的手段之一：优雅地关闭 Pod。

## Pod驱逐的生命周期

默认情况下，`kubectl drain`命令驱逐节点上的 Pod 时会遵循 Pod 的生命周期，这意味着整个过程会遵守以下规则：

-   `kubectl drain`将向控制中心发出删除目标节点上的 Pod 的请求。随后，请求将通知目标节点上的 `kubelet` 开始关闭 Pod。
    
-   节点上的`kubelet` 将会调用 Pod 里的 `preStop` 钩子。
    
-   当 `preStop` 钩子执行完成后，节点上的`kubelet` 会向Pod容器中运行的程序发送 `TERM`信号 （SIGTERM）。
    
-   节点上的`kubelet`将最多等待指定的宽限期（在pod上指定，或从命令行传入；默认为30秒）然后关闭容器，然后强行终止进程（使用SIGKILL）。注意，这个宽限期包括执行 `preStop`钩子的时间。
    

> 译注：Kubelet 终止Pod前的等待宽限期有两种方式指定
> 
> 1.  在Pod定义里通过Pod模板的spec.terminationGracePeriodSeconds 设定
>     
> 2.  kubectl delete pod {podName} --grace-period=60
>     

基于此流程，我们可以利用应用程序 Pod 中的`preStop`钩子和信号处理来正常关闭应用程序，以便在最终终止应用程序之前对其进行“清理”。例如，假如有一个工作进程从队列中读取信息然后处理任务，我们可以让应用程序捕获 TERM 系统信号，以指示该应用程序应停止接受新任务，并在所有当前任务完成后停止运行。或者，如果运行的应用程序无法修改以捕获 TERM 信号（例如第三方应用程序），则可以使用`preStop`钩子来实现该服务提供的自定义API，来正常关闭应用。

在我们的示例中，Nginx 默认情况下不能处理 TERM 信号，因此，我们将改为依靠 Pod 的 `preStop`钩子实现正常停止Nginx。我们将修改资源定义，将生命周期钩子添加到容器的`spec`定义中，如下所示：

```auto
lifecycle:  preStop:    exec:      command: [        # Gracefully shutdown nginx        "/usr/sbin/nginx", "-s", "quit"      ]
```

应用此配置后，在将 TERM 信号发送给容器中的Nginx进程之前，`kebulet` 调用 Pod 的生命周期钩子发出命令 `/ usr / sbin / nginx -s quit`。请注意，由于该命令将会正常停止 Nginx 进程和 Pod，因此 TERM 信号实际上在这个例子中是一个空操作。

在定义文件添加了生命周期钩子后，整个 Deployment 资源的定义变成了下面这样

```auto
---apiVersion: apps/v1kind: Deploymentmetadata:  name: nginx-deployment  labels:    app: nginxspec:  replicas: 2  selector:    matchLabels:      app: nginx  template:    metadata:      labels:        app: nginx    spec:      containers:      - name: nginx        image: nginx:1.15        ports:        - containerPort: 80        lifecycle:          preStop:            exec:              command: [                # Gracefully shutdown nginx                "/usr/sbin/nginx", "-s", "quit"              ]
```

## 停机后的后续流量

使用上面的`preStop`钩子正常关闭 Pod 可以确保 Nginx 在处理完现存流量有才会停止。但是，你可能会发现，Nginx 容器在关闭后仍会继续接收到流量，从而导致服务出现停机时间。

为了了解造成这个问题的原因，让我们来看一个示例图。假定该节点已接收到来自客户端的流量。应用程序会产生一个工作线程来处理请求。我们用在 Nginx Pod 示例图内的圆圈表示该工作线程。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5khlYtJDtMLXRb4XfQ6lFdGzD5F8Cq7yt2SibA6g7CWAUNGo3mLQSnWoVicRhTlrQvhKBhrwLcQXxg/640?wx_fmt=png)

正在处理请求的Nginx

假设在工作线程处理请求的同时，集群的运维人员决定对 `Node1` 进行维护。运维运行了`kubectl drain node-1` 后，节点上的`kubelet` 会执行 Pod 设置的`preStop`钩子，开始进入Nginx进程正常关闭的流程。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5khlYtJDtMLXRb4XfQ6lFdKIItj2t6HibFwVJicc7r2CNx4aCDibdicsygk0bSh1KyRKdmQ4n9EmZpLg/640?wx_fmt=png)

对节点进行维护，清出节点上的Pod时会先执行preStop钩子

由于 Nginx 仍要处理已存流量的请求，所以进入正常关闭流程后 Nginx 不会马上终止进程，但是会拒绝处理后续到达的流量，向新请求返回错误。

在这个时间点，假设一个新的服务请求到达了 Pod 上层的 Service，因为此时 Pod 仍然是上层 Service 的Endpoint，所以这个即将关闭的 Pod 仍然可能会接收到 Service 分发过来的请求。如果 Pod 真的接收到了分发过来的新请求 Nginx 就会拒绝处理并返回错误。

> 译注：推荐阅读[学练结合快速掌握K8s Service控制器](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&scene=21#wechat_redirect)

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5khlYtJDtMLXRb4XfQ6lFdnNo8J18erYvwnEVN2MVe1j00V8f3S2E8GTicK0iaUzKcweEFXCngK9Qw/640?wx_fmt=png)

Nginx处于关闭流程时会拒绝新来的请求

最终 Nginx 将完成对原始已存请求的处理，随后`kubelet`会删除 Pod，节点完成排空。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5khlYtJDtMLXRb4XfQ6lFdLuGgKnzyn9N4K8LMHQyJuQC3Pa8maGrm8A3hGabsNlicE3wu06aPT5w/640?wx_fmt=png)

Nginx 处理完已存请求后终止进程

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5khlYtJDtMLXRb4XfQ6lFdW5Wz32ZcicQexbadl3cZ710nJTRz4nsOQvbJLEV805wY27UbX4Bvwfg/640?wx_fmt=png)

Pod停止运行，kubelet删除Pod

为什么会这样呢？如何避免在Pod执行关闭期间接受到来自客户端的请求呢？在本系列的下一部分中，我们会更详细地介绍 Pod 的生命周期，并给出如何在 `preStop` 钩子中引入延迟为 Pod 进行摘流，以减轻来自 Service 的后续流量的影响。


## Pod关闭序列


> 译注：我的理解是要在Pod真正停止运行前，要先把它从Service上拿掉，也就是摘流。

那么，是什么情况会导致 Pod 从 Service 中注销掉呢？要了解这一点，我们需要更深入一层，来了解从集群中删除Pod时都发生了什么。

通过 Kubernetes 的 API 将 Pod 从群集中删除后，该 Pod 在元数据服务器中被标记为要删除。这会向所有相关子系统发送一个 Pod 删除通知，然后处理该通知：

> 译注：这里说的元数据服务器，指的应该是Kubernetes APIServer，而子系统则是Kubernetes的一些核心组件。

-   Pod 所在节点上的`kubelet`将启动上一篇文章中描述的 Pod 关闭序列。
    
-   所有节点上运行的`kube-proxy`守护程序将从 iptables 中删除 Pod的 IP 地址。
    
-   端点控制器将从有效端点列表中删除该 Pod，反映到我们这个例子中来就是从 Service 管控的端点（endpoint）列表中移除这个 Pod 端点。
    

我们无需了解每个系统的详细信息。这里的重点是涉及多个子系统，这些系统可能在不同的节点上运行，而上面列举的这些操作是会并行发生的。因此，在将 Pod 从所有活动列表中删除之前，Pod 很有可能已经开始执行 preStop 钩子并接收到了 TERM 信号。这就是即使 Pod 在启动关闭序列后，仍继续接收到流量的原因。

## 摘流方案

从表面上看，我们可以将上面那些事件序列串联起来，禁止他们并行进行，直到从所有相关子系统注销了要删除的 Pod 之后，再开始 Pod 的关闭序列。但是，由于 Kuberenetes 系统的分布式性质，在实践中很难做到这一点。如果节点之一遇到网络阻隔会怎样？是否要无限期地等待事件传播？如果该节点重新恢复联机怎么办？如果你的Kubernetes 集群有成千上万个节点怎么办？

不幸的是，现实情况是并不存在防止所有中断的完美解决方案。但是，我们可以做的是在 Pod 关闭序列中引入足够的延迟，以捕获99％的情况。为此，我们在preStop挂钩中引入了一个 sleep指令，以延迟 Pod 关闭序列。接下来，让我们看看在我们的例子中它是如何工作的。

我们会更新一直以来使用的资源定义文件，使用sleep 命令引入延迟来作为要执行的 preStop 钩子的一部分。在「 Kubernetes in Action」中，作者 `Lukša` 建议使用5到10秒的延迟，因此在这里我们将使用5秒延迟作为 preStop 钩子的一部分：

```auto
lifecycle:  preStop:    exec:      command: [        "sh", "-c",        # Introduce a delay to the shutdown sequence to wait for the        # pod eviction event to propagate. Then, gracefully shutdown        # nginx.        "sleep 5 && /usr/sbin/nginx -s quit",      ]
```

现在，让我们推演一下这个示例关闭序列中会发生什么。像上一篇文章的分析一样，我们将使用`kubectl drain`逐出节点上的 Pod。这将会发送一个Pod deletion 事件，该事件会同时通知给 kubelet 和 Endpoint Controller（端点控制器，这里指的是 Pod 上层的 Service控制器）。在此，我们假设 preStop 钩子在 Service 从自己可用列表中移除 Pod 之前启动。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f6qb5bA4pttCzpRolT0K7mL56m2MOr8W0khDCtViaQtpOtrOVZqMibWgYhkTicjd8AGPvp9TSB6icCzIw/640?wx_fmt=png)

驱逐节点上的Pod，会发送一个Pod Deletion事件

在 preStop 钩子执行时，首先会延迟5秒执行第二条关闭Nginx的命令。在这个期间，Service 将会从自己的可用列表中移除 Pod。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f6qb5bA4pttCzpRolT0K7mLkicHvfomqS8pNtkBJh9uvNaQlvjCiaX2CbrLUrQzfV5O2JQSoLIW9JBQ/640?wx_fmt=png)

关闭程序被延迟的同时Service会从列表中去掉要关闭的Pod

在此延迟期间，Pod 仍处于运行状态，因此即使其接收到新的连接请求，它仍能够处理连接。此外，在将 Pod 从Service 中移除后，客户端的连接请求，将不会路由到将要关闭的 Pod 上。因此，在这种情况下，假如 Service 在延迟期间内处理完这些事件，集群将不会有停机时间。

最后，preStop 钩子进程从休眠中醒来并执行关闭 Nginx 容器，从节点中删除容器：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f6qb5bA4pttCzpRolT0K7mLPqDBNibaOUibZj21hYV3YZpYuQ7E8g4dgkpwrI1EWjSic5ZN1k9k68PKw/640?wx_fmt=png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f6qb5bA4pttCzpRolT0K7mLjXM0IyjYqficScPK4aDiaFcDqPT4L3icpoeiaUzEWLLLggHTJrPNRSleFw/640?wx_fmt=png)

此时，我们就可以安全地在Node1上进行任何升级，包括重启节点加载新的内核版本。如果我们已经启动了一个新节点来容纳Node1运行的工作负载，那么我们也可以关闭Node1节点。

## 重新创建Pod

如果你已经看到了这里，你可能想知道如何重新创建最初被调度到维护节点上的 Pod。现在，我们知道了如何正常关闭 Pod，但是如果要维持运行中的 Pod 的数量，该怎么办？这就要靠 Deployment 控制器发挥作用了。

Deployment 控制器负责在集群上维护指定的期望状态，如果你能回想起我们的资源定义文件里的内容，你会发现我们不是直接创建的 Pod。取而代之的是我们通过给 Deployment 提供创建Pod的模板，让它来自动为我们管理 Pod。下面是定义中的 Pod 模板部分的内容：

```auto
template:    metadata:      labels:        app: nginx    spec:      containers:      - name: nginx        image: nginx:1.15        ports:        - containerPort: 80
```

模板指定了创建一个运行`nginx:1.15`镜像的容器的Pod，指定 Pod 携带的标签为`app:nginx`，向外暴露80端口。

除了Pod模板之外，我们还为 Deployment 资源提供了一个配置，用于指定应维护的 Pod 副本数：

```auto
spec:  replicas: 2
```

这会通知 Deployment 控制器它应始终维持有两个 Pod 运行在集群上。每当运行的 Pod 数量下降时，Deployment 控制器都会自动创建一个新的Pod来替换它。因此，在我们这个例中，当我们使用 `kubectl drain` 操作从节点上驱逐 Pod 时，Deployment 控制器会在其他可用节点上自动重新创建 Pod，保持当前状态与定义里指定的期望状态一直。




> PDB是针对Voluntary Disruption场景设计的，属于Kubernetes可控的范畴之一，而不是为Involuntary Disruption（非自愿中断设计）设计的，自愿中断主要是一些系统维护和升级更新的操作，而非自愿中断一般都是些硬件和网络故障导致的中断。一些集群会对Node进行自动管理，因此需要使用PDB来保障应用的HA。

## PDB：预算可容忍的故障数

Pod 中断预算（PDB）是一种在给定时间可容忍的中断数量（故障预算）的指标。每当计算出服务中的 Pod 中断会导致服务降至PDB以下时，操作就会暂停，直到可以维持PDB为止。这意味着在等待更多 Pod 可用之前，可以暂时停止逐出Pod，以免驱逐 Pod 而超出预算。

要配置一个PDB，我们需要在 Kubernetes 里创建一个`PodDisruptionBudgets`资源对象（后面简称PDB对象）用来匹配服务中的Pod。举个例子来说，我们想要创建一个PDB对象让我们之前使用Deployment创建的Nginx应用始终保持至少一个Pod可用，那么我们可以应用下面的配置：

```auto
apiVersion: policy/v1beta1kind: PodDisruptionBudgetmetadata:  name: nginx-pdbspec:  minAvailable: 1  selector:    matchLabels:      app: nginx
```

这会告诉 Kubernetes 我们想要在任意时间点都有至少一个匹配标签`app: niginx`的Pod在集群中可用。使用此方法，我们可以促使Kubernetes 保证在自愿中断（更新/ 维护）进行时服务至少有一个Pod是可用的，避免服务停机。

## PDB的工作原理

为了说明 PDB 是如何工作的，让我们回到我们的一直以来使用的示例。为了简单起见，在示例中，我们将忽略任何 preStop 钩子，就绪性探针和服务请求。我们还将假设我们要对集群节点进行一对一替换。这意味着我们将通过使节点数量加倍，在新节点上运行重建的 Pod。

在图示中我们从两个节点的原始群集开始：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncdJSu9fuaO4j3ruyPwLwyfWBamSV7IzhRzQwFA6WMrfu9fr113s2ribA/640?wx_fmt=png)

我们提供了两个额外的节点来运行新的虚拟机镜像。最终将会在新节点上创建 Pod 替换运行在旧节点上的Pod。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncMAQwKKHfmHribXZ2bvCG4hJ80kdjichEDibGnejYMROYj5aCBG9rMDAsQ/640?wx_fmt=png)

要替换服务Pod所在的节点，我们首先需要清空旧节点。在此示例中，让我们看看如果同时向运行Nginx Pod的两个节点发出`kubectl drain`命令时会发生什么。排空Node上Pod的请求将在两个线程中发出（实践时，可以使用两个终端分别运行`kubectl drain`命令），每个线程管理一个节点的排空执行序列。

注意，在这里我们，假设 kubectl drain 命令会立即发出驱逐请求。实际上，drain 操作首先会涉及对节点进行标记（给节点打上 NoSchedule的 标记），以便不会把 Pod 重新调度到旧节点上去。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncnibpmVgUeEacoPJZFqSzmq1oZCdgOL60U85buJ3oOuWfYPueKBrtY9g/640?wx_fmt=png)

标记节点不可调用

节点标记完成后，负责排空节点的线程开始逐出节点上的Pod。这个过程中线程首先会去控制中心查询，看驱逐 Pod 是否会导致服务可用Pod数下降到配置的 PDB 以下。

这里需要注意的是，控制台会将并发请求串行化，一次处理一个PDB查询。这样，在这种情况下，控制平面将成功响应其中一个请求，而使另一个请求失败。这是因为第一个请求基于两个可用Pod的。允许此请求会将可用的 Pod 数量减少到1，PDB 得以维持。当控制中心允许请求继续进行时，便会将其中一个容器逐出，从而变得不可用。之后，当处理第二个请求时，控制平面将拒绝它，因为允许该请求会将可用Pod的数量降至0，低于我们配置的PDB。

鉴于此，在示例中，我们假定节点1是获得成功响应的节点。在这种情况下，节点1负责排空操作的线程将继续逐出 Pod，而节点2的排空线程将会等待并在稍后重试：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncTwcdteJld1EWlesbSRZ7ibp6skibJBRtoHv560C2pksy9fibSYG3gZvEg/640?wx_fmt=png)

串行化逐出请求，允许线程1的请求，因为不满足PDB拒绝线程2的请求

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncJsFg70c0Xiang5JaCkUnAgGQ25q3XbVUBCb2j8Dpr1puiav2VpmAAfUw/640?wx_fmt=png)

驱逐Node1上的Nginx Pod

当节点1上的Nginx Pod被驱逐后，Pod 会立即被 Deployment 重建出来并调度到集群的节点上。因为我们集群的旧节点都已经被打上了`NoSchedule`的标记，所以调度器会选择一个新节点进行调度。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncClLYZ5Z0nA5ms4n6rYia3CGKdRrbRwWXNBcDXweTicrAgpoM0OydAiceA/640?wx_fmt=png)

重建Pod被调度到了Node3这个新节点上

至此，成功在新节点上完成了Pod更换，并且排空了原始节点Node1，用于排空Node1的线程就完成任务了。

从现在开始，当 Node2 的排空线程再次去控制中心查询 PDB 时，将会得到成功响应。这是因为有一个正在运行的Pod （刚才在Node3上新建的 Pod）不在考虑驱逐的序列中，因此，让 Node2 的排空线程继续前进不会将可用Pod的数量降到 PDB 以下。所以线程2会继续前进逐出 Node2 上的 Pod，完成驱逐过程：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncqAcibGHN4icNicGWW02IpzSPXx4TSwdbEicvKc55X885yoCDF0edvxZdNg/640?wx_fmt=png)

线程2再次查询，可以满足PDB后开始驱逐Node2上的Pod

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncuiaw0icm390pHcHELXPz0Y7FBaVo7zlFnJKaBlehaXaRpzxBXnDicqzhQ/640?wx_fmt=png)

驱逐Node2上的Nginx Pod

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f404D2rYNHrgYVaAIIPfvncMb9ADEWI6FGOcBU6OiakYRCtkcEVOT1W84EBKNPD1Ev1pBlYshLiaruA/640?wx_fmt=png)

在Node4上新建Pod，完成整个集群Node升级过程

至此，我们就成功地将两个 Pod 都迁移到了新节点上，而没有遇到无可用 Pod 可以为应用程序提供服务的情况。而且，我们不需要在两个负责排空节点的线程之间有任何协调逻辑，Kubernetes 会根据我们提供的配置为我们处理所有工作！

