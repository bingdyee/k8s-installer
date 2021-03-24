# k8s-installer
Install Kubernetes(k8s) Cluster on CentOS 7 using Kubeadm.


## CentOS7 下 SSH登录慢

```shell
vi /etc/ssh/sshd_config
GSSAPIAuthentication no
UseDNS no

systemctl restart sshd
```