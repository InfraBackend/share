## 介绍

前面的几篇文章从概念层面介绍了`Kubernetes`是什么，它的内部架构是怎样的。并且也在电脑上安装了`Minikube`\--拥有一个单节点的`Kubernetes`集群，让我们能够在自己的电脑上开始体验`Kubernetes`。今天的文章我准备和大家一起一步步地尝试做一个`Go`应用程序的`Docker`镜像，把它部署到`Minikuebe`上运行。今天的文章不需要什么基础，`Kubernetes`的新手朋友们先一起上车学起来。

## 应用程序代码

我们用`Go`写一个简单的`HTTP Server`，`Server`侦听3000端口包含`"/"`和`"/health_check"`两个路由，今天文章的关注点不在怎么用`Go`开发程序所以都是`Hello World`级别的代码，就不更多解释了，直接看代码吧。

```auto
package mainimport ( "fmt" "net/http")func index(w http.ResponseWriter, r *http.Request) { fmt.Fprintf(w, "<h1>Hello World</h1>")}func check(w http.ResponseWriter, r *http.Request) { fmt.Fprintf(w, "<h1>Health check</h1>")}func main() { http.HandleFunc("/", index) http.HandleFunc("/health_check", check) fmt.Println("Server starting...") http.ListenAndServe(":3000", nil)}
```

## 制作程序镜像

接下来开始制作包含应用程序代码的`Docker`镜像。关于`docker`相关的使用方法和如何编写`Dockerfile`，可以在公众号里回复关键字**docker**获取完整的参考笔记。

### dockerfile

在应用程序的根目录添加名为`Dockerfile`的文件，在文件里添加如下指令：

```auto
FROM golang:alpineRUN mkdir /appCOPY . /appWORKDIR /appRUN go build -o main . CMD ["/app/main"]
```

### build 镜像

在`Dockerfile`所在的目录下执行`docker build`构建镜像

```auto
➜  docker build -t go-app-img .
```

`docker`会依据`Dockerfile`里的指令构建镜像，整个构建的过程类似下面：

```auto
➜  docker build -t go-app-img .Sending build context to Docker daemon  9.216kBStep 1/6 : FROM golang:alpinealpine: Pulling from library/golangdf20fa9351a1: Pull complete ed8968b2872e: Pull complete a92cc7c5fd73: Pull complete 9e0cccf56431: Pull complete cbe0275821fc: Pull complete Digest: sha256:6042b9cfb4eb303f3bdcbfeaba79b45130d170939318de85ac5b9508cb6f0f7eStatus: Downloaded newer image for golang:alpine ---> 3289bf11c284Step 2/6 : RUN mkdir /app ---> Running in b34dccb1f3deRemoving intermediate container b34dccb1f3de ---> 1fa1a1c21aa2Step 3/6 : COPY . /app ---> 815660da9d1aStep 4/6 : WORKDIR /app ---> Running in 49dc25fe6bb7Removing intermediate container 49dc25fe6bb7 ---> 14776702ccf7Step 5/6 : RUN go build -o main . ---> Running in 3bd4dc1e2bf6Removing intermediate container 3bd4dc1e2bf6 ---> 59aa7f96ee42Step 6/6 : CMD ["/app/main"] ---> Running in 6309f604d662Removing intermediate container 6309f604d662 ---> 023baffdcb28Successfully built 023baffdcb28Successfully tagged go-app-img:latest
```

### 验证镜像

这一步其实可以省略，不过为了确保制作的镜像是没有问题，我们通过`docker run`命令用这个镜像运行容器验证一下。

```auto
➜ docker run -d -p 3333:3000 --rm --name go-app-container go-app-img
```

在这里，我们指示`docker`从镜像`go-app-img`运行容器，将主机端口`3333`绑定到容器的内部端口`3000`，以后台模式（-d）运行容器，给此容器命名为`go-app-container`，并在容器结束运行后自动删除容器（--rm）。

打开浏览器输入`localhost:3333`访问到的页面输出会是：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7p03GKREyf79thozoKNiaDUY2HYQnc2Xs9Ykw8ufmyIDdY0I3ia94v73OiakLn0dvjeuOJsVOGsGdVQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图片

### 推送镜像到DockerHub

测试镜像没问题后，将镜像推送到`DockerHub`，到时候`Kubernetes`在部署应用时会根据指定的镜像名称从`DockerHub`上拉取镜像（镜像源是可配置的，不一定非得是`DockerHub`，可以是私有镜像仓库）。

```auto
➜  docker build -t kevinyan001/kube-go-app .   ...➜  docker push  kevinyan001/kube-go-app...
```

用`Dockerfile`重新构建镜像，指定镜像仓库名。构建完成后将镜像然后推送到`DockerHub`上。

上面仓库名中的`kevinyan001`是我自己的`DockerHub`账号，你们可以直接使用下面的命令拉取我的镜像使用，不过还是建议每个人动手制作自己的镜像。

```auto
docker pull kevinyan001/kube-go-app:latest
```

## Kubernetes部署应用  

部署应用开始需要先定义预期状态，就是在`yaml`文件里声明具体的`Kubernetes`对象的各种预期的状态。然后让`Kubernetes`创建对象，之后它会始终驱动集群的当前状态向预期状态移动（比如有节点挂了，会新起节点替代挂掉的节点）。部署完应用后后我们还需要通过`Service`向外部暴露应用，这样才能访问运行在`Kubernetes`集群里的应用。

下面我们来一步步递进地执行这三个步骤。

开始之前我们需要启动一下`Minikube`

```auto
minikube start
```

如果你还没有安装可以参照《[**Minikube-运行在笔记本上的Kubernetes集群**](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485235&idx=1&sn=0cdd31c25b13790b336e0a8222f00b64&chksm=fa80d6a4cdf75fb27a64c7f06f9a33ed4e644624d3fcf73d791274d03b071467ebb4e292ac95&token=1954888916&lang=zh_CN&scene=21#wechat_redirect)》里的安装步骤

### 定义预期状态

在部署清单文件（`deployment.yaml`）中定义预期状态

```auto
apiVersion: apps/v1kind: Deploymentmetadata:  name: my-go-appspec:  replicas: 1  selector:    matchLabels:      app: go-app  template:    metadata:      labels:        app: go-app    spec:      containers:        - name: go-app-container          image: kevinyan001/kube-go-app          resources:            limits:              memory: "128Mi"              cpu: "500m"          ports:            - containerPort: 3000
```

`Kubernetes Deployment` 对象（清单文件的`kind`里指定的）表示运行在集群中的应用。文件里还指定了应用需要一个副本运行（`replicas`），以及运行的容器名和容器的镜像、资源大小等信息。

`Deployment`是`Kubernetes`对象的一种，还有其他很多种对象分别对应`Kubernetes`里的不同类型的资源。

### 部署应用

使用上面定义的`deployment.yaml`创建`Deployment`对象来运行`Go`应用程序的容器：

```auto
➜ kubectl create -f deployment.yamldeployment.apps/my-go-app created➜ kubectl get deploymentsNAME        READY     UP-TO-DATE   AVAILABLE   AGEmy-go-app   1/1       1            1           24s➜ kubectl get podsNAME                         READY     STATUS    RESTARTS   AGEmy-go-app-5bb8767f6d-2pdtk   1/1       Running   0          43s
```

### 暴露应用

应用部署完后还不能从外部直接访问，需要把刚才`Deployment`对象运行的应用程序作为`Kubernetes`的一个`Service`对外暴露。

```auto
➜ kubectl expose deployment my-go-app --type=NodePort --name=go-app-svc --target-port=3000 service/go-app-svc exposed➜ kubectl get svcNAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGEgo-app-svc   NodePort    10.104.190.231   <none>        3000:31425/TCP   40hkubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          6d13h➜ minikube ip192.168.64.4
```

通过`kubectl get svc`查询`Service`可以得到主机的`31425`端口映射到了`Kubernetes`运行着应用的容器的`3000`端口。在浏览器里使用`Kubernetes`集群IP加`NodePort`即可访问到`Kubernetes`部署的`Go`应用程序。

打开浏览器通过`192.168.64.4:31425`（以自己实践时查到的IP和端口为准）访问应用程序定义的两个路由的结果如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7p03GKREyf79thozoKNiaDUcVkvGHKxkyftnonnaEkF2jueTykzdXrkV0aQxI1jjbfiaBVoBJa2fdg/640?wx_fmt=png)

IndexPage

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f7p03GKREyf79thozoKNiaDUm1ia4CwJFc7tMkiaxMB3vMYPO2sibsYjB58d8FUKQ1UR2jFDYw3zFBb1g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

HealthCheckPage

