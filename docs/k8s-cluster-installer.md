# Installing Kubernetes Cluster

## Perform the following steps on all nodes

### Step 1: Configure DNS

```shell
# ip link
# sudo cat /sys/class/dmi/id/product_uuid
# netstat -nlp | grep "8080|6443"
# netstat -nlp | grep "2379|2380"
# netstat -nlp | grep "10250|10251|10252"

sudo vi /etc/hosts
# 192.168.137.11 k8s-master
# 192.168.137.12 k8s-worker1
# 192.168.137.13 k8s-worker2
```

### Step 2: Disable SELinux

```shell
sudo setenforce 0
sudo cp -p /etc/selinux/config /etc/selinux/config.bak
sudo sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
sudo getenforce
```

### Step 3: Turn off Swap

```shell
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a
```

### Step 4: Update Iptables Settings

```shell
sudo cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Step 5: Install Docker

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install -y docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io-1.3.9
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

### Step 6: Using aliyun Kubernetes source

```shell
sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum clean all
yum makecache -y
```

### Step 7: Install Kubernetes

```shell
sudo yum install -y kubelet-1.19.8 kubeadm-1.19.8 kubectl-1.19.8 --disableexcludes=kubernetes

sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
```



## Perform the following steps on master node

### Step 1: Set hostname

```shell
sudo hostnamectl set-hostname k8s-master
```

### Step 2: Configure Firewall

```shell
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```

### Step 3: Pull Docker images (optional)

```shell
# Aliyun Docker repo: registry.cn-hangzhou.aliyuncs.com
# k8s image: registry.aliyuncs.com/google_containers

docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.19.8
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.19.8
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.19.8
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.19.8
docker pull registry.aliyuncs.com/google_containers/pause:3.2
docker pull registry.aliyuncs.com/google_containers/etcd:3.4.9-1
docker pull registry.aliyuncs.com/google_containers/coredns:1.7.0
docker pull calico/cni:v3.18.1
docker pull calico/pod2daemon-flexvol:v3.18.1
docker pull calico/node:v3.18.1
docker pull calico/kube-controllers:v3.18.1
```

### Step 4: Create Cluster with Kubeadm

```shell
kubeadm reset -f
kubeadm init \
  --kubernetes-version=v1.19.8 \
  --pod-network-cidr=192.168.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --upload-certs
  
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 5: Install calico network

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```



## Perform the following steps on worker nodes

### Step 1: Set hostname

```shell
sudo hostnamectl set-hostname k8s-worker
```

### Step 2: Configure Firewall

```shell
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --reload
```

### Step 3: Pull Docker images (optional)

```
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.19.8
docker pull registry.aliyuncs.com/google_containers/pause:3.2
```

### Step 4: Join worker nodes to master node

```shell
kubeadm join 192.168.137.11:6443 --token xxx \
    --discovery-token-ca-cert-hash sha256:xxx

mkdir -p $HOME/.kube
scp k8s-master:$HOME/.kube/config $HOME/.kube
```

## Issues

