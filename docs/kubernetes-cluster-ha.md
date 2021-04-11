# 安装高可用 Kubernetes 集群

## 机器配置要求
| 主机 | IP地址 | CPU | RAM | HD | 备注 |
| :---: | :---: | :---: | :---: | :---: | :---: |
| master1 | 192.168.137.40 | 2C | 2G | 50G |
| master2 | 192.168.137.41 | 2C | 2G | 50G |
| node1 | 192.168.137.42 | 2C | 2G | 50G |
| 虚拟IP（VIP） | 192.168.137.43 | 2C | 2G | 50G |


## 在 k8s-master1、k8s-master2 节点上执行以下操作

### 环境准备
```shell
# cat >> /etc/hosts << EOF
cat <<EOF | sudo tee -a /etc/hosts
192.168.137.40 master1
192.168.137.41 master2
192.168.137.42 node1
192.168.137.43 master.k8s.io
EOF

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

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

sudo yum install -y ntpdate yum-utils
ntpdate time.windows.com
```

### Step 1：安装 keepalive
```shell
sudo yum install -y conntrack-tools libseccomp libtool-ltdl
sudo yum install -y keepalived

cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
  router_id k8s
}

vrrp_script check_haproxy {
  script "killall -0 haproxy"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
  state MASTER
  interface enp0s3
  virtual_router_id 51
  priority 250
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass ceb1b3ec013d66163d6ab
  }

  virtual_ipaddress {
    192.168.137.43
  }

  trach_script {
    check_haproxy
  }

}
EOF

sudo systemctl enable --now keepalived

# ip a s enp0s3
```


### Step 2: 安装 HAProxy
```shell
sudo yum install -y haproxy

cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#---------------------------------------------------------------------
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server  master1 192.168.137.40:6443 check
    server  master2 192.168.137.41:6443 check

#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF

sudo systemctl enable --now haproxy

sudo firewall-cmd --permanent --add-masquerade \
  && firewall-cmd --permanent --add-port=6443/tcp \
  && firewall-cmd --permanent --add-port=2379-2380/tcp \
  && firewall-cmd --permanent --add-port=10250/tcp \
  && firewall-cmd --permanent --add-port=10251/tcp \
  && firewall-cmd --permanent --add-port=10252/tcp \
  && firewall-cmd --permanent --add-port=10255/tcp \
  && firewall-cmd --reload
```

### Step 3: 安装kubeadm、kubelet、CRI
```

```

### Step 3：在 k8s-master1 上初始化
```shell
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
  advertiseAddress: 192.168.137.40
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  name: master1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
    - master1
    - master2
    - master.k8s.io
    - 192.168.137.43
    - 192.168.137.40
    - 192.168.137.41
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: master.k8s.io:16443
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

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Step 4：将 k8s-master2 加入集群

```shell
# 将 master1 密钥复制到 master2
mkdir -p /etc/kubernetes/pki/etcd
scp master1:/etc/kubernetes/admin.conf /etc/kubernetes/
scp master1:/etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} /etc/kubernetes/pki
scp master1:/etc/kubernetes/pki/etcd/ca.* /etc/kubernetes/pki/etcd

kubeadm join master.k8s.io:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f4fc85f1f6842e9030e06eaaddb75b2ea11e3afeb2e80fd7b379c56673ae2e98 \
    --control-plane

```

kubeadm join master.k8s.io:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f4fc85f1f6842e9030e06eaaddb75b2ea11e3afeb2e80fd7b379c56673ae2e98