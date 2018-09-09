# Build a 2-nodes Kubernetes Cluster
This is a demo project to record the process of how to build Kubernetes cluster on CentOS. 

## Prepare nodes

* Nodes design

| Hostname | IP Address | Note|
| --- | --- |
| docker101.fen9.li | 192.168.200.101 |	master node |
| docker102.fen9.li | 192.168.200.102 |	worker node |

* Ensure 2 nodes can see each other
```
[root@docker101 ~]# tail -2 /etc/hosts
192.168.200.101   docker101.fen9.li docker101
192.168.200.102   docker102.fen9.li docker102
[root@docker101 ~]#
```

### Setup docker on 2 nodes
* Install docker and start/enable docker service by running following commands
```
wget https://download.docker.com/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
systemctl start docker
systemctl enable docker
```

* Change docker cgroup driver to systemd by following below process
```
[root@docker101 ~]# cp /usr/lib/systemd/system/docker.service /etc/systemd/system        
[root@docker101 ~]# vim /etc/systemd/system/docker.service
[root@docker101 ~]# diff /usr/lib/systemd/system/docker.service /etc/systemd/system/docker.service
12c12
< ExecStart=/usr/bin/dockerd
---
> ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd
[root@docker101 ~]#
...
systemctl daemon-reload                                              
systemctl restart docker                                             
```

*Ensure docker is running and its Cgroup Driver is systemd
```
[root@docker101 ~]# systemctl is-active docker
active
[root@docker101 ~]# docker info | grep -i cgroup
Cgroup Driver: systemd
[root@docker101 ~]# docker --version
Docker version 18.06.1-ce, build e68fc7a
[root@docker101 ~]# ip addr show docker0
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:fc:af:a0:ac brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
[root@docker101 ~]#
```

### Install Kubernetes

* Enable br_netfilter Kernel Module
```
modprobe br_netfilter
echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.conf 
echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.conf
...
[root@docker101 ~]# sysctl -p
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
[root@docker101 ~]#
```

* Disable SWAP
```
swapoff -a
...
[root@docker101 ~]# cp /etc/fstab /etc/fstab.orig
[root@docker101 ~]# vim /etc/fstab
[root@docker101 ~]# diff /etc/fstab /etc/fstab.orig
11c11
< #/dev/mapper/centos-swap swap  swap    defaults        0 0
---
> /dev/mapper/centos-swap swap  swap    defaults        0 0
[root@docker101 ~]#
```

* Create kubernetes repo
```
[root@docker101 ~]# vim /etc/yum.repos.d/kubernetes.repo
[root@docker101 ~]# cat /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
[root@docker101 ~]#
...
yum repolist
```

* Install Kubernetes
```
[root@docker101 ~]# yum install -y kubelet kubeadm kubectl
...
Installed:
  kubeadm.x86_64 0:1.11.2-0               kubectl.x86_64 0:1.11.2-0               kubelet.x86_64 0:1.11.2-0

Dependency Installed:
  cri-tools.x86_64 0:1.11.0-0           kubernetes-cni.x86_64 0:0.6.0-0           socat.x86_64 0:1.7.3.2-2.el7

Complete!
[root@docker101 ~]#
...
systemctl reboot
systemctl start kubelet
systemctl enable kubelet
```

### Configure firewall rule on nodes
* On Kubernetes master node
```
[root@docker101 ~]# vim /etc/firewalld/services/kubernetes-master.xml
[root@docker101 ~]# cat /etc/firewalld/services/kubernetes-master.xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>kubernetes manager</short>
  <description>kubernetes manager </description>
  <port protocol="tcp" port="2379-2380"/>
  <port protocol="tcp" port="6443"/>
  <port protocol="tcp" port="10250-10252"/>
  <port protocol="tcp" port="10255"/>
</service>
[root@docker101 ~]# 
...
firewall-cmd --permanent --add-service={http,https,kubernetes-master} 
firewall-cmd --reload
...
[root@docker101 ~]# firewall-cmd --list-service
ssh dhcpv6-client kubernetes-master http https
[root@docker101 ~]#
```

* On Kubernetes worker node
```
[root@docker102 ~]# vim /etc/firewalld/services/kubernetes-worker.xml
[root@docker102 ~]# cat /etc/firewalld/services/kubernetes-worker.xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>kubernetes worker</short>
  <description>kubernetes worker</description>
  <port protocol="tcp" port="10250"/>
  <port protocol="tcp" port="10255"/>
  <port protocol="tcp" port="30000-32767"/>
</service>
[root@docker102 ~]# 
...
firewall-cmd --permanent --add-service=kubernetes-worker
firewall-cmd --reload
...
[root@docker102 ~]# firewall-cmd --list-service
ssh dhcpv6-client kubernetes-worker
[root@docker102 ~]#
```







