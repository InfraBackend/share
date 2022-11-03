

> Etcd现在是分布式服务架构中的重要组件，它由 CNCF 孵化托管， 在微服务和 Kubernates 集群中不仅可以作为服务注册与发现，还是一个用于配置共享的分布式键值存储，采用 raft 算法，实现分布式系统数据的可用性和一致性。

一般用Go语言开发的gRPC服务会使用Etcd实现服务发现和注册。此外一些重要的配置也会存储在Etcd里通过让程序监听Key的变更来实现无需重启应用的配置更新。

关于为什么要使用Etcd我们不做过多介绍，现在切入正题。安装Etcd的方式比较多，如果想直接把Etcd集群安装在机器上而不是Kubernetes里可以通过 `goreman` 工具。不过因为我电脑上安装了Minikube，所以想尽量把所有东西都运行在Kubernetes里这样未来换电脑也就不用发愁需要安装那么多工具集了。除了演示在Kubernetes里安装运行Etcd集群外，还会安装一个Etcd的Web UI服务，让我们能够通过浏览器查询和设置Etcd的Key-Value，这个Etcd Web UI服务同样是运行在Kubernetes里，相信通过今天文章的学习你也一定感受到Kubernetes的便捷和学到不少知识。



## Kubernetes 安装Etcd

在Kubernetes里安装Etcd的方法有两种，一种是原始的通过StatefulSet控制器，也就是有状态应用来编排Etcd的节点，这种需要配置Pod使用的镜像，配置文件和启动命令。通过无头服务，在集群内部为Pod提供名称到IP的映射，以及NodePort类型的服务向集群外暴露客户端端口。还有一种是使用coreos提供Etcd Operator直接安装，很多细节都为我们直接处理好了。

在这里我们使用第一种用StatefulSet创建Etcd节点和Service对外暴露客户端端口的安装方式。

### Service 设置DNS和暴露端口

首先我们来创建为Etcd集群的Pod提供Pod名称到IP映射的无头服务。

```auto
---apiVersion: v1kind: Servicemetadata:  name: etcd  namespace: etcd  annotations:    # Create endpoints also if the related pod isn't ready    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"spec:  ports:    - port: 2379      name: client    - port: 2380      name: peer  clusterIP: None  selector:    component: etcd
```

这里要创建的无头服务的名称是 etcd 。因为StatefulSet编排的Pod名称永远是PodName-序号，到时候集群内部各个Etcd节点配置的通信方式就可以用  "PodName-序号.etcd:2380" 这种形式来代替使用"节点IP:2380"。

2380是Etcd服务端的端口，而对外提供服务的客户端端口是2379，因此还需要有一个NodePort类型的Service向集群外部暴露客户端对2379端口的访问。

```auto
---apiVersion: v1kind: Servicemetadata:  name: etcd-client  namespace: etcdspec:  ports:    - name: http      nodePort: 30453      port: 2379      targetPort: 2379      protocol: TCP  type: NodePort  selector:    component: etcd
```

> 创建这两个Service：kubectl apply -f resources/services.yml -n etcd

关于Kubernetes Service 和 StatefulSet 的作用原理、各种配置的详细解释可以参考我公众号里以前的文章：

[Kubernetes Service学习笔记和实践练习](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&chksm=fa80db15cdf752039494992f71a3bc488cf386841bd1aaaa44115f5e7f155ba55ce468ec89ee&token=1423297622&lang=zh_CN&scene=21#wechat_redirect)

[深入理解StatefulSet，用Kubernetes编排有状态应用](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486737&idx=1&sn=e7d0689fa74b108bae734515837c68e1&chksm=fa80dc86cdf755909f2f29ee8cb9dce930b95a837cb045be5c9700a9000743c9cb7f8eed6731&token=1423297622&lang=zh_CN&scene=21#wechat_redirect)

### 配置Etcd节点Pod

我们通过StatefulSet编排创建3个Etcd节点的Pod，创建出来后上面的那两个Service会根据Pod的标签`component=etcd`找到它们，把节点加入到自己的服务端点列表中。

```auto
---apiVersion: apps/v1kind: StatefulSetmetadata:  name: etcd  labels:    component: etcdspec:  serviceName: etcd  replicas: 3  selector:    matchLabels:      component: etcd  template:    metadata:      name: etcd      labels:        component: etcd    spec:      volumes:        - name: etcd-storage          emptyDir: {}      containers:        - name: etcd          image: quay.io/coreos/etcd:latest          ports:            - containerPort: 2379              name: client            - containerPort: 2380              name: peer          volumeMounts:            - name: etcd-storage              mountPath: /var/run/etcd/default.etcd
```

Pod的启动命令里要配置上每个节点点的IP和端口，上面说了无头服务的配置可以通过PodName-序号.etcd 的方式解析出IP，这里对应的就是：

```auto
etcd-0.etcdetcd-1.etcdetcd-2.etcd
```

不过为了灵活性，我参考了国外一位网友分享的方式通过启动时执行shell脚本的方式，动态根据Etcd集群节点数量来设置启动命令里的 PeersUrl等配置

```auto
          env:            - name: CLUSTER_SIZE              value: "3"            - name: SET_NAME              value: "etcd"            - name: MINIKUBE_IP              value: "$MINIKUBE_IP"            - name: MINIKUBE_PORT              value: "$MINIKUBE_PORT"          command:            - /bin/sh            - -ecx            - |              IP=$(hostname -i)              PEERS=""              for i in $(seq 0 $((${CLUSTER_SIZE} - 1))); do                  PEERS="${PEERS}${PEERS:+,}${SET_NAME}-${i}=http://${SET_NAME}-${i}.${SET_NAME}:2380"              done              exec etcd --name ${HOSTNAME} \                --listen-peer-urls http://${IP}:2380 \                --listen-client-urls http://${IP}:2379,http://127.0.0.1:2379 \                --advertise-client-urls http://${HOSTNAME}.${SET_NAME}:2379,http://${MINIKUBE_IP}:${MINIKUBE_PORT} \                --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}:2380 \                --initial-cluster-token etcd-cluster-1 \                --initial-cluster ${PEERS} \                --initial-cluster-state new \                --data-dir /var/run/etcd/default.etcd
```

> StatefulSet的创建命令为：
> 
> ```auto
> cat resources/etcd.yml.tmpl | resources/config.bash | kubectl apply -n etcd -f -
> ```
> 
> 提供了一个shell脚本来设置和MINIKUBE\_PORT 这两个在容器里无法获得的环境变量。

创建Etcd集群所使用的 yaml 资源声明文件和具体的操作步骤都已经放到了Github上，大家可以按照里面的命令进行操作。链接地址：https://github.com/kevinyan815/LearningKubernetes/tree/master/etcd

### 测试安装成果

测试安装是否成功也简单，观测Pod在Kubernetes里都正常启动起来后我们往Etcd里 set 一个键值，看能不能再查询出来就行了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7zouSianPhKLrYbP4VQuZXRiaRzzWBfZyZcYnwfXS88diauj6hEnqC1GdgcTO6ouNiauiauFe5iaDRD7uQ/640?wx_fmt=png)

用etcdctl 测试成果

接下来我们在说说怎么给Etcd做个Web UI，毕竟一些为应用程序预准备的配置如果要用命令行一条条 set 进去的话也太麻烦了。

## 给Etcd集群做个Web UI

做这个Web UI，说来话长，花的时间比上面搭建Etcd集群多多了，大概花了两三天反复尝试才搞定。

Web UI我使用了 e3w 这个项目

> 项目地址：https://github.com/soyking/e3w

原本这个项目是有提供docker镜像和docker-compose容器编排运行的，不过我实在不想再在我电脑上安装这么多工具集了就一直想把它改造成能在 Kubernetes 里运行的方式。

看了下这个项目的源码，启动的时候会去读取 `/app/conf`  目录下的`config.default.ini` 配置文件，WebUI服务的端口号默认配置的是8080，有了这两个信息后我们就可以通过Deployment创建Pod来放置Web UI服务，通过Service暴露Web UI服务供集群外部访问的端口了。

```auto
---apiVersion: apps/v1kind: Deploymentmetadata:  name: e3w-deployment  namespace: etcd  labels:    app: e3wspec:  replicas: 1  selector:    matchLabels:      app: etcd-client-e3w  template:    metadata:      labels:        app: etcd-client-e3w    spec:      containers:        - name: e3w-app-container          image: soyking/e3w:latest          ports:            - name: e3w-server-port              containerPort: 8080---kind: ServiceapiVersion: v1metadata:  name: e3w-service  namespace: etcdspec:  type: NodePort  selector:    app: etcd-client-e3w  ports:    - protocol: TCP      targetPort: e3w-server-port      nodePort: 30081      port: 80
```

至于配置文件，我的设想是把配置放到ConfigMap里，再把ConfigMap里的配置项作为文件挂载在原来配置文件的路径上。这样做的好处有两个：

1.  不需要单独在机器上创建一个具体路径的配置文件，减少了Pod对宿主机上hostPath的依赖。
    
2.  不需要按照e3w项目里推荐的自定义方式那样修改项目里的配置文件重新打Docker镜像才能连接上自己的Etcd集群。
    

下面 ConfigMap 里的 e3w-config.default.ini 就是我们要作为文件挂载到容器里的配置项。

```auto
apiVersion: v1kind: ConfigMapmetadata:  name: e3w-configmap  namespace: etcd  labels:    config: e3w.inidata:  e3w-config.default.ini: |    [app]    port=8080    auth=false    [etcd]    root_key=root    dir_value=    addr=etcd-0.etcd.etcd.svc.cluster.local:2379,etcd-1.etcd.etcd.svc.cluster.local:2379,etcd-2.etcd.etcd.svc.cluster.local:2379    username=    password=    cert_file=    key_file=    ca_file=
```

不过，在这一步上实属费了不少时间，主要是把 ConfigMap 的一个配置项作为文件挂载到容器里除了需要在 volumeMounts.MountPath 上配置完整的目录和文件名外还需要用上 subPath 这个配置。

```auto
    spec:      containers:          volumeMounts:            - name: e3w-configmap-volume              mountPath: /app/conf/config.default.ini              subPath: config.default.ini      volumes:        - name: e3w-configmap-volume          configMap:            name: e3w-configmap            items:              - key: e3w-config.default.ini                path: config.default.ini
```

这又是一个小知识点，关于Volume挂载时 subPath 的应用场景等后面再说（大家想听的话下篇就安排，记得点赞啊）。

配置文件搞定后，再看一下Pod运行的状态，就不再是Error了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7zouSianPhKLrYbP4VQuZXR8INJ1nTK8rUiahJxI8hyZI0LkBUVIKhRfKf79Q9TucWc2fvibpl8wbhw/640?wx_fmt=png)

Etcd & WebUI Pod

上面创建的Etcd集群里的三个基点和e3w的WebUI服务都能正常运行。

通过WebUI我们可以查看Etcd集群的运行状态

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7zouSianPhKLrYbP4VQuZXRBZVVAo8DAyRCxdmnVErPaibWRVCzCnAnhS7VKC7BeeAn7wIKppumSiaw/640?wx_fmt=png)

节点状态界面

以及更方便地通过UI界面管理Key-Value：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7zouSianPhKLrYbP4VQuZXRjlWicMWrk4H8P5wUB8j9mDHeQXksXRMIjEMtpuIKws2nC2h4Vl9rhAw/640?wx_fmt=png)

键值管理界面

整体上感觉这个Web UI服务体验上还不错，很多功能都有。

> Etcd Web UI服务的 yaml 定义文件我也放到了GitHub上，链接地址：https://github.com/kevinyan815/LearningKubernetes/tree/master/e3w
