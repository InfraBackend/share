## 前言

`Minikube`是一个可以在本地电脑上运行`Kubernetes`的工具。`Minikube`会在笔记本电脑中的虚拟机上运行一个单节点的`Kubernetes`集群，让用户能对`Kubernetes`进行体验或者在之上进行`Kubernetes`的日常开发。

`Windows`，`MacOS`和`Linux`系统上都可以安装`Minikube`，不过在安装前需要确认系统的版本已经支持虚拟化（一般只要不是太老的系统版本都支持虚拟化）

## kubectl

在电脑上安装`Minikubne`前需要先安装`kubectl`，它是`Kubernetes`的命令行工具，可以使用`kubectl`部署应用程序，检查和管理集群资源以及查看日志。

### 安装kubectl

文章里我们演示的安装步骤都是macOS上的，如果是Linux和Windows系统只需要下载相应系统的二进制文件就行，我会在文章后边贴上官方的安装指南。

首先下载最新的稳定版本的`kubectl`二进制文件。

```auto
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl"
```

为`kubectl`授予可执行权限，然后将可执行文件放到系统的`PATH`目录中

```auto
chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
```

## 安装MiniKube

如果你的`macOS`上没有安装虚拟机监控程序的话在第一次启动`minikube`的时候会自动选择安装`HyperKit`作为虚拟机驱动，如果是以前电脑上有安装过`VirtualBox`那么可以在`Minikube`启动时加上`--vm-driver=virtualbox`来选择虚拟机驱动。

安装`minikube`的过程跟`kubectl`的过程差不多，也是下载`minikube`的二进制文件，赋予可执行权限后将其放入系统环境变量`PATH`对应的目录中。

不过由于大家都知道的网络访问原因，很多朋友无法直接使用`Kubernetes`官方提供的`minikube`进行实验，所以这里选择使用阿里云提供的`minikube`版本

```auto
curl -Lo minikube https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.11.0/minikube-darwin-amd64 \ && chmod +x minikube \ && sudo mv minikube /usr/local/bin/
```

如果是Linux和Window系统，安装流程类似只是软件的版本不同，具体可以参照官方文档里给的MiniKube的安装指南：

https://kubernetes.io/docs/tasks/tools/install-minikube

## 运行Minikube

启动`minikube`的方法非常简单，只要使用下面的命令

```auto
minikube start  --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'    
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f5HK3JWQYIYmvnDPK8WlwJQXM1tWiasGUd8iaozSTf7DicxC56E9QHkzZWuNXxs4h2KDdQ5LJoaPkRbw/640?wx_fmt=png)

启动minikube

在最新的`Minikube`中，已经提供了配置化的方式，可以帮助大家利用阿里云的镜像地址来获取所需的`Docker`镜像和配置。

## 测试Minikube

下面我们通过`minikube status`命令查看一下它的运行状态测试我们安装的`minikube`。

```auto
➜  minikube statusminikubetype: Control Planehost: Runningkubelet: Runningapiserver: Runningkubeconfig: Configured
```

通过`kubectl`查看集群的一些信息。

```auto
➜  kubectl get pods -ANAMESPACE     NAME                               READY   STATUS    RESTARTS   AGEkube-system   coredns-67c766df46-59rtb           1/1     Running   0          17mkube-system   coredns-67c766df46-jxmvf           1/1     Running   0          17mkube-system   etcd-minikube                      1/1     Running   0          16mkube-system   kube-addon-manager-minikube        1/1     Running   0          16mkube-system   kube-apiserver-minikube            1/1     Running   0          16mkube-system   kube-controller-manager-minikube   1/1     Running   0          17mkube-system   kube-proxy-ljppw                   1/1     Running   0          17mkube-system   kube-scheduler-minikube            1/1     Running   0          16mkube-system   storage-provisioner                1/1     Running   0          17m➜   kubectl get nodesNAME       STATUS   ROLES    AGE   VERSIONminikube   Ready    master   18m   v1.18.3➜   kubectl get namespacesNAME              STATUS   AGEdefault           Active   18mkube-node-lease   Active   18mkube-public       Active   18mkube-system       Active   18m
```
