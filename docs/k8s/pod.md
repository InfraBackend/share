`Pod`是`Kubernetes`项目里定义的最小可调度单元，是`Kubernetes`对应用程序的抽象。

## 什么是Pod

在`Kubernetes`的`API`对象模型中，`Pod`是最小的`API`对象，换一个专业点的的说法可以这样描述：`Pod`，是 `Kubernetes` 的原子调度单位。在集群中，`Pod`表示正在运行的应用进程。`Pod`的内部可以有一个或多个容器，同属一个`Pod`的容器将会共享：

-   网络资源
    
-   相同的IP
    
-   存储
    
-   应用到Pod上的自定义配置
    

可以看到`Pod`是`Kubernetes`定义出来的一个逻辑概念，可以用另外一种方式来理解`Pod`：一种特定于应用程序的“逻辑主机”，其中包含一个或多个紧密协作的容器。例如，假设我们在Pod中有一个应用程序容器和一个日志记录容器。日志记录容器的唯一工作是从应用程序容器中提取日志。将两个容器放置同一个`Pod`里可消除额外的通信时间，因为它们位于同一个"主机"，因此所有内容都是本地的并且它们共享所有资源，就跟在同一台物理服务器上执行这些操作一样。

此外也不是所有有“关联”的容器都属于同一个`Pod`。比如，应用容器和数据库虽然会发生访问关系，但并没有必要、也不应该部署在同一台机器上，它们更适合做成两个`Pod`。

## Pod的模型

根据`Pod`里的容器数量可以将`Pod`分为两种类型：

-   单容器模型。由于`Pod`是`Kubernetes`可识别的最小对象，`Kubernetes`管理调度`Pod`而不是直接管理容器，所以即使只有一个容器也需要封装到`Pod`里。
    
-   多容器模型。在这个模型中，`Pod`可以容纳多个紧密关联的容器以共享`Pod`里的资源。这些容器作为单一的，凝聚在一起的服务单元工作。
    

每个`Pod`运行应用程序的单个实例。如果需要水平扩展/缩放应用程序（例如运行多个副本），则可以为每个实例使用一个`Pod`。这与在单个`Pod`中运行同一应用程序的多个容器不同。

还需要提的一点是，`Pod`本身不具有调度功能。如果所在的节点发生故障或者你要维护节点，则`Pod`是不会自动调度到其他节点了。`Kubernetes`用一系列控制器来解决`Pod`的调度问题，`Deployment`就是最基础的控制器。通常我们都是在定义的控制器的配置里通过`PodTemplate`定义要控制的`Pod`，让控制器和所管控的`Pod`一起被创建出来（这部分内容后面单独写文章讨论）。

## Pod生命周期的阶段

一个`Pod`的状态会告诉我们它当前正处于生命周期的哪个阶段，`Pod`的生命周期有5个阶段：

-   Pending：等待状态表明至少有一个`Pod`内的容器尚未创建。
    
-   Running：所有容器已经创建完成，并且`Pod`已经被调度到了一个Node上。此时`Pod`内的容器正在运行，或者正在启动或重新启动。
    
-   Succeeded：`Pod`中的所有容器均已成功终止，并且不会重新启动。
    
-   Faild: 所有容器均已终止，至少有一个容器发生了故障。失败的容器以非零状态退出。
    
-   Unknown：无法获得`Pod`的状态。
    

## 在实践中使用Pod

我们已经讨论了`Pod`在理论上的含义，现在让我们看看它在实际中长什么样。我们将首先浏览一个简单的`Pod`定义`YAML`文件，然后部署一个示例应用程序来展示如何使用它。

### Pod的YAML文件

`Kubernetes`里所有的`API`对象都由四部分组成：

-   apiVersion -- 当前使用的`Kubernetes`的API版本。
    
-   kind -- 你想创建的对象的种类。
    
-   metadata -- 元数据，用于唯一表示当前的对象，比如name、namespace等。
    
-   spec -- 我们的`Pod`的指定配置，例如镜像名称，容器名称，数据卷等。
    

`apiVersion`，`kind`和`metadata`是必填字段，适用于所有`Kubernetes`对象，而不仅仅是`pod`。`spec`里指定的内容（`spec`也是必需字段）会因对象而异。下面的示例显示了Pod的YAML文件大概长什么样子。

```auto
apiVersion: "api version"             kind: "object to create"                 metadata:                     name: "Pod name"  labels:    app: "label value"spec:                         containers:  - name: "container name"    image: "image to use for container"
```


理解了`Pod`配置文件的模板后，接下来我们看看如何使用配置文件创建上面说的两种模型的`Pod`。

### 单容器Pod

下面的`pod-1.yaml`是个单容器`Pod`的清单文件。它会运行一个`Nginx`容器。

```auto
apiVersion: v1kind: Podmetadata:  name: first-pod  labels:    app: myappspec:  containers:  - name: my-first-pod    image: nginx
```

接下来，我们通过运行`Kubectl create -f pod-1.yaml`将清单文件部署到本地的`Kubernetes`集群中。然后，我们运行`kubectl get pods`以确认我们的`Pod`运行正常。

```auto
kubectl get podsNAME                                          READY     STATUS    RESTARTS   AGEfirst-pod                                      1/1       Running   0          45s
```

可以到`Pod`的`Nginx`容器里执行以下`service nginx status`命令确保`Nginx`在正常运行。

```auto
kubectl exec first-pod -- service nginx statusnginx is running.
```

这会在`Pod`里执行`service nginx status`指令，类似`docker exec`命令。

现在，我们通过运行`kubectl delete pod first-pod`删除刚才创建的`Pod`。

```auto
kubectl delete pod first-podpod "firstpod" deleted
```

### 多容器Pod

下面我们将部署一个更复杂的`Pod`：一个拥有两个容器的`Pod`，这些容器相互协作作为一个实体工作。其中一个容器每10秒将当前日期写入一个文件，而另一个`Nginx`容器则为我们展示这些日志。这个`Pod`的`YAML`如下：

```auto
apiVersion: v1kind: Podmetadata:  name: multi-container-pod # pod的名称spec:  volumes:  - name: shared-date-logs  # 为Pod里的容器创建一个共享数据卷    emptyDir: {}  containers:  - name: container-writing-dates # 第一个容器的名称    image: alpine # 容器的镜像    command: ["/bin/sh"]    args: ["-c", "while true; do date >> /var/log/output.txt; sleep 10;done"] # 每10秒写入当前时间    volumeMounts:    - name: shared-date-logs      mountPath: /var/log # 将数据卷挂在到容器的/var/log目录  - name: container-serving-dates # 第二个容器的名字    image: nginx:1.7.9 # 容器的镜像    ports:      - containerPort: 80 # 定义容器提供服务的端口    volumeMounts:    - name: shared-date-logs      mountPath: /usr/share/nginx/html # 将数据卷挂载到容器的/usr/share/nginx/html 
```

上面通过`volumes`指令定义了Pod内的数据卷

```auto
  volumes:  - name: shared-date-logs  # 为Pod里的容器创建一个数据卷    emptyDir: {}
```

第一个容器将数据卷挂载到了`/var/log/`每隔10秒往`output.txt`文件里写入时间，而第二个容器通过将数据卷挂载到`/usr/share/nginx/html`伺服了这个日志文件。

执行`kubectl create -f pod-2.yaml`创建这个多容器`Pod`：

```auto
kubectl create -f pod-2.yamlpod "multi-container-pod" created
```

然后确保`Pod`已经正确部署：

```auto
kubectl get podsNAME                                          READY     STATUS    RESTARTS   AGEmulti-container-pod                           2/2       Running   0          1m
```

通过运行`kubectl describe pod podName`，查看Pod的详细信息，里面会包含两个容器的信息。（下面的内容只截取了容器相关的信息）

```auto
Containers:  container-writing-dates:    Container ID:  docker://e5274fb901cf276ed5d94b...    Image:         alpine    Image ID:      docker-pullable://alpine@sha256:621c2f39...    Port:              Host Port:         Command:      /bin/sh    Args:      -c      while true; do date >> /var/log/output.txt; sleep 10;done    State:          Running      Started:      Sat, 1 Aug 2020 11:31:44 +0800    Ready:          True    Restart Count:  0    Environment:        Mounts:      /var/log from shared-date-logs (rw)      /var/run/secrets/Kubernetes.io/serviceaccount from default-token-8dl5j (ro)    container-serving-dates:    Container ID: docker://f9c85f3fe3...    Image:          nginx:1.7.9    Image ID:       docker-pullable://nginx@sha256:e3456c851...    Port:           80/TCP    Host Port:      0/TCP    State:          Running      Started:      Sat, 1 Aug 2020 11:31:44 +0800    Ready:          True    Restart Count:  0    Environment:        Mounts:      /usr/share/nginx/html from shared-date-logs (rw)      /var/run/secrets/Kubernetes.io/serviceaccount from default-token-8dl5j (ro)
```

两个容器都在运行，下面我们将进到`Pod`里确保两个容器都在执行分配的作业。

通过运行`kubectl exec -it multi-container-pod -c container-serving-dates -- bash`连接到`Nginx`容器里。

在容器内运行`curl'http://localhost:80/output.txt'`，它应该返回时间日志文件的内容给我们。（如果容器中未安装curl，请先运行`apt-get update && apt-get install curl`，然后再次运行`curl'http://localhost:80/output.txt'`。）

```auto
curl 'http://localhost:80/app.txt'Sat Aug 1  11:31:44 CST 2020Sat Aug 1  11:31:54 CST 2020Sat Aug 1  11:32:04 CST 2020
```

## SideCar模式

除了上面说的那些之外，我们可以在一个`Pod`中按照顺序启动一个或多个辅助容器，来完成一些独立于主进程（主容器）之外的工作，完成工作后这些辅助容器会依次退出，之后主容器才会启动，这种容器设计模式叫做`sidecar`。

比如对于前端`Web`应用，如果把构建后的`Js`项目放到`Nginx`镜像的`/usr/share/nginx/html`目录下，`Nginx`和`Js`应用做成一个镜像运行容器，每次应用有更新或者`Nginx`要做升级、更新配置操作都需要重新做一个镜像，非常麻烦。

有了`Pod`之后，这样的问题就很容易解决了。我们可以把前端`Web`应用和`Nginx`分别做成镜像，然后把它们作为一个`Pod`里的两个容器"组合"在一起。这个`Pod`的配置文件如下所示：

```auto
apiVersion: v1kind: Podmetadata:  name: web-2spec:  initContainers:  - image: kevinyan/front-app:v2    name: front    command: ["cp", "/www/application/*", "/app"]    volumeMounts:    - mountPath: /app      name: app-volume  containers:  - image: nginx:1.7.9    name: nginx    ports:      - containerPort: 80 # 定义容器提供服务的端口    volumeMounts:    - mountPath: /usr/share/nginx/html      name: app-volume  volumes:  - name: app-volume    emptyDir: {}
```

所有`spec.initContainers`定义的容器，都会比`spec.containers`定义的用户容器先启动。并且，`Init`容器会按顺序逐一启动，直到它们都启动并且退出了，用户容器才会启动。所以，这个`Init`类型的容器启动后，执行了一句"cp /www/application/\* /app"，把应用包拷贝到"/app"目录下，然后退出。这个"/app"目录，挂载了一个名叫`app-volume` 的`Volume`。接下来`Nginx`容器，同样声明了挂载`app-volume`到自己的"/usr/share/nginx/html"目录下。由于这个`Volume` 是被`Pod`里的容器共享的所以等`Nginx`容器启动时，它的目录下就一定会存在前端项目的文件。这个文件正是上面的`Init`容器启动时拷贝到`Volume`里面的。

这就是容器设计模式里最常用的一种模式：`sidecar`。顾名思义，`sidecar`指的就是我们可以在一个`Pod`中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。


使用`Kubernetes`的主要好处之一是它具有管理和维护集群中容器的能力，几乎可以提供服务零停机时间的保障。在创建一个`Pod`资源后，`Kubernetes`会为它选择worker节点，然后将其调度到节点上运行`Pod`里的容器。`Kubernetes`强大的功能可使应用程序的容器保持连续运行，还可以根据需求的增长自动扩展系统。除此之外在`Pod`或容器出现故障时`Kubernetes`还可以让系统实现"自愈"。在本文中，我们将介绍如何使用`Kubernetes`内置的`livenessProbe`和`readinessProbe`来管理和控制应用程序的运行状况。

## Pod的重启策略

`Kubernetes`自身的系统修复能力有一部分是需要依托`Pod`的重启策略的， 重启策略也叫`restartPolicy`。它是`Pod` 的`Spec`部分的一个标准字段（pod.spec.restartPolicy），默认值是Always，即：任何时候这个容器发生了异常，它一定会被重新创建。

可以通过设置 restartPolicy，改变 Pod 的恢复策略。除了 Always，它还有 OnFailure 和 Never 两种情况。

-   Always：在任何情况下，只要容器不在运行状态，就自动重启容器；
    
-   OnFailure: 只在容器 异常时才自动重启容器；
    
-   Never: 从来不重启容器。
    

在实际使用时，我们需要根据应用运行的特性，合理设置这三种恢复策略。

对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数，比如：

```auto
$ kubectl get pod test-liveness-execNAME           READY     STATUS    RESTARTS   AGEliveness-exec   0/1       Running   1          1m
```

如果一个 Pod 里只有一个容器，然后这个容器异常退出了。那么，只有当 restartPolicy=Never 时，这个 Pod 才会进入 Failed 状态。而其他情况下，由于 Kubernetes 都可以重启这个容器，所以 Pod 的状态保持`Running` 不变，RESTARTS信息统计了`Pod`的重启次数。需要注意的是：虽然是重启，但背后其实是`Kubernetes`用重新创建的容器替换了旧容器。

## Pod怎么实现自我修复？

将`Pod`调度到某个节点后，该节点上的Kubelet将运行其中的容器，并在`Pod`的生命周期内保持它们的运行。如果容器的主进程崩溃，`kubelet`将重新启动容器。但是，如果容器内的应用程序抛出错误导致其不断重启，则`Kubernetes`可以通过使用正确的诊断程序并遵循`Pod`的重启策略来对其进行修复。

`Kubernetes`可以对两种健康检查做出应对：

-   Liveness：活性检查，`kubelet`使用活性探针（livenessProbe）的返回状态作为重新启动容器的依据。一个Liveness探针用于在应用运行时检测容器的问题。容器进入此状态后，`Pod`所在节点的`kubelet`可以通过`Pod`策略来重启容器。
    
-   Readiness：就绪检查，这种类型的探测（readinessProbe）用于检测容器是否准备好接受流量。你可以使用这种探针来管理哪些`Pod`会被用作服务的后端。如果`Pod`尚未准备就绪，则将其从服务的后端列表中删除。
    

`Kubernetes`把放在`Pod`里的健康检查处理程序叫做探针（Probe），比喻成医学手术上探测病变的探针，还是很形象的。

## 探针处理程序

为了使健康检查能够对`Pod`的运行状况进行诊断，`kubelet`会调用容器中为探针实现的处理程序，这些处理程序分为三大类：

-   Exec：在容器内执行命令。
    
-   TCPSocket：对指定端口上，容器的IP地址执行TCP检查。
    
-   HTTPGet：在容器的IP上执行HTTP GET请求。
    

处理程序的返回状态也分为三种：

-   Success：容器通过诊断。
    
-   Failed：容器无法通过诊断。
    
-   Unknown：诊断失败，状态不确定，将不采取任何措施。
    

聊完了探针程序的种类和返回值接下来我们来了解一下这两种探针的使用案例。

## 使用案例

活性和就绪探针都在`Pod`的`YAML`文件中配置。每种类型都有不同的用例。

### livenessProbe

如前所述，活性探针用于诊断不健康的容器。他们可以在服务无法继续进行时检测到服务中的问题，并会根据其重启策略重启有问题的容器，期望通过这种方式来解决服务的问题。

例如，在容器内包含一个Exec活性探针，以检测应用程序何时转换为`Broken`状态。

```auto
apiVersion: v1kind: Podmetadata:  labels:    test: liveness  name: liveness-execspec:  containers:  - name: liveness    image: k8s.gcr.io/busybox    args:    - /bin/sh    - -c    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600    livenessProbe:      exec:        command:        - cat        - /tmp/healthy      initialDelaySeconds: 5      periodSeconds: 5
```

这个`YAML`定义一个单容器的`Pod`。它表示`kubelet`在容器启动完成后5秒进行第一次健康检查（initialDelaySeconds：5），之后每5秒都会执行一次检查（periodSeconds: 5）。`kubelet`在容器中执行命令"cat/tmp/healthy"，如果成功，则返回0，指示它是健康的。如果返回非零值，则`kubelet`将kill掉该容器并重新启动它。

这个例子是官方教程里给的，除了在容器中执行命令外，发起 HTTP 或者 TCP 请求的livenessProbe，教程里也给出了示例：

```auto
...    livenessProbe:      httpGet:        path: /healthz        port: 8080        httpHeaders:        - name: Custom-Header          value: Awesome      initialDelaySeconds: 3      periodSeconds: 3
```

```auto
...    livenessProbe:      tcpSocket:        port: 8080      initialDelaySeconds: 15      periodSeconds: 20
```

在官方教程里可以找到更多示例，链接：https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

### readinessProbe

就绪探针与活性探针（GET请求，TCP连接和命令执行）进行相同类型的检查。但是，纠正措施有所不同。它不会重启未通过检查的容器的`Pod`，而是从`Service`上摘除`Pod`，暂时将其与流量隔离。

比如，有一个`Pod`可能正在做大量计算或正在进行繁重的操作，从而增加了服务的响应延迟。让我们定义一个就绪探针，通过探针的timeoutSeconds参数定义超过两秒响应GET请求的`Pod`的状态为非健康状态

```auto
apiVersion: v1kind: Podmetadata: name: nodedelayedspec: containers:   - image: k8s.gcr.io/liveness     name: liveness     ports:       - containerPort: 8080         protocol: TCP     readinessProbe:       httpGet:         path: /         port: 8080       timeoutSeconds: 2
```

当探测响应时间超过两秒钟时，kubelet不会重新启动Pod，而是将流量路由到其他健康的`Pod`上。等到`Pod`不再过载后，`kubelet`会将`Pod`重新加回到原来的`Service`中。

## 总结

默认情况下，`Kubernetes`提供两种健康检查：readinessProbe 和 livenessProbe。它们都使用相同类型的探针处理程序（HTTP GET请求，TCP连接和命令执行）。他们对未通过检查的`Pod`做出的纠错措施有所不同。livenessProbe将重新启动容器，预期重启后错误不再发生。readinessProbe会将Pod与流量隔离，直到故障原因消失。

通过在同一个`Pod`中使用这两种健康检查，可以确保流量不会到达尚未准备就绪的`Pod`，并且确保`Pod`在发生故障时能重新启动。

良好的应用程序设计应同时记录足够的信息，尤其是在引发异常时。它还应公开必要的API端点，这些端点将会传达重要的运行状况和状态指标，以供监控系统（如Prometheus）使用。


