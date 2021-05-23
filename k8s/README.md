# k8s集群搭建

```
k8s-master 192.168.66.10
k8s-node01 192.168.66.20
k8s-node02 192.168.66.21
```

## 操作系统版本

```
uname -r
4.18.0-193.el8.x86_64
cat /etc/os-release 
NAME="CentOS Linux"
VERSION="8 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="8"
PLATFORM_ID="platform:el8"
PRETTY_NAME="CentOS Linux 8 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:8"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-8"
CENTOS_MANTISBT_PROJECT_VERSION="8"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="8"
```

## 关闭图形界面

```
查看当前的默认目标，运行：
systemctl get-default

设置默认目标，运行：
systemctl set-default multi-user.target   #关闭图形界面
systemctl set-default graphical.target    #打开图形界面
```

## 环境准备

### 主机命名

```
k8s-master01：
hostnamectl set-hostname k8s-master01
vim /etc/hosts
192.168.66.10 k8s-master01
192.168.66.20 k8s-node01
192.168.66.21 k8s-node02

k8s-node01：
hostnamectl set-hostname k8s-node01
vim /etc/hosts
192.168.66.10 k8s-master01
192.168.66.20 k8s-node01
192.168.66.21 k8s-node02

k8s-node02：
hostnamectl set-hostname k8s-edge02
vim /etc/hosts
192.168.66.10 k8s-master01
192.168.66.20 k8s-node01
192.168.66.21 k8s-node02
```

### 关闭防火墙、swap、selinux

```
# 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld
# 关闭swap
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
# 关闭 selinux
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

###  安装依赖项 & 更新yum仓库 & 缓存信息

```
yum install -y chrony conntrack ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git

yum makecache

yum clean all

yum makecache
```

###  配置时间同步服务

```
vim /etc/chrony.conf
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#pool 2.centos.pool.ntp.org iburst
server time1.aliyun.com iburst     
server time2.aliyun.com iburst
server time3.aliyun.com iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
keyfile /etc/chrony.keys

# Get TAI-UTC offset and leap seconds from the system tz database.
leapsectz right/UTC

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
```

```
systemctl restart chronyd && systemctl enable chronyd && systemctl status chronyd 

# 设置系统时区为 中国/上海
timedatectl set-timezone Asia/Shanghai
date
# 将当前的 UTC 时间写入硬件时钟
timedatectl set-local-rtc 0
```

###  安装iptables & 配置内核参数

```
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save

cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
# net.ipv4.tcp_tw_recycle=0 # centos8 取消了tcp_tw_recycle，只保留了tcp_tw_reuse
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统OOM时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

sysctl --system
modprobe br_netfilter
```

### 配置ipvs

```
# 设置 rsyslogd 和 systemd journald
# 持久化保存日志的目录
mkdir /var/log/journal
mkdir /etc/systemd/journald.conf.d

# 设置配置参数
cat <<EOF >/etc/systemd/journald.conf.d/99-prophet.conf
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SynIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=10G
# 单日志文件最大 200M 
SystemMaxFileSize=200M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到
syslog ForwardToSyslog=no
EOF

systemctl restart systemd-journald && systemctl status systemd-journald

# 配置ipvs
vim /etc/rc.local 
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
```

##  安装docker

```
# set up the docker yum repository.
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# install docker-ce-cli docker-ce
yum list docker-ce-cli --showduplicates

yum install -y docker-ce-cli-19.03.15 docker-ce-19.03.15

systemctl enable docker.service && systemctl start docker
systemctl status docker


cat <<EOF >/etc/docker/daemon.json 
{
  "registry-mirrors": [
      "https://60nwgi45.mirror.aliyuncs.com",
      "https://reg-mirror.qiniu.com",
      "https://docker.mirrors.ustc.edu.cn",
      "https://dockerhub.azk8s.cn",
      "https://hub-mirror.c.163.com",
      "https://registry.docker-cn.com"
  ],
  "insecure-registries": [],
  "debug": true,
  "experimental": true,
  "features": {
      "buildkit": true
  }
}
EOF

systemctl restart docker
systemctl status docker
```

```
安装docker报错requires containerd.io >= 1.2.2-3, but none of the providers can be installed时：

dnf install  https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.13-3.1.el7.x86_64.rpm
```

## 安装 k8s

```
cat <<EOF >/etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

```
yum install -y kubelet-1.19.10-0 kubeadm-1.19.10-0 kubectl-1.19.10-0

vim /etc/rc.local 
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local

modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
```

### k8s-master

```
kubeadm init --apiserver-advertise-address=192.168.66.10 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v1.19.8 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=Swap

mkdir ~/.kube
cd ~/.kube
cp -i /etc/kubernetes/admin.conf ./config

mkdir ~/k8s/flannel -p
cd ~/k8s/flannel
curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f ~/k8s/flannel/kube-flannel.yml

# 验证节点
kubectl get node
kubectl get all --all-namespaces -o wide
```

```
当https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml连接失败时，需要手动上外网下载
```

### k8s-node

```
kubeadm join 192.168.66.10:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxxx
```

```
# 验证节点
kubectl get node
[root@ke-cloud1 ~]# kubectl get node
NAME        STATUS   ROLES    AGE   VERSION
k8s-master01   Ready      master       6h48m   v1.19.10
k8s-node01     Ready      <none>       6h37m   v1.19.10
k8s-node02     NotReady   <none>       6h37m   v1.19.10
```

```
k8s-node重启后，如果还是处于not-ready状态，需要手动启动kubelet:
#systemctl stop kubelets
systemctl start kubelet
systemctl status kubelet
```



[参考一](https://gitee.com/Hu-Lyndon/kubeedge/blob/master/01.%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/01.k8s-docker-centos8.md)

[k8s集群功能验证](https://mp.weixin.qq.com/s/zGTwwFzqOqvOHO-V2JzmWQ)