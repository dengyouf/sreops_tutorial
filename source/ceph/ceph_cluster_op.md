# Ceph 集群运维

## 1. 集群管理

```shell
# 查看集群的状态
ceph -s 
ceph -w 
ceph health

# 列出节点上所有ceph实例
sudo systemctl  list-units|grep "ceph"|awk '{print $1}' |grep service
ceph-crash.service
ceph-mgr@ceph-node01.service
ceph-mon@ceph-node01.service
ceph-osd@0.service
ceph-osd@1.service
ceph-radosgw@rgw.ceph-node01.service
```

## 2. 存储池管理

```shell
# 列出pool
ceph osd lspools
rados lspools

# 创建 pool
ceph osd pool create mypool 16
ceph osd pool create rep_pool 16 16 replicated 

# 删除存储池 在配置文件 [mon] 段添加内容 mon allow pool delete = true
ceph osd pool rm mypool mypool  --yes-i-really-really-mean-it

# 查看存储池使用情况
ceph df

# 获取存储池副本数
ceph osd dump|grep "replicated size"

# 设置副本数
 ceph osd pool set mypool size 2
```
 