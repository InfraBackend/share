

## ReplicaSet是什么

光从`ReplicaSet`这个控制器的名字（副本集）也能想到它是用来控制副本数量的，这里的每一个副本就是一个`Pod`。`ReplicaSet`它是用来确保我们有指定数量的`Pod`副本正在运行的`Kubernetes`控制器，意在保证系统当前正在运行的`Pod`数等于期望状态里指定的`Pod`数目。

**一般来说，`Kubernetes`建议使用`Deployment`控制器而不是直接使用`ReplicaSet`，`Deployment`是一个管理`ReplicaSet`并提供`Pod`声明式更新、应用的版本管理以及许多其他功能的更高级的控制器。所以`Deployment`控制器不直接管理`Pod`对象，而是由 `Deployment` 管理`ReplicaSet`，再由`ReplicaSet`负责管理`Pod`对象。**

明白了这个逻辑后就明白为什么要在学习`Deployment`前先来了解`ReplicaSet`这个控制器对象了。

## ReplicaSet 怎么管理Pod

`ReplicaSet`会通过标签选择器（Label-Selector）管理所有带有与选择器匹配的标签的容器。创建`Pod`时，它会认为所有`Pod` 是一样的，是无状态的，所以在创建顺序上不会有先后之分。如果使用相同的标签选择器创建另一个`ReplicaSet`，则之前的`ReplicaSet`会认为是它创建了这些`Pod`，会触发控制循环里的逻辑删掉多余的`Pod` ，新的`ReplicSet`又会再次创建`Pod`。双方的当前状态始终不等于期望状态，这就会引发问题，因此确保`ReplicaSet`标签选择器的唯一性这一点很重要。

## 定义ReplicaSet对象

下面这个配置文件的模板里罗列了定义`ReplicaSet`对象的`YAML`文件内容的主要组成部分：

```auto
apiVersion: apps/v1kind: ReplicaSet   metadata:   name: some-name  labels:    app: some-App    tier: some-Tierspec:   replicas: 3 # 告诉k8s需要多少副本  Selector: # 用于匹配Pod的标签选择器    matchLabels:      tier: someTier  template:    metadata:      labels:        app: some-App        tier: someTier    spec: # 与Pod对象的spec定义类似      Containers: 
```

`Kubernetes`里所有的`API`对象都由四部分组成：

-   apiVersion -- 当前使用的`Kubernetes`的API版本。
    
-   kind -- 你想创建的对象的种类。
    
-   metadata -- 元数据，用于唯一表示当前的对象，比如name、namespace等。
    
-   spec -- 当前对象的指定配置。
    

但是`ReplicaSet`的spec部分的定义看起来与其他对象的spec定义略有不同。上面的示例中有两个`spec`字段。第一个`spec`声明的是`ReplicaSet`的属性--定义有多少个`Pod`副本（默认将仅部署1个`Pod`）、匹配`Pod`标签的选择器等。第二个`spec`用于`Pod`里的容器属性等配置。

**实际上" .spec.template"里的内容就是声明`Pod`对象时要定义的各种属性，所以这部分也叫做PodTemplate（Pod模板）**。还有一个值得注意的地方是：在`.spec.selector`中定义的标签选择器必须能够匹配到`spec.template.metadata.labels`里定义的`Pod`标签，否则`Kubernetes`将不允许创建`ReplicaSet`。

## ReplicaSet使用示例

了解了什么是副本集，以及如何编写副本集的声明文件后，接下来我们动手创建一个副本集。在此示例中，我们将通过`replica.yaml`文件创建一个具有3个`Pod`副本的`nginx`应用。

```auto
# replica.yamlapiVersion: apps/v1kind: ReplicaSetmetadata:  name: myapp-replicas  labels:    app: myapp    tier: frontendspec:  replicas: 3  selector:    matchLabels:      tier: frontend    matchExpressions:      - {key: tier, operator: In, values: [frontend]}  template:    metadata:      labels:        app: myapp        tier: frontend    spec:      containers:      - name: nginx        image: nginx        ports:        - containerPort: 80
```

首先我们运行kubectl create命令使用上面定义的声明文件创建对象。

```auto
$ kubectl create -f replica.yamlreplicaset.apps "myapp-replicas" created
```

接下来通过命令确保副本集创建成功

```auto
$ kubectl get replicasetNAME                            DESIRED   CURRENT   READY     AGEmyapp-replicas                  3         3         3         15s
```

上面的命令返回显示了`myapp-replicas`副本集部署了三个副本，三个都是就绪状态。接下来我们再看一下每个`Pod`副本的运行状态：

```auto
$ kubectl get podNAME                            READY     STATUS    RESTARTS   AGEmyapp-replicas-67rkp            1/1       Running   0          33smyapp-replicas-6kfd8            1/1       Running   0          33smyapp-replicas-s96sg            1/1       Running   0          33s
```

所有3个都处于运行状态，重启次数都是0，这意味着目前为止我们的应用程序没有崩溃过。我们还可以**使用`kubectl describe`命令查询副本集对象的详细信息，信息里的`Events`部分详细显示了`ReplicaSet`控制器进行过哪些编排动作。**

```auto
$ kubectl describe replicaset myapp-replicasName:         myapp-replicasNamespace:    defaultSelector:     tier=frontend,tier in (frontend)Labels:       app=myapp              tier=frontendAnnotations:  Replicas:     3 current / 3 desiredPods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 FailedPod Template:  Labels:  app=myapp           tier=frontend  Containers:   nginx:    Image:        nginx    Port:         80/TCP    Host Port:    0/TCP    Environment:      Mounts:         Volumes:        Events:  Type    Reason            Age   From                   Message  ----    ------            ----  ----                   -------  Normal  SuccessfulCreate  12m   replicaset-controller  Created pod: myapp-replicas-6kfd8  Normal  SuccessfulCreate  12m   replicaset-controller  Created pod: myapp-replicas-67rkp  Normal  SuccessfulCreate  12m   replicaset-controller  Created pod: myapp-replicas-s96sg
```

## 总结

总体来说`ReplicaSet`需要掌握的基本概念就这么多，从上面的`YAML`文件中我们可以看到，一个 `ReplicaSet`对象的定义，其实就是由副本数目的定义和一个 Pod 模板组成的。等下篇文章讲到`Deployment`对象时你就会发现，`ReplicaSet`对象定义文件里的内容其实是`Deployment`对象的子集。

实际应用中我们不直接创建使用`ReplicaSet`对象，而是直接使用更高级的控制器对象`Deployment`，由 `Deployment` 管理`ReplicaSet`。

