# 基于 Ingress 实现启程调度和负载均衡

## 1. Ingress 安装

### 1.1 Helm 安装

- 添加 charts repo 

```shell
~# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
~# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈Happy Helming!⎈
```

- 获取指定版本的charts

```shell
~/ingress-nginx# helm search repo ingress-nginx  -l |grep 4.12.0 
ingress-nginx/ingress-nginx     4.12.0          1.12.0           Ingress controller for Kubernetes using NGINX a...
~# helm pull ingress-nginx/ingress-nginx  --version  4.12.0 --untar
```

-  配置values.yaml

```shell
~/ingress-nginx# cp values.yaml  values.yaml.bak
~/ingress-nginx# vim values.yaml
# 修改hostNetwork为 true
  hostNetwork: true
# 修改 dnsPolicy 为 ClusterFirstWithHostNet
  dnsPolicy: ClusterFirstWithHostNet
# 修改kind类型为 DaemonSet
  kind: DaemonSet
# 修改 service 类型 nodePort
  service:
    ...
    type: NodePort
    nodePorts:
      # -- Node port allocated for the external HTTP listener. If left empty, the service controller allocates one from the configured node port range.
      http: "80"
      # -- Node port allocated for the external HTTPS listener. If left empty, the service controller allocates one from the configured node port range.
      https: "443"
      # -- Node port mapping for external TCP listeners. If left empty, the service controller allocates them from the configured node port range.
```

> kubernetes 具有 NodePort 可见性的服务保留的端口范围默认为30000-32767，此处要想使用80和443端口，需要修改api-server的配置参数如下：

```shell
vim /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 172.16.192.31:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.16.192.31
    ...
    - --service-node-port-range=1-65535
```

- 应用charts

```shell
~/ingress-nginx# helm upgrade --install ingress-nginx -f values.yaml . --namespace ingress-nginx --create-namespace
```

### 1.2 使用 YAML 清单部署

```shell
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml
# 定制 yaml
diff deploy.yaml.bak deploy.yaml
356a357
>     nodePort: 80 # 添加
361a363
>     nodePort: 443 # 添加
366c368
<   type: LoadBalancer
---
>   type: NodePort # 使用NodePort的方式
```


