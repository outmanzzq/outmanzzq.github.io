---
layout: post
title: 通过 kind 快速构建 Kubernetes （K8S）测试集群
categories: kind,k8s,devops
description: 通过 kind 工具快速搭建 K8S 测试集群，并将集群内部微服务暴露到宿主机，以方便外部访问。
keywords: kind,k8s,devops
---
> 通过 kind 工具快速搭建 K8S 测试集群，并将集群内部微服务暴露到宿主机，以方便外部访问。


# 一、kind 简介

- 官网：https://kind.sigs.k8s.io
- 文档：https://kind.sigs.k8s.io/docs/user/quick-start/

## kind 是什么？

k8s 集群的组成比较复杂，如果纯手工部署的话易出错且时间成本高。而本文介绍的 kind 工具，能够快速的建立起可用的 k8s 集群，降低初学者的学习门槛。

Kind是 Kubernetes In Docker 的缩写，顾名思义，看起来是把 k8s 放到 docker 的意思。

kind 创建 k8s 集群的基本原理就是：提前准备好 k8s 节点的镜像，通过 docker 启动容器，来模拟 k8s 的节点，从而组成完整的 k8s 集群。

> 需要注意: kind 创建的集群仅可用于开发、学习、测试等，不能用于生产环境。

## kind 有什么特点？

- 创建、启动 k8s 集群非常快速，资源消耗相比传统虚拟机消耗极低。
- 支持快速创建多节点的 k8s 集群，包括高可用模式。
- kind 支持 Linux, macOS and Windows
- 它是 CNCF 认证的 k8s 集群安装方式之一

## 如何安装 kind？

### 二进制安装

- Linux

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

-  MacOS

```sh
# for Intel Macs
[ $(uname -m) = x86_64 ]&& curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-darwin-amd64
# for M1 / ARM Macs
[ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-darwin-arm64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

-  Windows(PowerShell)
 
```sh
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.17.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
```

### 通过软件包管理器安装

- MacOS

```sh
brew install kind
```

- Windows (Chocolatey) (https://chocolatey.org/packages/kind)

```sh
choco install kind
```

> 具体安装参考官网：https://kind.sigs.k8s.io/docs/user/quick-start/#installation

## 如何通过 kind 新建 k8s 集群？

kubectl 是与 k8s 交互的客户端命令工具，因此需要先安装此工具。

```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

然后通过一行命令就能够快速的创建 k8s 集群：

```
root@e5pc-vm-01:~# kind create cluster --name myk8s-01
Creating cluster "myk8s-01" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-myk8s-01"
You can now use your cluster with:

kubectl cluster-info --context kind-myk8s-01

Have a nice day! 👋

```

至此已得到一个可用的k8s集群了：

```sh
root@e5pc-vm-01:~# kubectl get nodes
NAME                     STATUS   ROLES                  AGE   VERSION
myk8s-01-control-plane   Ready    control-plane,master   66s   v1.21.1
```

## kind 创建 k8s 集群的内幕

先查看多出来的 docker 镜像和容器：

```sh
root@e5pc-vm-01:~# docker images
REPOSITORY     TAG       IMAGE ID       CREATED        SIZE
ubuntu         latest    2b4cba85892a   2 days ago     72.8MB
kindest/node   <none>    32b8b755dee8   9 months ago   1.12GB
 
root@e5pc-vm-01:~# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                       NAMES
b4bde05b0190   kindest/node:v1.21.1   "/usr/local/bin/entr…"   12 minutes ago   Up 12 minutes   127.0.0.1:42267->6443/tcp   myk8s-01-control-plane

```

创建过程大概是：

1. 先获取镜像 kindest/node:v1.21.1
2. 然后启动容器 myk8s-01-control-plane

启动的容器就是这个 k8s 集群的 master 节点，显然此集群只有 master 节点。

同时，kind create cluster 命令还是将此新建的 k8s 集群的连接信息写入当前用户 (root)的 kubectl 配置文件中，从而让 kubectl 能够与集群交互。

配置文件内容如下：

```sh
root@e5pc-vm-01:~# cat  .kube/config 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: xxxx
    server: https://127.0.0.1:42267
  name: kind-myk8s-01
contexts:
- context:
    cluster: kind-myk8s-01
    user: kind-myk8s-01
  name: kind-myk8s-01
current-context: kind-myk8s-01
kind: Config
preferences: {}
users:
- name: kind-myk8s-01
  user:
    client-certificate-data: xxxx
    client-key-data: xxxx
```

## kind创建k8s集群的架构图为：

![ ](/images/20221118-kind-portmappings-01.png)

> 说明：
> 1. 实际 kind 镜像内部采用更轻量化的 Containerd 替代 Docker 做为容器引擎
> 2. 对应的 docker 相关命令通过 crictl 替代

## kind的高级用法 （通过 YAML 配置文件创建集群）

运行方式：

`kind create cluster --config  xxx.yaml`

1. 创建单节点 K8S 集群，实现如下功能：

- 挂载宿主机目录到 kind 容器中  # 经测试，挂载到 kind 容器的宿主机目录，会与宿主机同步实时变更！
- 自定义 Pod/Service 网段
  - podSubnet: "10.244.0.0/16"
  - serviceSubnet: "10.96.0.0/12"
- 将 kube-proxy 策略由默认 iptalbes 切换为 ipvs
- 将 kind 容器 30000 端口映射到宿主机 80 端口

```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: app-1-cluster
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  kubeProxyMode: "ipvs"
nodes:
- role: control-plane
  extraMounts:
  - hostPath: ./
    containerPath: /home/kind
    readOnly: true
  extraPortMappings:
  - containerPort: 30000
    hostPort: 80
```

```sh
 kind create cluster --config kind-demo.yaml                                                                                           
Creating cluster "app-1-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.25.2) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-app-1-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-app-1-cluster

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂

☺  kubectl get node                                                                                                                      
NAME                          STATUS     ROLES           AGE   VERSION
app-1-cluster-control-plane   NotReady   control-plane   21s   v1.25.2                                                                                                      

☺  docker ps                                                                                                                             
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                                              NAMES
028cfc53d113   kindest/node:v1.25.2   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:59297->6443/tcp, 0.0.0.0:80->30000/tcp   app-1-cluster-control-plane

☺  docker exec -it app-1-cluster-control-plane ls /home/kind                                                                             
Ingress  kind-3n-1m2w.yaml        kind-demo.yaml     kind-ha-config-aliyun.yaml
app      kind-config-aliyun.yaml  kind-ha-3m3w.yaml  kind-mapping-ports.yaml
```

2. 创建一主多从 K8S 集群

```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

```sh
☺  cat kind-3n-1m2w.yaml                                                                                                                 
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker

☺  time kind create cluster --config kind-3n-1m2w.yaml                                                                                   
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.2) 🖼 
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
kind create cluster --config kind-3n-1m2w.yaml  5.98s user 2.90s system 15% cpu 55.544 total

☺  kind get clusters                                                                                                                     
kind

☺  kubectl get node                                                                                                                      
NAME                 STATUS   ROLES           AGE    VERSION
kind-control-plane   Ready    control-plane   117s   v1.25.2
kind-worker          Ready    <none>          94s    v1.25.2
kind-worker2         Ready    <none>          94s    v1.25.2
```

3. 创建高可用 K8S 集群

- 自定义 Service 网段
  - Service：10.0.0.0/16
- 采用阿里云自定义容器镜像

```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta1
  kind: ClusterConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
  imageRepository: registry.aliyuncs.com/google_containers
  nodeRegistration:
    kubeletExtraArgs:
      pod-infra-container-image: registry.aliyuncs.com/google_containers/pause:3.1
- |
  apiVersion: kubeadm.k8s.io/v1beta1
  kind: InitConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
  imageRepository: registry.aliyuncs.com/google_containers
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```

```sh
☺  kind create cluster --config kind-ha-config-aliyun.yaml                                                                               
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.2) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦 📦 📦  
 ✓ Configuring the external load balancer ⚖️ 
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining more control-plane nodes 🎮 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊

☺  kind get clusters                                                                                                                     
kind

☺  kubectl get node                                                                                                        
NAME                  STATUS   ROLES           AGE     VERSION
kind-control-plane    Ready    control-plane   2m17s   v1.25.2
kind-control-plane2   Ready    control-plane   113s    v1.25.2
kind-control-plane3   Ready    control-plane   41s     v1.25.2
kind-worker           Ready    <none>          28s     v1.25.2
kind-worker2          Ready    <none>          28s     v1.25.2
kind-worker3          Ready    <none>          28s     v1.25.2
```

# 二、实战

## 实战 01

目标：

- 将 kind 容器（集群）以下端口暴露到宿主机（ kind -> 宿主机）
  - 80 -> 80
  - 443 -> 443
  - 30000 -> 30000
- 部署 Nginx 服务

### 1. 构建集群

运行下列命令创建新的k8s cluster

```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: tsk8s
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
```

```sh
 kind create cluster --config kind-http.yaml                                                                                           
Creating cluster "tsk8s" ...
 ✓ Ensuring node image (kindest/node:v1.25.2) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-tsk8s"
You can now use your cluster with:

kubectl cluster-info --context kind-tsk8s

Thanks for using kind! 😊

~/Documents/server/kind
☺  kind get clusters                                                                                                                     
tsk8s

☺  kubectl get node                                                                                                                      
NAME                  STATUS   ROLES           AGE   VERSION
tsk8s-control-plane   Ready    control-plane   63s   v1.25.2
```

这时可以看到80、443、30000端口已经暴露出来了

```sh
☺  docker ps                                                                                                                             
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                                                                                           NAMES
bbefb4a4c82a   kindest/node:v1.25.2   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:30000->30000/tcp, 127.0.0.1:59559->6443/tcp   tsk8s-control-plane
```

> extraPortMappings # 把 k8s 容器（相当于K8s所在的服务器）端口暴露出来，这里暴露了 80、443、30000
> node-labels       # 只允许 Ingress controller 运行在有 "ingress-ready=true" 标签的 node 上

### 2. 部署 Deployment、Service

#### 部署 Deployment

新建文件 my-dep.yaml，添加以下内容

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-dep
spec:
  replicas: 1  # number of replicas of frontEnd application
  selector:
    matchLabels:
      app: httpd-app
  template:
    metadata:
      labels: # Must match 'Service' and 'Deployment' labels
        app: httpd-app
    spec:
      containers:
      - name: httpd
        image: httpd # docker image of frontend application
        ports:
        - containerPort: 80
```

> 说明：
> 
> - Deployment 的名称为 “httpd-dep”
> - 管理的 Pods 需要带有 “app: httpd-app” 标签
> - Pod 模板中指定运行的镜像为 Docker 公共仓库中的 httpd


运行以下命令创建 Deployment

```sh
☺  kubectl apply -f app/my-dep.yaml                                                                                                      
deployment.apps/httpd-dep configured

~/Documents/server/kind
☺  kubectl get po -o wide                                                                                                                
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE                  NOMINATED NODE   READINESS GATES
httpd-dep-786dbbf544-mhstm   1/1     Running   0          30s   10.244.0.8   tsk8s-control-plane   <none>           <none>
```

> 说明：可以看到 Pod 被分配了一个 K8s 内部 IP，这个 IP 不能从 K8s 外部访问到，而且这个 IP 在 Pod 重建后会变化

### 部署 Service

```YAML
kind: Service
apiVersion: v1
metadata:
  name: httpd-svc
spec:
  selector:
      app: httpd-app
  ports:
  - port: 80
```

运行以下命令创建 Service

```sh
☺  kubectl apply -f app/my-svc.yaml                                                                                                      
service/httpd-svc created

☹  kubectl get svc/httpd-svc -o wide                                                                                                     
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
httpd-svc   ClusterIP   10.96.172.174   <none>        80/TCP    82s   app=httpd-app
```

> 说明：Service 的 IP 是不会变化的（除非重建 Service ）

查看 Service 详细信息, 发现 Type 为 Cluster (只能集群内部访问)

```sh
☺  kubectl describe svc/httpd-svc                                                                                                        
Name:              httpd-svc
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=httpd-app
**Type:              ClusterIP**
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.172.174
IPs:               10.96.172.174
Port:              <unset>  80/TCP
TargetPort:        80/TCP
**Endpoints:         10.244.0.8:80**
Session Affinity:  None
Events:            <none>
```

> 说明：
> 
> 可以看到这里Service关联的Endpoint的IP和端口就是上面Pod的IP和端口。每次Pod重建后这里的Endpoint就会刷新为新的IP。
> 目前这个Service只有内部IP和端口，所以这个Service还只能用在K8s内部暴露Pods。
> 
> 下面我们修改Service配置，使用K8s外部也可以访问到这个Service

### 更改Serivce（nodePort）

修改my-svc.yaml，添加以下内容

```yaml
kind: Service
apiVersion: v1
metadata:
  name: httpd-svc
spec:
  selector:
      app: httpd-app
  type: NodePort #1
  ports:
  - port: 80
    nodePort: 30000 #2
```

> 说明：
> #1 Service type 默认为 ClusterIP，即只有内部 IP。改为 NodePort 后，Service 会把 K8s 内部的端口映射到集群所在的服务器上
> #2 指定 Service 映射到服务器上的端口为 30000

再次运行 kubectl apply 命令,查看 Service 信息，可以看到端口中多了一个 30000

```sh
☺  kubectl get svc/httpd-svc                                                                                                             
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
httpd-svc   NodePort   10.96.172.174   <none>        80:30000/TCP   9m56s

☺  kubectl describe svc/httpd-svc                                                                                                        
Name:                     httpd-svc
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=httpd-app
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.172.174
IPs:                      10.96.172.174
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30000/TCP
Endpoints:                10.244.0.8:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

30000 这个端口被映射到了 K8s 集群所在的服务器上（即 K8s 运行的容器），而我们在创建 kind K8s 时把容器的 30000 端口又映射到本地，

所以现在我们可以在本地用浏览器访问 30000 端口。

在本地可以访问到 30000 端口上的 httpd 应用

```sh
☺  curl -I localhost:30000                                                                                                               
HTTP/1.1 200 OK
Date: Fri, 18 Nov 2022 11:00:35 GMT
Server: Apache/2.4.54 (Unix)
Last-Modified: Mon, 11 Jun 2007 18:53:14 GMT
ETag: "2d-432a5e4a73a80"
Accept-Ranges: bytes
Content-Length: 45
Content-Type: text/html
```

## 实战 02 创建 MySQL 数据库服务

先删除前面的服务，因为需要使用 30000 端口。

```sh
kubectl delete -f my-dep.yaml
kubectl delete -f myl-svc.yaml
```

创建 MySQL 服务步骤：

- 1、创建一个新的 namespace
- 2、创建对应服务
- 3、验证是否成功

### 1、创建一个新的 namespace

创建 namespace ,命令行直接创建

`kubectl create namespace test`

### 2、在该 namespace 下创建（ ENV 中设置了 MySQL 的 root 用户的密码为 mysql ）

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.6
        imagePullPolicy: IfNotPresent
        args:
          - "--ignore-db-dir=lost+found"
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "1234"
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: test
  labels:
    name: mysql-svc
spec:
  selector:
    app: mysql
  type: NodePort
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
    name: http
    nodePort: 30000
  selector:
    app: mysql
```

创建对应 service

```sh
kubectl create -f mysql-svc.yaml --record

```

### 3、验证是否成功

- 主机：service 对应的 pod 所在的 node 的 ip
- 端口：上面 service 中的 nodeport 端口号 30000
- 密码：deployment 文件 env 中设置的 root 用户的密码）

```sh
☹  mysql -h 192.168.11.106 -P 30000 -uroot -p1234 -e "show databases"                                                                    
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

# 三、kind 常用命令

```sh

Usage:
  kind [command]

Available Commands:
  build       构建节点镜像
  completion  Output shell completion code for the specified shell (bash, zsh or fish)
  create      创建一个或多个集群
  delete      删除一个或多个集群
  export      导出集群配置信息或日志等
  get         获取集群相关信息（kubeconfig 等）
  help        Help about any command
  load        加载镜像到 Node 节点中
  version     打印版本号
```

> 参考链接：<https://www.cnblogs.com/yakniu/p/16456367.html>
