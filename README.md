# k8s_v1.10.0_install

参考：https://www.kubernetes.org.cn/4041.html

## 1.准备
### 1.1	环境准备
OS：Linux CentOS 7 3.10.0-327.el7.x86_64<br/>
我是3个Linux虚拟机：<br/>
192.168.80.11  master<br/>
192.168.80.12  node1<br/>
192.168.80.13  node2<br/><br/>
master（IP：192.168.80.11）

|组件|版本|部署方式|安装目录或访问路口|
|:---|:---|:---|:---|
|etcd|3.3.2|二进制|/usr/local/kubernetes/bin/etcd|
|docker|18.03.0-ce|二进制|/usr/bin/docker|
|flannel|0.10.0|二进制|/usr/local/kubernetes/bin/flanneld|
|Kubernetes|1.10.0|二进制|/usr/local/kubernetes/bin<br/>（kube-apiserver、kube-controller-manager、kubectl、kube-proxy、kube-scheduler）|

node（IP：192.168.80.12/13）

|组件|版本|部署方式|安装目录或访问路口|
|:---|:---|:---|:---|
|etcd|3.3.2|二进制|/usr/local/kubernetes/bin/etcd|
|docker|18.03.0-ce|二进制|/usr/bin/docker|
|flannel|0.10.0|二进制|/usr/local/kubernetes/bin/flanneld
|Kubernetes|1.10.0|二进制|/usr/local/kubernetes/bin<br/>（Kubectl、kubelet、kube-proxy）|

### 1.2	系统配置
配置host<br/>
```shell
# cat /etc/hosts
192.168.80.11  master
192.168.80.12  node1
192.168.80.13  node2
```
ping一下能ping通才行。<br/>
关闭主机防火墙：<br/>
```shell
# systemctl stop firewalld
# systemctl disable firewalld
```
注：
生产部署可以开放指定端口，无需关防火墙，端口参考：https://kubernetes.io/docs/setup/independent/install-kubeadm/ 之 “Check required ports”章节。<br/>
禁用SELINUX<br/>
```shell
# setenforce 0
# vi /etc/selinux/config
```
修改SELINUX为disabled<br/>

### 1.3	下载安装介质
将介质下载到本地，上传到linux<br/>
我将介质上传到/software下，新建software文件夹:<br/>
```shell
mkdir -p /software
```

## 2.部署ETCD
etcd需要在3台主机上都安装。<br/>
### 2.1	修改install_etcd.sh脚本
注意：编辑install_etcd.sh脚本中ifconfig获取本机ip地址的命令，按照实际情况进行修改以正确过滤获得主机IP<br/>
获取主机名示例：<br/>
```shell
# ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eno16777736: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:92:c7:7a brd ff:ff:ff:ff:ff:ff
    inet 192.168.80.11/24 brd 192.168.80.255 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe92:c77a/64 scope link 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 52:54:00:f2:34:bb brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 500
    link/ether 52:54:00:f2:34:bb brd ff:ff:ff:ff:ff:ff
5: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 5a:79:0f:f2:f5:e5 brd ff:ff:ff:ff:ff:ff
    inet 172.18.85.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::5879:fff:fef2:f5e5/64 scope link 
       valid_lft forever preferred_lft forever
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:53:db:29:c7 brd ff:ff:ff:ff:ff:ff
    inet 172.18.85.1/24 brd 172.18.85.255 scope global docker0
       valid_lft forever preferred_lft forever

```
修改shell文件中如下2处：<br/>
 
 ### 2.2 执行install_etcd.sh脚本
在3台主机上分别执行如下命令：<br/>
```shell
# ./install_etcd.sh etcd01 etcd01=http://192.168.80.11:2380,etcd02=http://192.168.80.12:2380,etcd03=http://192.168.80.13:2380
# ./install_etcd.sh etcd02 etcd01=http://192.168.80.11:2380,etcd02=http://192.168.80.12:2380,etcd03=http://192.168.80.13:2380
# ./install_etcd.sh etcd03 etcd01=http://192.168.80.11:2380,etcd02=http://192.168.80.12:2380,etcd03=http://192.168.80.13:2380
```

注意：<br/>
执行上述命令时，不能等一台完全执行成功了再去下一台执行，因为etcd启动后会进行选举leader投票，如果各etcd启动间隔过大，会导致etcd集群启动失败

 ### 2.3 拷贝/usr/local/kubernetes/bin/etcd*文件到/usr/bin
```shell
# cp /usr/local/kubernetes/bin/etcd* /usr/bin
```

 ### 2.4	确认etcd集群
 ```shell
 # etcdctl member list
 aca05a37b3249961: name=etcd02 peerURLs=http://192.168.80.12:2380 clientURLs=http://192.168.80.12:2379 isLeader=true
 e496d796aa0990dc: name=etcd03 peerURLs=http://192.168.80.13:2380 clientURLs=http://192.168.80.13:2379 isLeader=false
 eb2b0e6d1a23a1fe: name=etcd01 peerURLs=http://192.168.80.11:2380 clientURLs=http://192.168.80.11:2379 isLeader=false
 ```
说明集群已经启动，其中12是leader。
## 3.安装Flannel
Flannel需要在所有主机上安装。<br/>
### 3.1 编辑install_flannel.sh脚本设置正确的网卡接口及etcd键值
 
### 3.2	在3台主机上分别执行下面命令
```shell
# ./install_flannel.sh  http://192.168.80.11:2379,http://192.168.80.12:2379,http://192.168.80.13:2379
```
### 3.3	确认
```shell
# ifconfig flannel.1
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 172.18.89.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::6854:deff:fe7c:f389  prefixlen 64  scopeid 0x20<link>
        ether 6a:54:de:7c:f3:89  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 29 overruns 0  carrier 0  collisions 0
```        
说明安装成功。
## 4.部署Docker
docker需要在所有主机上安装，采用二进制安装方式进行。<br/>
### 4.1 执行安装脚本
```shell
./install-docker.sh  docker-18.03.0-ce.tgz
```
### 4.2	确认
```shell
# docker version
Client:
 Version:	18.03.0-ce
 API version:	1.37
 Go version:	go1.9.2
 Git commit:	0520e24
 Built:	Wed Mar 21 23:05:52 2018
 OS/Arch:	linux/amd64
 Experimental:	false
 Orchestrator:	swarm

Server:
 Engine:
  Version:	18.03.0-ce
  API version:	1.37 (minimum version 1.12)
  Go version:	go1.9.4
  Git commit:	0520e24
  Built:	Wed Mar 21 23:14:54 2018
  OS/Arch:	linux/amd64
  Experimental:	false
```
## 5.部署kubernetes
### 5.1	部署master节点
在192.168.80.11主机上执行下面命令：
```shell
# ./install_k8s_master.sh 192.168.80.11 http://192.168.80.11:2379,http://192.168.80.12:2379,http://192.168.80.13:2379
```
### 5.2	部署node节点
在192.168.80.12/13主机上执行下面命令：
```shell
#  ./install_k8s_node.sh 192.168.80.11
```
### 5.3	将 /usr/local/kubernetes/bin 下的kube*文件复制到 /usr/bin
```shell
# cp /usr/local/kubernetes/bin/kube* /usr/bin 
```

### 5.4	确认集群状态
查看集群状态<br/>
```shell
# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
controller-manager   Healthy   ok
```
查看node状态<br/>
```shell
# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
node1     Ready     <none>    24m       v1.10.0
node2     Ready     <none>    23m       v1.10.0
```
至此，一个简单的kubernetes集群就搭建完成了。

## 6.	部署Busybox容器
### 6.1	准备容器镜像
对于无法上网的环境，可以使用node上的本地镜像来启动容器。除了要busybox容器外，还需要pause镜像才能启动busybox容器。<br/>
镜像使用上传的tar文件，用docker  load命令导入，如下：<br/>
```shell
# docker load -i /software/k8s_v1.10.0_install/kubernetes/images/busybox.tar
# docker load -i /software/k8s_v1.10.0_install/kubernetes/images/gcr.io~google_containers~pause-amd64~3.0.tar.gz
```

### 6.2	编写busybox_rc.yaml
具体内容详见安装介质。

### 6.3	在master上运行rc
```shell
# kubectl create -f /software/k8s_v1.10.0_install/kubernetes/images/busybox-rc.yaml
```

### 6.4	确认
在master执行kubectl get pods可以看到已经启动了2个pod，状态是运行中：<br/>
```shell
# kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
busybox-rc-jpdlw   1/1       Running   0          14s
busybox-rc-xkfl4   1/1       Running   0          14s
```

容器在node上运行，可以在node上执行docker ps –a 看到已经运行的容器：<br/>
```shell
# docker ps -a
CONTAINER ID        IMAGE                                      COMMAND             CREATED             STATUS              PORTS               NAMES
190f043094bb        8ac48589692a                               "sleep 3600"        2 minutes ago       Up 2 minutes                            k8s_busybox_busybox-rc-jpdlw_default_576a33f6-cc75-11e8-94ed-000c2992c77a_0
50f80a93d164        gcr.io/google_containers/pause-amd64:3.0   "/pause"            2 minutes ago       Up 2 minutes                            k8s_POD_busybox-rc-jpdlw_default_576a33f6-cc75-11e8-94ed-000c2992c77a_0
```
docker exec -it {容器ID} sh进入其中一个容器，执行命令进行验证。<br/>

## 7.	部署kube-dns和DashBoard服务
### 7.1	准备镜像
从安装介质中load镜像：<br/>
```shell
# docker load -i /software/k8s_v1.10.0_install/kubernetes/images/k8s-dns-dnsmasq-nanny-amd64_v1.14.7.tar
# docker load -i /software/k8s_v1.10.0_install/kubernetes/images/k8s-dns-kube-dns-amd64_1.14.7.tar
# docker load -i /software/k8s_v1.10.0_install/kubernetes/images/k8s-dns-sidecar-amd64_1.14.7.tar
# docker load -i /software/k8s_v1.10.0_install/kubernetes/images/kubernetes-dashboard-amd64_v1.8.3.tar
```
注：注意使用的是kube-dns1.14.7版本镜像，1.14.1镜像在docker ce 18.03下跑不起来<br/>
### 7.2	部署kube-dns服务
Kube-dns作为集群内部的域名解析服务，没有它就只能使用ip地址来相互访问。<br/>
执行如下命令部署启动kube-dns服务：
```shell
# kubectl create -f kube-dns.yaml
```
注：注意修改kube-dns.yaml中的clusterIP地址和kube-master-url地址。<br/>

以下3个命令可以查看kube-dns的状态：<br/>
```shell
# kubectl get pod -n kube-system -o wide
# kubectl get service -n kube-system -o wide
# kubectl get deployment -n kube-system -o wide
```

### 7.3 部署DashBoard服务
执行命令部署启动dashboard服务：
```shell
# kubectl create -f kubernetes-dashboard.yaml
```
注：注意修改kubernetes-dashboard.yaml中的API server地址和开放的Node port。<br/>

4)	确认
执行命令：kubectl get pods -o wide –namespace kube-system
```shell
# kubectl get pods -o wide --namespace kube-system
NAME                                    READY     STATUS    RESTARTS   AGE       IP            NODE
kube-dns-7cf564b67d-x9w92               3/3       Running   0          39m       172.18.89.3   node2
kubernetes-dashboard-848f7b85f8-txfhj   1/1       Running   0          8m        172.18.7.3    node1
```
使用http://{NodeIP}:30090访问即可，如：http://192.168.80.12:30090
 

