

-   什么是`Service`对象，在`Kubernetes`里它是干什么用的；
    
-   `Kubernetes`里怎么发现`Service`；
    
-   如何创建和使用`Service`；
    
-   **nodePort，port，targetPort**都是啥；
    

文章前面半部分理论知识多一点，稍显枯燥，后半部分会用一个实践练习给之前用`Deployment`部署好的应用`Pod`们加上`Service`，让外部请求能访问到`Kubernetes`集群里的应用，并为`Pod`提供负载均衡。

## Kubernetes Service

和之前文章里介绍的[Pod](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485464&idx=1&sn=00ca443bbcd4b2996efdede396b6c667&chksm=fa80d98fcdf7509944d63f618264e36cd8082a77e23aa36428a3d57a2f4189bcce4e52986967&token=2051310148&lang=zh_CN&scene=21#wechat_redirect)，[ReplicaSet](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485541&idx=1&sn=59d41bef81615420319b9a721a78ecee&chksm=fa80d9f2cdf750e4250ee59d842501a55375c4c8eacf971eb72adbc3a3cad27259cbd6c27200&token=2051310148&lang=zh_CN&scene=21#wechat_redirect)，[Deployment](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485643&idx=1&sn=6460bf2e170e4b2e8ebb2882bfe7c60f&chksm=fa80d95ccdf7504ad9b5e3ba7ad3dad6a25347a7b0aad4636523cb1ba878cebbc480bf2153a0&token=2051310148&lang=zh_CN&scene=21#wechat_redirect)一样，**Service**也是**Kubernetes**里的一个API对象，而 **Kubernetes** 之所以需要 **Service**，一方面是因为`Pod` 的 IP 不是固定的，另一方面则是因为一组`Pod` 实例需要`Service`提供复杂均衡功能。**所以`Service`是在逻辑抽象层上定义了一组`Pod`，为他们提供一个统一的固定IP和访问这组`Pod`的负载均衡策略**。

下面是`Service`对象的常用属性设置：

-   使用label selector，在集群中查找目标`Pod`;
    
-   ClusterIP设置Service的集群内IP让`kube-proxy`使用;
    
-   通过prot和targetPort将访问端口与目标端口建议映射（不指定targetPort时默认值和port设置的值一样）;
    
-   Service支持多个端口映射
    
-   Service支持HTTP（默认），TCP和UDP协议;
    

下面是一个典型的`Service`定义：

```auto
apiVersion: v1kind: Servicemetadata:  name: hostnamesspec:  selector:    app: hostnames  ports:  - name: default    protocol: TCP    port: 80    targetPort: 9376
```

## 都有哪些类型的Service

`Kubernetes`中有四种Service类型：

-   **ClusterIP**。这是默认的Service类型，会将Service对象通过一个内部IP暴露给集群内部，这种类型的Service只能够在集群内部使用`<ClusterIP>:<port>`访问。
    
-   **NodePort**。会在每个宿主机节点的一个指定的固定端口上暴露Service，与此同时还会自动创建一个ClusterIP类型的Service，NodePort类型的Service会将集群外部的请求路由给ClusterIP类型的Service。你可以使用`<NodeIP>:<NodePort>`访问NodePort类型的Service，NodePort的端口范围为30000-32767。
    
-   **LoadBalancer**。适用于公有云上的`Kubernetes`服务，使用公有云服务的`CloudProvider`创建**LoadBalancer**类型的**Service**，同时会自动创建**NodePort**和**ClusterIP**类型的**Service**，**LoadBalancer**会把请求路由到**NodePort**和**ClusterIP**类型的**Service**上。
    
-   **ExternalName**。ExternalName 类型的 Service，是在 kube-dns 里添加了一条 CNAME 记录。这个CNAME记录是在Service的spec.externalName里指定的，
    

以上四种类型除了`ExternalName`，`Kubernetes`的`kube-proxy`组件都会为`Service`提供VIP（虚拟IP），`kube-proxy`支持两种模式：**iptables**和**ipvs**。涉及到不少知识，感兴趣的可以去极客时间上看这篇文章：Service, DNS与服务发现\[1\]

上面的第三和第四种类型的`Service`在本地试验不了，所以后面的例子我们主要通过`NodePort`类型的`Service`学习它的基本用法。

## 怎么发现Service

在`Kubernetes`里的内部组件`kube-dns`会监控Kubernetes API，当有新的`Service`对象被创建出来后，`kube-dns`会为`Service`对象添加DNS A记录（从域名解析 IP 的记录）

对于 `ClusterIP` 模式的 `Service` 来说，它的 A 记录的格式是:

**serviceName.namespace.svc.cluster.local**，当你访问这条 A 记录的时候，它解析到的就是该 Service 的 VIP 地址。

对于指定了 clusterIP=None 的 Headless Service来说，它的A记录的格式跟上面一样，但是访问记录后返回的是Pod的IP地址集合。Pod 也会被分配对应的 DNS A 记录，格式为：**podName.serviceName.namesapce.svc.cluster.local**

我们会在后面的实践练习里通过`nslookup`印证DNS记录是否符合这里说的格式

## 创建和使用Service

跟其他`Kubernetes`里的API对象，`Service`也是通过`YAML`文件定义然后提交给`Kubernetes`后由`ApiManager`创建完成。一个典型的`NodePort`类型的`Service`的定义如下所示：

```auto
apiVersion: v1kind: Servicemetadata:  name: app-servicespec:  type: NodePort  selector:    app: go-app  ports:    - name: http      protocol: TCP      nodePort: 30080      port: 80      targetPort: 3000
```

这里定义的Service对象会去管控我们在之前的文章《[K8s上的Go服务怎么扩容、发版更新、回滚、平滑重启？教你用Deployment全搞定！](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485643&idx=1&sn=6460bf2e170e4b2e8ebb2882bfe7c60f&chksm=fa80d95ccdf7504ad9b5e3ba7ad3dad6a25347a7b0aad4636523cb1ba878cebbc480bf2153a0&token=2057376102&lang=zh_CN&scene=21#wechat_redirect)》里用`Deployment`创建的`Go`应用的三个`Pod`副本。

```auto
➜ kubectl get pods -l app=go-app NAME                         READY   STATUS    RESTARTS   AGEmy-go-app-864496b67b-6hm7r   1/1     Running   1          16dmy-go-app-864496b67b-d87kl   1/1     Running   1          16dmy-go-app-864496b67b-qxrsr   1/1     Running   1          16d➜ 
```

我们用**kubectl apply -f service.yaml**命令把定义好的`Service`提交给`Kubernetes`：

```auto
➜ kubectl apply -f service.yaml service/app-service created
```

被`Service`的`selector`选中的`Pod`，就称为`Service` 的 `Endpoints`，可以使用 **kubectl get ep** 命令看到它们，如下所示：

```auto
➜  kubectl get ep app-serviceNAME          ENDPOINTS                                         AGEapp-service   172.17.0.6:3000,172.17.0.7:3000,172.17.0.8:3000   8m38s
```

需要注意的是，只有处于`Running`状态，且 [readinessProbe](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485500&idx=1&sn=6197d294cc4f409c2a62a7997c431b68&chksm=fa80d9abcdf750bd6735e7b7481c225d4cbfaad2159427b049d8b229877392f4bef11469363e&token=2057376102&lang=zh_CN&scene=21#wechat_redirect) 检查通过的`Pod`，才会出现在`Service`的 `Endpoints` 列表里。当某一个`Pod`出现问题时，`Kubernetes` 会自动把它从 `Service` 里摘除掉。

使用 **kubectl get svc**可以查看到刚才看到的`Service`的信息和状态。

```auto
➜ kubectl get svcNAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGEapp-service   NodePort    10.108.26.155   <none>        80:30080/TCP   116mkubernetes    ClusterIP   10.96.0.1       <none>        443/TCP        89d
```

## nodePort 、port、targetPort都是啥

上面我们创建了一个`NodePort`类型的`Service`，在下面的端口映射**spec.ports**配置里，每个端口映射里出现了三种**port**：nodePort、port、targetPort。那这三种`port`都代表的什么意思呢？

-   **port**：指定在集群内部暴露`Service` 所使用的端口，集群内部使用`<ClusterIP>:<port>`访问`Service`的`EndPoints` （`Service`选中的Pod）。
    
-   **nodePort**：指定向集群外部暴露`Service` 所使用的端口，从集群外部使用`<NodeIp>:<NodePort>`访问`Service`的`EndPoints`。如果你不显式地声明 `nodePort` 字段，会随机分配可用端口来设置代理。这个端口的范围默认是 30000-32767。
    
-   **targetPort**：`targetPort`是后面的`Pod`监听的端口，容器里的应用也应该监听这个端口，`Service`会把请求发送到这个端口。
    

所以结合刚才我们创建的app-service这个`Service`的信息，在集群内部使用**10.108.26.155:80** 访问`Pod`里的应用。因为我们试验使用的`minikube`是个单节点的集群，`NodeIP`可以通过 **minikube ip**命令获得。

```auto
➜ minikube ip192.168.64.4
```

所以从集群外部，通过**192.168.64.4:30080**访问`Pod`里的应用。

```auto
➜ curl 192.168.64.4:30080Hello WorldHostname: my-go-app-75d6d768ff-mlqnh%                                                                                                                    ➜ curl 192.168.64.4:30080Hello WorldHostname: my-go-app-75d6d768ff-4x8p8%                                                                                                                    ➜  curl 192.168.64.4:30080Hello WorldHostname: my-go-app-75d6d768ff-vt7dx%                                                                                                                    
```

通过多次访问，我们可以看到请求会通过`Service`发给不同的应用`Pod`。`Pod`里的应用就是在原来的文章里一直使用的例子的基础上加了一行获取系统`Hostname`的代码：

```auto
...func index(w http.ResponseWriter, r *http.Request) { fmt.Fprintln(w, "Hello World") hostname, _ := os.Hostname() fmt.Fprintf(w, "Hostname: %s", hostname)}...
```

最后我们进到`Pod`里看一下`Service`创建后`kube-dns`组件在集群里为`app-service`这个`Service`对象创建的DNS A记录，因为`Service`定义里指定的名字是`app-service`，命名空间的话因为没有指定就是默认的`default`命名空间，所以我们使用**nslookup app-service.default.svc.cluster.local** 查看一下这条`DNS`记录，进入到其中一个`Pod`里，执行上述查询的结果如下：

```auto
nslookup app-service.default.svc.cluster.local  Server:         10.96.0.10Address:        10.96.0.10:53Name:   app-service.default.svc.cluster.localAddress: 10.108.26.155
```

对于`Service`的`EndPoints` 也是有`DNS`记录的，因为不是`Headless Service`，所以需要用nslookup \*.app-service.default.svc.cluster.local查询`DNS`记录。

```auto
nslookup *.app-service.default.svc.cluster.local  Server:         10.96.0.10Address:        10.96.0.10:53Name:   *.app-service.default.svc.cluster.localAddress: 172.17.0.8Name:   *.app-service.default.svc.cluster.localAddress: 172.17.0.6Name:   *.app-service.default.svc.cluster.localAddress: 172.17.0.7
```

上面查询出来三条`DNS`记录，正好跟`Service`管控的`Pod`数量能够对上。

## 总结

今天的文章里我结合实例讲述了`Kubernetes`里`Service`对象的基本使用方法和对象本身的一些原理，其实需要计算机网络知识掌握的好才能从更深层次了解各种模式的`Service`的实现原理，这方面的内容推荐极客时间里的专栏文章深入剖析Kubernetes Service\[2\]。

`Kubernetes`支持多种将外部流量引入集群的方法。`ClusterIP`、`NodePort`和`Ingress`是三种广泛使用的资源，它们都在路由流量中发挥作用。每一个都允许您使用一组独特的功能和折衷方案来公开服务。

## 背景

默认情况下，`Kubernetes`上运行的服务都是在自己的 [Pod](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485464&idx=1&sn=00ca443bbcd4b2996efdede396b6c667&chksm=fa80d98fcdf7509944d63f618264e36cd8082a77e23aa36428a3d57a2f4189bcce4e52986967&token=2075750696&lang=zh_CN&scene=21#wechat_redirect) 里过着与世隔绝的生活，外部无法打扰他们。我们可以通过创建  [Service](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&chksm=fa80db15cdf752039494992f71a3bc488cf386841bd1aaaa44115f5e7f155ba55ce468ec89ee&token=2075750696&lang=zh_CN&scene=21#wechat_redirect) 使容器供外部世界可见，这个“外部世界” 即可以整个集群、也可以是整个互联网。

`Service`将流量路由到`Pod`内的容器中。`Service`是一种用于在网络上公开`Pod`的抽象机制。每个`Service`有一个类型——`ClusterIP`、`NodePort`或`LoadBalancer`。这些定义了外部流量如何到达服务。

但是光有`Service`也不行 ，有时候我们需要将不同域名和URL路径上的流量路由到集群内部，这就需要`Ingress`帮助才行。

## ClusterIP

ClusterIP 是默认的`Service`类型，不指定`Type`时默认就是`ClusterIP`类型的`Service`。`ClusterIP`在集群内提供网络连接。它通常无法从外部访问。我们将这些`ClusterIP Service`用于服务之间的内部网络。

```auto
apiVersion: v1kind: Servicespec: metadata:   name: my-service  selector:    app: my-app  type: ClusterIP  ports:    - name: http      port: 80      targetPort: 8080      protocol: TCP
```

上面的示例定义了一个`ClusterIP Service`。到`ClusterIP` 上端口 80 的流量将转发到你的`Pod` 上的端口 8080  (targetPort配置项)，携带 `app: my-app`标签的 Pod 将被添加到 `Service`中作为作为服务的可用端点。

可以通过运行 `kubectl get svc my-service` 查看分配的 IP 地址。集群中的其他服务可以使用 10.96.0.1:80 与这个的 Service 管控的服务进行交互。

```auto
➜  kubectl get svc app-serviceNAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGEmy-service        ClusterIP   10.96.0.1        <none>        8080:80/TCP       63d
```

可以使用 `spec.clusterIp` 字段手动将 ClusterIP 设置为特定 IP 地址：

```auto
spec:  type: ClusterIP  clusterIp: 123.123.123.123
```

## NodePort

`NodePort`在固定端口号上公开向集群外部暴露服务，它允许从集群外部访问该服务，在集群外部需要使用集群的 IP 地址和`NodePort`指定的端口才能访问。创建`NodePort Service`将在集群中的每个`Node`上开放该端口。`Kubernetes`会自动将端口流量路由到它所连接的服务。下面是一个 `NodePort Service` 的示例：

```auto
apiVersion: v1kind: Servicespec: metadata:   name: my-service  selector:    app: my-app  type: NodePort  ports:    - name: http      port: 80      targetPort: 8080      protocol: TCP
```

`NodePort`的定义与`ClusterIP Service`具有相同的属性。唯一的区别是把类型设置成了："NodePort"。`targetPort`字段仍然是必需的，因为`NodePort`由`ClusterIP`提供支持。

创建`NodePort Service`的同时还会自动创建一个`ClusterIP` 类型的`Service`，`NodePort`会将端口上的流量路由给`ClusterIP` 类型的 Service。

这也就是为什么下面我们查看`NodePort Service`时发现他也是有 ClusterIP 的原因：

```auto
➜  kubectl get svc my-serviceNAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGEmy-service        NodePort    10.96.44.244     <none>        8080:30176/TCP    56d
```

使用上述例子创建`NodePort Service`，`Kubernetes`将会从30000-32767这个范围随机分配一个端口作为`NodePort`端口，不过我们可以通过设置`ports.nodePort`字段来手动指定端口：

```auto
spec:  ports:    - name: http      port: 80      targetPort: 8080      nodePort: 32000      protocol: TCP
```

这个会将 32000 端口上的流量通过`Service`最终路由给`Pod`里容器的 8080 端口。

您可以使用`NodePort`快速设置用于开发环境的服务或在其上公开`TCP`或`UDP`服务，但是对于公开`HTTP`服务来说`NodePort`不是一个的理想选择，因为其使用的都是非`HTTP`标准的端口，我们需要使用其他替代方案。

## Ingress

`Ingress` 实际上是与`Service`完全不同的资源，算是`Service`上面的一层代理，通常在 `Service`前使用`Ingress`来提供`HTTP`路由配置。它让我们可以设置外部 URL、基于域名的虚拟主机、SSL 和负载均衡。

给`Service`前面加`Ingress`，你的集群中需要有`Ingress-Controller`才行。有多种控制器可供选择。大多数主要的云提供商都有自己的`Ingress-Controller`，与他们的负载平衡基础设施相集成。如果是自建K8S集群，通常使用`nginx-ingress`作为控制器，它使用`NGINX`服务器作为反向代理来把流量路由给后面的`Service`。

> 关于控制器Nginx-Ingress的安装部署参考：https://kubernetes.github.io/ingress-nginx/deploy/ 后面介绍`Ingress`实践的文章也会再细说。

可以使用`Ingress` 资源类型创建`Ingress`。kubernetes.io/ingress.class 注释可让你指明正在创建的`Ingress`分类。如果集群里安装了多个`Ingress-Controller`这将很有用，也可以将不同的`Service`分别挂在不同分类的`Ingress`下面，增加一些高可用性。

```auto
apiVersion: networking.k8s.io/v1beta1kind: Ingressmetadata:  name: my-ingress  annotations:    kubernetes.io/ingress.class: nginxspec:  rules:    - host: example.com      http:        paths:          - path: /            backend:              serviceName: my-service              servicePort: 80    - host: another-example.com      http:        paths:          - path: /            backend:              serviceName: second-service              servicePort: 80
```

上面定义了两个`Ingress`端点。第一个主机规则将 example.com 流量路由到`my-service`服务上的端口 80。第二条规则将 another-example.com 流量路由到`second-service`。

如果想使用 `HTTPs` 访问服务，可以通过在`Ingress` 规范中设置`tls`字段来配置 `SSL`：

```auto
spec:  tls:    - hosts:      - example.com      - secretName: my-secret
```

不过前提是在集群中需要通过`Secret`对象配置这些域名的证书信息。

当需要处理来自多个域名 和 URL 路径的流量时，应该使用`Ingress`。它让我们可以使用声明性语句配置路由和`Service`。`Ingress`控制器将提供你的路由并将它们映射到服务。
