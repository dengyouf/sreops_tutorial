# Kubernetes Storage

## 1. NFS CSI driver for Kubernetes

- [在Kubernetes集群中安装 NFS Server](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/example/nfs-provisioner/README.md)

```shell
~# wget https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/nfs-provisioner/nfs-server.yaml
~# kubectl  label node k8s-worker01 nfs=yes
~# diff nfs-server.yaml.bak nfs-server.yaml
38c38
<         "kubernetes.io/os": linux
---
>               "nfs": "yes" # 修改nfs-service调度到这个节点
60c60
<             path: /nfs-vol  
---
>             path: /data/nfs-vol  # 配置nfs的存储路径

~# kubectl apply -f nfs-server.yaml

~# kubectl  get pod -l app=nfs-server
NAME                          READY   STATUS    RESTARTS   AGE
nfs-server-6564d66655-k58jj   1/1     Running   0          24m
```

- [在集群中安装 NFS CSI driver](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.8.0.md)

```shell
~# wget https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.8.0/deploy/install-driver.sh 
~# cat install-driver.sh | bash -s v4.8.0 --

~# kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
NAME                                  READY   STATUS    RESTARTS      AGE   IP              NODE           NOMINATED NODE   READINESS GATES
csi-nfs-controller-7694bcc6cc-hvtkb   4/4     Running   1 (10m ago)   10m   172.16.192.42   k8s-worker02   <none>           <none>
~# kubectl -n kube-system get pod -o wide  -l  app=csi-nfs-node
NAME                 READY   STATUS    RESTARTS      AGE     IP              NODE           NOMINATED NODE   READINESS GATES
csi-nfs-node-5m5xz   3/3     Running   0             10m     172.16.192.42   k8s-worker02   <none>           <none>
csi-nfs-node-jsv8x   3/3     Running   1 (37m ago)   46m     172.16.192.41   k8s-worker01   <none>           <none>
csi-nfs-node-k4xtl   3/3     Running   0             8m25s   172.16.192.31   k8s-master01   <none>           <none>
csi-nfs-node-ldxzv   3/3     Running   1 (35m ago)   46m     172.16.192.43   k8s-worker03   <none>           <none>

```

- 验证存储可用性

```shell
# 创建 Storage Class 存储类
~# wget https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/refs/heads/master/deploy/v4.8.0/storageclass.yaml
~# kubectl apply -f storageclass.yaml
~# kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
~# kubectl  get sc
NAME                PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-csi (default)   nfs.csi.k8s.io   Delete          Immediate           false                  11s

# 部署应用验证服务可用性
~# wget https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/refs/heads/release-4.0/deploy/example/nfs-provisioner/nginx-pod.yaml
~# kubectl apply -f nginx-pod.yaml
~# kubectl  get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nginx   Bound    pv-nginx   10Gi       RWO                           4m50s
~# kubectl  get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pv-nginx   10Gi       RWO            Delete           Bound    default/pvc-nginx                           4m53s
~# nginx-nfs-example
~# kubectl  get pod  nginx-nfs-example
NAME                READY   STATUS    RESTARTS   AGE
nginx-nfs-example   1/1     Running   0          5m25s
```