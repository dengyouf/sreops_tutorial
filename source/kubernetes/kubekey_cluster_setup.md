# KubeKey

## 1. Kubekey 安装 kubernetes集群

基于 Kubeadm 部署Kubernetes集群。操作系统为 Ubuntu 20.04 LTS，用到的各相关程序版本如下：

- 系统要求

| 最低配置 | 内核版本 |
| --- | ---|
| CPU：2 核，内存：4 G，硬盘：40 G | 4.15+ |

| 主机 IP         | 	主机名	         |角色|
|---------------|---------------|---|
| 192.168.1.111 | 	k8s-master01 |	master, etcd,worker|
| 192.168.0.121 | 	k8s-worker01 |	worker|
| 192.168.0.122 | 	k8s-worker02 |	worker|
| 192.168.0.123 | 	k8s-worker03 |	worker|

> /var/lib/docker 路径主要用于存储容器数据，在使用和操作过程中数据量会逐渐增加。因此，在生产环境中，建议为 /var/lib/docker 单独挂载一个硬盘

### 1.1 安装依赖项

在 Ubuntu 操作系统上，执行以下命令为服务器安装依赖项

```shell
~# sudo apt install socat conntrack ebtables ipset -y
```

### 1.2 安装kubekey

```shell
~# export KKZONE=cn
~# curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.13 sh -
~# chmod +x kk
```

>运行 `./kk version --show-supported-k8s`，查看能使用 KubeKey 安装的所有受支持的 Kubernetes 版本 

### 1.3 配置规划集群

```shell
~# ./kk create config --with-kubernetes  v1.26.5  [--with-kubesphere v3.2.1] 
~# ls config-sample.yaml
config-sample.yaml
```
```shell
~# cat config-sample.yaml

apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: k8s-master01, address: 192.168.1.111, internalAddress: 192.168.1.111, user: root, password: "root"}
  - {name: k8s-worker01, address: 192.168.1.121, internalAddress: 192.168.1.121, user: root, password: "root"}
  - {name: k8s-worker02, address: 192.168.1.122, internalAddress: 192.168.1.122, user: root, password: "root"}
  - {name: k8s-worker03, address: 192.168.1.123, internalAddress: 192.168.1.123, user: root, password: "root"}
  roleGroups:
    etcd:
    - k8s-master01
    control-plane:
    - k8s-master01
    worker:
    - k8s-worker01
    - k8s-worker02
    - k8s-worker03
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers
    internalLoadbalancer: haproxy

    domain: lb.linux.io
    address: ""
    port: 6443
  kubernetes:
    version: v1.26.5
    clusterName: cluster.local
    autoRenewCerts: true
    containerManager: containerd
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []
```

### 1.4 创建集群

```shell
~# ./kk create cluster -f config-sample.yaml
```

## 2. 验证集群

```shell
~# kubectl get pod -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-86d646c976-mq4wh   1/1     Running   0          4m55s
kube-system   calico-node-7462s                          1/1     Running   0          4m55s
kube-system   calico-node-8hvvd                          1/1     Running   0          4m55s
kube-system   calico-node-vm58l                          1/1     Running   0          4m55s
kube-system   calico-node-xzx5t                          1/1     Running   0          4m55s
kube-system   coredns-d9d84b5bf-h8bz8                    1/1     Running   0          5m12s
kube-system   coredns-d9d84b5bf-zmmc7                    1/1     Running   0          5m12s
kube-system   haproxy-k8s-worker01                       1/1     Running   0          4m53s
kube-system   haproxy-k8s-worker02                       1/1     Running   0          4m54s
kube-system   haproxy-k8s-worker03                       1/1     Running   0          4m53s
kube-system   kube-apiserver-k8s-master01                1/1     Running   0          5m24s
kube-system   kube-controller-manager-k8s-master01       1/1     Running   0          5m24s
kube-system   kube-proxy-8mb6z                           1/1     Running   0          4m56s
kube-system   kube-proxy-b64zz                           1/1     Running   0          4m56s
kube-system   kube-proxy-hvcfm                           1/1     Running   0          4m56s
kube-system   kube-proxy-w4f98                           1/1     Running   0          4m56s
kube-system   kube-scheduler-k8s-master01                1/1     Running   0          5m24s
kube-system   nodelocaldns-8h5tf                         1/1     Running   0          4m57s
kube-system   nodelocaldns-dgcdh                         1/1     Running   0          4m57s
kube-system   nodelocaldns-nv5qc                         1/1     Running   0          5m12s
kube-system   nodelocaldns-thxpd                         1/1     Running   0          4m57s
```
```shell
~# kubectl  get nodes -o wide
NAME           STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master01   Ready    control-plane   5m54s   v1.26.5   192.168.1.111   <none>        Ubuntu 20.04.6 LTS   5.4.0-204-generic   containerd://1.6.4
k8s-worker01   Ready    worker          5m23s   v1.26.5   192.168.1.121   <none>        Ubuntu 20.04.6 LTS   5.4.0-204-generic   containerd://1.6.4
k8s-worker02   Ready    worker          5m23s   v1.26.5   192.168.1.122   <none>        Ubuntu 20.04.6 LTS   5.4.0-204-generic   containerd://1.6.4
k8s-worker03   Ready    worker          5m23s   v1.26.5   192.168.1.123   <none>        Ubuntu 20.04.6 LTS   5.4.0-204-generic   containerd://1.6.4
```