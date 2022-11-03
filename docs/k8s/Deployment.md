

-   什么是`Deployment`控制器
    
-   `Deployment`的工作原理。
    
-   怎么创建`Deployment`。
    
-   如何使用`Deployment`滚动更新应用。
    
-   如何使用`Deployment`进行应用的版本回滚。
    

## 什么是Deployment

在`Kubernetes`中，建议使用`Deployment`来部署`Pod` 和 `RS`，因为它具有很多方便管理集群的内置功能，比如：

-   轻松部署RS（副本集）
    
-   清理不再需要的旧版RS
    
-   扩展/缩小RS里的Pod数量
    
-   动态更新`Pod`（根据Pod模板定义的更新用新Pod替换旧Pod）
    
-   回滚到以前的`Deployment`版本
    
-   保证服务的连续性
    

以下面这个`Deployment`对象的定义为例，第一部分是自己的元信息（name, labels）的定义，第二部分是`ReplicaSet`对象的定义(spec.replica=3....)，`ReplicaSet`定义里又包含了`Pod`的定义(spec.template)：

```auto
apiVersion: apps/v1kind: Deploymentmetadata:  name: nginx-deployment  labels:    app: nginxspec:  replicas: 3  selector:    matchLabels:      app: nginx  template:    metadata:      labels:        app: nginx    spec:      containers:      - name: nginx        image: nginx:1.7.9        ports:        - containerPort: 80
```

在具体的实现上，这个`Deployment`，与`ReplicaSet`，以及`Pod` 的关系和管理层级我们可以用一张图把它描述出来：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/z4pQ0O5h0f5Iw7YXQNvoPR2rp5aPupfm9ZgbOPTicefwVXhKt053jePkWgHJqe1FFUGrSGlaVUib9NhPsFGfMJiag/640?wx_fmt=jpeg)

Deployment、RS和Pod的关系

## Deployment的工作原理

`Kubernetes`里有很多种控制器，每一个控制器，都以独有的方式负责某种编排功能。Deployment，正是这些控制器中的一种。它们都遵循 Kubernetes 项目中的一个通用编排模式，即：控制循环（control loop），每种控制器负责的编排功能就是它们自己在控制循环里实现的逻辑。

接下来，还是以上面定义的`Deployment` 为例，我和你简单描述一下的工作原理：

-   Deployment 控制器从 Etcd 中获取到所有携带了"app: nginx"标签的 Pod，然后统计它们的数量，这就是实际状态；
    
-   Deployment 对象的 Replicas 字段的值就是期望状态，Deployment 控制器将两个状态做比较；
    
-   根据比较结果，`Deployment`确定是创建 Pod，还是删除已有的 Pod，还是什么不干；
    

这是针对`Pod`副本数的编排，至于`Pod`的动态更新和`Deployment`对象版本的回滚文章下面再说。总而言之，控制器的核心思想就是通过控制循环不断地将实际状态调谐成定义的期望状态，一旦期望状态有更新就会触发控制循环里的调谐逻辑。

## 怎么创建Deployment

创建`Deployment`前需要先声明它的对象定义，我们拿以前文章《[**Kubernetes入门实践--部署运行Go项目**](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485328&idx=1&sn=1615cbb8d60b0df38cf59375cca6cef2&chksm=fa80d607cdf75f11cf025730a492646f3ea375a81fca5e70c46703f37aa2e7d929950693cae6&token=845896787&lang=zh_CN&scene=21#wechat_redirect)》里用到过的`Deployment`定义简单解释下每部分的含义：

```auto
apiVersion: apps/v1kind: Deploymentmetadata: # Deployment的元数据  name: my-go-appspec:  replicas: 1 # ReplicaSet部分的定义  selector:    matchLabels:      app: go-app  template: # Pod 模板的定义    metadata:      labels:        app: go-app    spec: # Pod里容器相关的定义      containers:        - name: go-app-container          image: kevinyan001/kube-go-app          resources:            limits:              memory: "128Mi"              cpu: "100m"          ports:            - containerPort: 3000
```

-   apiVersion 声明了对象的API版本，Kubernetes会去对应的包里加载库文件。
    
-   kind声明对象的种类，其实就是告诉Kubernetes去加载什么对象。
    
-   metadata就是我们这个对象的元数据。
    
-   spec.replicas 定义副本集有多少个Pod副本，而spec.selectors则是副本集匹配Pod的规则。
    
-   spec.template是Pod模板的定义，其中的内容就是一个完整的Pod对象的定义。
    
-   spec.template.spec是关于Pod里容器相关的定义。
    

具体里面每个字段的意思和用途我就不多说了，前面的文章里都讲过，重点强调一下容器配置里**limits.memory**的`128Mi`代表的是内存分配给容器128兆，而**limits.cpu**的1000m = 1核心。100m就是分配给容器0.1核，这个在自己电脑上实践的时候尽量别分配太大，不然根本启动不起来。

写好声明文件后，使用`kubectl create`命令创建`Deployment`对象，`Kubernetes`里所有的API对象都是这么创建的。

```auto
➜  kubectl create -f deployment.yaml --recorddeployment.apps/my-go-app created➜   
```

对于在笔记本上实践的同学，需要先安装Minikube，具体的安装步骤可以参考：[Minikube-运行在笔记本上的Kubernetes集群](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485235&idx=1&sn=0cdd31c25b13790b336e0a8222f00b64&chksm=fa80d6a4cdf75fb27a64c7f06f9a33ed4e644624d3fcf73d791274d03b071467ebb4e292ac95&token=1401073673&lang=zh_CN&scene=21#wechat_redirect)。

在继续使用`Deployment`进行更高级的编排工作前，我们先用下面两个命令确保一下`Deployment`的运行状态：

-   **kubectl rollout status deployment** 告诉我们`Deployment`对象的状态变化。
    
    ```auto
    ➜ kubectl rollout status deployment my-go-appdeployment "my-go-app" successfully rolled out
    ```
    
-   **kubectl get deployment** 显示期望的副本数和正在更新的副本数，以及当前可提供服务的`Pod`数量。因为我们在定义里只指定了一个副本，所以当前只有一个`Pod`。
    
    ```auto
    kubectl get deployment my-go-appNAME        READY   UP-TO-DATE   AVAILABLE   AGEmy-go-app   1/1     1            1           13m
    ```
    
-   **kubectl get replicaset** 查看`Deployment`为Pod创建的`ReplicaSet`的状态。
    
    ```auto
    kubectl get replicaset          NAME                   DESIRED   CURRENT   READY   AGEmy-go-app-864496b67b   1         1         1       19m
    ```
    
    默认情况下，Deployment会将pod-template-hash添加到它创建的`ReplicaSet`的名称中。比如这里的**my-go-app-864496b67b**
    
-   最后 **kubectl get pod** 命令可以查看`ReplicaSet`创建出来的`Pod`副本的状态。
    
    ```auto
    NAME                         READY   STATUS             RESTARTS   AGEmy-go-app-864496b67b-ctkf9   1/1     Running            0          25m
    ```
    

## 使用Deployment滚动更新应用

`Deployment` 通过**"控制器模式"**，来操作`ReplicaSet` 的个数和属性，进而实现**"水平扩展 / 收缩"** 和 **"滚动更新"** 这两个编排动作。

### 水平扩展/收缩

"水平扩展 / 收缩"非常容易实现，`Deployment` 只需要修改它所控制的`ReplicaSet` 的 `Pod` 副本个数就可以了。比如，把这个值从 1 改成 3，那么 `Deployment` 所对应的 `ReplicaSet`，就会根据修改后的值自动创建两个新的`Pod`，"水平收缩"则反之。这个操作的指令也非常简单，就是 **kubectl scale**，比如：

```auto
➜ kubectl scale --replicas=3 deployment my-go-app --recorddeployment.apps/my-go-app scaled
```

如果你手快点还能通过上面说的命令 **kubectl rollout status deployment my-go-app** 看到扩展过程中`Deployment`对象的状态变化：

```auto
kubectl rollout status deployment my-go-app    Waiting for deployment "my-go-app" rollout to finish: 1 of 3 updated replicas are available...Waiting for deployment "my-go-app" rollout to finish: 2 of 3 updated replicas are available...deployment "my-go-app" successfully rolled out
```

可以通过下面的命令观察到ReplicaSet的Name没有发生变化：

```auto
➜   kubectl get replicaset                         NAME                   DESIRED   CURRENT   READY   AGEmy-go-app-864496b67b   3         3         3       53m
```

这证明了 `Deployment`水平扩展和收缩副本集是不会创建新的ReplicaSet的，但是涉及到Pod模板的更新后，比如更改容器的镜像，那么Deployment会用创建一个新版本的ReplicaSet用来替换旧版本。

### 滚动更新

在上面的`Deployment`定义里，Pod模板里的容器镜像设置的是**kevinyan001/kube-go-app**，接下来比如我们的`Go`项目代码更新了，用最新的代码打包了镜像 **kevinyan001/kube-go-app:v0.1**，部署Go项目的新镜像的过程就会触发`Deployment`的滚动更新。

有两种方式更新镜像，一种是更新`deployment.yaml`里的镜像名称，然后执行 **kubectl apply -f deployment.yaml**。一般公司里的`Jenkins`等持续继承工具用的就是这种方式。还有一种就是使用**kubectl set image** 命令，为了方便演示我们这里就是用第二种方式进行`Pod`的滚动更新。

```auto
➜  kubectl set image deployment my-go-app go-app-container=kevinyan001/kube-go-app:v0.1 --recorddeployment.apps/my-go-app image updated
```

执行滚动更新后通过命令行查看`ReplicaSet`的状态会发现`Deployment`用新版本的`ReplicaSet`对象替换旧版本对象的过程。

```auto
➜  kubectl get replicaset                                                                NAME                   DESIRED   CURRENT   READY   AGEmy-go-app-6749dbc697   3         3         2       19smy-go-app-864496b67b   1         1         1       72m➜  kubectl get replicasetNAME                   DESIRED   CURRENT   READY   AGEmy-go-app-6749dbc697   3         3         3       24smy-go-app-864496b67b   0         0         0       72m
```

通过这个Deployment的Events可以查看到这次滚动更新的详细过程：

```auto
➜  kubectl describe deployment my-go-appName:                   my-go-appNamespace:              defaultCreationTimestamp:      Sat, 29 Aug 2020 00:31:56 +0800Events:.....  Normal  ScalingReplicaSet  37h                deployment-controller  Scaled up replica set my-go-app-6749dbc697 to 1  Normal  ScalingReplicaSet  37h                deployment-controller  Scaled down replica set my-go-app-864496b67b to 2  Normal  ScalingReplicaSet  37h                deployment-controller  Scaled up replica set my-go-app-6749dbc697 to 2  Normal  ScalingReplicaSet  37h (x2 over 37h)  deployment-controller  Scaled down replica set my-go-app-864496b67b to 1  Normal  ScalingReplicaSet  37h                deployment-controller  Scaled up replica set my-go-app-6749dbc697 to 3  Normal  ScalingReplicaSet  37h                deployment-controller  Scaled down replica set my-go-app-864496b67b to 0
```

当你修改了`Deployment`里的`Pod`定义之后，`Deployment` 会使用这个修改后的 `Pod` 模板，创建一个新的 `ReplicaSet`（hash=6749dbc697），这个新的`ReplicaSet` 的初始`Pod`副本数是：0。然后`Deployment` 开始将这个新的`ReplicaSet`所控制的`Pod` 副本数从 0 个变成 1 个，即：**"水平扩展"**出一个副本。紧接着`Deployment`又将旧的 `ReplicaSet`（hash=864496b67b）所控制的旧 Pod 副本数减少一个，即：**"水平收缩"**成两个副本。如此交替进行就完成了这一组`Pod` 的版本升级过程。像这样，将一个集群中正在运行的多个 `Pod` 版本，交替地逐一升级的过程，就是 **"滚动更新"**。

用示意图描述这个过程的话就像下图这样

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/z4pQ0O5h0f5Iw7YXQNvoPR2rp5aPupfmYzKC1dOxzLoRTQu1j5HL8ZYkGUY2hMibpwKO6plawDHkPgLesXPu3lQ/640?wx_fmt=jpeg)

Deployment滚动更新的过程

为了保证服务的连续性，`Deployment` 还会确保，在任何时间窗口内，只有指定比例的`Pod` 处于离线状态。同时，它也会确保，在任何时间窗口内，只有指定比例的新 `Pod` 被创建出来。这两个比例的值都是可以配置的，默认都是期望状态里`spec.relicas`值的 25%。所以，在上面这个 `Deployment` 的例子中，它有 3 个 `Pod` 副本，那么控制器在“滚动更新”的过程中永远都会确保至少有 2 个`Pod` 处于可用状态，至多只有 4 个 `Pod` 同时存在于集群中。这个策略可以通过`Deployment` 对象的一个字段，**RollingUpdateStrategy**来设置：

```auto
apiVersion: apps/v1kind: Deployment...spec:...  strategy:    type: RollingUpdate    rollingUpdate:      maxSurge: 1      maxUnavailable: 1
```

## 回滚Deployment版本

上面执行变更命令的时候都使用了**\--record** 参数，这个参数能让`Kubernetes`在这个`Deployment`的变更记录里记录上产生变更当时执行的命令。

执行**kubectl rollout history deployment my-go-app** 就能看到这个`Deployment`的更新记录：

```auto
➜  kubectl rollout history deployment my-go-app                                   deployment.apps/my-go-app REVISION  CHANGE-CAUSE1         kubectl scale deployment my-go-app --replicas=3 --record=true2         kubectl set image deployment my-go-app go-app-container=kevinyan001/kube-go-app:v0.1 --record=true
```

假如刚才那个滚动更新的Go项目镜像有问题，我们想回退到以前的版本。借助**\--record**参数帮我们记录的执行命令和更新记录里的修订号就可以找到想要回滚的版本修订号。

一旦确定了修订号后我们**kubectl rollout undo**命令就能完成`Deployment`对象的版本回滚。

```auto
kubectl rollout undo  deployment my-go-app --to-revision=1deployment.apps "my-go-app"
```

执行完后我们会发现一个非常有意思的事情，以前那个版本的`ReplicaSet`(hash=864496b67b)的Pod的数又变回了3，新`ReplicaSet`(hash=6749dbc697)的Pod数变成了0。

```auto
➜ kubectl get rsNAME                   DESIRED   CURRENT   READY   AGEmy-go-app-6749dbc697   0         0         0       3m33smy-go-app-864496b67b   3         3         3       4m30s
```

证明`Deployment`在上次滚动更新后并不会把旧版本的`ReplicaSet`删掉，而是留着回滚的时候用，所以`ReplicaSet`相当于一个基础设施层面的应用的版本管理。

回滚后在看变更记录，发现已经没有修订号1的内容了，而是多了修订号为3的内容，这个版本的变更内容其实就是回滚前修订号1里的变更内容。

```auto
➜ kubectl rollout history deployment my-go-app   deployment.apps/my-go-app REVISION  CHANGE-CAUSE2         kubectl set image deployment my-go-app go-app-container=kevinyan001/kube-go-app:v0.1 --record=true3         kubectl scale deployment my-go-app --replicas=3 --record=true
```

## 控制ReplicaSet的版本数量

你可能已经想到了一个问题：我们对`Deployment` 进行的每一次更新操作，都会生成一个新的`ReplicaSet` 对象，是不是有些多余，甚至浪费资源？所以，Kubernetes 项目还提供了一个指令，使得我们对 Deployment 的多次更新操作，最后只生成一个`ReplicaSet`对象。具体的做法是，在更新`Deployment`前，你要先执行一条 **kubectl rollout pause** 指令。它的用法如下所示：

```auto
➜ kubectl rollout pause deployment my-go-appdeployment.apps/my-go-app paused
```

这个命令的作用，是让这个`Deployment`进入了一个"暂停"状态。由于此时`Deployment`正处于“暂停”状态，所以我们对`Deployment`的所有修改，都不会触发新的“滚动更新”，也不会创建新的`ReplicaSet`。而等到我们对 `Deployment` 修改操作都完成之后，只需要再执行一条 **kubectl rollout resume** 指令，就可以把这个 它恢复回来，如下所示：

```auto
➜ kubectl rollout resume deployment my-go-appdeployment.apps/my-go-app resumed
```

随着应用版本的不断增加，`Kubernetes`会为同一个`Deployment`保存很多不同的`ReplicaSet`。`Deployment` 对象有一个字段，叫作 **spec.revisionHistoryLimit**，就是 `Kubernetes` 为 `Deployment` 保留的"历史版本"个数。如果把它设置为 0，就再也不能做回滚操作了。

## 总结

`Kubernetes` 项目对 `Deployment` 的设计，代替我们完成了对**应用**的抽象，让我们可以用一个`Deployment` 对象来描述应用，使用 **kubectl rollout** 命令控制应用的版本。

`Deployment` 还会保证服务的连续性，确保滚动更新时在任何时间窗口内，只有指定比例的`Pod` 处于离线状态，同时也只有指定比例的新 `Pod` 被创建出来，这样就保证了服务能平滑更新。用`Go`写的`HTTP`服务举例子来说，我们不需要再在代码里自己实现`HTTP Server`平滑重启的功能，因为这些功能都由`Deployment`在应用抽象层面替我们实现了。
