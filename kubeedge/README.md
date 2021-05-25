# kubeedge集群搭建

```
k8s-master（kubeedge-cloud） 192.168.66.10
kube-edge01 192.168.66.30
kube-edge02 192.168.66.31
```

## 边缘端

### 关闭防火墙及 selinux （边缘端）

```
#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
#禁用 selinux
setenforce 0
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
```

### 关闭 swap （边缘端）

```
sed -i 's/.*swap.*/#&/' /etc/fstab
echo vm.swappiness=0 >> /etc/sysctl.conf
```

### 设置主机名 （边缘端）

```
k8s-master01:
vim /etc/hosts
192.168.66.10 k8s-master01
192.168.66.20 k8s-node01
192.168.66.21 k8s-node02
192.168.66.30 kube-edge01
192.168.66.31 kube-edge02

kube-edge01:
hostnamectl set-hostname kube-edge01
vim /etc/hosts
192.168.66.10 k8s-master01
192.168.66.30 kube-edge01
192.168.66.31 kube-edge02

kube-edge02:
hostnamectl set-hostname kube-edge02
vim /etc/hosts
192.168.66.10 k8s-master01
192.168.66.30 kube-edge01
192.168.66.31 kube-edge02
```

### 安装docker （边缘端）

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
reboot
```

## 云端

```
云端主要是负责编译 kubeEdge 的相关组件与运行 cloudcore，所以需要准备 golang 环境，以及需要去官方 github 上拉取源码，进行编译
```

### 配置 golang 环境

```
wget https://golang.google.cn/dl/go1.14.4.linux-amd64.tar.gz
tar -zxvf go1.14.4.linux-amd64.tar.gz -C /usr/local
```

```
vim /etc/profile
文件末尾追加
# golang env
export GOROOT=/usr/local/go
export GOPATH=/data/gopath
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

source /etc/profile
mkdir -p /data/gopath && cd /data/gopath
mkdir -p src pkg bin
```

### 配置 cloudcore

```
#下载 kubeEdge 源码
git clone https://github.com/kubeedge/kubeedge $GOPATH/src/github.com/kubeedge/kubeedge

#编译 keadm
cd $GOPATH/src/github.com/kubeedge/kubeedge
# 查看远端分支
git branch
git branch -r
# 切换远端1.5分支到本地
git checkout -b release-1.5 origin/release-1.5
# 安装gcc编译器
yum install -y make gcc
# 编译安装keadm 其实只需要编译keadm就可以了
make all WHAT=keadm
# 编译安装cloudcore
make all WHAT=cloudcore
# 编译安装edgecore
make all WHAT=edgecore
# 编译安装admission
make all WHAT=admission
# 编译安装csidriver
CGO_ENABLED=0 go build -v -o /data/gopath/src/github.com/kubeedge/kubeedge/_output/local/bin/csidriver github.com/kubeedge/kubeedge/cloud/cmd/csidriver

#将编译好的二进制文件 copy 到/usr/local/bin 中
cp _output/local/bin/* /usr/local/bin/
#创建 cloudcore 节点
keadm init --advertise-address="192.168.66.10" --kubeedge-version=1.15.0 

```

## Edge 端配置

```
k8s-master01（云端）:
#Edge 端也通过 keadm 进行配置，可以将 cloud 端编译生成的二进制文件 scp 到 Edge 端
scp -r _output/local/bin/* root@192.168.66.30:/usr/local/bin/
scp -r _output/local/bin/* root@192.168.66.31:/usr/local/bin/
keadm gettoken
```

```
kube-edge01、kube-edge02：
keadm join --kubeedge-version=1.5.0 --cloudcore-ipport=192.168.66.10:10000 --token=fdc16c8aead0d9e94604e01773dae30954c41ce288bc3436eb2c76c9f507a636.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjE4NTcyOTZ9.77zNeEj6UOX8mDWqDT3QOwPD4Zt-CLCDozT2iqmF1F0
```

### 问题解决1：

```
云端日志查看：

需要先添加系统服务
cp $GOPATH/src/github.com/kubeedge/kubeedge/build/tools/cloudcore.service /etc/systemd/system/cloudcore.service 
cp $GOPATH/src/github.com/kubeedge/kubeedge/_output/local/bin/cloudcore /etc/kubeedge/cloudcore  
systemctl enable cloudcore 
systemctl start cloudcore

#查看cloudcore状态
systemctl status cloudcore

#查看最近100条日志
journalctl -u cloudcore -n 100
```

```
边缘端日志查看：
#查看edgecore状态
c
#查看最近100条日志
journalctl -u edgecore -n 100
```

### 问题解决2：

```
一般来说如果不能科学上网的话，就会提示连接失败。

比如/etc/kubeedge/ 路径下，xx文件下载不了

命令行提示什么下载不下来，手动去下载就行，并且上传到指定位置就行，当然需要一台能够科学上网的机器
```

### 问题解决3：

```
F1104 18:03:19.701956 15614 keadm.go:27] failed to run command(cd /etc/kubeedge/ && sudo -E wget -k --no-check-certificate https://raw.githubusercontent.com/kubeedge/kubeedge/release-1.4/build/tools/edgecore.service), err:exit status 4

cd /etc/kubeedge
vim edgecore.service

[Unit]
Description=edgecore.service

[Service]
Type=simple
ExecStart=/usr/local/bin/edgecore #注意路径 如果采用官方提供的路径，就需要将相应的edgecore复制到/etc/kubeedge/目录下
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 云端主要目录与cloudcore状态：

![5](../docs/imgs/5.png)

![6](../docs/imgs/6.png)

### 边缘端主要目录与edgecore状态：

![7](../docs/imgs/7.png)

![8](../docs/imgs/8.png)

### 集群状态：

![9](../docs/imgs/9.png)

[参考与kubeedge集群功能验证](https://mp.weixin.qq.com/s/4IjRxo8uQn7ZKyPvM35ZsA)

[参考二](https://blog.csdn.net/weixin_42142364/article/details/111084493)

[参考三](https://blog.csdn.net/MSSC_/article/details/114866906)