# kubeedge集群搭建

## 云端
[参考文章](https://blog.csdn.net/ma726518972/article/details/115025774)
```angular2
#注意顺序
kubectl create -f 01-namespace.yaml
kubectl create -f 02-serviceaccount.yaml
kubectl create -f 03.clusterrole.yaml
kubectl create -f 04.clusterrolebinding.yaml
kubectl create -f 09-devices_v1alpha2_device.yaml
kubectl create -f 10-devices_v1alpha2_devicemodel.yaml
kubectl create -f 11-cluster_objectsync_v1alpha1.yaml
kubectl create -f 12-objectsync_v1alpha1.yaml
#注意顺序
kubectl create -f 05-configmap.yaml
kubectl create -f 07-deployment.yaml
kubectl create -f 08-service.yaml
```
```
#常用命令
kubectl get nodes
kubectl get pod -n kubeedge
kubectl delete pod podid -n kubeedge
kubectl describe pod podid -n kubeedge
kubectl logs -f pod podid -n kubeedge
#master节点去污点
kubectl taint nodes master1 node-role.kubernetes.io/master:NoSchedule-
```
## 边缘端 - 树莓派版
###树莓派系统烧录
```
百度网盘网址：链接：https://pan.baidu.com/s/1TkqfFEskuL0Aht0mj1_WYA 提取码：u8uq

解压后有三个文件：格式化SD卡、Win32DiskImager-0.9.5-binary、2019-09-26-raspbian-buster-full

解压2019-09-26-raspbian-buster-full.zip文件，
将树莓派的SD卡插入到笔记本电脑中，通过格式化SD卡.zip里的软件，
格式化SD卡，然后通过Win32DiskImager-0.9.5-binary.zip压缩包里软件选择格式化好的SD卡和解压好树莓派操作系统2019-09-26-raspbian-buster-full，
点击write后，等待一段时间后，系统就可以烧录完成。

在烧录好系统盘里的boot目录下，创建一个名字为SSH的文件（不带任何后缀）。

将烧录好的SD卡插到树莓派的SD卡槽中，供电，开机。
```

###树莓派准备操作
```
#root用户的设置
sudo passwd root
su root

#修改主机名
nano /etc/hostname #保存并退出为：Ctrl + X > Y（YES）> 回车
修改成你需要的主机名如raspberrypi-edge-01,这个主机名会作为边缘节点的名称
echo "127.0.0.1 raspberrypi-edge-01" > /etc/hosts
```

###树莓派镜像源配置
```
sudo nano /etc/apt/sources.list
注释原有的内容，新添加以下两条内容
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main contrib non-free rpi
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main contrib non-free rpi
保存并退出为：Ctrl + X > Y（YES）> 回车

sudo nano /etc/apt/sources.list.d/raspi.list
注释原有的内容，新添加以下两条内容
deb http://mirrors.ustc.edu.cn/archive.raspberrypi.org/debian/ buster main ui
保存并退出为：Ctrl + X > Y（YES）> 回车

更新软件源
sudo apt-get update
更新软件
sudo apt-get upgrade
```

###树莓派安装Docker
```
sudo apt-get install \
apt-transport-https \
ca-certificates \
curl \
gnupg2 \
software-properties-common

curl -fsSL https://download.docker.com/linux/raspbian/gpg | sudo apt-key add -

echo "deb [arch=armhf] https://download.docker.com/linux/raspbian \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update

apt-cache madison docker-ce 获取docker可用版本,选择合适版本安装
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
例如：sudo apt-get install docker-ce=5:19.03.11~3-0~raspbian-buster docker-ce-cli=5:19.03.11~3-0~raspbian-buster containerd.io
docker version
```

###树莓派安装mosquitto
```
docker run -d --name mosquito -p 1883:1883 eclipse-mosquitto:2.0.7
```

###树莓派安装edgecore
```
#获取edgecore
方法一：编译源码
git clone https://github.com/kubeedge/kubeedge.git $GOPATH/src/github.com/kubeedge/kubeedge
cd $GOPATH/src/github.com/kubeedge/kubeedge
git tag
git checkout v1.5.0
make all WHAT=edgecore 比较慢

将编译后的二进制文件复制出来
cd $GOPATH/src/github.com/kubeedge/kubeedge/edge 
mkdir -p /root/cmd
cp edgecore /root/cmd/

方法二：下载编译好的edgecore
wget https://github.com/kubeedge/kubeedge/releases/download/v1.5.0/kubeedge-v1.5.0-linux-amd64.tar.gz
tar -zxvf kubeedge-v1.5.0-linux-amd64.tar.gz

cd /root/kubeedge-v1.5.0-linux-arm/edge
cp edgecore /root/cmd/
```
```
#创建edgecore配置文件
mkdir -p /etc/kubeedge/config/
/root/cmd/edgecore --minconfig > /etc/kubeedge/config/edgecore.yaml
vim /etc/kubeedge/config/edgecore.yaml

看nodeIP是否正确
将edgehub.websocket.server:更改成云端公网IP地址+:10000
看modules.edged.podSandboxImage:是否是kubeedge/pause-arm:3.1
填写token（或将云端生成的证书，拖拽到树莓派的 /etc/kubeedge 下）
token获取方式
k8s master执行命令
kubectl get secret -n kubeedge tokensecret -oyaml -o jsonpath={".data.tokendata"} | base64 -d
```
```
启动edgecore
cd /root/cmd
./edgecore 或
nohup ./edgecore > edgecore.log 2>&1 &
```
```
查看edgenode
kubectl get nodes
```

r&s
【解决】 Streaming server stopped unexpectedly: listen tcp: lookup localhost on 114.114.114.114:53: no such host
/etc/hosts配置
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

解决error: system validation failed - Following Cgroup subsystem not mounted: [memory]
#修改/boot/cmdline.txt
sudo vim /boot/cmdline.txt
cgroup_enable=memory cgroup_memory=1
添加在同一行的最后面,接着内容后空格后添加, 注意:不要换行添加
#重启机器配置生效
reboot