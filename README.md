# k8s-installer
Install Kubernetes(k8s) Cluster on CentOS 7 using Kubeadm.

## 添加sudo用户
```
sudo usermod -aG [sudo | wheel] username
```

## yum 源更新
```shell
sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
sudo curl -L http://mirrors.aliyun.com/repo/Centos-7.repo -o /etc/yum.repos.d/CentOS-Base.repo
sudo yum clean all && yum makecache
```

## CentOS7 下 SSH登录慢

```shell
vi /etc/ssh/sshd_config
GSSAPIAuthentication no
UseDNS no

systemctl restart sshd
```