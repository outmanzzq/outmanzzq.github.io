---
layout: post
title: K8S(13)监控实战-部署 Prometheus
categories: k8s,docker
description: 由于 Docker 容器的特殊性，传统的 Zabbix 无法对 K8S 集群内的 Docker 状态进行监控，所以需要使用 Prometheus 来进行监控
keywords: k8s,docker,devops
---

> Prometheus 是由 SoundCloud 开源监控告警解决方案，从 2012 年开始编写代码，2015 年 GitHub 上开源，2016 年 Prometheus 成为继 Kubernetes 之后，成为 CNCF （Cloud Native Computing Foundation）中的第二个项目成员，也是第二个正式毕业的项目。


## 1 Prometheus 前言相关

由于 Docker 容器的特殊性，传统的 Zabbix 无法对 K8S 集群内的 Docker 状态进行监控，所以需要使用 Prometheus 来进行监控

- Prometheus 官网：<https://prometheus.io/>

### 1.1 Prometheus 的特点

- 多维度数据模型,使用时间序列数据库 TSDB 而不使用 Mysql。
- 灵活的查询语言 PromQL。
- 不依赖分布式存储，单个服务器节点是自主的。
- 主要基于 HTTP 的 Pull 方式主动采集时序数据
- 也可通过 Pushgateway 获取主动推送到网关的数据。
- 通过服务发现或者静态配置来发现目标服务对象。
- 支持多种多样的图表和界面展示，比如 Grafana 等。

### 1.2 基本原理

#### 1.2.1 原理说明

Prometheus 的基本原理是通过各种 Exporter 提供的 HTTP 协议接口

周期性抓取被监控组件的状态，任意组件只要提供对应的 HTTP 接口就可以接入监控。

不需要任何 SDK 或者其他的集成过程,非常适合做虚拟化环境监控系统，比如 VM、Docker、Kubernetes 等。

互联网公司常用的组件大部分都有 Exporter 可以直接使用，如 Nginx、MySQL、Linux 系统信息等。

#### 1.2.2 架构图:

![img](/images/20220425-k8s13-prometheus-01.png)

#### 1.2.3 三大套件

- Server 主要负责数据采集和存储，提供 PromQL 查询语言的支持。
- Alertmanager 警告管理器，用来进行报警。
- Push Gateway 支持临时性 Job 主动推送指标的中间网关。

#### 1.2.4 架构服务过程

1. Prometheus Daemon 负责定时去目标上抓取 Metrics (指标)数据

   每个抓取目标需要暴露一个http服务的接口给它定时抓取。

   支持通过配置文件、文本文件、Zookeeper、DNS SRV Lookup等方式指定抓取目标。

2. PushGateway 用于 Client 主动推送 Metrics 到 PushGateway

   而 Prometheus 只是定时去 Gateway 上抓取数据。

   适合一次性、短生命周期的服务

3. Prometheus 在 TSDB 数据库存储抓取的所有数据

   通过一定规则进行清理和整理数据，并把得到的结果存储到新的时间序列中。

4. Prometheus 通过 PromQL 和其他 API 可视化地展示收集的数据。

   支持 Grafana、Promdash 等方式的图表数据可视化。

   Prometheus 还提供 HTTP API 的查询方式，自定义所需要的输出。

5. Alertmanager 是独立于 Prometheus 的一个报警组件

   支持 Prometheus 的查询语句，提供十分灵活的报警方式。

#### 1.2.5 常用的 Exporter

Prometheus 不同于 Zabbix，没有 Agent，使用的是针对不同服务的 Exporter

正常情况下，监控 K8S 集群及 Node，Pod，常用的 Exporter 有四个：

- **kube-state-metrics**

  收集 K8S 集群 master&etcd 等基本状态信息

- **node-exporter**

  收集 K8S 集群 Node 信息

- **cadvisor**

  收集 K8S 集群 Docker 容器内部使用资源信息

- **blackbox-exporte**

  收集 K8S 集群 Docker 容器服务是否存活

## 2 部署 4 个 Exporter

老套路，下载 Docker 镜像，准备资源配置清单，应用资源配置清单：

### 2.1 部署 kube-state-metrics

#### 2.1.1 准备 Docker 镜像

```sh
docker pull quay.io/coreos/kube-state-metrics:v1.5.0
docker tag quay.io/coreos/kube-state-metrics:v1.5.0 harbor.zq.com/public/kube-state-metrics:v1.5.0
docker push harbor.zq.com/public/kube-state-metrics:v1.5.0
```

准备目录

```sh
mkdir /data/k8s-yaml/kube-state-metrics
cd /data/k8s-yaml/kube-state-metrics
```

#### 2.1.2 准备 RBAC 资源清单

```YAML
cat >rbac.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs:
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
EOF
```

#### 2.1.3 准备 Deploy 资源清单

```yaml
cat >dp.yaml <<'EOF'
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
  labels:
    grafanak8sapp: "true"
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
spec:
  selector:
    matchLabels:
      grafanak8sapp: "true"
      app: kube-state-metrics
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        grafanak8sapp: "true"
        app: kube-state-metrics
    spec:
      containers:
      - name: kube-state-metrics
        image: harbor.zq.com/public/kube-state-metrics:v1.5.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http-metrics
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
      serviceAccountName: kube-state-metrics
EOF
```

#### 2.1.4 应用资源配置清单

任意 Node 节点执行

```sh
kubectl apply -f http://k8s-yaml.zq.com/kube-state-metrics/rbac.yaml
kubectl apply -f http://k8s-yaml.zq.com/kube-state-metrics/dp.yaml
```

**验证测试**

```sh
kubectl get pod -n kube-system -o wide|grep kube-state-metrics
~]# curl -I http://172.7.21.4:8080/healthz
ok
```

返回 OK 表示已经成功运行。

### 2.2 部署 Node-exporter

**由于node-exporter是监控 Node 的，需要每个节点启动一个，所以使用 DaemonSet 控制器**

#### 2.2.1 准备 Docker 镜像

```sh
docker pull prom/node-exporter:v0.15.0
docker tag  prom/node-exporter:v0.15.0 harbor.zq.com/public/node-exporter:v0.15.0
docker push harbor.zq.com/public/node-exporter:v0.15.0
```

准备目录

```sh
mkdir /data/k8s-yaml/node-exporter
cd /data/k8s-yaml/node-exporter
```

#### 2.2.2 准备 DaemonSet 资源清单

```YAML
cat >ds.yaml <<'EOF'
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    daemon: "node-exporter"
    grafanak8sapp: "true"
spec:
  selector:
    matchLabels:
      daemon: "node-exporter"
      grafanak8sapp: "true"
  template:
    metadata:
      name: node-exporter
      labels:
        daemon: "node-exporter"
        grafanak8sapp: "true"
    spec:
      volumes:
      - name: proc
        hostPath: 
          path: /proc
          type: ""
      - name: sys
        hostPath:
          path: /sys
          type: ""
      containers:
      - name: node-exporter
        image: harbor.zq.com/public/node-exporter:v0.15.0
        imagePullPolicy: IfNotPresent
        args:
        - --path.procfs=/host_proc
        - --path.sysfs=/host_sys
        ports:
        - name: node-exporter
          hostPort: 9100
          containerPort: 9100
          protocol: TCP
        volumeMounts:
        - name: sys
          readOnly: true
          mountPath: /host_sys
        - name: proc
          readOnly: true
          mountPath: /host_proc
      hostNetwork: true
EOF
```

> 主要用途就是将宿主机的 `/proc`,`sys` 目录挂载给容器,是容器能获取 Node 节点宿主机信息

#### 2.2.3 应用资源配置清单：

任意 Node 节点

```sh
kubectl apply -f http://k8s-yaml.zq.com/node-exporter/ds.yaml
kubectl get pod -n kube-system -o wide|grep node-exporter
```

### 2.3 部署 Cadvisor

#### 2.3.1 准备 Docker 镜像

```sh
docker pull google/cadvisor:v0.28.3
docker tag google/cadvisor:v0.28.3 harbor.zq.com/public/cadvisor:v0.28.3
docker push harbor.zq.com/public/cadvisor:v0.28.3
```

准备目录

```sh
mkdir /data/k8s-yaml/cadvisor
cd /data/k8s-yaml/cadvisor
```

#### 2.3.2 准备 DaemonSet 资源清单

Cadvisor 由于要获取每个 Node 上的 Pod 信息,因此也需要使用 Daemonset 方式运行

```YAML
cat >ds.yaml <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: kube-system
  labels:
    app: cadvisor
spec:
  selector:
    matchLabels:
      name: cadvisor
  template:
    metadata:
      labels:
        name: cadvisor
    spec:
      hostNetwork: true
#------Pod 的 Tolerations 与 Node 的 Taints 配合,做 Pod 指定调度----
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
#-------------------------------------
      containers:
      - name: cadvisor
        image: harbor.zq.com/public/cadvisor:v0.28.3
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: docker
          mountPath: /var/lib/docker
          readOnly: true
        ports:
          - name: http
            containerPort: 4194
            protocol: TCP
        readinessProbe:
          tcpSocket:
            port: 4194
          initialDelaySeconds: 5
          periodSeconds: 10
        args:
          - --housekeeping_interval=10s
          - --port=4194
      terminationGracePeriodSeconds: 30
      volumes:
      - name: rootfs
        hostPath:
          path: /
      - name: var-run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /data/docker
EOF
```

> 人为影响 K8S  调度策略的三种方法：
>
> - 污点、容忍度方法
>   - 污点：运算节点 Node 上的污点
>   - 容忍度：Pod 是否能够容忍污点
> - nodename：让 Pod 运行在指定的 node 上
> - nodeSelector：通过标签选择器，让 Pod 运行在指定的一类 Node 上
>
> Tolerations:  预期结果
>
> - 当遇到 key = xx  这个标签时，执行 effect 动作（不调度）
>
> 具体参考：<https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/>



#### 2.3.3 应用资源配置清单：

应用清单前,先在每个 Node 上做以下软连接,否则服务可能报错

```sh
mount -o remount,rw /sys/fs/cgroup/
ln -s /sys/fs/cgroup/cpu,cpuacct /sys/fs/cgroup/cpuacct,cpu
```

应用清单

```sh
kubectl apply -f http://k8s-yaml.zq.com/cadvisor/ds.yaml
```

检查：

```sh
kubectl -n kube-system get pod -o wide|grep cadvisor
```

### 2.4 部署 blackbox-exporter

#### 2.4.1 准备 Docker 镜像

```sh
docker pull prom/blackbox-exporter:v0.15.1
docker tag  prom/blackbox-exporter:v0.15.1  harbor.zq.com/public/blackbox-exporter:v0.15.1
docker push harbor.zq.com/public/blackbox-exporter:v0.15.1
```

准备目录

```sh
mkdir /data/k8s-yaml/blackbox-exporter
cd /data/k8s-yaml/blackbox-exporter
```

#### 2.4.2 准备 ConfigMap 资源清单

```yaml
cat >cm.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
  namespace: kube-system
data:
  blackbox.yml: |-
    modules:
      http_2xx:
        prober: http
        timeout: 2s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2"]
          valid_status_codes: [200,301,302]
          method: GET
          preferred_ip_protocol: "ip4"
      tcp_connect:
        prober: tcp
        timeout: 2s
EOF
```

#### 2.4.3 准备 Deployment 资源清单

```yaml
cat >dp.yaml <<'EOF'
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: blackbox-exporter
  namespace: kube-system
  labels:
    app: blackbox-exporter
  annotations:
    deployment.kubernetes.io/revision: 1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      volumes:
      - name: config
        configMap:
          name: blackbox-exporter
          defaultMode: 420
      containers:
      - name: blackbox-exporter
        image: harbor.zq.com/public/blackbox-exporter:v0.15.1
        imagePullPolicy: IfNotPresent
        args:
        - --config.file=/etc/blackbox_exporter/blackbox.yml
        - --log.level=info
        - --web.listen-address=:9115
        ports:
        - name: blackbox-port
          containerPort: 9115
          protocol: TCP
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 50Mi
        volumeMounts:
        - name: config
          mountPath: /etc/blackbox_exporter
        readinessProbe:
          tcpSocket:
            port: 9115
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
EOF
```

#### 2.4.4 准备 Service 资源清单

```yaml
cat >svc.yaml <<'EOF'
kind: Service
apiVersion: v1
metadata:
  name: blackbox-exporter
  namespace: kube-system
spec:
  selector:
    app: blackbox-exporter
  ports:
    - name: blackbox-port
      protocol: TCP
      port: 9115
EOF
```

#### 2.4.5 准备 Ingress 资源清单

```yaml
cat >ingress.yaml <<'EOF'
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: blackbox-exporter
  namespace: kube-system
spec:
  rules:
  - host: blackbox.zq.com
    http:
      paths:
      - path: /
        backend:
          serviceName: blackbox-exporter
          servicePort: blackbox-port
EOF
```

#### 2.4.6 添加域名解析

这里用到了一个域名，添加解析

```sh
vi /var/named/zq.com.zone
blackbox       A    10.4.7.10

systemctl restart named

# 测试
[root@hdss7-11 ~]# dig -t A blackbox.zq.com +short
10.4.7.10
```

#### 2.4.7 应用资源配置清单

```sh
kubectl apply -f http://k8s-yaml.zq.com/blackbox-exporter/cm.yaml
kubectl apply -f http://k8s-yaml.zq.com/blackbox-exporter/dp.yaml
kubectl apply -f http://k8s-yaml.zq.com/blackbox-exporter/svc.yaml
kubectl apply -f http://k8s-yaml.zq.com/blackbox-exporter/ingress.yaml
```

#### 2.4.8 访问域名测试

访问 <http://blackbox.zq.com>，显示如下界面,表示 Blackbox 已经运行成

![mark](/images/20220425-k8s13-prometheus-02.png)


> 参考链接：<https://www.cnblogs.com/noah-luo/p/13501841.html>
