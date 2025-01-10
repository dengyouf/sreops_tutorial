# Kubernetes Network

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
~/ingress-nginx#ingress-nginx# diff  values.yaml.bak  values.yaml
80c80
<   dnsPolicy: ClusterFirst
---
>   dnsPolicy:  ClusterFirstWithHostNet # 修改dns策略
224c224
<   kind: Deployment
---
>   kind: DaemonSet # 修改为 DaemonSet
495c495
<     type: LoadBalancer
---
>     type: NodePort # 修改为 Service类型
553c553
<       http: ""
---
>       http: "80" # 映射宿主机80端口， 需配合 --service-node-port-range 参数使用
555c555
<       https: ""
---
>       https: "443" # 映射宿主机443端口，需配合 --service-node-port-range 参数使用
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

## 2. Ingress 基础使用

### 2.1 公开后端服务

- 创建第一个 Deployment 

```shell
echo "---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp-deploy-v10
  labels:
    app: demoapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp
      version: v1.0
  strategy: {}
  template:
    metadata:
      labels:
        app: demoapp
        version: v1.0
    spec:
      containers:
      - name: demoapp-v10
        image: ikubernetes/demoapp:v1.0"|tee demoapp-deploy-v10.yaml|kubectl apply -f -
```

- 创建Service-`demoapp-v10`
```shell
echo "---
apiVersion: v1
kind : Service
metadata:
  name: demoapp-v10-svc
  labels:
    app: demoapp
spec:
  type: ClusterIP
  ports:
  - name: http-80
    port: 80
    targetPort: 80
  selector:
    app: demoapp
    version: v1.0"|tee demoapp-v10-svc.yaml|kubectl apply -f -
```

- 创建ingress规则

```shell
~# kubectl  get svc
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
demoapp-v10-svc   ClusterIP   10.108.203.213   <none>        80/TCP    4m13s

echo "---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demoapp-v10-ingress
spec:
  rules:
  - host: demoapp-v10.linux.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demoapp-v10-svc
            port:
              number: 80
  ingressClassName: nginx"|tee demoapp-v10-ingress.yaml|kubectl apply -f -
```

- 验证
- 
```shell
~  curl -H 'Host: demoapp-v10.linux.io' 172.16.192.31                                                                                            ✔  674  10:25:55
iKubernetes demoapp v1.0 !! ClientIP: 10.244.3.43, ServerName: demoapp-deploy-v10-659484dcb5-2gp67, ServerIP: 10.244.3.48!
```

### 2.2 URL重写

- 创建第二个Deployment 
```shell
echo "---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp-deploy-v11
  labels:
    app: demoapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp
      version: v1.1
  strategy: {}
  template:
    metadata:
      labels:
        app: demoapp
        version: v1.1
    spec:
      containers:
      - name: demoapp-v11
        image: ikubernetes/demoapp:v1.1"|tee demoapp-deploy-v11.yaml|kubectl apply -f -

echo "---
apiVersion: v1
kind : Service
metadata:
  name: demoapp-v11-svc
  labels:
    app: demoapp
spec:
  type: ClusterIP
  ports:
  - name: http-80
    port: 80
    targetPort: 80
  selector:
    app: demoapp
    version: v1.1"|tee demoapp-v11-svc.yaml|kubectl apply -f -
```

```shell
# kubectl  get svc
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
demoapp-v10-svc   ClusterIP   10.108.203.213   <none>        80/TCP    20m
demoapp-v11-svc   ClusterIP   10.107.12.202    <none>        80/TCP    7s
```

- url重写规则


```shell
echo "---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demoapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: demoapp.linux.io
    http:
      paths:
      - path: /v10(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: demoapp-v10-svc
            port:
              number: 80
      - path: /v11(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: demoapp-v11-svc
            port:
              number: 80
  ingressClassName: nginx" |tee demoapp-ingress.yaml|kubectl apply -f -
```

- 验证

```shell
~#  curl -H 'Host: demoapp.linux.io' 172.16.192.31/v10
iKubernetes demoapp v1.0 !! ClientIP: 10.244.3.43, ServerName: demoapp-deploy-v10-659484dcb5-2gp67, ServerIP: 10.244.3.48!

~#  curl -H 'Host: demoapp.linux.io' 172.16.192.31/v11
iKubernetes demoapp v1.1 !! ClientIP: 10.244.2.84, ServerName: demoapp-deploy-v11-55db5ccd8d-jvsfm, ServerIP: 10.244.2.88!
```

### 2.3 HTTPS

- 创建证书

```shell
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=http-svc.linux.io/O=http-svc.linux.io"
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

- 部署服务

```shell
echo "apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: http-svc
  template:
    metadata:
      labels:
        app: http-svc
    spec:
      containers:
      - name: http-svc
        image: dengyouf/echoserver:2.3
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP

---

apiVersion: v1
kind: Service
metadata:
  name: http-svc
  labels:
    app: http-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: http-svc
" |tee http-svc.yaml|kubectl apply -f -
```

- 创建ingress规则

```shell
echo "
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: http-svc-tls-ingress
spec:
  tls:
    - hosts:
      - http-svc.linux.io
      # This assumes tls-secret exists and the SSL
      # certificate contains a CN for foo.bar.com
      secretName: tls-secret
  ingressClassName: nginx
  rules:
    - host: http-svc.linux.io
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            # This assumes http-svc exists and routes to healthy endpoints
            service:
              name: http-svc
              port:
                number: 80
"|tee http-svc-tls-ingress.yaml|kubectl apply -f -
```

- 验证

```shell
~]# curl -H 'Host:  http-svc.linux.io' -k  https://172.16.192.31
```

