

k8s  ubuntu 集群安装部署



# ubuntu系统卸载与安装docker



## 卸载

在ubuntu环境下，卸载docker，命令如下：

```sh
sudo apt-get remove docker docker-engine docker-ce docker.io
```

卸载Docker后,/var/lib/docker/目录下会保留原Docker的镜像,网络,存储卷等文件. 如果需要全新安装Docker,需要删除/var/lib/docker/目录。

```
rm -fr /var/lib/docker/
```

## 安装

在ubuntu环境下安装docker，下载以阿里云镜像源地址为例，步骤如下：

```sh
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
apt-cache madison docker-ce

# 出现的查找结果如下
docker-ce | 17.03.1~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.03.0~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages

# Step 2: 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.1~ce-0~ubuntu-xenial)
sudo apt-get -y install docker-ce=[VERSION]

```

## 验证docker

**1、查看docker服务是否启动：**

```sh
$ systemctl status docker
$ docker version
```

**2、若未启动，则启动docker服务：**

```sh
sudo systemctl start/restart docker
```

**3、经典的hello world：**

```sh
sudo docker run hello-world
```

# k8s ubuntu18 安装

### 准备工作

1. ##### 关闭防火墙

   ```bash
   sudo ufw disable
   ```

2. ##### 关闭系统swap

   ```undefined
    sudo swapoff -a
   ```

3. ##### 同时在每台机器的  vi /etc/hosts 配置如下 

   **ifconfig** 查看机器IP地址

   ```css
   192.168.105.128 master
   192.168.105.133 node1
   192.168.105.130 node2
   
   ```

### 安装docker

建议版本: docker-ce-18.0x

#### 下载K8S 使用

master节点和所有node节点

```tsx
$ apt-get update && apt-get install -y apt-transport-https
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF 
$ apt-get update
# 5. 查找版本
apt-cache madison xxx(比如kubeadm)

# 6. 安装
# 指定版本安装 apt-get install -y kubelet=[version] kubectl=[version] kubeadm=[version]

$ apt-get install -y kubelet kubeadm kubectl
# apt-get install -y kubelet=1.15.3-00 kubectl=1.15.3-00 kubeadm=1.15.3-00

# 启动服务并设置开机自启
$ systemctl enable kubelet && systemctl start kubelet

重新初始化：
$ kubeadm reset

```

初始化： master节点

```sh
sudo kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.15.3 --pod-network-cidr=10.244.0.0/16



PS：kubeadm init很容易出错，如果出错可以运行kubeadm reset重置，然后就可以重新kubeadm init
$ kubeadm reset

```

安装成功后：

```javascript
如果忘记了Master节点的token，可以使用下面命令查看（master节点上操作）：
# kubeadm token list

```

复制备份, 节点加入集群使用:

```sh
kubeadm join 192.168.105.128:6443 --token 4ph0jr.wt66grc5h4dzetg6 \
    --discovery-token-ca-cert-hash sha256:f1841311e6fd2f7d6dc0d2bae6b7b9e30ce9f2b04fd4b829a5bb9af7b84989c1 --ignore-preflight-errors=Swap
    
# node上的kubelet的启动参数中去掉了必须关闭swap的限制，所以同样需要–ignore-preflight-errors=Swap这个参数

```

安装成功后执行

```ruby
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

在安装完Master节点后，查看节点信息（ `kubectl get nodes`)会发现节点的状态为noready。`journalctl -xeu kubelet`[ **journalctl -f -u kubelet.service** 或者 **journalctl -f -u kubelet** ]查看noready的原因发现`network plugin is not ready: cni config uninitialized`是由于cni插件没有配置。其实这是由于还没有配置网络。

#### 安装flannel网络配件

Pod 之间必须有一个网络来进行彼此通信

```ruby
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

发现还是不好使，最后Google 使用

```ruby
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml


应用官方的yaml文件之后，过一会查看所有pod已经正常running状态了，也分配出了对应IP：
 watch kubectl get pod -n kube-system -owide

```

插件安装完成后，可以通过检查`coredns pod`的运行状态来判断网络插件是否正常运行。等待`coredns pod`的状态变成Running，就可以继续添加从节点了。

**kubectl get nodes** 查看节点状态

使用`kubectl get nodes`查看已加入的节点时，出现了`Status`为`NotReady`的情况。

```ruby
root@master1:~# kubectl get nodes
NAME      STATUS      ROLES    AGE    VERSION
master1   NotReady    master   152m   v1.15.0
worker1   NotReady    <none>   94m    v1.15.0

```

这种情况是因为有某些关键的 pod 没有运行起来，首先使用如下命令来看一下`kube-system`的 pod 状态：

```csharp
kubectl get pod -n kube-system

```

**watch kubectl get pod -n kube-system -owide**  查看pod节点的状态

你也可以通过`kubectl describe pod -n kube-system <服务名>`来查看某个服务的详细情况，如果 pod 存在问题的话，你在使用该命令后在输出内容的最下面看到一个`[Event]`条目，如下：

```php
root@master1:~# kubectl describe pod kube-flannel-ds-amd64-9trbq -n kube-system

...

Events:
  Type     Reason                  Age                 From              Message
  ----     ------                  ----                ----              -------
  Normal   Killing                 29m                 kubelet, worker1  Stopping container kube-flannel
  Warning  FailedCreatePodSandBox  27m (x12 over 29m)  kubelet, worker1  Failed create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "kube-flannel-ds-amd64-9trbq": Error response from daemon: cgroup-parent for systemd cgroup should be a valid slice named as "xxx.slice"
  Normal   SandboxChanged          19m (x48 over 29m)  kubelet, worker1  Pod sandbox changed, it will be killed and re-created.
  Normal   Pulling                 42s                 kubelet, worker1  Pulling image "quay.io/coreos/flannel:v0.11.0-amd64"

```

**方案一：修改yml文件的镜像地址**

```
# 提取 https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml 的内容,名为：kube-flannel.yml， 修改quay.io/coreos/flannel:v0.11.0-amd64 为阿里的镜像地址，registry.cn-hangzhou.aliyuncs.com/colorv/flannel:v0.11.0-amd64 ，对应版本到阿里镜像库搜索。
# 在master vi kube-flannel.yml
$ vi kube-flannel.yml
root@master1:~# kubectl apply -f kube-flannel.yml


```

应用官方的yaml文件之后，过一会查看所有pod已经正常running状态了，也分配出了对应IP：
 **watch kubectl get pod -n kube-system -owide**

**方案二：手动拉取镜像**

`flannel`的镜像可以使用如下命令拉到，如果你是其他镜像没拉到的话，百度一下就可以找到国内的镜像源地址了，这里记得把最后面的版本号修改成你自己的版本，具体的版本号可以用上面说的`kubectl describe`命令看到：

```undefined
docker pull registry.cn-hangzhou.aliyuncs.com/colorv/flannel:v0.10.0-amd64

```

等镜像拉取完了之后需要把镜像名改一下，改成 k8s 没有拉到的那个镜像名称，我这里贴的镜像名和版本和你的不一定一样，注意修改：

```undefined
docker tag registry.cn-hangzhou.aliyuncs.com/colorv/flannel:v0.10.0-amd64 quay.io/coreos/flannel:v0.10.0-amd64

```

修改完了之后过几分钟 k8s 会自动重试，等一下就可以发现不仅`flannel`正常了，其他的 pod 状态也都变成了`Running`，这时再看 node 状态就可以发现问题解决了：

```ruby
root@master1:~# kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   3h27m   v1.15.0
worker1   Ready    <none>   149m    v1.15.0

```



### 装 dashboard

```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

# registry.cn-hangzhou.aliyuncs.com/kuberneters/kubernetes-dashboard-amd64:v1.10.0

# 查看token
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
$ kubectl proxy

```



- 生成p12文件

```ruby
# 生成client-certificate-data
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt

# 生成client-key-data
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key

# 生成p12，这里会要求输入密码，记住这个密码
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"

```

- 导入p12文件

使用`kubectl get secret --all-namespaces|grep dashboard`查看dashboard关联的身份令牌token。

# 工作节点加入失败

在子节点执行`kubeadm join`命令后返回超时错误，如下：

```ruby
root@worker2:~# kubeadm join 192.168.56.11:6443 --token wbryr0.am1n476fgjsno6wa --discovery-token-ca-cert-hash sha256:7640582747efefe7c2d537655e428faa6275dbaff631de37822eb8fd4c054807
[preflight] Running pre-flight checks
error execution phase preflight: couldn't validate the identity of the API Server: abort connecting to API servers after timeout of 5m0s

```

在`master`节点上执行`kubeadm token create --print-join-command`重新生成加入命令，并使用输出的新命令在工作节点上重新执行即可。

注意 运行完命令时，只是将flannel镜像pull到docker中，你还需要通过`kubectl get nodes` 查看Pod的STATUS是否是Ready, flannel的启动需要几秒钟的时间
最后在使用   `ifconfig`



![img](https://upload-images.jianshu.io/upload_images/2656066-58cc0a3ebf8c2839.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

看到cni 和flannel 网卡启动 他们的IP都是10.244.1.1 和相对应即Pod网络



```sh
# 查看
kubectl get pods --namespace kube-system
kubectl get svc --namespace kube-system

# 查看状态日志
kubectl describe -n kube-system pod kube-dns-2247936740-c5fy2

```



#### 节点加入集群

在节点也需要安装`kubelet kubeadm kubectl`
加入集群

```sh
kubeadm join 192.168.105.128:6443 --token 4ph0jr.wt66grc5h4dzetg6 \
    --discovery-token-ca-cert-hash sha256:f1841311e6fd2f7d6dc0d2bae6b7b9e30ce9f2b04fd4b829a5bb9af7b84989c1

```



### node1 NotReady

```sh
$ kubectl get node node1 -oyaml 
# 看 condition
[root@node01 ~]# service kubelet status
$ kubectl describe node node1 # 查看node信息

```

```
[root@master ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}

```

若上面命令结果的STATUS字段为“Healthy”，则表示组件处于健康运行状态，否则需要检查其错误所在，必要时可使用“kubeadm reset”命令重置之后重新进行集群初始化。

```
其中“getcompontsstatuses”即能显示出集群组件当前的状态，也可使用其简写格式“get cs”

```



输入 **systemctl status kubelet** 命令查看 **kubelet** 的情况，发现 **kubelet** 确实没有启动：



# kubelet服务失败：Unable to update cni config: No networks found in /etc/cni/net.d

小故障排除提示：

“kubeadm reset”并尝试重新启动集群但由于kubelet服务未运行而失败。

执行此命令“**journalctl -xeu kubelet**”

错误消息“无法更新cni配置：在/etc/cni/net.d中找不到网络”

我检查了github：https：//github.com/kubernetes/kubernetes/issues/54918

应用解决方法“**chmod 777 /etc/cni/net.d**”并尝试启动该服务。

```sh
systemctl start kubelet.service

```

### Master 是ready状态，node为noready状态：

```sh
$ systemctl status -l kubelet
$ sed -i  '2a\  "cniVersion":"0.2.0",' /etc/cni/net.d/10-flannel.conflist
$ systemctl restart kubelet

```

# 卸载kubernetes-dashboard

```tsx
$ sudo kubectl -n kube-system delete $(sudo kubectl -n kube-system get pod -o name | grep dashboard)
pod "kubernetes-dashboard-3313488171-7706x" deleted
pod "kubernetes-dashboard-3313488171-ddkqd" deleted
pod "kubernetes-dashboard-3313488171-dpf9t" deleted
pod "kubernetes-dashboard-3313488171-jdz1n" deleted
pod "kubernetes-dashboard-3313488171-sxc9n" deleted

```





### 参考资料

------

[K8s折腾日记(零) -- 基于 Ubuntu 18.04 安装部署K8s集群](https://tomoyadeng.github.io/blog/2018/10/12/k8s-in-ubuntu18.04/index.html)

[10分钟看懂Docker和K8S](https://my.oschina.net/jamesview/blog/2994112)

[Kubernetes 部署](https://www.jianshu.com/p/7d0fc257b9cf)

[安装kuberneters](http://www.imooc.com/article/273302?block_id=tuijian_wz)

[K8s - 解决主机重启后kubelet无法自动启动问题](https://www.hangge.com/blog/cache/detail_2419.html)

[[k8s]kubernetes安装dashboard步骤](https://www.jianshu.com/p/073577bdec98)





```sh
mkdir key && cd key
#生成证书
openssl genrsa -out dashboard.key 2048 
openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=192.168.246.200'
openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt
#删除原有的证书secret
kubectl delete secret kubernetes-dashboard-certs -n kube-system
#创建新的证书secret
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kube-system
#查看pod
kubectl get pod -n kube-system
#重启pod
kubectl delete pod <pod name> -n kube-system


```







https://www.jianshu.com/p/d18f7d3df53b

kubeadm join 192.168.105.128:6443 --token iwjuyc.c007ireiklq8jchq \
    --discovery-token-ca-cert-hash sha256:e3b05228e495602034a68a0b5334e784ed4577d024b6510b487c939dfe363d33
