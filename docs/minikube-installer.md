# Installing Minikube

## Step 1: Disable SELinux

```shell
sudo setenforce 0
sudo sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

### Step 2: Turn off Swap

```shell
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a
```

## Step 3: Install Docker

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install -y docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io-1.3.9

mkdir -p /etc/docker

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

sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
```

## Step 4: Configure Firewall

```shell
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --add-port=8443/tcp --permanent
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --add-port=30000-32767/tcp --permanent
sudo firewall-cmd --reload
```

## Step 5: Install conntrack、socat、kubectl

```shell
sudo yum -y install conntrack socat

KUBE_VERSION=curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt
curl -LO https://dl.k8s.io/release/$KUBE_VERSION/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client -o json
```

## Step 6: Install Minikube

```shell
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -ivh minikube-latest.x86_64.rpm
minikube version
```

## Step 7: Start Minikube

```shell
minikube start --driver=none \
	--image-mirror-country='cn' \
	--image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
	
kubectl get nodes
```

https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/
https://minikube.sigs.k8s.io/docs/start/

