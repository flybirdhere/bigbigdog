![img](https://upload-images.jianshu.io/upload_images/25267841-8b4319e8ad384031.png?imageMogr2/auto-orient/strip|imageView2/2/w/1078/format/webp)

![img](https://upload-images.jianshu.io/upload_images/25267841-34215061a6e2bfdb.png?imageMogr2/auto-orient/strip|imageView2/2/w/871/format/webp)



![img](https://upload-images.jianshu.io/upload_images/25267841-db2e89b3e6f07dd8.png?imageMogr2/auto-orient/strip|imageView2/2/w/855/format/webp)





虚拟机错误提示解决

```
Error:
This virtual machine appears to be in use.

If this virtual machine is not in use, press the "Take Ownership"  button to obtain ownership of it. Otherwise, press the 'Cancel' button to avoid damaging it.

Solution:
Turn off the VM.
Close VMware Workstation.
Open the folder where VMware files are located indicated in the error message.
Remove any .lck or .lock files.
Run VMware Workstation.
Start the VM.
```

把文件发给另外的节点（例如设置各机器的host解析）

```
scp -rp /etc/hosts 10.0.0.13:/etc/hosts
```

为了让克隆出来的虚拟机变成一个独立的新虚拟机，需要更改其主机名和IP地址

```
克隆的机子的uuid(可以删除)和IP地址，还有mac地址(虚拟机中可以设置)都需要更改

输入hostname可查看当前本机的主机名
输入vi /etc/hostname 可进入VI编辑模式按I改本机的主机名，然后reboot重启虚拟机，主机名就更改了。
输入vi /etc/sysconfig/network-scripts/ifcfg-ens33进入VI编辑模式按I改本机的IP地址 uuid可以删除
输入vi /etc/hosts 进去VI编辑模式按I添加被克隆的虚拟机的IP地址与克隆后的虚拟机的IP地址
输入命令systemctl restart network 重启网络
两台虚拟机都开启后，使用ping命令即可查看网络是否连通

---------
DEVICE=eth0【网卡名称】

　　HWADDR=00:07:E9:05:E8:B4 #对应的网卡网卡地址,即mac地址（文件里可以没有）

　　TYPE=Ethernet#表示网络类型是以太网

　　UUID：网卡的UUID（文件里可以没有）

　　ONBOOT=yes【开机加载】

　　BOOTPROTO=static【是否自动获取，static是静态地址】

　　IPADDR=192.168.146.200【配置你的本地IP】

　　NETMASK=255.255.255.0【子网掩码】

　　GATEWAY=192.168.146.2【默认网关】
---------------
```

k8s集群搭建，首先准备三台虚拟机

```
k8s-master	192.168.178.110
k8s-node1	192.168.178.111
k8s-node2	192.168.178.112

# 注意做好host解析
```

安装kubernetes master节点

```
需要配置
etcd
api-server
controller-manager
scheduler

首先安装etcd服务
yum install ectd -y

etcd配置文件
vim  /etc/etcd/etcd.conf
第6行和第21行需要修改
启动etcd服务
systemctl start etcd.service
systemctl enable etcd.service

安装kubernetes master节点
yum install kubernetes-master.x86_64 -y

api-server配置文件
vim /etc/kubernetes/apiserver
需要修改多行，参考文件

controller-manager和scheduler都共用一个配置文件 
vim /etc/kubernetes/config

启动api-server服务
systemctl start kube-apiserver.service

再启动controller-manager服务
systemctl start kube-controller-manager.service 

再启动scheduler服务
systemctl start kube-scheduler.service

最后，最好把这些服务都设置成开机自启，以免机器意外停机
systemctl enable kube-apiserver.service
systemctl enable kube-controller-manager.service
systemctl enable kube-scheduler.service

服务都启动后，执行一条命令检测它的状态
kubectl get componentstatus 

至此，k8s-master节点就安装好了

```

安装kubernetes node节点

```
k8s-master节点本身也可以作为一个node节点

yum install kubernetes-node.x86_64 -y

api-server配置文件
vim /etc/kubernetes/config
需要修改22行
vim /etc/kubernetes/kubelet
需要修改多行，参考文件

systemctl start kubelet.service
systemctl enable kubelet.service
systemctl start kube-proxy.service
systemctl enable kube-proxy.service

在node1节点上安装
yum install kubernetes-node.x86_64 -y
vim /etc/kubernetes/config
vim /etc/kubernetes/kubelet

systemctl start kubelet.service
systemctl enable kubelet.service
systemctl start kube-proxy.service
systemctl enable kube-proxy.service

同理在node2节点上安装
yum install kubernetes-node.x86_64 -y
vim /etc/kubernetes/config
vim /etc/kubernetes/kubelet

systemctl start kubelet.service
systemctl enable kubelet.service
systemctl start kube-proxy.service
systemctl enable kube-proxy.service

```

所有node节点配置flannel网络插件

```
yum install flannel -y
配置flannel
vim /etc/sysconfig/flanneld

创建一个key (只有master节点上有这个命令，其它节点上操作会失败)
etcdctl set /atomic.io/network/config '{"Network": "172.16.0.0/16"}'
systemctl start flanneld.service
systemctl enable flanneld.service

重启docker服务，插件才会生效
systemctl restart docker

查看日志
tail -f /var/log/messages


sed -i 's#http://127.0.0.1:2379#http://192.168.178.110:2379#g' /etc/sysconfig/flanneld
etcdctl mk /atomic.io/network/config '{"Network": "172.16.0.0/16"}'

master节点

node节点
vim /etc/sysconfig/flanneld
FLANNEL_ETCD_ENDPOINTS="http://192.168.178.110:2379"

重启docker服务，插件才会生效
systemctl restart docker

```

测试跨宿主机容器之间的互通性

```
所有节点执行
docker pull busybox
docker run -it busybox
ifconfig
节点之间互ping

有可能ping不同，原因是iptables规则因为版本原因有些地方默认被改变了，修改即可
把 Chain FORWARD(policy DROP) 修改为 Chain FORWARD(policy ACCEPT)

iptables -P FORWARD ACCEPT
其它ping不同的节点也需要修改

加入docker的启动文件中使iptables规则开机自动启动
systemctl status docker 找到路径
vim /usr/lib/systemd/system/docker.service
加入一条指令（用iptables的全路径 which iptables）
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT

另外节点也需要添加

因为修改了docker的配置文件，所以要想生效需要执行
systemctl daemon-reload


```

创建pod

```
[root@k8s-master pod]# vi nginx_pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.13
      ports:
        - containerPort: 80
~                           

kubectl create -f nginx_pod.yaml
提示servertimeout
vim /etc/kubernetes/apiserver
删掉 ServiceAccount 即可正确创建pod

改完后需要重启
systemctl restart kube-apiserver.service 

错误解决方法
首先找出被assign到哪个节点上去
kubectl get pods -o wide
kubectl describe pod nginx

然后到那个节点 修改配置文件
vim /etc/kubernetes/kubelet
# pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=docker.io/tianyebj/pod-infrastructure:latest"

修改完配置文件后需要重启服务
systemctl restart kubelet.service

ls /var/lib/docker/tmp/

镜像加速
vim /etc/sysconfig/docker
OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false --registry-mirror=https://registry.docker-cn.com --insecure-registry=192.168.178.110:5000'

导入镜像的命令
docker load -i ***tar.gz

```

节点多的时候需要用到私有仓库，节省时间 和流量带宽

私有仓库有官方的registry，还有一个第三方的harbor（推荐）

```
先从官网下载私有仓库registry的镜像包
然后运行registry image
然后更改相关的配置文件（其它docker节点想使用私有仓库的话都需要更改配置文件），重启
vim /etc/sysconfig/docker
为了让我们的节点从私有仓库pull镜像，还需要更改kubelet的配置,让kubelet从配置文件中写的私有仓库地址中pull镜像，重启
vim /etc/kubernetes/kubelet
若上传到私有仓库提示错误，需要修改如下配置文件，重启
vim /etc/docker/daemon.json
{
"registry-mirrors": ["https://registry.docker-cn.com"],
"insecure-registries": ["192.168.178.110:5000"]
}
重启
systemctl daemon-reload
systemctl restart docker


----如何查看私有仓库中的镜像列表------
```

查找kubectl的帮助信息

```
kubectl explain pod.spec.containers
```

一个pod资源至少2个容器，一般四个容器，三个业务容器，一个基础容器，共用一个IP

如果一个pod删不掉，可以加参数强制删除 --force --grace-period=0(垃圾回收周期)

pod的常用操作

```
kubectl create -f nginx.yaml
kubectl get pods
kubectl describe pod nginx
kubectl delete pod test
kubectl delete pod tese --force --grace-period=0
kubectl explain pod.spec.containers
kubectl apply -f nginx.yaml

```

replication controller (RC)

```
vim nginx_rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 2
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: 192.168.178.110:5000/nginx:1.13
        ports:
        - containerPort: 80
```

RC与pod的关联 --Label

```
修改pod 
kubectl edit pod nginx

滚动升级
先复制一份yaml文件
cp nginx-rc.yaml nginx-rc2.yaml
修改yaml信息
vim nginx-rc2.yaml 
:%s#myweb#myweb2#g

kubectl rolling-update myweb -f nginx-rc2.yaml --update-period=30s（参数设置更新间隔时间，多长时间升级一次）

升级可以设置有间隔，回滚最好是一键回滚
kubectl rolling-update myweb2 -f nginx-rc.yaml --update-period=1s

升级到一半遇到bug，取消升级，回滚到上一个版本
kubectl rolling-update myweb myweb2 --rollback

没有指定升级间隔的话，默认是一分钟

```

复制文件到pod中

```
kubectl cp index.html myweb-bc9b3:/usr/share/nginx/html/index.html

修改nodeport默认范围，端口默认必须30000以上，当然也可以修改
vi /etc/kubernetes/apiserver
KUBE_API_ARGS="--service-node-port-range=10000-60000"
systemctl restart kube-apiserver

查看apiserver的启动文件
cat /usr/lib/systemd/system/kube-apiserver

```

至少需要一个rc来保证高可用，还需要一个service来保证能够被外界（比如tomcat）访问

手动改svc的配置文件

```
kubectl edit sve myweb
rc在进行版本升级后会造成svc短时间内访问不了，deployment解决了这个问题

Deployment是一个定义及管理多副本应用（即多个副本 Pod）的新一代对象，与Replication Controller相比，它提供了更加完善的功能，使用起来更加简单方便

创建deployment
vi nginx-deploy.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 192.168.178.110:5000/nginx:1.13
        ports:
        - containerPort: 80

kubectl create -f nginx-deploy.yaml

为了防止干扰，删掉rc资源
kubectl delete rc myweb
kubectl delete svc myweb

一个deploy会先起一个rs，然后rs再起3个po

暴露端口
kubectl expose deployment nginx-deployment --port=80 --type=NodePort

升级版本测试
修改配置文件
kubectl edit deployment nginx-deployment
也可以回滚
kubectl rollout undo deployment nginx-deployment

deployment最佳实践
版本发布
kubectl run nginx --image=192.168.178.110:5000/nginx:1.13 --replicas=3 --record
版本升级
kubectl set image deployment nginx nginx=191.168.178.110:5000/nginx:1.15
版本回滚（可回滚到指定版本）
kubectl rollout undo deployment nginx --to-revision=1

历史版本查询
kubectl rollout history deployment nginx-deployment 

deployment比rc更简单更好用，而且不依赖配置文件，生产环境中deployment是主流，rc用的越来越少



```

pod和pod之间如何通信

```
https://www.qstack.com.cn/tomcat_demo.zip

```

dns

```
wget https://www.qstack.com.cn/skydns.zip
kubectl get pod --namespace=kube-system 

要想所有容器都可以使用这个dns的话，还需要更改配置文件

需要测试检测一下svc的vip是否能解析出来

kubectl get pod 默认查看的是default命名空间下的pod
如果想查看其它命名空间下的pod，需要加参数指定命名空间
kubectl get pod --namespace=kube-system

kubectl get all --namespace=kube-system
查看kube-system命名空间内的所有资源

由于image被某个container引用（拿来运行），如果不将这个引用的container销毁（删除），那image肯定是不能被删除。

所以想要删除运行过的images必须首先删除它的container。继续来看刚才的例子

删除容器 docker rm 容器名
删除镜像 docker rmi 镜像名

```

namespace

```
创建namespace 
kubectl create namespace qiangge

查看namespace
kubectl get namespace

删除namespace，非常危险的命令，会删除该namespace下运行的所有k8s服务
kubectl delete namespace qiangge

查看所有namespce的资源
kubectl get all --all-namespaces


```

pv和pvc是一一绑定的。

```
服务端 客户端
所有节点都需要安装 yum install nfs-utils -y
配置文件
[root@k8s-master ~]# cat  /etc/exports
/data 192.168.178.0/24(rw,async,no_root_squash,no_all_squash)

systemctl restart rpcbind
systemctl restart nfs

showmount -e 192.168.178.110

vi test-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: test
  labels:
    type: test
spec:
  capacity:
    storage: 10Gi 
  accessModes:
    - ReadWriteMany 
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path:  "/data/k8s"
    server: 192.168.178.110
    readOnly: false

kubectl create -f test-pv.yaml 
kubectl get pv
创建pvc
kubectl get pvc

```

暴露端口

```
kubectl expose rc myweb --port=80 --type=NodePort
```

持久化训练

```
cp -a tomcat_demo tomcat_demo_pv
删除service
kubectl delete service mysql
删除pod 
kubectl delete pod mysql
删除所有svc
kubectl delete svc --all

service 和 endpoint和名字进行关联，所以需要名字一致
kubectl get all --all-namespaces



```

**YAML 是一种非常简洁/强大/专门用来写配置文件的语言！**

为了解决软件交付过程中的环境依赖，同时提供一种更加轻量的虚拟化技术，Docker出现了。

Docker是一种CS架构的软件产品，可以把代码及依赖打包成镜像作为交付介质，并且把镜像启动成为容器，提供容器生命周期的管理。

/etc/docker/daemon.json  #这个文件是固定的，是docker默认会去加载的文件，不可以改到其它路径

docker三个核心要素：镜像、容器、仓库

容器是动态的，容器运行时可以对外提供服务。本质上讲是利用namespace和cgroup等技术在宿主机中创建的独立的虚拟空间。

仓库分为公有仓库（Docker Hub，阿里，网易。。）私有仓库（Docker Registry，Harbor）

公有仓库中一般存在一下几种类型的镜像
  - 操作系统镜像
  - 中间件（nginx，redis，mysql，tomcat）
  - 语言编译环境（python，java，golang）
  - 业务镜像（django-demo...）

导出镜像到文件中，从而可以把镜像拷贝给别人
  docker save -o nginx-alpine.tar nginx:alpine

收到别人拷贝过来的包如何用呢，就是从文件中加载镜像
	docker load -i nginx-alpine.tar

docker history images 镜像名 #可以查看镜像是如何构建出来的

 docker run -d -p 5000:5000 --restart always -v
/opt/registry:/var/lib/registry --name registry registry:2
# 把宿主机当前目录/opt/registry挂载到容器内的目录/var/lib/registry

apiVersion: v1  #apiVersion：脚本的版本，Pod 通常使用 v1 这个版本
kind: Pod  #脚本的类型，这里就是pod

在 kubernetes（k8s）中， 使用 Pod 脚本部署 pod，只能单节点部署，无法高可用。
因此需要用到 Deployment ，Deployment 可以指定 Pod 的副本数，通常情况是使用 Deployment 部署。

上一步我们使用 deployment 部署了多个 Pod 实例，但我们无法访问到 Pod 中的 Nginx。
此时就要借助 Service。

YAML 是一种非常简洁/强大/专门用来写配置文件的语言！

容器数据持久化的方法主要有两个。
  1. 挂载主机目录

    docker run --name wangxd -d -v /opt:/opt -v /var/log:/var/log nginx:alpine
  2. 使用volumes卷

    docker volume ls
    docker volume create my-vol
    docker run --name wangxd -d -v my-vol:/opt/my-vol nginx:alpine

> 如何理解构建镜像的过程？
>
> Dockerfile是一堆指令，在docker build的时候，按照该指令进行操作，最终生成我们期望的镜像。

> Namespace 资源隔离
>
> CGroup 资源限制
>
> UnionFS 联合文件系统
>
> 对容器层的操作，主要利用了写时复制（CoW）技术。CoW就是copy-on-write，表示只在需要写时才去复制。
>
> UnionFS 其实是一种为 Linux 操作系统设计的用于把多个文件系统联合到同一个挂载点的文件系统服务。  它能够将不同文件夹中的层联合（Union）到了同一个文件夹中，整个联合的过程被称为联合挂载（Union Mount）

> 为了解决软件交付过程中的环境依赖，同时提供一种更加轻量的虚拟化技术，Docker出现了。
> Docker是一种CS架构的软件产品，可以把代码及依赖打包成镜像作为交付介质，并且把镜像启动成为容器，提供容器生命周期的管理。
> /etc/docker/daemon.json  #这个文件是固定的，是docker默认会去加载的文件，不可以改到其它路径
> docker三个核心要素：镜像、容器、仓库
> 容器是动态的，容器运行时可以对外提供服务。本质上讲是利用namespace和cgroup等技术在宿主机中创建的独立的虚拟空间。
> 仓库分为公有仓库（Docker Hub，阿里，网易。。）私有仓库（Docker Registry，Harbor）
> 公有仓库中一般存在一下几种类型的镜像
>   - 操作系统镜像
>   - 中间件（nginx，redis，mysql，tomcat）
>   - 语言编译环境（python，java，golang）
>   - 业务镜像（django-demo...）
> 导出镜像到文件中，从而可以把镜像拷贝给别人
>     docker save -o nginx-alpine.tar nginx:alpine
> 收到别人拷贝过来的包如何用呢，就是从文件中加载镜像
> 	docker load -i nginx-alpine.tar
> docker history images 镜像名 #可以查看镜像是如何构建出来的
>
>  docker run -d -p 5000:5000 --restart always -v
> /opt/registry:/var/lib/registry --name registry registry:2
> # 把宿主机当前目录/opt/registry挂载到容器内的目录/var/lib/registry
>
> apiVersion: v1  #apiVersion：脚本的版本，Pod 通常使用 v1 这个版本
> kind: Pod  #脚本的类型，这里就是pod
> 在 kubernetes（k8s）中， 使用 Pod 脚本部署 pod，只能单节点部署，无法高可用。
> 因此需要用到 Deployment ，Deployment 可以指定 Pod 的副本数，通常情况是使用 Deployment 部署。
> 上一步我们使用 deployment 部署了多个 Pod 实例，但我们无法访问到 Pod 中的 Nginx。
> 此时就要借助 Service。

> 容器有创建，运行，停止，暂停和删除五种状态。

> pd  
>
> kd get po -o wide

> ```
> # COPY指令能够保留源文件的元数据，如权限，访问时间等等，这点很重要
> ```
