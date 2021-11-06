
kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像
- 禁止swap分区

2. 学习目标

1. 在所有节点上安装Docker和kubeadm
2. 部署Kubernetes Master
3. 部署容器网络插件
4. 部署 Kubernetes Node，将节点加入Kubernetes集群中
5. 部署Dashboard Web页面，可视化查看Kubernetes资源

3. 准备环境

```
关闭防火墙：
$ systemctl stop firewalld
$ systemctl disable firewalld

关闭selinux：
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0

关闭swap：
$ swapoff -a  $ 临时
$ vim /etc/fstab  $ 永久
cd /etc/yum.repos.d/
mv CentOS-Base.repo CentOS-Base.repo.ori
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

添加主机名与IP对应关系（记得设置主机名）：
如hostnamectl set-hostname k8s-master01
cat /etc/hosts
192.168.56.135 k8s-master01
192.168.56.136 k8s-node01
192.168.56.137 k8s-node02

将桥接的IPv4流量传递到iptables的链：
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

4. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 4.1 安装Docker
为了防止k8s不兼容最新版docker，这里安装18版本的
yum install -y yum-utils device-mapper-persistent-data lvm2
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce-19.03.12-3.el7
systemctl enable docker && systemctl start docker


### 4.2 添加阿里云YUM软件源


cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


### 4.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：


yum install -y kubelet-1.20.0 kubeadm-1.20.0 kubectl-1.20.0
systemctl enable kubelet


5. 部署Kubernetes Master

在192.168.56.135（Master）执行。

```
$ kubeadm init \
  --apiserver-advertise-address=192.168.67.40 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.20.0 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
```

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

上一步完成后执行如下（master端会有提示）
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


$ kubectl get nodes
```

6. 安装Pod网络插件（CNI）

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

确保能够访问到quay.io这个registery。

如果下载失败，可以改成这个镜像地址：lizhenliang/flannel:v0.11.0-amd64

7. 加入Kubernetes Node

在192.168.56.136/137（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
$ kubeadm join 192.168.56.135:6443 --token 8dbunt.xmqng5i4tk1nrta7 \
    --discovery-token-ca-cert-hash sha256:8ce8d089b11a7eaff9d18405cdfa052d3e4f5e09f50bc88e06a8884de634b68c
```
稍等片刻，在master查看节点信息。
[root@k8s-master01 ~]# kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    master   13m   v1.20.0
k8s-node01     Ready    <none>   84s   v1.20.0
k8s-node02     Ready    <none>   66s   v1.20.0

8. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
[root@k8s-master01 ~]# kubectl get pod,svc
NAME                         READY   STATUS              RESTARTS   AGE
pod/nginx-554b9c67f9-4bcw5   0/1     ContainerCreating   0          20s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.1.0.1       <none>        443/TCP        16m
service/nginx        NodePort    10.1.232.178   <none>        80:30778/TCP   9s


访问地址：http://192.168.56.136:30778  

9. 部署 Dashboard   2.0

https://github.com/tuwei1314/k8s/blob/main/recommended.yaml  github地址下载



访问地址：https://192.168.56.136:30001
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/admin/{print $1}')
```
使用输出的token登录Dashboard。

10.  k8s 命令自动补全 
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
安装k8s 1.20 安装nfs插件实现pv自动供给，
unexpected error getting claim reference: selfLink was empty, can't make reference 导致pvc无法绑定挂载
etc/kubernetes/manifests/kube-apiserver.yaml 添加"--feature-gates=RemoveSelfLink=false"
