# 安装 Kubernetes 集群

## 在所有节点中执行以下操作

### Step 1: 环境设置
```shell
cat <<EOF | sudo tee -a /etc/hosts
192.168.137.30 k8s-master
192.168.137.31 k8s-worker1
192.168.137.32 k8s-worker2
EOF

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

# 允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo modprobe overlay && modprobe br_netfilter
sudo sysctl --system

# 禁用 selinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 关闭虚拟内存
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a

sudo firewall-cmd --permanent --add-port=80/tcp \
 && firewall-cmd --permanent --add-port=443/tcp \
 && firewall-cmd --reload
```

### Step 2: 安装容器运行时（containerd）
```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sudo yum install -y containerd.io-1.4.4
sudo containerd config default> /etc/containerd/config.toml

sudo vi /etc/containerd/config.toml

# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
#  ...
#  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#    SystemdCgroup = true
# sand_box=registry.aliyuncs.com/google_containers/pause:3.2

sudo systemctl enable --now containerd
sudo systemctl status containerd

sudo cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum clean all && yum makecache -y

# 安装 kubeadm、kubelet、kubectl
sudo yum install -y kubelet-1.20.5 kubeadm-1.20.5 kubectl-1.20.5 --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

## 再主节点中执行以下操作

### Step 1: 环境准备
```shell
sudo hostnamectl set-hostname k8s-master

sudo firewall-cmd --permanent --add-masquerade \
  && firewall-cmd --permanent --add-port=443/tcp \
  && firewall-cmd --permanent --add-port=6443/tcp \
  && firewall-cmd --permanent --add-port=2379-2380/tcp \
  && firewall-cmd --permanent --add-port=10250/tcp \
  && firewall-cmd --permanent --add-port=10251/tcp \
  && firewall-cmd --permanent --add-port=10252/tcp \
  && firewall-cmd --permanent --add-port=10255/tcp \
  && firewall-cmd --reload
```

### Step 2: 初始化主节点
```shell
# kubeadm config print init-defaults > kubeadm-config.yaml
sudo cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.137.30
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  name: k8s-master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.20.5
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF

kubeadm init --config kubeadm-config.yaml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# kubeadm join 192.168.137.30:6443 --token abcdef.0123456789abcdef \
#    --discovery-token-ca-cert-hash sha256:b05e8dc6f7d333794d5bdc802b5bf5a7b0ee4e13ddee732fc6c4094853aae2d2
```

### Step 3: 安装集群网络
```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## 在工作节点中执行以下操作

### Step 1: 环境准备
```shell
sudo hostnamectl set-hostname k8s-worker1

sudo firewall-cmd --permanent --add-port=30000-32767/tcp \
 && firewall-cmd --permanent --add-port=10250/tcp \
 && firewall-cmd --reload
```

### Step 2: 加入集群
```shell
kubeadm join 192.168.137.30:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:b05e8dc6f7d333794d5bdc802b5bf5a7b0ee4e13ddee732fc6c4094853aae2d2

# 启用 kubectl
mkdir -p $HOME/.kube
scp k8s-master:$HOME/.kube/config $HOME/.kube
```

## 使用Docker容器运行时
```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install -y docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io-1.4.4
mkdir -p /etc/docker

# "registry-mirrors": ["https://sqx5pgui.mirror.aliyuncs.com"]
sudo cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
```

## 安装 Ingress
```yaml



```

## 问题排查
```shell
# 获取节点情况
kubectl get nodes
# 或
kubectl describe node $node_name

# 日志查看
journalctl -xefu kubelet

# 查看镜像下载情况
crictl images
# 或
ctr -n k8s.io images ls

# 端口占用查询
ss -antulp | grep 6443

# 新建镜像标签
ctr --namespace=k8s.io i tag $old_tag $new_tag

# 先下载k8s镜像
kubeadm config images pull

# 删除节点
kubectl delete node $node_name

# 设计节点为 node 角色
kubectl label nodes $node_name node-role.kubernetes.io/node=

# 主节点不负载
kubectl taint nodes k8s-master node-role.kubernetes.io/master=true:NoSchedule

kubectl get po -n ingress-nginx -o wide

## 手动拉取基本镜像
crictl pull registry.aliyuncs.com/google_containers/kube-proxy:v1.20.5
crictl pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.5
crictl pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.5
crictl pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.5
crictl pull registry.aliyuncs.com/google_containers/pause:3.2
crictl pull registry.aliyuncs.com/google_containers/etcd:3.4.9-1
crictl pull registry.aliyuncs.com/google_containers/coredns:1.7.0
crictl pull calico/cni:v3.18.1
crictl pull calico/pod2daemon-flexvol:v3.18.1
crictl pull calico/node:v3.18.1
crictl pull calico/kube-controllers:v3.18.1
```