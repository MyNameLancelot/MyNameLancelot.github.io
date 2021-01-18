---
layout: post
title: "kubernetes的使用"
date: 2021-01-18 15:37:27
categories: docker
---

## 一、kubernetes 概述

### 基本介绍

kubernetes简称K8s，是用8代替"ubernete"而成的缩写。是一个开源的用于管理云平台中多个主机上的容器化的应用。Kubernetes的目标是让部署容器化的应用简单并且高效（powerful），Kubernetes 提供了应用部署，规划，更新，维护的一种机制。

### 功能

- 自动装箱：基于容器对应用运行环境的资源配置要求自动部署应用容器
- 自我修复：当容器终止运行时会对容器进行重启。当Node节点有问题时会对容器进行重新部署和重新调度
- 水平扩展：通过简单的命令、用户UI界面或基于CPU等资源使用情况，对应用容器进行规模扩大或规模剪裁
- 服务发现：用户不需使用额外的服务发现机制，就能够基于Kubernetes自身能力实现服务发现和负载均衡
- 滚动更新：可以根据应用的变化，对应用容器运行的应用，进行一次性或批量式更新
- 版本回退：可以根据应用部署情况，对应用容器运行的应用，进行历史版本即时回退
- 密钥和配置管理：在不需要重新构建镜像的情况下，可以部署和更新密钥和应用配置，类似热部署
- 存储编排：自动实现存储系统挂载及应用，特别对有状态应用实现数据持久化非常重要存储系统可以来自于本地目录、网络存储(NFS、Gluster、Ceph 等)、公共云存储服务
- 批处理：提供一次性任务，定时任务调度

### 架构

<img src="/img/k8s/k8s架构图.png" alt="img" style="zoom:80%;" />

**Master Node**：k8s 集群控制节点对集群进行调度管理，接受集群外用户的集群操作请求。Master Node由`API Server`、`Scheduler`、`ClusterState Store（ETCD 数据库）`和`Controller MangerServer`所组成
**Worker Node**：集群工作节点，运行用户业务应用容器。Worker Node包含`kubelet`、`kube proxy`和`ContainerRuntime`；

## 二、kubernetes 搭建

一般采用kubeadm方式，二进制安装无法纳入容器管理，不能自愈

**<span id="kubeadm-1">第一步：系统初始化</span>**

- 关闭防火墙

    ```shell
    # 临时关闭
    systemctl stop firewalld
    # 永久关闭
    systemctl disable firewalld
    ```

- 关闭selinux

    ```shell
    # 临时关闭
    setenforce 0
    # 永久关闭
    sed -i 's/enforcing/disabled/' /etc/selinux/config
    ```

- 关闭swap

    ```shell
    # 临时关闭
    swapoff -a
    # 永久关闭
    sed -ri 's/.*swap.*/#&/' /etc/fstab
    ```

- 设置主机名

    ```shell
    hostnamectl set-hostname <hostname>
    ```

- 节点添加主机映射

    ```shell
    cat >> /etc/hosts << EOF
    192.168.22.165 centos165
    192.168.22.166 centos166
    192.168.22.167 centos167
    EOF
    ```

- 将桥接的IPv4流量传递到iptables的链

    ```shell
    cat > /etc/sysctl.d/k8s.conf << EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    
    # 生效
    sysctl --system
    ```

- 时间同步

    ```shell
    yum install ntpdate -y
    ntpdate time.windows.com
    ```

- 开启ipvs

    ```shell
    yum install -y ipvsadm
    modprobe br_netfilter
    
    cat > /etc/sysconfig/modules/ipvs.modules <<EOF
    #!/bin/bash
    modprobe -- ip_vs
    modprobe -- ip_vs_rr
    modprobe -- ip_vs_wrr
    modprobe -- ip_vs_sh 
    modprobe -- nf_conntrack_ipv4
    EOF
    chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules &&
    lsmod | grep -e ip_vs -e nf_conntrack_ipv4
    ```
    
- 设置`rsyslogd`和`systemd journald`

    ```shell
    #docker容器日志可以再  /var/log/containers查看
    
    mkdir /var/log/journal #持久化保存日志的目录
    mkdir /etc/systemd/journald.conf.d
    cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
    [Journal]
    #持久化保存到磁盘
    Storage=persistent
    
    #压缩历史日志
    Compress=yes
    SyncIntervalSec=5m
    RateLimitInterval=30s
    RateLimitBurst=1000
    
    #最大占用空间10G
    SystemMaxUse=10G
    
    #单日志文件最大200M
    SystemMaxFileSize=200M
    
    #日志保存时间2周
    MaxRetentionSec=2week
    
    #不将日志转发到syslog
    ForwardToSyslog=no
    EOF
    
    systemctl restart systemd-journald
    ```

**<span id="kubeadm-2">第二步：软件安装</span>**

- 安装Docker

    ```shell
    wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
    
    # 查看可安装的docker版本
    yum list docker-ce --showduplicates | sort -r
    
    yum -y install docker-ce-[version]
    
    systemctl enable docker && systemctl start docker
    ```

- 为docker添加阿里云镜像加速

    ```shell
    sudo mkdir -p /etc/docker
    sudo tee /etc/docker/daemon.json <<-'EOF'
    {
      "registry-mirrors": ["https://do6wervs.mirror.aliyuncs.com"]
    }
    EOF
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

- 添加yum源

    ```shell
    cat > /etc/yum.repos.d/kubernetes.repo << EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=0
    repo_gpgcheck=0
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
    https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF
    ```

- 安装kubeadm，kubelet 和kubectl

    ```shell
    yum install -y kubelet kubeadm kubectl
    systemctl enable kubelet
    ```

**第三步：部署Kubernetes Master**

- 初始化kubeadm

    ```shell
    # pod-network-cidr必须为10.244.0.0/16，因为flannel使用此网段
    kubeadm init \
    --apiserver-advertise-address=[masterIp] \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version v1.20.1 \
    --service-cidr=10.96.0.0/12 \
    --pod-network-cidr=10.244.0.0/16
    
    # 执行完成之后会有让执行的命令
    # master节点执行的命令
    # To start using your cluster, you need to run the following as a regular user:
    # mkdir -p $HOME/.kube
    # sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    # sudo chown $(id -u):$(id -g) $HOME/.kube/config
    # Then you can join any number of worker nodes by running the following on each as root:
    # worker节点执行的命令
    # kubeadm join 192.168.22.165:6443 --token glkg5s.ae3ofk7bh1x2ot0e \
    #    --discovery-token-ca-cert-hash 
    # sha256:0cc790ef2d73484c78f4a10081da322b97f72b9d3a91999e3f76b0fd84d4c75c 
    ```

- 执行完成之后让执行的命令

    ```shell
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    kubectl get nodes
    ```

**<span id="kubeadm-4">第四步：安装Pod 网络插件（CNI）</span>**

- 安装[flannel](https://github.com/coreos/flannel)【在master节点执行】，<a href="#flannel-work">flannel作用</a>

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```

- 加入Kubernetes Node【在work节点执行】

    ```shell
    # 向集群添加新节点，执行在kubeadm init 输出的kubeadm join命令
    kubeadm join 192.168.22.165:6443 --token glkg5s.ae3ofk7bh1x2ot0e \
    --discovery-token-ca-cert-hash sha256:0cc790ef2d73484c78f4a10081da322b97f72b9d3a91999e3f76b0fd84d4c75c 
    ```

**第五步：测试kubernetes 集群**

```shell
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc

# 访问地址：http://NodeIP:Port
```

## 三、kubernetes 组件介绍

### Pod

**概念**

​	`Pod`是k8s中可以创建和管理的最小单元，其他的资源对象都是用来支撑或者扩展`Pod`对象功能的，比如`Controller`对象是用来管控`Pod`对象的，`Service`或者`Ingress`资源对象是用来暴露`Pod`引用对象的，`PersistentVolume`资源对象是用来为`Pod`提供存储等等。k8s不会直接处理docker容器，而是操作`Pod`。<span style="color:#669999">一个`Pod`是由一个或多个docker容器组成。</span>

​	每一个`Pod`都有一个特殊的被称为"根容器"的`Pause`容器，其他容器则为业务容器。业务容器共享`Pause`的网络栈和Volume挂载卷

---

**特性**

1）资源共享

一个`Pod`里的多个容器可以共享存储和网络，可以看作一个逻辑的主机。共享的如namespace,cgroups或者其他的隔离资源

2）生命周期短暂

`Pod`属于生命周期比较短暂的组件，比如当`Pod`所在节点发生故障，那么该节点上的`Pod`会被调度到其他节点，但被重新调度的`Pod`是一个全新的`Pod`,跟之前的`Pod`没有关系

3）平坦的网络

K8s集群中的所有`Pod`都在同一个共享网络地址空间中，也就是说每个`Pod`都可以通过其他`Pod`的IP地址来实现访问

---

**分类**

1）普通`Pod`

普通`Pod`一旦被创建，就会被放入到`etcd`中存储，随后会被`Kubernetes Master`调度到某个具体的Node上并进行绑定，随后该`Pod`对应的Node上的kubelet进程实例化成一组相关的Docker 容器并启动起来。

2）静态`Pod`

静态`Pod`是由kubelet进行管理的仅存在于特定Node上的`Pod`,它们不能通过`API Server`进行管理，用户不能管理。

---

**创建Pod的流程图**

<img src="/img/k8s/creat-pod.png" style="zoom:67%;" />

### Controller

**ReplicationController & ReplicaSet & Deployment**

- ReplicationController用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的Pod来替代。而如果异常多出来的容器也会自动回收。在新版本中建议使用ReplicaSet来取代ReplicationController
- ReplicaSet跟ReplicationController没有本质的不同只是名字不一样，只是ReplicaSet支持集合式的selector
- 虽然ReplicaSet可以独立使用，但一般还是使用Deployment来管理ReplicaSet（比如ReplicaSet不支持滚动升级)。Deployment经典的应用场景：
    - 定义Deployment来创建Pod和ReplicaSet
    - 滚动升级和回滚应用
    - 扩容和缩容
    - 暂停和继续Deployment

> 水平自动伸缩仅适用于Deployment和ReplicaSet ，根据Pod的metric扩缩容

---

**StatefulSet**

StatefulSet是为了解决有状态服务的问题(对应Deployments 和ReplicaSets是为无状态服务而设计)，其应用场景包括:

- 稳定的持久化存储。即`Pod`重新调度后还是能访问到相同的持久化数据（基于PVC来实现）
- 稳定的网络标志。即`Pod`重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）实现
- 有序部署，有序扩展。即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0 到 N-1，在下一个Pod运行之前所有之前的Pod必须都是Running 和Ready状态）
- 有序收缩，有序删除（即从N-1到 0）

---

**DaemonSet**

DaemonSet确保全部或者一些Node上运行一个`Pod`的副本。当有Node 加入集群时，也会为他们新增一个Pod。当有Node从集群移除时，这些Pod也会被回收。删除DaemonSet 将会删除它创建的所有`Pod`使用DaemonSet 的一些典型用法:

- 运行集群存储daemon， 例如在每个Node上运行glusterd、 ceph
- 在每个Node上运行日志收集daemon, 例如fluentd、 logstash
- 在每个Node上运行监控daemon, 例如Prometheus Node Exporter

---

**Job & Cron Job**

`Job`负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个`Pod`成功结束

`Cron Job`管理基于时间的Job，即在给定时间点只运行一次或周期性地在给定时间点运行

### <span id="flannel-work">容器网络通讯</span>

Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务，简单来说它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址。而且它还能在这些IP地址之间建立一个覆盖网络(Overlay Network) ，通过这个覆盖网络，将数据包原封不动地传递到目标容器内

<img src="/img/k8s/Flannel.png" style="zoom:70%;" />

> ETCD之Flannel作用说明：①存储管理Flannel可分配的IP地址段资源；②建立维护Pod节点路由表

**同一个Pod内部通讯**：同一个`Pod`共享同一个网络命名空间，共享同一个Linux协议栈

**Pod1与Pod2不在同一台主机**：`Pod`的地址是与docker0在同一个网段的，但docker0网段与宿主机网卡是两个完全不同的IP网段，并且不同Node之间的通信只能通过宿主机的物理网卡进行。将`Pod`的IP和所在Node的IP关联起来，通过这个关联让`Pod`可以互相访问

**Pod1与Pod2在同一台机器**：由Docker0 网桥直接转发请求至Pod2，不需要经过Flannel

**Pod至Service的网络**：目前基于性能考虑，全部为iptables 维护和转发

**Pod到外网：**`Pod`向外网发送请求，查找路由表，转发数据包到宿主机的网卡，宿主网卡完成路由选择后，iptables执行Masquerade，把源IP更改为宿主网卡的IP， 然后向外网服务器发送请求

**外网访问Pod：**Service

## 四、命令

**kubectl create/apply **

例：kubectl create deployment nginx-deployment --image=nginx --replicas=1

​        kubectl apply -f nginx-deployment .yaml

解释：使用nginx镜像创建一个pod，副本数为1，replicas默认为1

---

**kubectl get node ** 

例：kubectl get node -o wide

解释：查看node节点信息

---

**kubectl get pod** 

例：kubectl get pod -o wide --namespace=default -w

解释：查看default名称空间下pod详细信息

**kubectl get [all/deployment/rs/daemonSet/job/cronJob/svc/pv/pvc/namespace]**

例：kubectl get deployment --namespace=default

解释：查看deployment控制器信息

---

**kubectl delete pods**

例：kubectl delete pods nginx-deployment

解释：删除指定pod

---

**kubectl scale**

例：kubectl scale --replicas=3 deployment/nginx-deployment 

解释：伸缩pod

---

**kubectl expose**  

例：kubectl expose deployment nginx-deployment --port=8888 --target-port=80

解释：创建一个service端口是8888代理容器端口80，并实现了负载均衡

---

**kubectl edit**

例：kubectl edit svc nginx-deployment

解释：可修改pod的svc

---

**kubectl explain 【pod/jobs/persistentvolumes/...】**

例：kubectl explain pod

解释：查看资源清单可配置项目，可使用`kubectl api-resource`查看列出各项

---

**kubectl describe**

例：kubectl describe pod my-pod

解释：查看指定pod的状态详情

---

**kubectl logs**

例：kubectl logs -f --tail=20 my-pod -c init-myservice

解释：查看指定pod的指定容器的日志

---

**kubectl exec**

例：kubectl exec readness-http-pod -c readness-http-pod  -i -t -- bash

解释：查看进入指定pod的指定容器

## 五、资源清单

### 资源分类

**名称空间级别**

工作负载型：`Pod`、`ReplicaSet`、`Deployment`、`StatefulSet`、`DeamonSet`、`Job`、`CronJob`

服务发现及负载均衡资源：`service`、`Ingress`

配置与存储资源：`Volumn`、`CSI`【容器存储接口，用于扩展第三方存储】

特殊类型的存储卷：`ConfigMap`、`Secret`、`DownwardAPI`

**集群级资源**

`Namespace`、`Node`、`Role`、`ClusterRole`、`RoleBinding`、`ClusterRoleBinding`

**元数据级别**

`HPA`、`PodTemplate`、`LimitRange`

### 常用字段

**必须存在的属性**

| **参数名**             | **字段类型** | **说明**                                                     |
| ---------------------- | ------------ | ------------------------------------------------------------ |
| version                | String       | K8S API 的版本，目前基本是v1，可以用 kubectl api-version命令查询 |
| kind                   | String       | 这里指的是 yaml 文件定义的资源类型和角色, 比如: Pod          |
| metadata               | Object       | 元数据对象，固定值写 metadata                                |
| metadata.name          | String       | 元数据对象的名字，这里由我们编写，比如命名Pod的名字          |
| metadata.namespace     | String       | 元数据对象的命名空间，由我们自身定义                         |
| Spec                   | Object       | 详细定义对象，固定值写Spec                                   |
| spec.container[]       | list         | 这里是Spec对象的容器列表定义，是个列表                       |
| spec.container[].name  | String       | 这里定义容器的名字                                           |
| spec.container[].image | String       | 这里定义要用到的镜像名称                                     |

**spec主要对象**

| **参数名**                                  | **字段类型** | **说明**                                                     |
| ------------------------------------------- | ------------ | ------------------------------------------------------------ |
| spec.containers[].name                      | String       | 定义容器的名字                                               |
| spec.containers[].image                     | String       | 定义要用到的镜像的名称                                       |
| spec.containers[].imagePullPolicy           | String       | 定义镜像拉取策略，有 Always，Never，IfNotPresent 三个值课选<br/>（1）Always：默认值，意思是每次尝试重新拉取镜像<br/>（2）Never：表示仅使用本地镜像 <br/>（3）IfNotPresent：如果本地有镜像就是用本地镜像，没有就拉取在 |
| spec.containers[].command[]                 | List         | 指定容器启动命令，因为是数组可以指定多个，不指定则使用镜像打包时使用的启动命令 |
| spec.containers[].args[]                    | List         | 指定容器启动命令参数，因为是数组可以指定多个                 |
| spec.containers[].workingDir                | String       | 指定容器的工作目录                                           |
| spec.containers[].volumeMounts[]            | List         | 指定容器内部的存储卷配置                                     |
| spec.containers[].volumeMounts[].name       | String       | 指定可以被容器挂载的存储卷的名称                             |
| spec.containers[].volumeMounts[].mountPath  | String       | 指定可以被容器挂载的容器卷的路径                             |
| spec.containers[].volumeMounts[].readOnly   | String       | 设置存储卷路径的读写模式，true 或者 false，默认为读写模式    |
| spec.containers[].ports[]                   | List         | 指定容器需要用到的端口列表                                   |
| spec.containers[].ports[].name              | String       | 指定端口名称                                                 |
| spec.containers[].ports[].containerPort     | String       | 指定容器需要监听的端口号                                     |
| spec.containers[].ports.hostPort            | String       | 指定容器所在主机需要监听的端口号，默认跟上面 containerPort 相同，注意设置了 hostPort 同一台主机无法启动该容器的相同副本（因为主机的端口号不能相同，这样会冲突） |
| spec.containers[].ports[].protocol          | String       | 指定端口协议，支持TCP和UDP，默认值为TCP                      |
| spec.containers[].env[]                     | List         | 指定容器运行千需设置的环境变量列表                           |
| spec.containers[].env[].name                | String       | 指定环境变量名称                                             |
| spec.containers[].env[].value               | String       | 指定环境变量值                                               |
| spec.containers[].resources                 | Object       | 指定资源限制和资源请求的值（这里开始就是设置容器的资源上限） |
| spec.containers[].resources.limits          | Object       | 指定设置容器运行时资源的运行上限                             |
| spec.containers[].resources.limits.cpu      | String       | 指定CPU的限制，单位为 core 数，将用于 docker run --cpu-shares 参数 |
| spec.containers[].resources.limits.memory   | String       | 指定 MEM 内存的限制，单位为 MIB，GIB                         |
| spec.containers[].resources.requests        | Object       | 指定容器启动和调度室的限制设置                               |
| spec.containers[].resources.requests.cpu    | String       | CPU请求，单位为 core 数，容器启动时初始化可用数量            |
| spec.containers[].resources.requests.memory | String       | 内存请求，单位为 MIB，GIB 容器启动的初始化可用数量           |

**额外的参数项**

| **参数名**            | **字段类型** | **说明**                                                     |
| --------------------- | ------------ | ------------------------------------------------------------ |
| spec.restartPolicy    | String       | 定义Pod重启策略，可以选择值为Always、OnFailure<br/>（1）Always：默认，Pod终止运行将被重启<br/>（2）OnFailure：只有Pod以非零退出码终止时，kubelet 才会重启该容器<br/>（3）Never：Pod终止后，kubelet将退出码报告给Master，不会重启该Pod |
| spec.nodeSelector     | Object       | 定义Node的Label过滤标签，以key:value格式指定                 |
| spec.imagePullSecrets | Object       | 定义pull镜像是使用secret名称，以name:secretkey格式指定       |
| spec.hostNetwork      | Boolean      | 定义是否使用主机网络模式，默认值为false。设置true表示使用宿主机网络，不使用docker网桥，同时设置了true将无法在同一台宿主机上启动第二个副本 |

### Pod的生命周期

<img src="/img/k8s/pod生命周期.png"  style="zoom:67%;" />

经历过程：①容器初始化环境；②运行容器的pause基础容器；③init c容器（init c可以有多个，串行运行）；④main c启动主容器（启动之初允许执行一个执行命令或一个脚本，在结束的时候允许执行一个命令）；⑤readless监控；⑥liveness监控

#### Init C

Pod能够具有多个容器，应用运行在容器里面，但是它也可能有一个或多个先于应用容器启动的Init容器

Init容器与普通的容器非常像，除了如下两点:

- Init容器总是运行到成功完成为止
- 每个Init容器都必须在下一个Init容器启动之前成功完成，如果Pod的Init容器失败，Kubernetes会不断地重启该Pod，直到Init容器成功为止。如果Pod对应的restartPolicy 为Never, 它不会重新启动

**init C使用示例**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my
spec:
  containers:
  - name: my-container
    image: busybox:1.33.0
    command: ['sh', '-c', 'echo the app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.33.0
    # 寻找myservice域名解析，如果失败休息2s
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.33.0
    # 寻找mydb域名解析，如果失败休息2s
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
     
---    

apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376

---

apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9366
```

- 在Pod启动过程中，Init容器会按顺序在网络和数据卷初始化【即Pause容器】之后启动。每个容器必须在下一个容器启动之前成功退出，如果由于运行时或失败退出，将导致容器启动失败，它会根据Pod的restartPolicy指定的策略进行重试

- 在所有的Init容器没有成功之前，Pod将不会变成Ready状态。Init容器的端口将不会在Service中进行聚集。正在初始化中的Pod处于Pending状态，但应该会将Initializing 状态设置为true
- 如果Pod重启，所有Init 容器必须重新执行
- 对Init容器spec 的修改被限制在容器image字段，修改其他字段都不会生效。更改image字段，等价于重启该Pod
- Init容器具有应用容器的所有字段。除了readinessProbe，因为Init容器无法定义不同于完成(completion)的就绪(readiness) 之外的其他状态。这会在验证过程中强制执行在Pod中的每个app和Init容器的名称必须唯一

#### 容器探针

探针是由kubelet对容器执行的定期诊断。kubelet调用由容器实现的Handler即节点自动调用。

**类型**

- ExecAction: 在容器内执行指定命令，如果命令退出返回码为0则认为成功
- TCPSocketAction：对指定端口上的容器的IP地址做TCP检查，如果端口打开则被认为成功
- HTTPGetAction: 对指定端口和路径上的容器IP指定HTTP Get请求，如状态码大于等于200且小于400则被认为成功

**方式**

- livenessProbe： 容器是否正在运行。如果存活探测失败，则kubelet会杀死容器并根据RestarPolicy策略执行相应操作。如果容器不提供存活探针，默认状态为success
- readnessProbe： 容器是否准备好服务请求，如果失败，端点控制器从与Pod匹配的所有Service的端点中删除该Pod的ip地址，初始延迟之前就绪状态为Failure。如果容器不提供就绪探针，默认状态为success

**探针使用示例**

- readnessProbe-HTTPGetAction联合使用

    如果nginx下存在readiness.html则就绪检测完成

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: readness-http-pod
    spec:
      containers:
      - name: readness-http-pod
        image: nginx:1.19
        imagePullPolicy: IfNotPresent
        readinessProbe:
          httpGet:
            port: 80
            path: /readiness.html
          initialDelaySeconds: 1
          periodSeconds: 3
    ```

    容器一直处于READY未就绪状态，当使用`echo "page" >> readiness.html`创建文件之后即可变为就绪状态

- livenessProbe-ExecAction联合使用

    创建文件60秒之后删除，动态检测这个文件，如果文件不存则重启Pod

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness-exec-pod
      namespace: default
    spec:
      containers:
      - name: liveness-exec-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh", "-c", "touch /tmp/live; sleep 60; rm -f /tmp/live;sleep 3600"]
        livenessProbe:
          exec:
            command: ["test", "-e", "/tmp/live"]
          initialDelaySeconds: 1
          periodSeconds: 3
    ```

- livenessProbe-HTTPGetAction联合使用

    启动容器如果index.html无法访问则重启

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness-httpget-pod
      namespace: default
    spec:
      containers:
      - name: liveness-httpget-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            port: 80
            path: /index.html
          initialDelaySeconds: 1
          periodSeconds: 3
          timeoutSeconds: 10
    ```

- livenessProbe-TCPSocketAction联合使用

    检测容器80端口是否打开，

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness-tcp-pod
      namespace: default
    spec:
      containers:
      - name: liveness-tcp-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent
        restartPolicy: always
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 1
          periodSeconds: 3
          timeoutSeconds: 10
    ```

#### 启动和退出动作

在容器启动前后可执行操作

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle
spec:
  containers:
  - name: lifecycle-container
    image: nginx:1.19
    lifecycle:
      # 启动后运行
      postStart:
        exec:
          command: ["/bin/sh","-c","echo Hello from the postStart handler> /usr/share/message"]
      # 退出前运行
      preStop:
        exec:
          command: ["/bin/sh","-c","echo Hello from the poststop handler> /usr/share/message"]
```

## 六、控制器

Kubernetes中内建了很多controller (控制器)，这些控制器相当于一个状态机，用来控制Pod的具体状态和行为

### ReplicationController

ReplicationController (RC)用来确保容器应用的副本数始终保持在用户定义的副本数。即如果有容器异常退出，会自动创建新的Pod来替代，而如果异常多出来的容器也会自动回收。

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: nginx-rs
  template:
    metadata:
      labels:
        tier: nginx-rs
    spec:
      containers:
        - name: nginx-container
          image: nginx:1.19
```

### Deployment

Deployment为Pod和ReplicaSet提供了一个声明式定义(declarative)方法，用来替代以前的ReplicationController来方便的管理应用。

<img src="/img/k8s/deployment-scale.png"  style="zoom:60%;" />

- 定义Deployment来创建Pod和ReplicaSet

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 3
      # 保存回滚历史数，默认为10
      revisionHistoryLimit: 10
      selector:
        matchLabels:
          tier: nginx-deployment
      template:
        metadata: 
          labels:
            tier: nginx-deployment
        spec:
          containers: 
            - name: nginx-container
              image: nginx:1.19
    ```

- 滚动升级和回滚应用

    ```shell
    # 更改镜像版本，也可以使用kubectl edit deployment/nginx-deployment修改版本
    kubectl set image deployment/nginx-deployment nginx-container=nginx:1.18
    
    # 查看可回滚的版本，如果rs已经不存在，则历史版本也不存在
    kubectl rollout history deployment/nginx-deployment
    
    # 回滚
    kubectl rollout undo deployment/nginx-deployment [--to-revision=1]
    
    # 查看回滚状态
    kubectl rollout status deployment/nginx-deployment
    
    # 查看当前deployments详细信息，内有RollingUpdateStrategy更新策略，默认每次25%
    kubectl describe deployments
    ```

- 扩容和缩容

    ```shell
    kubectl scale deployment nginx-deployment --replicas=5
    ```

- 暂停和继续Deployment

    ```shell
    # 暂停更新
    kubectl rollout pause deployment/nginx-deployment
    
    # 继续更新
    kubectl rollout resume deployment/nginx-deployment
    ```

### DaemonSet

DaemonSet确保全部（或者一些）Node上运行一个Pod的副本。当有Node加入集群时，也会为他们新增一个Pod。当有Node从集群移除时，这些Pod也会被回收。删除DaemonSet将会删除它创建的所有Pod使用DaemonSet的一些典型用法

- 运行集群存储daemon，例如在每个Node上运行`glusterd`、`ceph`
- 在每个Node上运行日志收集daemon，例如`fluentd`、`logstash`
- 在每个Node上运行监控daemon，例如`Prometheus Node Exporter`、 `collectd`、 `Datadog代理`、`New Relic代理`，或Ganglia `gmond`

### StateFulSet

StatefulSet作为Controller为Pod提供唯一的标识。它可以保证部署和scale的顺序

StatefulSet是为了解决有状态服务的问题(对应Deployments和ReplicaSets是为无状态服务而设计) ，其应用场景包括:

- 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
- 稳定的网络标志,即Pod重新调度后其PodName和HostName不变，基于Headless Service (即没有Cluster IP的Service)来实现
- 有序部署，有序扩展，即Pod是有顺序的，在部或者扩展的时候要依据定义的顺序依次依次进行(即从到N-1,在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态)，基于init containers来实现
- 有序收缩，有序删除(即从N-1到0)

<a href="#pv-eg">StateFulSet一半与PV/PVC联用，详细案例</a>

### Job/CronJob

Job负责批处理任务，即仅执行一次的任务， 它保证批处理任务的一个或多个Pod成功结束。

- ` job.spec.template.spec.restartPolicy`仅支持`Never`或`OnFailure`
- `job.spec.completions`标志Job结束需要成功运行的Pod个数，默认为1
- `job.spec.parallelism`标志并运行的Pod个数，默认为1
- `job.spec.activeDeadlineSeconds`标志失败Pod的重试最大时间，超过不继续重试

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  completions: 1
  activeDeadlineSeconds: 3600
  parallelism: 1
  template:
    metadata:
      name: pi
    spec:
      restartPolicy: Never
      containers:
        - name: pi-container
          image: perl:5.32.0
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```

Cron Job管理基于时间的Job，原理是根据cron表达式调度Job，可以在给定时间点只运行一次，也可以周期性地在给定时间点运行

- `cronJob.spec.startingDeadlineSeconds`：启动Job的期限(秒级别)，该字段是可选的。如果因为任何原因而错过了被调度的时间，那么错过执行时间的Job将被认为是失败的。如果没有指定，则没有期限。

- `cronJob.spec.suspend`：挂起，该字段是可选的。如果设置为true，后续所有执行都会被挂起。它对已经开始执行的Job不起作用。默认值为false.
- `cronJob.spec.concurrencyPolicy`：并发策略，该字段是可选的。它指定了如何处理被Cron Job创建的Job的并发执行。
    - Allow（默认）：允许并发运行Job
    - Forbid：禁止井发运行,如果前一个还没有完成，则直接跳过下一个
    - Replace：取消当前正在运行的Job,用一个新的来替换

- `cronJob.spec.successfulJobsHistoryLimit`和`cronJob.spec.failedJobsHistoryLimit`：历史限制，是可选的字段。它们指定了可以保留多少完成和失败的Job。默认情况下，它们分别设置为3和1

典型的用法如下所示:

- 在给定的时间点调度Job运行
- 创建周期性运行的Job,例如:数据库备份、发送邮件

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello-corn
spec:
  concurrencyPolicy: Allow
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello-container
            image: busybox:1.33.0
            args:
            - "/bin/sh"
            - "-c"
            - "date;echo Hello From Kubernetes Cluster"
          restartPolicy: OnFailure
```

### Horizontal Pod Autoscaling

应用的资源使用率通常都有高峰和低谷的时候，如何削峰填谷，提高集群的整体资源利用率，让service中的Pod个数自动调整呢?这就有赖于Horizontal Pod Autoscaling了,顾名思义，使Pod水平自动缩放。

使用此功能需要先部署`metrics-server `实现指标监控，[部署方式详见](https://github.com/kubernetes-incubator/metrics-server)

```yaml
########################### HorizontalPodAutoscaler/v1 扩容【仅限CPU指标】 ###########################
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-autoscaling
  namespace: default
spec:
  scaleTargetRef:       
    apiVersion: apps/v1
    kind: Deployment   
    name: nginx-deployment
  minReplicas: 1                
  maxReplicas: 3
  # cpu利用率超过50%扩容，基于Pod设置的CPU Request值进行计算，例如该值为200m，那么系统将维持Pod的实际CPU使用值为100m。
  targetCPUUtilizationPercentage: 50
  
########################### HorizontalPodAutoscaler/v2beta1 扩容 ###########################

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-autoscaling
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment        # 基于Deployment进行扩缩
    name: nginx-deployment  # Deployment名
  minReplicas: 1   # 最小实例数
  maxReplicas: 2   # 最大实例数
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50  # CPU阈值设定50%
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 200Mi  # 内存设定200M
  - type: Object
    object:
      metricName: requests-per-second
      target:
        apiVersion: extensions/v1beta1
        kind: Ingress
        name: main-route
      targetValue: 10k   # 每秒请求量
      
---

# 若需要指标生效，需要一定注明该pod的request cpu和memory

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: nginx-deployment
  template:
    metadata: 
      labels:
        tier: nginx-deployment
    spec:
      containers: 
        - name: nginx-container
          image: nginx:1.19
          resources: 
            # 软限制
            requests:
              memory: 50Mi
              cpu: 1500m #1000m为一个CPU
            # 硬限制
            limits:
              memory: 100Mi  
              cpu: 2000m
```

## 七、Service

### Service的概念

`Service`是一种抽象，它定义了一组`Pods`的逻辑集合和一个用于访问它们的策略。

### Service的类型

- `ClusterIp`，默认类型，自动分配一个仅Cluster内部可以访问的虚拟IP

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deploy
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
         app: nginx-app
      template:
        metadata:
          labels:
            app: nginx-app
        spec:
          containers:
          - name: nginx-container
            image: nginx:1.19
            imagePullPolicy: IfNotPresent
    
    ---
    
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      namespace: default
    spec:
      type: ClusterIP
      selector:
        app: nginx-app
      ports:
      - name: http
        # 自己想暴露出的端口
        port: 30001
        # 对应容器暴露的端口
        targetPort: 80
    ```

    使用`kubectl get svc`可查看生成的随机IP，使用生产的随机IP加上指定的3001端口在集群内可访问` curl 随机IP:30001`

- `Headless Service`，有时不需要负载均衡，以及单独的Service IP。这时可以通过指定`spec.clusterlP`的值为"None"来创建Headless Service。这类Service并不会分配Cluster IP，`kube-proxy`不会处理它们，而且平台也不会为它们进行负载均衡和路由

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deploy
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
         app: nginx-app
      template:
        metadata:
          labels:
            app: nginx-app
        spec:
          containers:
          - name: nginx-container
            image: nginx:1.19
            imagePullPolicy: IfNotPresent
    
    ---
    
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      namespace: default
    spec:
      clusterIP: None
      selector:
        app: nginx-app
      ports:
      - name: http
        # 自己想暴露出的端口
        port: 30001
        # 对应容器暴露的端口
    ```

- `NodePort`，在ClusterlP基础上为Service在<span style="color:red">每台机器上</span>绑定一个随机端口，这样就可以通过NodePort来访问该服务

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deploy
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
         app: nginx-app
      template:
        metadata:
          labels:
            app: nginx-app
        spec:
          containers:
          - name: nginx-container
            image: nginx:1.19
            imagePullPolicy: IfNotPresent
    
    ---
    
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      namespace: default
    spec:
      type: NodePort
      selector:
        app: nginx-app
      ports:
      - name: http
        # 自己想暴露出的端口
        port: 30001
        # 对应容器暴露的端口
        targetPort: 80
    ```

    会生成一个随机端口再次映射到指定端口，这是可以用"[集群任意主机IP]:随机端口"访问服务

- `LoadBalancer`，loadBalancer和nodePort其实是同一种方式。区别在于loadBalancer比nodePort多了一步，就是可以调用云服务厂商去创建LB向节点导流，但是需要付费

- `Endpoint`，是k8s集群中的一个资源对象，存储在etcd中，用来记录一个service对应的所有pod的访问地址

    ```yaml
    apiVersion: v1
    kind: Endpoints
    apiVersion: v1
    metadata:
      #此名字需与service文件中的metadata.name的值一致
      name: remote-mysql-endpoint
      namespace: default
    subsets:
    - addresses:
      # 真实的IP
      - ip: 192.168.22.1
      # 随便写的公网IP
      - ip: 220.181.38.148
      ports:
      - port: 3306
    
    ---
    
    apiVersion: v1
    kind: Service
    metadata:
      #此名字需与endpoints文件中的metadata.name的值一致
      name: remote-mysql-endpoint
      namespace: default
    spec:
      ports:
      - port: 3306
    ```

    以后可用随便写的那个公网IP代替真实IP进行访问，如果真实IP发生变化不影响程序。此种方式只适合Ip访问，对于像阿里云等数据库的。需要用域名。则需要用`ExternalName`方式不而不是`Endpoints`方式。

- `ExternalName`，把集群外部的服务引入到集群内部中，适用于集群内部容器访问外部资源，没有任何类型代理被创建。

    ```yaml
    # 在集群任意的pod中使用external-name-service均会被解析为www.baidu.com
    # 即相当于创建了SVC_NAME.NAMESPACE.svc.cluster.local到指定域名、ip的DNS解析
    kind: Service
    apiVersion: v1
    metadata:
      name: external-name-service
      namespace: default
    spec:
      type: ExternalName
      externalName: www.baidu.com
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
    ```

    使用external-name-service代替百度域名，可用于云数据库

### Ingress

管理对集群中的服务（通常是HTTP）的外部访问的API对象。Ingress可以提供负载平衡、SSL终端和基于名称的虚拟主机。

**安装**

```shell
# 安装nginx-controller
# 复制 https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/deploy/static/mandatory.yaml
kubectl apply -f mandatory.yaml

# 复制https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
# 使用nodeport的方式暴露nginx-controller
kubectl apply -f service-nodeport.yaml

# 查看是否安装成功
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch

# 查看nginx ingress controller暴露的端口
kubectl get svc -n ingress-nginx
```

**nginx-Ingress原理图**

<img src="/img/k8s/nginx-Ingress原理图.png"  style="zoom:80%;" />

**Http代理访问**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
     app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent

---

# 最好创建的是无头svc
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP 
  selector:
    app: nginx-app
  ports:
  - name: http
    port: 80
    
---

# 访问www.kun.com:[ingress映射端口]即可实现负载均衡的访问效果，可以匹配多个主机和多个service
# 必须通过域名访问不能通过IP访问
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: www.kun.com
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service: 
            name: nginx-service
            port:
              number: 80
```

**Https代理访问**

①、创建证书

```shell
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj
"/CN=nginxsvc/0=nginxsvc"
```

②、创建secret

```shell
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

③、使用ingress

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
     app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent

---

# 最好创建的是无头svc
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP 
  selector:
    app: nginx-app
  ports:
  - name: http
    port: 80
    
---

# 访问www.kun.com:[ingress映射端口]即可实现负载均衡的访问效果，可以匹配多个主机和多个service
# 必须通过域名访问不能通过IP访问
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  tls: 
  # 要和下面的域名匹配
  - hosts: 
    - www.kun.com
    secretName: tls-secret
  rules:
  - host: www.kun.com
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service: 
            name: nginx-service
            port:
              number: 80
```

**auth认证**

①、创建认证

```shell
yum -y install httpd
htpasswd -c auth [userName]
kubectl create secret generic basic-auth --from-file=auth
```

②、创建Ingress

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
     app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent

---

# 最好创建的是无头svc
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP 
  selector:
    app: nginx-app
  ports:
  - name: http
    port: 80
    
---

# 访问www.kun.com:[ingress映射端口]即可实现负载均衡的访问效果，可以匹配多个主机和多个service
# 必须通过域名访问不能通过IP访问
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - kun'
spec:
  rules:
  - host: www.kun.com
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service: 
            name: nginx-service
            port:
              number: 80	
```

## 八、存储

### ConfigMap

许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。ConfigMap给提供了向容器中注入配置信息的机制的功能，可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制大对象

**准备工作**

```txt
文件：doc/user.properties
userName=TBB
age=19

文件：doc/game.properties
name=lol
age=18+

文件：doc/conf/file.conf
appConfigMap
```

**创建-使用目录**

```shell
kubectl create configmap file-config --from-file=doc/conf
```

**创建-使用文件**

```shell
kubectl create configmap user-config --from-file=doc/user.properties
kubectl create configmap game-config --from-file=doc/game.properties
```

**创建-使用字面值**

```shel
kubectl create configmap special-config --from-literal=special.city=BJ --from-literal=special.province=BJ
```

**创建-使用yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: yaml-config
data:
  special.how: Lili
  special.doing: working
# 执行 kubectl create  -f  config.yaml
```

**使用-代替环境变量，启动参数**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: confgmap-pod
  namespace: default
spec:
  containers:
  - name: confgmap-pod-container
    image: busybox:1.33.0
    command: ['sh', '-c', 'echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY);sleep 3600']
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: yaml-config
          key: special.how
    - name: SPECIAL_DOING
      valueFrom:
         configMapKeyRef:
           name: yaml-config
           key: special.doing
    envFrom:
    - configMapRef:
        name: file-config

# 使用env命令查看环境变量
# SPECIAL_LEVEL_KEY=Lili
# SPECIAL_DOING=working
# file.conf=appConfigMap
```

### Secret

Secret解决了密码、token、密钥等敏感数据的配置问题，不需要把这些数据暴露到镜像或者`pod.spec`中。Secret可以以Volume或者环境变量的方式使用

**Secret有三种类型**

- `Service Account`：来访问Kubernetes API，由Kubernetes自动创建并且会自动挂载到Pod的`/run/secrets/kubernetes.io/ serviceaccount`目录中，用户不使用【内部组件用于访问Kubernetes时使用】。
- `Opaque`：base64编码格式的Secret，用来存储密码、密钥等
- `kubernetes.io/dockerconfigison`：来存储私有docker registry的认证信息

#### Opaque Secret

**创建**

```yaml
# echo -n "admin" | base64
# YWRtaW4=
# echo -n "123456" | base64
# MTIzNDU2

apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
  
# 创建命令 kubectl create -f my-secret.yaml
```

**在环境变量中使用**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: secret-env-container
    image: nginx:1.19
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
# env | grep SECRET
# SECRET_USERNAME=admin
# SECRET_PASSWORD=123456
```

**挂在到volume中使用**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: secret-volume-pod-container
    image: nginx:1.19
    volumeMounts:
    - name: secrets
      mountPath: "/secret"
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
# 会用key生成文件名，value变为内容
# ls /secret
# password  username
```

#### kubernetes.io/dockerconfigison

使用Kuberctl创建docker registry认证的secret

```shell
kubectl create secret docker-registry docker-wyk-reg \
--docker-server=[DOCKER_REGISTRY_SERVER] \
--docker-username=[DOCKER_USER] \
--docker-password=[DOCKER_ PASSWORD] \
--docker-email=[DOCKER_EMAIL] 
```

在创建Pod的时候，通过imagePullSecrets来引用

```yaml
apiVersion: v1
kind: Pod
metadata :
  name: secret-image
spec:
  containers:
  - name: nginx
    image: wangyukun/nginx:v1
  imagePullSecrets :
  - name: docker-wyk-reg
```

### Volume

容器磁盘上的文件的生命周期是短暂的，这就使得在容器中运行重要应用时会出现一些问题。 首先，当容器崩溃时kubelet会重启它，但是容器中的文件将丢失【容器以干净的状态（镜像最初的状态）重新启动】。其次，在Pod中同时运行多个容器时，这些容器之间通常需要共享文件。Kubernetes中的Volume抽象就很好的解决了这些问题。

Kubernetes中的volume寿命与封装它的Pod相同。当Pod不再存在时,卷也将不复存在。

#### emptyDir

当Pod分派到某个node上时，emptyDir卷在node上会被创建，并且在Pod在该节点上运行期间，卷最初是空的，卷一直存在。尽管Pod中的容器挂载emptyDir卷的路径可能相同也可能不同，这些容器都可以读写emptyDir卷中相同的文件。当Pod 因为某些原因被从节点上删除时，emptyDir卷中的数据也会被永久删除。一般用于存储缓存数据

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      tier: nginx-deployment
  template:
    metadata: 
      labels:
        tier: nginx-deployment
    spec:
      containers: 
        - name: nginx-container
          image: nginx:1.19
          volumeMounts:
          - mountPath: /cache
            name: cache-volume
      volumes:
      - name: cache-volume
        emptyDir: {}
```

#### hostPath

hostPath能将主机节点文件系统上的文件或目录挂载到Pod中。除了必需的path属性之外，用户可以选择性地为hostPath卷指定 type

| 值                | 行为                                                         |
| ----------------- | ------------------------------------------------------------ |
| 空字符串（默认）  | 用于向后兼容，这意味着在安装hostPath卷之前不会执行任何检查   |
| DirectoryOrCreate | 如果在给定路径上什么都不存在则创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息 |
| Directory         | 在给定路径上必须存在的目录                                   |
| FileOrCreate      | 如果在给定路径上什么都不存在，则创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权 |
| File              | 在给定路径上必须存在的文件                                   |
| Socket            | 在给定路径上必须存在的 UNIX 套接字                           |
| CharDevice        | 在给定路径上必须存在的字符设备                               |
| BlockDevice       | 在给定路径上必须存在的块设备                                 |

注意事项：由于每个节点上的文件都不同，具有相同配置的Pod在不同节点上的行为可能不同

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      tier: nginx-deployment
  template:
    metadata: 
      labels:
        tier: nginx-deployment
    spec:
      containers: 
        - name: nginx-container
          image: nginx:1.19
          volumeMounts:
          - mountPath: /cache
            name: cache-volume
      volumes:
      - name: cache-volume
        hostPath:
          path: /data
          type: Directory
```

> node可以使用NFS挂在目录，这样所有pod的数据均一样

### <span id="pv-eg">Persistent Volume</span>

`Volume`主要是为了存储一些有必要保存的数据，而`Persistent Volume`主要是为了管理集群的存储，`Persistent Volume`对具体的存储进行配置和分配，而Pods等则可以使用`Persistent Volume`抽象出来的存储资源，不需要知道集群的存储细节。

`Persistent Volume`和`Persistent Volume Claim`类似Pods和Nodes的关系，`Persistent Volume Claim`提出需要的存储标准，然后从现有存储资源中匹配或者动态建立新的资源，最后将两者进行绑定。

Pods使用的是`PersistentVolumeClaim`而非`PersistentVolume`

**PV访问模式**

PersistentVolume可以以资源提供者支持的任何方式挂载到主机上。每个PV都有一套自己的用来描述特定功能的访问模式

- `ReadWriteOnce`：该卷可以被单个节点以读/写模式挂载，缩写为RWO
- `ReadOnlyMany`：该卷 可以被多个节点以只读模式挂载，缩写为ROX
- `ReadWriteMany`：该卷 可以被多个节点以读/写模式挂载，缩写为RWX

**回收策略**

- Retain（保留）：手动回收
- Recycle（回收，已废弃）：基本擦除( rm -rf /thevolume/* )
- Delete（删除）：关联的存储资产（例如AWS EBS、 GCE PD、Azure Disk和OpenStack Cinder卷）将被删除

> 当前，只有NFS和HostPath支持Recycle策略。AWS EBS、GCE PD、Azure Disk和Cinder卷支持Delete策略

**PV状态**

- Available（可用）：空闲资源还没有被任何声明绑定
- Bound（已绑定）：卷已经被声明绑定
- Released（已释放）：声明被删除，但是资源还未被集群重新声明
- Failed（失败）：该卷的自动回收失败

**PV的分类**

- 静态

    集群管理员创建一些PV。它们带有可供群集用户使用的实际存储的细节。它们存在于Kubernetes API中，可用于消费。

- 动态
    当管理员创建的静态PV都不匹配用户的PVC时，集群会尝试动态地为PVC创建卷。此配置基于`StorageClasses`来对接存储，并且管理员必须创建并配置该类才能进行动态创建。声明该类为""可以有效地禁用其动态配置，要启用基于存储级别的动态存储配置，集群管理员需要启用API server上的`DefaultStorageClass`准入控制器

**案例：搭建NFS，抽象出各种存储对象（PV），创建Pod使用PVC绑定PV**

①、搭建NFS

```shell
yum install -y nfs-common nfs-utils rpcbind
mkdir /nfsdata
chmod 666 /nfsdata
chown nfsnobody /nfsdata

vim /etc/exports 
/nfsdata *(rw,no_root_squash,no_all_squash,sync)

systemctl start rpcbind
systemctl start nfs

# 测试
showmount -e centos165
```

②、创建PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001nfs
spec:
  storageClassName: manualSpeed
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: centos165
    path: "/nfsdata"
    
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002nfs
spec:
  storageClassName: manualSpeed
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: centos165
    path: "/nfsdata"

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv003nfs
spec:
  storageClassName: slowSpeed
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: centos165
    path: "/nfsdata"
```

③、创建服务并绑定PVC

```yaml
# 创建无头service
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  clusterIP: None
  selector:
    app: nginx-app
  ports:
  - name: http
    # 自己想暴露出的端口
    port: 80
    # 对应容器暴露的端口
    
---
# 访问www.kun.com:[ingress映射端口]即可实现负载均衡的访问效果，可以匹配多个主机和多个service
# 必须通过域名访问不能通过IP访问
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: www.kun.com
    http:
      paths:
      - path: /
        pathType: Exact
        backend:
          service: 
            name: nginx-service
            port:
              number: 80
              
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: nginx-service
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 3
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: "manualspeed"
      resources:
        requests:
          storage: 5Gi
```















## 九、集群调度

调度分为几个部分：首先是过滤掉不满足条件的节点，这个过程称为`Predicate`然后对通过的节点按照优先级排序，这个是`Priority`最后从中选择优先级最高的节点。如果在`Predicate`过程中没有合适的节点，Pod会一直在pending状态，不断重试调度直到有节点满足条件。经过这个步骤，如果有多个节点满足条件，就继续`Prioritiy`过程，按照优先级大小对节点排序

`Predicate`有一系列的算法可以使用：

- `PodFitsResources`：节点上剩余的资源是否大于pod请求的资源

- `PodFitsHost`：如果pod指定了NodeName，检查节点名称是否和NodeName匹配

- `PodFitsHostPorts`：节点上已经使用的port是否和pod申请的port冲突

- `PodselectorMatches`：过滤掉和pod指定的label不匹配的节点

- `NoDiskConflict`：已经mount的volume和pod指定的volume不冲突，除非它们都是只读

`Prioritiy`由一系列键值对组成，键是该优先级项的名称，值是它的权重（该项的重要性）。这些优先级选项包括：

- `LeastRequestedPriority`：通过计算CPU和Memory的使用率来决定权重，使用率越低权重越高
- `BalancedResourceallocation`：点上CPU和Memory使用率越接近，权重越高
- `ImagelocalityPriority`：倾向于已经有要使用镜像的节点，镜像总大小值越大，权重越高

### 节点亲和性

查看&设置节点标签

```shell
# 设置标签
kubectl label node centos166 ssd=true city=BJ
# 查看标签
 kubectl get nodes --show-labels
```

`pod.spec.affinity.nodeAffinity`【节点亲和性设置】

- preferredDuringSchedulingIgnoredDuringExecution【软策略，策略是偏向于，更想(不)落在某个节点上，但如果实在没有，落在其他节点也可以】

    ```yaml
    # Pod最优部署在节点标签ssd="true"节点
    apiVersion: v1
    kind: Pod
    metadata:
      name: affinity-preferred
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 10
            preference:
              matchExpressions:
              - key: ssd
                # In、Exists、NotIn、DoesNotExist、Gt（字符串比较）、Lt（字符串比较）
                operator: In
                values:
                - "true"
    ```

- requiredDuringSchedulingIgnoredDuringExecution【硬策略，硬策略是必须(不)落在指定的节点上，如果不符合条件，则一直处于Pending状态】

    ```yaml
    # Pod不会部署在centos166节点
    apiVersion: v1
    kind: Pod
    metadata:
      name: affinity-required
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.19
        imagePullPolicy: IfNotPresent
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
                - key: kubernetes.io/hostname
                  operator: NotIn
                  values:
                  - centos166
    ```

### Pod亲和性

查看Pod标签

```shell
 kubectl get pod --show-labels
```

`pod.spec.affinity.podAffinity`（POD亲和性）/`pod.spec.affinity.podAntiAffinity`（POD反亲和性）

- `requiredDuringSchedulingIgnoredDuringExecution`硬策略

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-required
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - "demo"
        # 表示使用主机匹配
        topologyKey: kubernetes.io/hostname
  containers:
  - name: pod-affinity
    image: nginx:1.19
    imagePullPolicy: IfNotPresent
```

- `preferredDuringSchedulingIgnoredDuringExecution`软策略

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-preferred
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 10
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: NotIn
              values:
              - "demo"
          # 表示使用主机匹配
          topologyKey: kubernetes.io/hostname
  containers:
  - name: pod-affinity
    image: nginx:1.19
    imagePullPolicy: IfNotPresent
```

### 污点

`Taint`使节点能够排斥一类特定的pod

**污点（Taint）的组成**

使用`kubectl taint`命令可以给某个Node节点设置污点，Node被设置上污点之后就和Pod之间存在了一种相斥的关系，可以让Node拒绝Pod的调度执行，甚至将Node已经存在的Pod驱逐出去。

每个污点的组成为`key=value:effect`，每个污点有一个key和value作为污点的标签，value可以为空，effect描述污点的作用。effect支持以下三个选项：

- `NoSchedule`：表示k8s将不会将Pod调度到具有该污点的Node上
- `PreferNoSchedule`：表示k8s将尽量避免将Pod调度到具有该污点的Node上
- `NoExecute`：表示k8s将不会将Pod调度到具有该污点的Node上，同时会将Node上已经存在的Pod驱逐出去

```shell
#设置污点
kubectl taint nodes [nodeName] [key]=[value]:[effect]

#节点说明中，查找Taints字段
kubectl describe node [podName]

#去除污点
kubectl taint nodes [nodeName] [key]=[value]:[effect]-
```

> 有多个Master存在时，防止资源浪费，可以设置
> `kubectl taint nodes [nodeName] node-role.kubernetes.io/master:PreferNoSchedule`

### 容忍

设置了污点的Node将和Pod发生互斥。Pod将在一定程度上不会被调度到Node上，但我们可以在Pod上设置容忍（Toleration），设置了容忍的Pod将可以容忍污点的存在，可以被调度到存在污点的Node上。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      tier: nginx-deployment
  template:
    metadata: 
      labels:
        tier: nginx-deployment
    spec:
      tolerations:
        # 当不指定key值时，表示容忍所有的污点key
      - key: "disk"
        # operator的值为Exists将会忽略value值
        operator: "Equal"
        value: "error"
        # 当不指定effect值时，表示容忍所有的污点作用
        effect: "NoExecute"
        # 如果被别的条件触发驱逐，保留的运行时间
        tolerationSeconds: 3600
      containers: 
        - name: nginx-container
          image: nginx:1.19
```

### 固定节点

`pod.spec.nodeName`将Pod直接调度到指定的Node节点上，会跳过Scheduler的调度策略，该匹配规则是强制匹配

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      tier: nginx-deployment
  template:
    metadata: 
      labels:
        tier: nginx-deployment
    spec:
      nodeName: centos166
      containers: 
        - name: nginx-container
          image: nginx:1.19
```

`pod.spec.nodeSelector` 由调度器调度策略匹配label，而后调度Pod到目标节点，该四配规则属于强制约束

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      tier: nginx-deployment
  template:
    metadata: 
      labels:
        tier: nginx-deployment
    spec:
      nodeSelector: 
        ssd: "true"
      containers: 
        - name: nginx-container
          image: nginx:1.19
```

## 十、安全

### 机制说明

Kubernetes作为一个分布式集群的管理工具，保证集群的安全性是其一个重要的任务。 API Server是集群内部各个组件通信的中介，也是外部控制的入口。所以Kubernetes的安全机制基本就是围绕保护API Server来设计的。Kubernetes使用了认证（Authentication）、鉴权（Authorization）、准入控制（AdmissionControl）三步来保证API Server的安全。

### 认证

集群和节点之间是通过双向HTTPS证书认证的，在Kubernetes中需要认证的节点分为两种

- Kubenetes组件对API Server的访问：kubectl、Controller Manager、Scheduler、kubelet、kube-proxy
- Kubernetes管理的Pod对容器的访问: Pod 

安全性说明

- Controller Manage、Scheduler与API Server在同一台机器，所以直接使用API Server的非安全端口访问
- kubectl、kubelet、kube-proxy问API Server就都需要证书进行HTTPS双向认证

证书颁发

- 手动签发：通过k8s集群的跟ca进行签发HTTPS证书
- 自动签发：kubelet首次访问API Server时，使用token做认证，通过后Controller Manager会为kubelet生成一个证书,以后的访问都是用证书做认证了

**kubeconfig**

kubeconfig文件包含集群参数（CA证书、API Server地址）客户端参数(上面生成的证书和私钥)，集群context信息(集群名称、用户名)。Kubenetes 组件通过启动时指定不同的kubeconfig文件可以切换到不同的集群。

**ServiceAccount**

Pod中的容器访问API Server。因为Pod的创建、销毁是动态的,所以要为它手动生成证书就不可行了。Kubenetes使用了Service Account解决Pod问API Server的认证问题

**Secret 与SA的关系**

Kubernetes设计了一种资源对象叫做Secret，分为两类一种是用于ServiceAccount的service-account-token，另一种是用于保存用户自定义保密信息的Opaque。ServiceAccount 中用到包含三个部分: Token、ca.crt、namespace

- token是使用 API Server私钥签名的JWT。于访问API Server时，Server端认证

- ca.crt, 根证书。于Client端验证API Server发送的证书

- namespace, 标识这个service-account-token的作用域名空间

    ```shell
    kubectl get secret --all-namespaces
    kubectl describe secret default-token-5gm9r --namespace-kube-system
    ```

    默认情况下，每个namespace都会有一个ServiceAccount，如果Pod在创建时没有指定ServiceAccount，就会使用Pod所属的namespace的ServiceAccount

<img src="/img/k8s/认证流程.png" style="zoom:67%;" />

### 鉴权

RBAC（Role-Based Access Control）基于角色的访问控制,在Kubernetes 1.5中引入，现行版本成为默认标准

**RBAC的API资源对象说明**

<img src="/img/k8s/rbac.jpg" style="zoom:43%;" />

- `Role`：表示一组规则权限，权限只会增加（累加权限）

    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata :
      # 只有在指定的命名空间有效
      namespace: default
      name: pod-reader-role
    rules:
    - apiGroups: [""] # 空字符串""表明使用 core API group，就是每个YMAL声明得apiVersion的“/”前面得组
      resources: ["pods"]
      verbs: ["get", "watch", "list"]
    ```

- `ClusterRole`：可以授予与Role对象相同的权限，但由于它们属于集群范围对象，也可以使用它们授予对以下几种资源的访问权限

    - 集群级别的资源控制（例如node访问权限）
    - 非资源型endpoints（例如/healthz访问）
    - 所有命名空间资源控制（例如pods）

    ```yaml
    ind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      # ClusterRole 是集群范围对象，没有 "namespace" 区分
      name: secrets-cluster-role
    rules:
    - apiGroups: [""]
      resources: ["secrets"]
      verbs: ["get", "watch", "list", "create", "delete"]
    ```

- `RoloBinding`：把Role或ClusterRole中定义的各种权限映射到User，Service Account或者Group，从而让这些用户继承角色在 namespace中的权限。`RoloBinding`也能绑定`ClusterRole`，但是只能在指定名称空间下。

    ```yaml
    # 以下角色绑定定义将允许用户 "jane" 从 "default" 命名空间中读取pod
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-pods-binding
      # 只有在指定的命名空间有效
      namespace: default
    subjects:
    - kind: User
      name: jane
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: pod-reader-role
      apiGroup: rbac.authorization.k8s.io
    
    ---
    
    # 以下角色绑定允许用户"dave"读取"development"命名空间中的secret。
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-secrets-binding
      # 这里表明仅授权读取"development"命名空间中的资源。
      namespace: development
    subjects:
    - kind: User
      name: dave
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: secrets-cluster-role
      apiGroup: rbac.authorization.k8s.io
    ```

- `ClusterRoleBinding`：和`RoloBinding`功能类似，让用户继承ClusterRole在整个集群中的权限

    ```yaml
    # 以下`ClusterRoleBinding`对象允许在Linux用户组"manager"中的任何用户都可以读取集群中任何命名空间中的secret。
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-secrets-global-binding
    subjects:
    - kind: Group
      name: manager
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: secrets-cluster-role
      apiGroup: rbac.authorization.k8s.io
    ```

**对资源的引用**

大多数资源由代表其名字的字符串表示，例如"pods"。就像它们出现在相关API endpoint的URL中一样。然而，有一些Kubernetes API还 包含了"子资源"，比如 `pod` 的 `logs`。在Kubernetes中，pod logs endpoint的URL格式为：

```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

在这种情况下，"pods"是命名空间资源，而"log"是pods的子资源。可以使用`resource`实现更加细粒度的控制。

```yaml
# 拥有这个角色的用户或者用户组，可以使用kubectl get pods 和 kubectl get pods [podname] log
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

**示例：创建一个用户只能管理指定名称空间**

①、创建json配置

```json
{
  # 用户名
	"CN": "devuser",
	"hosts": [],
	"key": {
		"algo": "rsa",
		"size": 2048
	},
	"names": [{
		"C": "CN",
		"ST": "BeiJing",
		"L": "BeiJing",
    # 用户组名
		"O": "k8s",
		"OU": "System"
	}]
}
```

②、下载证书生成工具

```shell
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

chmod 755 /usr/local/bin/cfssl*


#生成证书
cd /etc/kubernetes/pki
cfssl gencert -ca=ca.crt -ca-key=ca.key -profile=kubernetes [jsonPath] | cfssljson -bare devuser
```

③、设置各种参数

```shell
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server="https://192.168.22.200:16443" \
  --kubeconfig=[userName].kubeconfig

# 设置客户端认证参数
kubectl config set-credentials [userName] \
  --client-certificate=/etc/kubernetes/pki/[userName].pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/pki/[userName]-key.pem \
  --kubeconfig=[userName].kubeconfig
  
# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes\
  --user=[userName]\
  --namespace=[namespace] \
  --kubeconfig=[userName].kubeconfig
  
# 创建命名空间
kubectl create namespace [namespace]

# 新建用户
useradd devuser
passwd devuser
mkdir /home/devuser/.kube
mv ./devuser.kubeconfig /home/devuser/.kube/config
chown -R devuser:devuser /home/devuser/.kube

kubectl create rolebinding devuser-admin-binding --clusterrole=admin --user=[userName] --namespace=[namespace]

# 切换到devuser设置默认上下文
cd ~/.kube
kubectl config use-context kubernetes --kubeconfig=config
```

### 准入控制

准入控制是API Server的插件集合，通过添加不同的插件，实现额外的准入控制规则。甚至于API Server的一些主要的功能都需要通过Admission Controllers实现，比如ServiceAccount。

官方文档上有针对不同版本的准入控制器推荐列表[每个版本不一样]

列举插件功能

- NamespaceLifecycle：防止在不存在的 namespace上创建对象，防止删除系统预置namespace,删除namespace时，连带删除它的所有资源对象
- LimitRanger：确保请求的资源不会超过资源所在Namespace的LimitRange的限制
- ServiceAccount：实现了自动化添加ServiceAccount
- ResourceQuota：确保请求的资源不会超过资源的ResourceQuota限制

## <span id="helm">十一、Helm</span>

### 概念

在没使用helm之前，向kubernetes部署应用，我们要依次部署deployment、svc等步骤较繁琐。况且随着很多项目微服务化，复杂的应用在容器中部署以及管理显得较为复杂。helm通过打包的方式类似包管理工具，支持发布的版本管理和控制,很大程度上简化Kubernetes应用的部署和管理。

**Chart**

chart是创建一个应用的信息集合，包括各种Kubernetes对象的配置模板、参数定义、依赖关系、文档说明等。chart是应用部署的自包含逻辑单元。可以将chart想象成yum中的软件安装包

**Release**

release是chart的运行实例，代表了一个正在运行的应用。当chart被安装到Kubernetes集群就生成一个release。chart能够多次安装到同一个集群，每次安装都是一个release

**helm和k8s交互原理**

<img src="/img/k8s/helm-架构.png" style="zoom:87%;" />

### 部署

①、[下载helm](https://github.com/helm/helm/releases)

②、解压&copy

```shell
tar -zxvf helm-[version]-linux-amd64.tar.gz
mv linux-amd64/helm /usr/bin/
```

③、添加仓库源

```shell
# 添加存储库
helm repo add azure http://mirror.azure.cn/kubernetes/charts
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
# 更新仓库
helm repo update

# 查看配置的存储库
helm repo list
helm search repo azure

# 删除存储库
helm repo remove azure
```

④、命令补全

```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(helm completion bash)
```

### 常用命令

| 命令                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| helm list                                                    | 查看当前安装的charts                                         |
| helm pull [repo/charName] [--version xxx]                    | 将Chart包下载到本地，缺省下载的是最新的Chart版本，并且是tgz包，可指定版本 |
| helm inspect chart [repo/charName]                           | 查看package详细信息                                          |
| helm install [charName]                                      | 安装本地的helm                                               |
| helm uninstall [charName] --keep-history                     | 卸载，--keep-history是可选参数，如果有代表保存，以后可以恢复否则是删除 |
| helm delete [charName]                                       | 删除，不可恢复                                               |
| helm repo add [repoName] [url]<br/>helm repo add --username [admin]--password [password] [repoName] [url] | 增加repo                                                     |
| helm repo update                                             | 更新repo仓库资源                                             |
| helm repo list                                               | 查看加到本地的仓库列表                                       |
| helm repo remove []                                          | 删除存储库                                                   |
| helm search hub [helmName]<br/>helm search repo [helmName]   | search hub，从hub中查找Chart，这些Chart来自于注册到Helm Hub中的各个仓库<br/>search repo，从所有加到本地的仓库中查找应用，这些仓库加到本地时Chart清单文件已被存放到Kubernetes中，所以查找应用时无需联网 |

### Helm编写示例

①、创建文件夹

```shell
mkdir hello-helm
cd hello-helm
```

②、创建`Chart.yaml`文件

```yaml
# 必须存在Chart.yaml文件，且文件内必须要有name、version定义
# vim Chart.yaml
name: hello-helm
version: 1.0.0
```

③、创建`values.yaml`文件

```yaml
# vim values.yaml
image:
  repo: nginx
  tag: 1.19
```

④、创建模板文件，用于生成Kubernetes资源清单(manifests)

```yaml
# mkdir templates
# cd templates
# vim Deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
     app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: {{ .Values.image.repo }}:{{ .Values.image.tag }}
        imagePullPolicy: IfNotPresent

# vim Service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: nginx-app
  ports:
  - name: http
    # 自己想暴露出的端口
    port: 30001
    # 对应容器暴露的端口
    targetPort: 80
```

⑤、安装运行

```shell
# 不指定helm名称随机生成
helm install . --generate-name
# 指定helm的名称
helm install hello-helm
```

### 部署应用

#### Prometheus【监控集群的工具】

①、下载Chart

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm pull prometheus-community/kube-prometheus-stack
```

②、解压下载好的文件

③、修改`Values.yaml`文件，里面有`grafana`的密码，和SVC配置

## 十二、高可用kubernetes 集群搭建

**kubernetes master节点组件在集群中的状态**

- `ApiService`：官方没有给出解决方案，但是它只是一个Restful风格的web服务，很好处理
- `etcd`：会自动和所有Master节点组成ETCD集群，不用关心
- `Controller-Manager`：只会启动一个，不用关心
- `Scheduler`：只会启动一个，不用关心
- `kubelet`：只在当前节点工作，不用关心
- `Proxy`：只在当前节点工作，不用关心

**第一步：环境准备，<a herf="#kubeadm-1">与kubernetes搭建第一步相同</a>**

**第二步：所有master节点部署keepalived**

①、安装相关包

```shell
yum install -y conntrack-tools libseccomp libtool-ltdl
yum install -y keepalived
```

②、配置master节点

```shell
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    # 其它从机为BACKUP
    state MASTER 
    interface enp0s3 
    virtual_router_id 51
    priority 250
    advert_int 1
    # 验证形象
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        192.168.22.200
    }
    track_script {
        check_haproxy
    }

}
EOF
```

③、启动和检查

```shell
# 启动keepalived
systemctl start keepalived.service
# 设置开机启动
systemctl enable keepalived.service
# 查看启动状态
systemctl status keepalived.service
```

④、检查网卡信息

```shell
ip a s enp0s3
```

**第三步：所有master节点部署haproxy**

①、安装

```shell
yum install -y haproxy
```

②、配置

```shell
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      centos165   192.168.22.165:6443 check
    server      centos168   192.168.22.168:6443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF
```

③、启动和检查

```shell
# 设置开机启动
systemctl enable haproxy
# 开启haproxy
systemctl start haproxy
# 查看启动状态
systemctl status haproxy

# 检查端口
netstat -lntup|grep haproxy
```

**第四步：所有节点安装Docker/kubeadm/kubelet，<a herf="#kubeadm-2">与kubernetes搭建第二步相同</a>**

**第五步：配置master节点的kubeadm【在具有VIP的master节点操作】**

①、创建kubeadm配置文件

```shell
mkdir /usr/local/kubernetes/manifests -p
cd /usr/local/kubernetes/manifests/

vim kubeadm-config.yaml
apiServer:
  certSANs:
    - centos165
    - centos168
    - k8sMaster
    - 192.168.22.165
    - 192.168.22.168
    - 192.168.22.200
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
# haproxy的代理端口
controlPlaneEndpoint: "192.168.22.200:16443"
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
  local:    
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.20.1
networking: 
  dnsDomain: cluster.local  
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.1.0.0/16
scheduler: {}
```

②、执行

```shell
kubeadm init --config kubeadm-config.yaml

# To start using your cluster, you need to run the following as a regular user:
#
#  mkdir -p $HOME/.kube
#  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#  sudo chown $(id -u):$(id -g) $HOME/.kube/config
# you can now join any number of control-plane nodes by copying certificate authorities
# and service account keys on each node and then running the following as root:
#
#  kubeadm join 192.168.22.200:16443 --token zhaun4.3esnwfuw7zr9xf3n \
#    --discovery-token-ca-cert-hash sha256:bc6177c8e33375671eb17e47c8336795d9eea071bb965d7004cac8e246ee730e \
#    --control-plane 

# Then you can join any number of worker nodes by running the following on each as root:

#kubeadm join 192.168.22.200:16443 --token zhaun4.3esnwfuw7zr9xf3n \
#    --discovery-token-ca-cert-hash #sha256:bc6177c8e33375671eb17e47c8336795d9eea071bb965d7004cac8e246ee730e 
```

③、按照提示配置环境变量，使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 检查
kubectl get nodes
```

**第六步：安装集群网络，<a herf="#kubeadm-4">与kubernetes搭建的第四步：安装Pod 网络插件（CNI）相同</a>**

**第七步：其它master加入**

```shell
# 把处理好的VIP的master节点密钥和相关文件复制到其它master节点
ssh root@192.168.22.168 mkdir -p /etc/kubernetes/pki/etcd
scp /etc/kubernetes/admin.conf root@192.168.22.168:/etc/kubernetes
scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@192.168.22.168:/etc/kubernetes/pki
scp /etc/kubernetes/pki/etcd/ca.* root@192.168.22.168:/etc/kubernetes/pki/etcd

# 执行第五步要求执行的命令
kubeadm join 192.168.22.200:16443 --token zhaun4.3esnwfuw7zr9xf3n \
--discovery-token-ca-cert-hash sha256:bc6177c8e33375671eb17e47c8336795d9eea071bb965d7004cac8e246ee730e \
--control-plane 
```

**第八步：node节点加入**

```shell
# 执行第五步要求执行的命令
kubeadm join 192.168.22.200:16443 --token zhaun4.3esnwfuw7zr9xf3n \
--discovery-token-ca-cert-hash sha256:bc6177c8e33375671eb17e47c8336795d9eea071bb965d7004cac8e246ee730e 

# 查看集群状态
kubectl get nodes
kubectl get cs

# 如果发现scheduler 和 controller-manager健康检查失败可开放他俩的非安全端口检查
# vim /etc/kubernetes/manifests/kube-scheduler.yaml
# vim /etc/kubernetes/manifests/kube-controller-manager.yaml
# 将port=0去掉
# systemctl restart kubelet
```

> 当一共有 3 个 master 节点时，至少需要 2 个节点可用，api-server 才可以正常工作
>



