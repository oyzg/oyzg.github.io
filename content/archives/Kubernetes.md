---
title: "Kubernetes"
date: 2025-10-15T15:35:36+08:00
draft: false
categories: [运维,云原生]
series: [学习笔记]
tags: [运维,Docker,K8s,Kubernetes,云原生]
summary: "容器编排"
---

# 容器
容器的本质其实就是一个特殊的进程。它通过Linux NameSpace、Linux CGroups、rootfs来实现对进程的隔离和约束。

Docker的核心步骤：
1、启用 Linux Namespace 配置；
2、设置指定的 Cgroups 参数；
3、切换进程的根目录（Change Root）。


#### NameSpace
NameSpace是用来修改进程视图的方法，通过NameSpace技术可以让这个进程只能看到自己这个namespace中的空间，使用方式则是通过添加相关参数。

```
int pid = clone(main_function, stack_size, SIGCHLD, NULL); 
# 添加CLONE_NEWPID后容器内看到的pid就是1了，除此外还有Mount、UTS、IPC、Network 和 User等Namespace
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```

#### CGroups
在NameSpace修改视图后，相比于虚拟化技术还是隔离得不够彻底，多个容器间使用的还是同一个操作系统内核，而且还有很多资源和对象不能NameSpace化，比如时间，在容器内修改时间是对宿主机可见的。

CGroups（Linux Control Group）技术是用来对进程进行资源限制的，限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等，从而防止一个进程把整个系统的资源吃光的情况。此外，还有对进程进行优先级设置、审计等作用。

它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下，通过修改该路径下的文件即可实现对进程的资源限制。

#### rootfs
在对进程进行隔离后，容器内看到的文件系统应该是完全独立的，让其不受宿主机和其他容器的影响。Mount Namespace 修改的是容器进程对文件系统“挂载点”的认知（对挂载点进行了隔离），即使开启了 Mount Namespace，容器进程看到的文件系统也跟宿主机完全一样，只有在容器执行挂载后，才会看到与宿主机不同的文件系统。在Linux中有一个chroot命令可以改变进程的根目录到你指定的位置。

但是我们一般需要在容器的根目录下挂载完整操作系统的文件系统，挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。

需要明确的是，rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。所有容器都共享宿主机的操作系统内核。

Docker在制作镜像时，引入了层的概念，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。rootfs由三部分组成，如图所示。只读层包含操作系统的基础文件，可读写层包含你的程序、依赖包等，如果你删除了只读层的文件，就会在可读写层生成一个foo文件，Init层主要包含一些操作系统的配置信息，如/etc/hosts、/etc/resolv.conf，这些文件输入只读层，但是我们需要对这些配置文件进行修改，并且希望这些配置信息在docker commit时不包含在内。

![rootfs](/images/kubernetes/rootfs.png)

# Docker

## Docker常用命令
![docker常用命令](/images/kubernetes/docker常用命令.png)

- 数据拷贝：docker cp [源路径] [目标路径]，对于容器内的文件采用[容器名或容器ID]:[路径]的形式，如docker cp a.txt 062:/tmp
- 共享目录：通过-v参数挂载，docker run -v [宿主机路径]:[容器内路径]
- 网络模式
    - null：容器与宿主机之间不通信
    - host：容器与宿主机共享网络，docker run时使用--net=host参数开启
    - bridge：容器与宿主机之间通过docker0网桥进行通信，使用--net=bridge开启，一般不需要，因为是默认的
    - 端口号映射需要bridge模式，在docker run 中指定-p参数，-p [宿主机端口]:[容器端口]



## Dockerfile
如前面rootfs部分所述，docker镜像是分成很多层的，可以用docker inspect [image name]来查看分层信息。

如果需要自己构建镜像则可以通过docker build命令进行构建，-f参数指定Dockerfile文件，没有就是Dockerfile文件，需要目录下只有这一个Dockerfile文件

Dockerfile中的常用命令有：
- FROM:基础镜像
- COPY:用于将本机的源码、配置文件等复制到镜像中，源文件必须在“构建上下文”路径中
- RUN:用于执行SHELL命令，一行只能有一条命令，所以末尾用\，命令之间使用&&相连，为了美观可以写一个sh文件，用COPY拷贝进来再RUN执行
- 变量:ARG和ENV，ARG变量只在镜像过程中可见，ENV变量在镜像构建和容器运行时均可见

TIP：Dockerfile每个指令都会生成一个镜像层，所以要精简


# Kubernetes
Kubernetes是一个容器编排工具。编排的意思就是能够按照用户意愿和系统规则，完全自动化的处理好容器之间的关系。调度是把容器按照某种规则，放在最佳节点上运行起来。

Kubunetes分为Master节点和note节点，Master节点由三个紧密协作的独立组件组合而成，它们分别是负责 API 服务的 kube-apiserver、负责调度的 kube-scheduler，以及负责容器编排的 kube-controller-manager。node节点最核心的部分是kubectl，用于和容器运行时打交道,此外还有kube-proxy用于管理容器的网络通信，ccontainer-runtime管理Pod的生命周期。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Etcd 中。


![rootfs](/images/kubernetes/rootfs.png)

除此之外，还有kubectl，是Kubernetes的命令行工具，用于和集群进行交互。

### minikube
minikube 是一个迷你版的Kubernetes集群，常用命令：
- minikube version： 查看minikube版本
- minikube start：启动minikube集群，--kubernetes-version=指定版本
- minikube stop：停止minikube集群
- minikube status：查看minikube集群状态
- minikube delete：删除minikube集群
- minikube node list：查看minikube集群节点
  
### kubectl
kubectl是Kubernetes的命令行工具，用于和集群进行交互，常用命令：
- kubectl get pods|nodes|services|deployments|namespaces：查看所有pod｜node｜service｜deployment｜namespace信息
- kubectl describe pods|nodes|services|deployments|namespaces：查看指定pod｜node｜service｜deployment｜namespace信息
- kubectl create|delete|edit -f [文件名]：创建|删除|编辑 指定资源
- kubectl apply -f [文件名]：使用文件创建资源
- kubectl logs [pod名]：查看指定pod的日志
- kubectl exec [pod名] [命令]：在指定pod中执行命令, kubectl exec [pod名] -it --/bin/bash进入pod
- kubectl run [pod名] --image=[镜像名]：运行一个pod
- kubectl get pod -n [namespace]：查看指定namespace下的所有pod
- kubectl api-resources：查看支持的所有API对象
- kubectl explain [API对象]：查看API对象， --dry-run=client -o yaml可以生成yaml样板

## API对象
由于Kubenetes的设计思路——“单一职责”和“组合优于继承”，所有对象都尽量只关注自己的职责。
![Pod](/images/kubernetes/Pod.png)

### Pod
Pod是Kubernetes的最小运行单位，一个Pod中可以运行多个容器，每个容器之间是相互隔离的，但是可以共享同一个IP地址和端口。yaml示例：
```
apiVersion: v1
kind: Pod
metadata:
  name: busy-pod
  labels:
    owner: chrono
    env: demo
    region: north
    tier: back
spec:
  containers:
    - image: busybox:latest
      name: busy
      imagePullPolicy: IfNotPresent
      env:
        - name: os
          value: "ubuntu"
        - name: debug
          value: "on"
      command:
        - /bin/echo
      args:
        - "$(os), $(debug)"
      
      resources: # 限制使用资源
        requests: # 要申请的资源
          cpu: 10m # 1000m = 1CPU时间
          memory: 100Mi
        limits: # 使用资源的上限
          cpu: 20m
          memory: 200Mi
```
由apiVersion、kind、metadata、spec四个基本组成部分，metadata包含pod的名称、标签、注解等信息，spec包含pod的运行参数，包括容器镜像、环境变量、命令、参数、端口映射、卷挂载、资源限制、调度策略等等。

Kubernetes 为检查应用状态定义了三种探针，它们分别对应容器不同的状态：
- Startup，启动探针，用来检查应用是否已经启动成功，适合那些有大量初始化工作要做，启动很慢的应用。如果Startup探针失败，会尝试反复重启
- Liveness，存活探针，用来检查应用是否正常运行，是否存在死锁、死循环。如果容器异常也会重启容器
- Readiness，就绪探针，用来检查应用是否已经准备好接收流量。如果失败会从Service负载均衡中移除，不再分配流量

应用程序先启动，加载完配置文件等基本的初始化数据就进入了 Startup 状态，之后如果没有什么异常就是 Liveness 存活状态，但可能有一些准备工作没有完成，还不一定能对外提供服务，只有到最后的 Readiness 状态才是一个容器最健康可用的状态

要使用探针需要预留“检查口”，如下，在ConfigMap中使用/ready作为检查口。
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngx-conf
data:
  default.conf: |
    server {
      listen 80;
      location = /ready {
        return 200 'I am ready';
      }
    }
```

探针的具体定义：
```
apiVersion: v1
kind: Pod
metadata:
  name: ngx-pod-probe
spec:
  volumes:
    - name: ngx-conf-vol
      configMap:
        name: ngx-conf
  containers:
    - image: nginx:alpine
      name: ngx
      ports:
        - containerPort: 80
      volumeMounts:
        - mountPath: /etc/nginx/conf.d
          name: ngx-conf-vol
      startupProbe: # 启动探针
        periodSeconds: 1 # 执行探测动作的时间间隔
        # timeoutSeconds字段 探测动作的超时时间
        # successThreshold字段 连续几次探测成功才认为是正常
        # failureThreshold字段 连续几次探测失败才认为是异常
        # 三种探测方式： exec、TCP Socket、HTTP GET
        exec: # 执行一个Linux命令
          command: ["cat", "/var/run/nginx.pid"]
      livenessProbe: # 存活探针
        periodSeconds: 10
        tcpSocket: # 使用 TCP 协议尝试连接容器的指定端口
          port: 80
      readinessProbe: # 就绪探针
        periodSeconds: 5 
        httpGet: # 连接端口并发送 HTTP GET 请求
          path: /ready
          port: 80
```

### Job和CronJob
- 在线业务：长时间运行的，如nginx
- 离线业务：短时间运行的，如定时任务
Job和CronJob都是用来处理离线业务的，Job是处理临时任务，CronJob是处理定时任务。
Job的yaml示例：
```
apiVersion: batch/v1
kind: Job
metadata:
  name: echo-job

spec:
  activeDeadlineSeconds: 15
  backoffLimit: 2
  completions: 4
  parallelism: 2

  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - image: busybox
        name: echo-job
        imagePullPolicy: IfNotPresent
        command: ["/bin/echo"]
        args: ["hello", "world"]
```
template定义了一个应用模版，里面用来嵌入Pod。
几个重要字段：
- activeDeadlineSeconds，设置 Pod 运行的超时时间。
- backoffLimit，设置 Pod 的失败重试次数。
- completions，Job 完成需要运行多少个 Pod，默认是 1 个。
- parallelism，它与 completions 相关，表示允许并发运行的 Pod 数量，避免过多占用资源

CronJob的yaml示例：
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: echo-cj

spec: # 这个spec是CronJob的配置
  schedule: '*/* * * * *' # 定时，每分钟执行一次
  jobTemplate: # 这个下面嵌套Job
    spec:
      template: # 这个下面嵌套Pod
        spec:
          restartPolicy: OnFailure
          containers:
          - image: busybox
            name: echo-cj
            imagePullPolicy: IfNotPresent
            command: ["/bin/echo"]
            args: ["hello", "world"]
```

### ConfigMap和Secret
ConfigMap和Secret用于保存配置信息，ConfigMap保存的是非敏感信息，如数据库连接信息，Secret保存的是敏感信息，如密码。这些信息都保存在etcd中。

ConfigMap的yaml示例：
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: info

data: # 用来存储数据，kv结构
  count: '10'
  debug: 'on'
  path: '/etc/systemd'
  greeting: |
    say hello to kubernetes.
```

Secret的yaml示例：
```
apiVersion: v1
kind: Secret
metadata:
  name: user

data: # 这些都是base64编码的，
  name: cm9vdA==  # root
  pwd: MTIzNDU2  # 123456
  db: bXlzY2Fw  # mysql
```
```
# 自己手动base64编码，-n是删除隐含的换行符
echo -n "123456" | base64
MTIzNDU2
```

使用方法：
1、环境变量的方式：利用pod.spec.containers.env.valueFrom以环境变量的形式注入Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: env-pod

spec:
  containers:
  - env:
    - name: COUNT
      valueFrom:
        configMapKeyRef: #ConfigMap对象
          name: info
          key: count
    - name: USERNAME
      valueFrom:
        secretKeyRef: #Secret对象
          name: user
          key: name
```

2、Volume的方式：类似docker run -v，在Pod中挂载一个Volume，然后把Volume中的内容映射到容器中。
```
apiVersion: v1
kind: Pod
metadata:
  name: vol-pod

spec:
  # 定义两个 Volume，分别引用 ConfigMap 和 Secret，名字是 cm-vol 和 sec-vol
  volumes:
    - name: cm-vol
      configMap:
        name: info
    - name: sec-vol
      secret:
        secretName: user

  # 然后在容器中挂载 Volume
  containers:
    - volumeMounts:
        - mountPath: /tmp/cm-items
          name: cm-vol
        - mountPath: /tmp/sec-items
          name: sec-vol
      image: busybox
      name: busy
      imagePullPolicy: IfNotPresent
      command: ["/bin/sleep", "300"]
```

### Deployment
Deployment是用来部署应用程序的，用来处理在线业务，让应用永不宕机。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ngx-dep
  name: ngx-dep
spec:
  replicas: 2  # 副本数，运行的Pod数量
  selector:
    matchLabels:
      app: ngx-dep   # 匹配标签，和下面的template中的标签一致
      # 通过这种labels，解除了Deployment和Pod的强绑定，把组合关系变成了“弱引用”
  template:
    metadata:
      labels:
        app: ngx-dep
    spec:
      containers:
        - image: nginx:alpine
          name: nginx
```
通过scale命令来扩容：
kubectl scale --replicas=5 deploy ngx-dep

### Daemonset
用于在集群的每个节点上运行且仅运行一个Pod，主要用于日志收集、数据采集等场景。

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: redis-ds
  labels:
    app: redis-ds

spec:
  selector:
    matchLabels:
      name: redis-ds

  template:
    metadata:
      labels:
        name: redis-ds
    spec:
      containers:
        - image: redis:5-alpine
          name: redis
          ports:
            - containerPort: 6379
```

**污点（taint）和容忍度（toleration**

Master默认不跑应用，需要配置taint和toleration来配置才能在master上运行pod。
方法一：去掉master节点的taint，末尾的-就是去除
kubectl taint node master node-role.kubernetes.io/master:NoSchedule-

方法二：在创建节点的时候，添加toleration
```
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
  operator: Exists
```

#### 静态Pod
默认在/etc/kubernetes/manifests下，Kubernetes 的 4 个核心组件 apiserver、etcd、scheduler、controller-manager都以静态 Pod 的形式存在的

### Service
用于服务发现和负载均衡，通过iptables实现。

生成yaml文件命令：kubectl expose deploy ngx-dep --port=80 --target-port=80 --dry-run=client -o yaml
```
apiVersion: v1
kind: Service
metadata:
  name: ngx-svc

spec:
  selector:
    app: ngx-dep

  ports:
    - port: 80 # 外部端口
      targetPort: 80 # 内部端口
      protocol: TCP # 协议
  
  type：ClusterIP # 还有“ExternalName”、“LoadBalancer”、"NodePort"，“NodePort”类型会创建一个专用映射端口，外部可访问
```
**namespace**

kubectl get ns查看所有的namespace，默认情况下所有API对象都在defalult namespace下。

基于 DNS 插件，可以以域名形式访问Service，Service对象的域名形式为：”对象. 名字空间.svc.cluster.local“，或者“对象. 名字空间”、“对象名”

### Ingress
Ingress是流量的总入口，统管集群的进出口数据，以及负载均衡（七层，Service是四层负载均衡）

#### Ingress Controller
Ingress Controller是Ingress资源的实际执行者，负责监听Ingress资源的变化并根据规则配置负载均衡器。
常用的Ingress Controller实现有Nginx Ingress Controller。

工作原理：
1. 用户创建Ingress资源定义路由规则
2. Ingress Controller持续监听API Server中的Ingress资源变化
3. 当Ingress资源发生变化时，Controller更新其内部的负载均衡配置
4. 外部流量通过Ingress Controller访问集群内的服务


#### IngreeClass
用于定义 Ingress 控制器的类型和配置，解决了集群中存在多个 Ingress 控制器时的路由冲突问题。

![Ingress](/images/kubernetes/Ingress.png)


### PersistentVolume
用于管理存储资源，属于系统资源，与Node平级，Pod只拥有其使用权

- PersistentVolumeClaim，简称 PVC，用来向Kubernetes申请存储资源，由Pod使用，代表Pod向系统申请PV，申请成功后会将PV和PVC绑定（bind）
- StorageClass，在PV和PVC之间，帮助PVC找到合适的PV。

PV的yaml：
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-10m-pv
spec:
  storageClassName: host-test # 对应StorageClass
  # 访问模式，ReadWriteOnce（可读可写，只能被一个Pod使用）、ReadOnlyMany（只可读，可以被多个Pod使用）、ReadWriteMany（可读可写，可以被多个Pod使用）
  accessModes: 
    - ReadWriteOnce
  capacity: # 存储容量
    storage: 10Mi
  hostPath: # 存储卷的本地路径，在节点上创建的目录
    path: /tmp/host-10m-pv/
```
PVC的yaml：
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-5m-pvc
spec:
  storageClassName: host-test
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Mi # 请求的存储容量
```

在Pod中使用PVC：
```
apiVersion: v1
kind: Pod
metadata:
  name: host-pvc-pod
spec:
  volumes:
    - name: host-pvc-vol
      persistentVolumeClaim:
        claimName: host-5m-pvc # 绑定PVC
  containers:
    - name: ngx-pvc-pod
      image: nginx:alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: host-pvc-vol
          mountPath: /tmp
```
HostPath 是最简单的一种 PV，数据存储在节点本地，速度快但不能跟随 Pod 迁移

要实现跨节点数据共享，需要使用NFS或者Ceph等网络存储系统
```
# NFS PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-1g-pv
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  nfs:
    path: /tmp/nfs/1g-pv # 需事先创建好
    server: 192.168.10.208
```

以上PV都是静态存储卷，需要手动创建，而Provisioner可以自动管理、创建PV，是动态存储卷。Provisioner以Pod形式运行，部署Provisioner有三个yaml文件，rbac.yaml、class.yaml和deployment.yaml。

使用Provisioner时就不需要定义PV了，只需要在PVC中指定StorageClass，StorageClass再关联到Provisioner。
StorageClass的yaml：
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # Provisioner名称
parameters:
  archiveOnDelete: "false" # "false"为自动回收存储空间
```

### StatefulSet
Deployment只能管理无状态的应用，StatefulSet管理有状态的，可以看作Deployment的特例。

StatefulSet启动的Pod是有顺序的，名字为XXXX-0, XXXX-1等，可以根据编号来决定依赖关系，

StatefulSet的yaml：
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sts
spec:
  serviceName: redis-svc # 和Service中metadata.name一致

  volumeClaimTemplates: # PVC，为每个Pod自动创建PVC
  - metadata:
      name: redis-100m-pvc
    spec:
      storageClassName: nfs-client
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 100Mi

  replicas: 2
  selector:
    matchLabels:
      app: redis-sts
  template:
    metadata:
      labels:
        app: redis-sts
    spec:
      containers:
        - image: redis:5-alpine
          name: redis
          ports:
          - containerPort: 6379

          volumeMounts:
          - name: redis-100m-pvc
            mountPath: /data
```
写 Service 对象的时候要小心一些，metadata.name 必须和StatefulSet 里的 serviceName 相同，selector 里的标签也必须和 StatefulSet 里的一致。StatefulSet创建的Pod的域名和其他Pod不同，格式为“Pod名.服务名.名字空间.svc.cluster.local”，简写”Pod名.服务名“。

### namespace
namespace是Kubernetes中的一个概念，可以将集群切分成多个区域，同时namespace也是一种API对象，简称ns。Kubernetes中默认有四个namespace：default、kube-system、kube-public、kube-node-lease。要将API对象放入指定namespace中可以在metadata里添加一个namespace字段。

### ResourceQuota
用于为namespace进行资源配额，yaml文件示例：
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-qt
  namespace: dev-ns
spec:
  hard: # 硬性全局限制
    requests.cpu: 10 # 10个CPU
    requests.memory: 10Gi
    limits.cpu: 10
    limits.memory: 20Gi
    requests.storage: 100Gi
    persistentvolumeclaims: 100
    pods: 100
    configmaps: 100
    secrets: 100
    services: 10
    count/jobs.batch: 1
    count/cronjobs.batch: 1
    count/deployments.apps: 1
```
在添加资源配额后，要求所有在里面运行的 Pod 都必须用字段 resources 声明资源需求，否则就无法创建。可以用LimitRange自动为Pod加上资源限制，LimitRange的yaml：
```
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev-ns
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 200m
        memory: 50Mi
      default:
        cpu: 500m
        memory: 100Mi
    - type: Pod
      max:
        cpu: 800m
        memory: 200Mi
```

### HorizontalPodAutoscaler
基于Metrics Server，它从 Metrics Server 获取当前应用的运行指标，主要是 CPU 使用率，再依据预定的策略增加或者减少 Pod 的数量
yaml:
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: ngx-hpa
spec:
  maxReplicas: 10
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ngx-hpa-dep
  targetCPUUtilizationPercentage: 5
```
在Deployment的spec下一定要写resources字段，否则无法实现自动扩缩容

#### Metrics Server
Metrics Server 是一个专门用来收集 Kubernetes 核心资源指标（metrics）的工具，它定时从所有节点的 kubelet 里采集信息,安装Metrics Server后可以使用kubectl top查看资源使用情况

#### Prometheus
相对于Metrics Server，Prometheus的指标更全一些。Prometheus 是云原生监控领域的“事实标准”，用 PromQL 语言来查询数据，配合 Grafana可以展示直观的图形界面，方便监控。

## kubuadm——一键部署
使用方法：

```
# 创建一个 Master 节点
$ kubeadm init
 
# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master 节点的 IP 和端口 >
```

Kubernetes在部署时，它的每一个组件都是一个需要被执行的、单独的二进制文件。所以部署十分麻烦，kubeadm的方案是把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。


## 滚动更新
Kubernetes中rsion使用Pod模版的hash值来作为版本号。
要实现滚动更新，主要有以下几个命令：
- kubectl apply -f xxx.yaml,修改yaml文件后应用即可，Kubernetes会进行滚动更新（逐步的创建新版本Pod，删除旧版本Pod，最后只剩新版本Pod，旧版本Pod全部删除）
- kubectl rollout pause暂停更新
- kubectl rollout resume恢复更新
- kubectl rollout history查看版本历史，加上参数--revision x，查看指定版本详细信息
- kubectl rollout undo，回滚到上一个版本，--to-revision参数指定版本
- Deployment 的 metadata添加字段annotations类似label，但是label用于外部对象，annotations用于内部对象，其中kubernetes.io/change-cause是更新版本说明。

## 网络模型
Docker有三种网络模式：null、host 和 bridge，但是只适用于单机，Kubernetes提出了自己的网络模型IP-per-pod：
- 集群里的每个 Pod 都会有唯一的一个 IP 地址。
- Pod 里的所有容器共享这个 IP 地址。
- 集群里的所有 Pod 都属于同一个网段。
- Pod直接可以基于IP地址访问另一个Pod，不需要做网络地址转换（NAT）

### CNI（Container Networking Interface）
CNI 为网络插件定义了一系列通用接口，CNI插件大致分为三种：
- Overlay：它构建了一个工作在真实底层网络之上的“逻辑网络”，把原始的 Pod 网络数据封包，再通过下层网络发送出去，到了目的地再拆包。因为这个特点，它对底层网络的要求低，适应性强，缺点就是有额外的传输成本，性能较低。
- Route：在底层网络之上工作，但它没有封包和拆包，而是使用系统内置的路由功能来实现Pod 跨主机通信。它的好处是性能高，不过对底层网络的依赖性比较强，如果底层不支持就没办法工作了
- Underlay： 直接用底层网络来实现 CNI，也就是说 Pod 和宿主机都在一个网络里，Pod和宿主机是平等的。它对底层的硬件和网络的依赖性是最强的，因而不够灵活，但性能最高。

常用的CNI插件：
- Flannel：最早是Overlay模式，后来用 Host-Gateway 技术支持了 Route 模式，简单易用、但是性能不是很好
- Calico：Route模式使用BGP 协议来维护路由信息，性能要比 Flannel 好，而且支持多种网络策略，具备数据加密、安全隔离、流量整形等功能
- Cilium：同时支持 Overlay 模式和 Route 模式，它的特点是深度使用了 Linux eBPF 技术，在内核层次操作网络数据，所以性能很高，可以灵活实现各种功能



参考资料：
《Kubernetes入门实战课》— 罗剑锋（极客时间）