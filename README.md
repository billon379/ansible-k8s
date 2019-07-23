# 使用kubeadm搭建k8s集群

## 一、集群配置信息
| 操作系统      |  主机名      | IP               | 组件                                                          |
| :---        | :---        | :---             | :---                                                          |
| CentOS 7.6  | k8s_master  | 192.168.217.129  | etcd、kube-apiserver、kube-controller-manager、kube-scheduler  |
| CentOS 7.6  | k8s_node01  | 192.168.217.136  | kubelet、kube-proxy                                           |
| CentOS 7.6  | k8s_node02  | 192.168.217.137  | kubelet、kube-proxy                                           |
----------------------------------------------------------------------------------------------------------------

## 二、全局操作（所有节点执行）
### 1、安装ntp服务
```
# yum -y install ntpdate
# ntpdate cn.pool.ntp.org
```
### 2、关闭firewalld
Kubernetes的master（管理主机）与node（工作节点）之间会有大量的网络通信，安全的做法是在防火墙上配置各组件需要相互通信的端口号。
在一个安全的内部网络环境中建议关闭防火墙服务，这里我们关闭防火墙来部署测试环境。
```
# systemctl disable firewalld
# systemctl stop firewalld
```
### 3、关闭SELinux
禁用SELinux的目的是让容器可以读取主机文件系统。
```
# sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
# sed -i 's/SELINUXTYPE=targeted/#&/' /etc/selinux/config
# setenforce 0
```
### 4、关闭swap
当前的Qos策略都是假定主机不启用内存Swap。如果主机启用了Swap，那么Qos策略可能会失效。
例如：两个Pod都刚好达到了内存Limits，由于内存Swap机制，它们还可以继续申请使用更多的内存。 
如果Swap空间不足，那么最终这两个Pod中的进程可能会被“杀掉”。目前Kubernetes和Docker尚不支持内存Swap空间的隔离机制。
```
# swapoff -a && sed -i "s/\/dev\/mapper\/centos-swap/\#\/dev\/mapper\/centos-swap/g" /etc/fstab
```
### 5、防止iptables被绕过
```
# modprobe br_netfilter
# cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# sysctl --system
```

## 三、安装docker-ce（所有节点执行）
### 1、安装所需的软件包
```
# yum install -y yum-utils device-mapper-persistent-data lvm2
```
### 2、设置稳定的库
```
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
### 3、通过yum安装docker-ce
1.安装最新版本的docker-ce（*如果需要安装特定的版本跳过此步骤，参考步骤2*）
```
# yum install -y docker-ce
```
2.查看库中可用的版本，安装特定的版本
```
# yum list docker-ce.x86_64  --showduplicates | sort -r
# yum install docker-ce-18.09.7-3.el7
```
注:版本号格式为docker-ce-详细版本号，如：docker-ce-17.06.2.ce-1.el7.centos
### 4、启动docker
```
# systemctl start docker
```
### 5、将服务加入到启动项
```
chkconfig docker on
```
### 6、修改docker参数
```
# tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
# systemctl daemon-reload
# systemctl restart docker
```

## 四、安装kubernetes初始化工具（所有节点执行）
### 1、使用阿里云的kubernetes源
```
# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### 2、安装工具
```
# yum install -y kubelet kubeadm kubectl
```
### 3、启动kubelet
```
# systemctl enable kubelet && systemctl start kubelet
```

## 五、集群初始化（master节点执行）
### 1、查看集群初始化所需镜像及对应依赖版本号
```
# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.15.0
k8s.gcr.io/kube-controller-manager:v1.15.0
k8s.gcr.io/kube-scheduler:v1.15.0
k8s.gcr.io/kube-proxy:v1.15.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```
### 2、因为这些重要镜像都被墙了，所以要预先单独下载好，然后才能初始化集群。脚本如下：
```
#!/bin/bash

set -e

KUBE_VERSION=v1.15.0
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.10
CORE_DNS_VERSION=1.3.1

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
```
### 3、初始化集群
```
# kubeadm init --kubernetes-version=v1.15.0 --pod-network-cidr=192.168.0.0/16
```
*注意：初始化之后会安装网络插件，这里选择了calico，所以修改 --pod-network-cidr=192.168.0.0/16*
```
[root@localhost ~]# kubeadm init --kubernetes-version=v1.15.0 --pod-network-cidr=192.168.0.0/16
[init] Using Kubernetes version: v1.15.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost localhost] and IPs [192.168.217.129 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost localhost] and IPs [192.168.217.129 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [localhost kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.217.129]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 17.507490 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node localhost as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node localhost as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 8pxu6r.zs445r86lm26icgn
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.217.129:6443 --token 8pxu6r.zs445r86lm26icgn \
    --discovery-token-ca-cert-hash sha256:9bd837fdc8bbac09dd0de9415dcef20fd36e60ed8efb94928a83232070306754
```
以上输出显示初始化成功，并给出了接下来的必要步骤和节点加入集群的命令，照着做即可。
```
[root@k8s-master ~]# mkdir -p $HOME/.kube
[root@k8s-master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
查看已经运行的pod
```
[root@localhost ~]# kubectl get pod -n kube-system -owide
NAME                                READY   STATUS    RESTARTS   AGE     IP                NODE        NOMINATED NODE   READINESS GATES
coredns-5c98db65d4-4slqh            0/1     Pending   0          5h20m   <none>            <none>      <none>           <none>
coredns-5c98db65d4-gbfjj            0/1     Pending   0          5h20m   <none>            <none>      <none>           <none>
etcd-localhost                      1/1     Running   0          5h19m   192.168.217.129   localhost   <none>           <none>
kube-apiserver-localhost            1/1     Running   0          5h19m   192.168.217.129   localhost   <none>           <none>
kube-controller-manager-localhost   1/1     Running   0          5h19m   192.168.217.129   localhost   <none>           <none>
kube-proxy-prd7b                    1/1     Running   0          5h20m   192.168.217.129   localhost   <none>           <none>
kube-scheduler-localhost            1/1     Running   0          5h19m   192.168.217.129   localhost   <none>           <none>
```
到这里，会发现除了coredns未ready，这是正常的，因为还没有网络插件，接下来安装calico后就变为正常running了。

## 六、安装calico（master节点执行）
```
# kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml
```
应用官方的yaml文件之后，过一会查看所有pod已经正常running状态了，也分配出了对应IP：
```
[root@localhost ~]# kubectl get pod -n kube-system -owide
NAME                                READY   STATUS    RESTARTS   AGE     IP                NODE        NOMINATED NODE   READINESS GATES
calico-node-glqpt                   1/1     Running   0          2m58s   192.168.217.129   localhost   <none>           <none>
coredns-5c98db65d4-295qc            1/1     Running   0          6m22s   192.168.0.9       localhost   <none>           <none>
coredns-5c98db65d4-td7l2            1/1     Running   0          6m22s   192.168.0.8       localhost   <none>           <none>
etcd-localhost                      1/1     Running   0          5m39s   192.168.217.129   localhost   <none>           <none>
kube-apiserver-localhost            1/1     Running   0          5m23s   192.168.217.129   localhost   <none>           <none>
kube-controller-manager-localhost   1/1     Running   0          5m36s   192.168.217.129   localhost   <none>           <none>
kube-proxy-7cn99                    1/1     Running   0          6m22s   192.168.217.129   localhost   <none>           <none>
kube-scheduler-localhost            1/1     Running   0          5m15s   192.168.217.129   localhost   <none>           <none>
```
查看节点状态
```
[root@localhost ~]# kubectl get node -owide
NAME        STATUS   ROLES    AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
localhost   Ready    master   6h14m   v1.15.0   192.168.217.129   <none>        CentOS Linux 7 (Core)   3.10.0-957.el7.x86_64   docker://18.9.7
```

## 七、加入集群（工作节点执行）
### 1、先在需要加入集群的节点上下载必要镜像，下载脚本如下：
```
#!/bin/bash

set -e

KUBE_VERSION=v1.15.0
KUBE_PAUSE_VERSION=3.1

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy-amd64:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
```
### 2、然后在主节点初始化输出中获取加入集群的命令，复制到工作节点执行即可：
```
[root@localhost ~]# kubeadm join 192.168.217.129:6443 --token 8pxu6r.zs445r86lm26icgn --discovery-token-ca-cert-hash sha256:9bd837fdc8bbac09dd0de9415dcef20fd36e60ed8efb94928a83232070306754
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## 八、使用ansible脚本执行上述操作
### 1、文件结构说明
```
ansible-k8s/
├── roles  #将执行流程分为多个role，分别完成相应功能
│   ├── docker  #docker安装脚本，对应上述安装步骤[三、安装docker-ce（所有节点执行）]
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   ├── templates
│   │   │   └── deamon.json.j2
│   │   └── vars
│   │       └── main.yml
│   ├── k8s-common  #通用操作执行脚本，对应上述安装步骤[二、全局操作（所有节点执行）][四、安装kubernetes初始化工具（所有节点执行）]
│   │   ├── handlers
│   │   ├── tasks
│   │   │   └── main.yml
│   │   ├── templates
│   │   │   └── kubernetes.repo.j2
│   │   └── vars
│   │       └── main.yml
│   ├── k8s-master  #master节点执行脚本，对应上述安装步骤[五、集群初始化（master节点执行）][六、安装calico（master节点执行）]
│   │   ├── handlers
│   │   ├── tasks
│   │   │   └── main.yml
│   │   ├── templates
│   │   │   └── k8s_master_images.sh.j2
│   │   └── vars
│   │       └── main.yml
│   └── k8s-node  #node节点执行脚本，对应上述安装步骤[七、加入集群（工作节点执行）]
│       ├── handlers
│       ├── tasks
│       │   └── main.yml
│       ├── templates
│       │   └── k8s_node_images.sh.j2
│       └── vars
│           └── main.yml
├── inventory  #主机清单
└── test.yml  #测试用的playbook
```
### 2、inventory文件内容如下
```
#k8s_master节点
[k8s_master]
192.168.217.129

#k8s_node节点
[k8s_node]
192.168.217.137
```
### 3、test.yml内容如下
```
#This playbook deploys the k8s cluster
#1.install k8s-common,docker on all nodes
- name: 1.install k8s-common,docker on all nodes
  hosts: k8s_master,k8s_node
  remote_user: root
  roles:
    - k8s-common
    - docker
  tags:
    - k8s-common

#2.install k8s-master on master
- name: 2.install k8s-master on master
  hosts: k8s_master
  remote_user: root
  roles:
    - k8s-master
  tags:
    - k8s-master

#3.install k8s-node on node
- name: 3.install k8s-node on node
  hosts: k8s_node
  remote_user: root
  roles:
    - k8s-node
  tags:
    - k8s-node
```
### 4、执行ansible脚本
```
[root@localhost ansible-k8s]# ansible-playbook test.yml
```
### 5、执行部分tag
```
[root@localhost ansible-k8s]# ansible-playbook test.yml --tags "k8s-common"
```