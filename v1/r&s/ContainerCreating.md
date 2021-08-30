
```
如果Pod的部署卡在`ContainerCreating`，基本是因为镜像无法正常下载的原因
```
#解决：手动下载镜像
```
#例如云端部署pod时出现这种情况

#边缘端执行如下操作
#下载镜像并打包
docker save -o kubeedge-pi-counter.tar kubeedge/kubeedge-pi-counter:v1.0.0
#load镜像包到本地
docker load -i kubeedge-pi-counter.tar

#云端重新部署即可
```

```
本地与服务器文件传输命令：
rz 上传文件到服务器
sz filename 从服务器下载文件到本地
```
