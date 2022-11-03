在今天的文章中我将介绍`Kubernetes`中的`ConfigMap`对象。它的主要用途什么，为什么要用`ConfigMap`以及在Kubernetes里通常是如何使用ConfigMap的管理应用配置的。


## 什么是ConfigMap

能够灵活管理应用的配置是一个系统能否长期成功运转的一个关键因素，尤其是在应用分布式微服务时更是如此。

再将应用部署到测试，开发和生产等多个环境时，由于环境不同，将配置放到应用程序的镜像里不是一个好的做法。理想情况下，你会希望将配置与应用程序镜像分开管理好匹配不同的部署环境。在`Kubernetes`项目里这就是`ConfigMap` 发挥作用的地方。

`ConfigMap`使您可以将应用配置从应用程序的镜像内容中分离出来。这使得你的容器化应用程序在`Kubernetes`里更具可移植性，而无需担心配置。用户和系统组件都可以在`ConfigMap`中存储配置数据。

`ConfigMap`与另外一种API对象`Secret`有点类似 (后面会写文章单独介绍)，但是它提供了一种管理非敏感信息的配置的方式。

## 怎么创建ConfigMap

`ConfigMap`的创建方式非常简单，你可以使用 `kubectl create configmap` 命令基于 目录、文件 或者字符串字面值来创建 ConfigMap：

```auto
kubectl create configmap <map-name> <data-source>
```

其中，`<map-name>` 是要设置的`ConfigMap` 名称，`<data-source>` 是要从中提取数据的目录、 文件或者字面值。

你可以使用`kubectl describe` 或者 `kubectl get`获取已创建的`ConfigMap`的信息。

下面我们来演示一下这三种创建`ConfigMap`的方式。

### 通过文件目录创建ConfigMap

要从目录创建ConfigMap，必须首先创建一个目存放配置文件的目录:

```auto
$ mkdir configmap-demo 
```

然后将示例配置文件下载到目录中

```auto
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties -O configmap-demo-gamewget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties  -O configmap-demo-ui
```

下面是这l两个配置文件的内容，如果因为网络问题下载不到这个文件的话可以自己创建一个config-demo文件把内容粘贴进去。

```auto
enemies=alienslives=3enemies.cheat=trueenemies.cheat.level=noGoodRottensecret.code.passphrase=UUDDLRLRBABASsecret.code.allowed=truesecret.code.lives=30-----文件分割-----color.good=purplecolor.bad=yellowallow.textmode=truehow.nice.to.look=fairlyNice
```

接下来使用`kubectl create configmap`命令执行创建：

```auto
$ kubectl create configmap demo-configmap --from-file=configmap-democonfigmap "demo-configmap" created
```

，使用`kubectl describe`可以查看`demo-configmap`这个ConfigMap的描述：

```auto
$ kubectl describe configmap demo-configmapName:         demo-configmapNamespace:    defaultLabels:       <none>Annotations:  <none>Data====game.properties:----enemies=alienslives=3enemies.cheat=trueenemies.cheat.level=noGoodRottensecret.code.passphrase=UUDDLRLRBABASsecret.code.allowed=truesecret.code.lives=30ui.properties:----color.good=purplecolor.bad=yellowallow.textmode=truehow.nice.to.look=fairlyNiceEvents:  <none>
```

可以看到ConfigMap会把两个文件的内容作为其数据条目。

### 通过文件创建ConfigMap

从文件创建ConfigMap与从目录创建非常相似。需要做的就是将文件名传递给–-from-file参数。通过这种方式创建ConfigMap时，你可以根据需要多次使用--from-file参数，将多个文件数据源添加到ConfigMap中。

```auto
kubectl create configmap configmap-demo-2 \--from-file=configmap-demo-game \--from-file=configmap-demo-ui
```

### 直接用字符串创建ConfigMap

通过这种方式创建ConfigMap意味着您可以直接从命令行指定配置，而无需创建任何文件或目录。比如使用命令 **kubectl create ConfigMap special-config –from-literal=special.level=very –from-literal=special.type=charm**。

你可以传入多个键值对。命令行中提供的每对键值在 ConfigMap 的 `data` 部分中均表示为单独的条目。

```auto
kubectl get configmap special-config -o yaml
```

输出类似以下内容:

```auto
apiVersion: v1kind: ConfigMapmetadata:  creationTimestamp: 2020-10-30T19:14:38Z  name: special-config  namespace: default  resourceVersion: "3"  selfLink: /api/v1/namespaces/default/configmaps/special-config  uid: fcddce046-d653-71eb-8f30-68f728db1988data:  special.level: very  special.type: charm
```

## 在Pod里使用ConfigMap

### 用 ConfigMap 中的数据定义容器环境变量

将上面用字符串键值对直接创建的`ConfigMap` 中定义的 `special.how` 值分配给下面YAML文件里定义的Pod环境变量 `SPECIAL_LEVEL_KEY` 。

```auto
apiVersion: v1kind: Podmetadata:  name: dapi-test-podspec:  containers:    - name: test-container      image: k8s.gcr.io/busybox      command: [ "/bin/sh", "-c", "env" ]      env:        # 定义环境变量        - name: SPECIAL_LEVEL_KEY          valueFrom:            configMapKeyRef:              # 通过名字指定要引用的ConfigMap对象              name: special-config              # 指定引用ConfigMap里的那个数据条目              key: special.level  restartPolicy: Never
```

上面在Pod的`spec.env`定义环境变量`SPECIAL_LEVEL_KEY`时通过`valueFrom`的`configMapKeyRef`键告诉Pod要从ConfigMap中引用值，具体使用哪个ConfigMap对象里的那个数据条目则是通过`name`和`key`再去进一步指定。

### 将 ConfigMap 挂载到数据卷

在 Pod 定义的 `spec.volumes` 字段下添加 ConfigMap对象的名称。这会将 ConfigMap 数据以文件的形式添加到容器定义部分 `volumeMounts.mountPath` 的指定的挂载目录中（在下面的例子中为 `/etc/config`）。在容器中即可通过目录`/etc/config`下的文件使用ConfigMap中定义的数据条目，比如这里定义的容器启动命令就是容器启动后使用`ls`查看`/etc/config`目录下配置文件：

```auto
// pod-configmap-volume.yamlapiVersion: v1kind: Podmetadata:  name: dapi-test-podspec:  containers:    - name: test-container      image: k8s.gcr.io/busybox      command: [ "/bin/sh", "-c", "ls /etc/config/" ]      volumeMounts:      - name: config-volume        mountPath: /etc/config  volumes:    - name: config-volume      configMap:        # Provide the name of the ConfigMap containing the files you want        # to add to the container        name: special-config  restartPolicy: Never
```

创建 Pod:

```auto
kubectl create -f pod-configmap-volume.yaml
```

Pod 运行时，命令 `ls /etc/config/` 产生下面的输出：

```auto
SPECIAL_LEVELSPECIAL_TYPE
```

在Pod的YAML定义文件里，ConfigMap引用配置中 使用`path` 字段为特定的 ConfigMap 项目指定预期的文件名。在这里，`SPECIAL_LEVEL` 将挂载在 `config-volume` 数据卷中 `/etc/config/keys` 路径下。

```auto
apiVersion: v1kind: Podmetadata:  name: dapi-test-podspec:  containers:    - name: test-container      image: k8s.gcr.io/busybox      command: [ "/bin/sh","-c","cat /etc/config/keys" ]      volumeMounts:      - name: config-volume        mountPath: /etc/config  volumes:    - name: config-volume      configMap:        name: special-config        items:        - key: SPECIAL_LEVEL          path: keys  restartPolicy: Never
```

当 `Pod` 运行时，命令 `cat /etc/config/keys` 产生以下输出：

```auto
very
```
