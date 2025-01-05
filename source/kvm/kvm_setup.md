# kvm虚拟化

##  1. 在Ubuntu 上安装 KVM

### 1.1 KVM Hypervisor 要求

验证 CPU 虚拟化扩展是否可用

```shell
~$ sudo egrep -c '(vmx|svm)' /proc/cpuinfo
32
```

确定 KVM 内核模块是否已加载

```shell
~$ lsmod | grep kvm
kvm_intel             380928  0
kvm                  1015808  1 kvm_intel
```

是否可以使用KVM加速

```shell
~$ sudo apt install cpu-checker
~$ sudo kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

### 1.2 安装KVM软件包

```shell
~$ sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils -y
```

### 1.3 授权用户

只有libvirt和kvm用户组的成员才能运行虚拟机。如果您希望特定用户运行虚拟机，请将他们添加到这些用户组。

```shell
sudo adduser dengyouf libvirt
sudo adduser dengyouf kvm
```

### 1.4 验证安装

```shell
~$ sudo virsh  list --all
 Id   Name   State
--------------------

~$ sudo systemctl enable --now libvirtd
~$ sudo systemctl status libvirtd
```

## 2. KVM 设置桥接网络

### 2.1 在服务器上创建一个新接口 br0

```shell
~$ cd /etc/netplan/
~$ sudo cp 01-network-manager-all.yaml  01-network-manager-all.yaml.bak
echo "
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp3s0:
      dhcp4: false
      dhcp6: false
  bridges:
    br0:
      interfaces: [enp3s0]
      addresses: [192.168.1.228/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
      parameters:
        stp: false
        forward-delay: 0
      dhcp4: no
      dhcp6: no
"|sudo tee 01-network-manager-all.yaml

~$ sudo netplan  apply
```

## 3. 安装虚拟机

### 3.1 使用 virt-manager 安装虚拟机


```shell
~$ sudo apt install virt-manager
```
