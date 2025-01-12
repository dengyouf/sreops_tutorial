## 1 GitLab

### 1.1 配置清华源
```shell
~# curl -fsSL https://packages.gitlab.com/gpg.key | gpg --dearmor > /usr/share/keyrings/gitlab_gitlab-ce-archive-keyring.gpg
~# cat >>  /etc/apt/sources.list.d/gitlab-ce.list << EOF
deb [signed-by=/usr/share/keyrings/gitlab_gitlab-ce-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu focal main
EOF
~# apt update
```

### 1.2 安装gitLab

```shell
~# apt install gitlab-ce
```

### 1.3 配置gitlab

```shell
echo  "192.168.240 gitlab.linux.io"  >> /etc/hosts
vim /etc/gitlab/gitlab.rb
external_url 'http://gitlab.linux.io'
```

### 1.4 重启服务

```shell
~# gitlab-ctl  reconfigure
~# gitlab-ctl  restart
```

### 1.5 获取初始密码

```shell
~# cat /etc/gitlab/initial_root_password
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: ZjcKz389K26BKk9+N/+f8omY1YEPyi2noszubzh9Asc=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

## 3 Harbor


### 3.1 Prerequisites

> 官网： `https://goharbor.io/docs/2.12.0/install-config/installation-prereqs/`

![img.png](img.png)
![img_1.png](img_1.png)

### 3.2 Harbor Installation Prerequisites

```shell
~# hostnamectl  set-hostname reg.linux.io
~# ufw disable
~# echo '192.168.1.250 reg.linux.io' >> /etc/hosts
```
### 3.3 Install docker

> 阿里云： `https://developer.aliyun.com/mirror/docker-ce?spm=a2c6h.13651102.0.0.3e221b11McV0m7`

```shell
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# step 2: 信任 Docker 的 GPG 公钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step 3: 写入软件源信息
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 
# Step 4: 安装Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
# Step 2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.1~ce-0~ubuntu-xenial)
sudo apt install docker-ce=5:26.1.3-1~ubuntu.20.04~focal
```

```shell
~# echo '
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://docker.rainbond.cc",
        "https://docker.1panel.live"
    ]

}
' |sudo tee /etc/docker/daemon.json

~# systemctl  daemon-reload  && systemctl restart docker
```
```shell
# docker compose version
Docker Compose version v2.32.1
```
### 3.4 Install docker-compose (可选)

> 官网： `https://docs.docker.com/compose/install/#install-compose`

如果使用 docker compose v2 (docker-compose-plugin) 的话，不需要单独安装docker-compose 

```shell
~# sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
~# chmod +x /usr/local/bin/docker-compose
~# docker-compose version
docker-compose version 1.23.2, build 1110ad01
docker-py version: 3.6.0
CPython version: 3.6.7
OpenSSL version: OpenSSL 1.1.0f  25 May 2017
```

### 3.5 Download and Unpack the Installer

```shell
~# wget https://github.com/goharbor/harbor/releases/download/v2.12.1/harbor-offline-installer-v2.12.1.tgz
~# tar -xf harbor-offline-installer-v2.12.1.tgz  -C /opt/

```

### 3.6 Configure HTTPS Access to Harbor

```
~]# mkdir /opt/harbor/ssl
~]# cd /opt/harbor/ssl
```

```
1. Generate a CA certificate private key.
ssl]# openssl genrsa -out ca.key 4096
```

```
2. Generate the CA certificate.
ssl]# openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=reg.linux.io" \
 -key ca.key \
 -out ca.crt
```

```
3. Generate a private key for harbor server.
ssl]# openssl genrsa -out reg.linux.io.key 4096
```

```
4. Generate a certificate signing request (CSR).
ssl]# openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=reg.linux.io" \
    -key reg.linux.io.key \
    -out reg.linux.io.csr
```

```
5. Generate an x509 v3 extension file.
ssl]# cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=reg.linux.io
DNS.2=reg.linux.io
DNS.3=192.168.1.250
EOF
```

```
6. Use the v3.ext file to generate a certificate for your Harbor host.
ssl]# openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in reg.linux.io.csr \
    -out reg.linux.io.crt
```

### 3.7 Configure the Harbor YML File

```shell
/opt/harbor/ssl# cp /opt/harbor/harbor.yml.tmpl /opt/harbor/harbor.yml
/opt/harbor/ssl# vim /opt/harbor/harbor.yml
hostname: reg.linux.io
# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /opt/harbor/ssl/reg.linux.io.crt
  private_key: /opt/harbor/ssl/reg.linux.io.key
```

### 3.8 Run the `prepare` script to enable HTTPS

```shell
~# cd /opt/harbor/ && ./prepare
~# cd /opt/harbor/ && ./install.sh
```

- stop harbor： `docker compose down -v `
- start harbor : `docker compose up -d `

```shell
vim /lib/systemd/system/harbor.service
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor
 
[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml up
ExecReload=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml restart
ExecStop=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml down
 
[Install]
WantedBy=multi-user.target
```

### 3.9 Provide the Certificates to Harbor and Docker

```shell
/opt/harbor# cd ssl/
/opt/harbor/ssl# openssl x509 -inform PEM -in reg.linux.io.crt -out reg.linux.io.cert

cp reg.linux.io.cert  /etc/docker/certs.d/reg.linux.io/
cp reg.linux.io.key  /etc/docker/certs.d/reg.linux.io/
cp ca.crt  /etc/docker/certs.d/reg.linux.io/
```

```shell
~#  docker login reg.linux.io -u admin -p Harbor12345
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credential-stores

Login Succeeded
```

### 3.10 Upload image to Harbor

```shell
docker pull ikubernetes/myapp:v1
docker tag ikubernetes/myapp:v1  reg.linux.io/library/myapp:v1
docker push reg.linux.io/library/myapp:v1
```

## 4 Docker-Register

### 4.1 启动register

```shell
~# docker run -d --name docker-register  -p 5000:5000 -v /data/registry:/var/lib/registry registry:2
```
### 4.2 push/pull 镜像报错解决

```shell
~# docker tag nginx:1.20 192.168.1.250:5000/nginx:1.20
~# docker push 192.168.1.250:5000/nginx:1.20
The push refers to repository [192.168.1.250:5000/nginx]
Get "https://192.168.1.250:5000/v2/": http: server gave HTTP response to HTTPS client

# 解决
cat /etc/docker/daemon.json

{
    ...
    "insecure-registries":["192.168.1.250:5000"]
}
~# docker push 192.168.1.250:5000/nginx:1.20
~# curl http://192.168.1.250:5000/v2/_catalog
{"repositories":["nginx"]}
```

## 5 Containerd 对接 Harbor

> 前提： k8s节点hosts文件需要能解析Harbor域名：192.168.1.250 reg.linux.io

```shell
~# kubectl  edit cm/coredns -n kube-system
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        hosts {
           192.168.1.250 reg.linux.io
           fallthrough
        }
        prometheus :9153
        ...
    }
    
~# kubectl delete pod -n kube-system -l k8s-app=kube-dns
```

### 5.1 创建hosts.toml文件或者证书文件存储的目录

首先我们需要创建hosts.toml文件或者证书文件存储的目录，这个创建的目录名称必须是Harbor的域名(如果不是则报x509)，然后将证书文件或者hosts.toml文件放入该目录下才会生效。

```shell
~# mkdir -pv /etc/containerd/certs.d/reg.linux.io
```

### 5.2 修改config.toml配置

找到 `[plugins."io.containerd.grpc.v1.cri".registry]`下的config_path，然后指定证书存储目录,改完需重启containerd。

根据 containerd 配置规则，当使用 config_path 指定了证书和私有仓库配置文件目录时，不允许再使用 mirrors 配置，解决办法有两种：
- 删除 mirrors 配置，这里使用此红配置方式
- 移除 config_path 并仅使用 mirrors
```shell
[plugins]
    ...
    [plugins."io.containerd.grpc.v1.cri".registry]
        config_path = "/etc/containerd/certs.d"
#      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
#        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
#          endpoint = ["https://registry-1.docker.io"]
```
![img_2.png](img_2.png)


### 5.3 配置认证方式

#### 5.3.1 忽略证书的方式

- 忽略证书，就是我们只需要在/etc/containerd/certs.d/reg.linux.io/目录下面创建文件hosts.toml即可，不需要Harbor认证的自签名证书,无需重启containerd

```shell
cat >> /etc/containerd/certs.d/reg.linux.io/hosts.toml << EOF
server = "https://reg.linux.io"

[host."https://reg.linux.io"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = false
  ca = ["ca.crt"]
EOF
```

#### 5.3.2 启用证书的方式

需要把自签名Harbor域名的CA证书上传到/etc/containerd/certs.d/reg.linx.io/目录下

```shell
~# ls /etc/containerd/certs.d/reg.linux.io
ca.crt  hosts.toml
```

在该目录下创建hosts.toml文件并指定CA证书

```shell
~# cat /etc/containerd/certs.d/reg.linux.io/hosts.toml 
server = "https://reg.linux.io"
[host."https://reg.linux.io"]
  capabilities = ["pull", "resolve","push"]
  skip_verify = false
  ca = ["ca.crt"]
```

## 6 Kubernetes对接Harbor

### 6.1 通过Secret使用镜像

```shell
~# kubectl create secret docker-registry harbor-secret \
--docker-server=reg.linux.io \
--docker-username=admin \
--docker-password=Harbor12345 \
-n default
```

```shell
~# cat myapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy: {}
  template:
    metadata:
      labels:
        app: myapp
    spec:
      imagePullSecrets:
      - name: harbor-secret
      containers:
      - image: reg.linux.io/privatelib/myapp:v1
        name: myapp
        resources: {}
~# cat myapp.yaml |kubectl  apply -f -
```

### 6.1 通过ServviceAccount使用镜像

```shell
# 创建 sa
~# cat serviceaccount-harbor.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: harbor-sa
  namespace: default
root@k8s-master01:~# kubectl  apply -f serviceaccount-harbor.yaml
# patch 使用补丁修改、更新资源的字段
~# kubectl patch serviceaccount harbor-sa -n default  -p '{"imagePullSecrets": [{"name": "harbor-secret"}]}'
```

```shell
~# cat myapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy: {}
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccount: harbor-sa
      containers:
      - image: reg.linux.io/privatelib/myapp:v1
        name: myapp
        resources: {}
~# kubectl  apply -f myapp.yaml
```