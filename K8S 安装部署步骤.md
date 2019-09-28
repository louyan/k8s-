



# K8S 安装部署步骤

### 安装docker

目标：master和所有node节点

```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum install-y docker-ce-18.09.6-3.el7 （kubelet目前最高支持docker版本为该版本，版本更新可替换新版）
# Step 4: 开启Docker服务
sudo service docker start

# 安装指定版本的Docker-CE:
# Step 1: 安装指定版本的docker-ce-selinux
# yum install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
# Step 2: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step 3: 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

### 配置daemon.json 

目标：master和所有node节点

```
vi /etc/docker/daemon.json
编辑内容：
{
    "registry-mirrors": ["https://registry.docker-cn.com"],
    "exec-opts": ["native.cgroupdriver=systemd"]
}
```

### 重启docker

目标：master和所有node节点

```
systemctl restart docker
```

### 配置部署

目标：master和所有node节点

```
step 1: 修改host
	vi /etc/host
	编辑所有节点的ip地址 和 角色名
step 2: 关闭防火墙、selinux和swap，分别运行：
	systemctl stop firewalld
	systemctl disable firewalld
	setenforce 0
	sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
	swapoff -a
	sed -i 's/.*swap.*/#&/' /etc/fstab
step 3:编辑 k8s.conf
	vi /etc/sysctl.d/k8s.conf
	便捷内容:
		net.bridge.bridge-nf-call-ip6tables = 1
		net.bridge.bridge-nf-call-iptables = 1
	运行sysctl --system
step 4:
	配置国内yum源，分别运行：
	mkdir /etc/yum.repos.d/bak && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak
    wget -O /etc/yum.repos.d/CentOS-Base.repo 				  	         http://mirrors.cloud.tencent.com/repo/centos7_base.repo 
    wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo
    yum clean all && yum makecache
   
    wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
step 5:
	配置国内Kubernetes源，vi /etc/yum.repos.d/kubernetes.repo，文件内容如下：
	[kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
step 6: 安装kubeadm、kubelet、kubectl
	yum install -y kubelet kubeadm kubectl
	systemctl enable kubelet
	systemctl start kubelet
	kubeadm reset//重启adm
```

### 部署master节点

目标：master

```
kubeadm init --kubernetes-version=1.14.3 --apiserver-advertise-address=192.168.31.141 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16
其中kubernetes-version参数是k8s版本
apiserver-advertise-address是master节点的ip
pod-network-cidr是定义POD的网段（不用想这个网段是否存在，因为这是k8s的内部虚拟的网络）
PS：kubeadm init很容易出错，如果出错可以运行kubeadm reset重置，然后就可以重新kubeadm init

执行成功后将：
kubeadm join 192.168.172.132:6443 --token 3witgq.prgufvc5mc58ssnl \
    --discovery-token-ca-cert-hash sha256:b95122dca37850136a0750339c5d160d37954f0d1f22b52cf4b2f2eafe45b4fb 
拷贝待用

配置kubectl工具，分别运行：
mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config
```

### 部署node节点

目标：node

```
执行拷贝的代码段：
kubeadm join 192.168.172.132:6443 --token 3witgq.prgufvc5mc58ssnl \
    --discovery-token-ca-cert-hash sha256:b95122dca37850136a0750339c5d160d37954f0d1f22b52cf4b2f2eafe45b4fb 
```

### 部署网络查件

目标：master

```
安装calico（在master节点上操作）运行：
kubectl apply -f \
https://docs.projectcalico.org/v3.8/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
  
  v3.8为当前calico最新版本，Calico官网：https://docs.projectcalico.org
  
  应用官方的yaml文件之后，过一会查看所有pod已经正常running状态了，也分配出了对应IP：
  watch kubectl get pod -n kube-system -owide 
```

### 部署Dashboard

目标：master

```
Dashboard是k8s自带的查看k8s集群运行信息的图形界面软件
PS：注意只可以查看而不能操作
创建Dashboard的yaml文件，分别运行：

PS：其中的30001是Dashboard的端口

wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

sed -i 's/k8s.gcr.io/loveone/g' kubernetes-dashboard.yaml

sed -i '/targetPort:/a\ \ \ \ \ \ nodePort: 30001\n\ \ type: NodePort' kubernetes-dashboard.yaml
部署Dashboard，分别运行：

kubectl create -f kubernetes-dashboard.yaml
创建完成后，检查相关服务运行状态，分别运行：

kubectl get deployment kubernetes-dashboard -n kube-system

kubectl get pods -n kube-system -o wide

kubectl get services -n kube-system

netstat -ntlp|grep 30001（Centos下netstat需要先 yum install net-tools）

创建一个用于Dashboard的用户以及获取用户的令牌（token），分别运行：

kubectl create serviceaccount  dashboard-admin -n kube-system

kubectl create clusterrolebinding  dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin

kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')

当运行最后一行，有输出token，注意token要保存好
 eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tMjZ3dGIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNTZhZjlhM2QtYjIyZC00MzUxLThiMDItZTIxZmQ5MjQ5MmZmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.vCev2FN_1cuF3vkWoc79JiDuWgsxJStiAl2kvW-1h3rE75W9nze3HkuXSwahZ5mZBJxrRAtIbHRhV-D7c_6zW287-D32DxGEuYh5jJTKxS-RWwfvPADHJKr1kyTsZ3_-yt526B9FrMiHmP-tImUnZSvNZgIAv7xDFWbZ_yKEFadyDcFJHVlsQ_dXWWmwQK8K911nFsjkN9DCt8JqTQAInik79X17Dy3PichJ0cK_qaKMFKNutiEegBY3c2zES6kXbFmtukg-kISCPPXgvuok-ZHiiQsXYiz6woypL_oEEvKIDYOPg5Cvn8OFaqn8VTtwIiMPzVzPh8XlbPTu1RpUiw

```

### 安装dashbord

目标：master

```
1. 在master节点直接运行命令
kubectl apply -f https://raw.githubusercontent.com/newgoo/k8s-dep/master/dashboard/dashboard.yml
2.创建访问账户
vi dashboard_service_account_admin.yaml 
	apiVersion: v1
	kind: ServiceAccount
	metadata:
  		name: admin-user
  		namespace: kube-system
vi dashboard_cluster_role_binding_admin.yaml 
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
 		 name: admin-user
	roleRef:
  		apiGroup: rbac.authorization.k8s.io
  		kind: ClusterRole
  		name: cluster-admin
	subjects:
	- kind: ServiceAccount
 		 name: admin-user
  		namespace: kube-system
  3.发布到系统中去：
  kubectl apply -f dashboard_service_account_admin.yaml 
  kubectl apply -f dashboard_cluster_role_binding_admin.yaml 
  4.获取用户登录Token
  kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
  （记下其中的Token值，登录要用base64解码使用）
  eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW5jNm1nIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI0MjgyOWRiZC05ZDAxLTRmODUtYjM2My0wYWU0ZWM2NjYxZmIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.mOJeNP4omtRficKDNQAodTOUGV4FMF-_5kbJIL2KzlX2rs62M0YYqSzr5LSGoBRByRuQObYsLU4ndr6chWKF3TLdV90CWp8f1DZHg9yA50F4MvH6v6abQz_q1AoejusKXdXdahUg1NWoepkJUSKcOqMmmNw-qXPTKOB4pH3l4aiGH6wH9ezEmvYfL_1coaVDbCuAqNx2IbHLmEfesTA6u_2_rD_N1UFnGZ2Xr0MSKLDlcDztY9BgVQ2TFACOelGoPWzS4ykFpnIbnMatI7GYEiER-OcC7UfsYlibu-llSZ86LL8G2f_59hIp8AGLL4-Uq530X7l4SSRtNpdNn4AvMg
  
  5. 创建导入浏览器的.p12文件
  grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
   grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
    openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-web-client"
    输入两次准备设置的密码，
    ll kubecfg.* 查看文件
    取出生成的kubecfg.p12文件，准备导入浏览器。
    
```

![img](https://img2018.cnblogs.com/blog/1016513/201904/1016513-20190429130742225-1011036315.png) 

