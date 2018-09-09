# Build a 2-nodes Kubernetes Cluster
This is a demo project to record the process of how to build Kubernetes cluster on CentOS. 

## Prepare nodes

* Nodes design

| Hostname | IP Address | Note|
| ---- | ---- | ---- |
| docker101.fen9.li | 192.168.200.101 |	master node |
| docker102.fen9.li | 192.168.200.102 |	worker node |

* Ensure 2 nodes can ping each other

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
[root@docker101 ~]# docker info | grep -i cgroup
Cgroup Driver: cgroupfs
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

* Ensure docker is running and its Cgroup Driver is systemd
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

## Setup Kubernetes cluster
### On Kubernetes master node
* Initiate cluster
```
[root@docker101 ~]# kubeadm init --apiserver-advertise-address=192.168.200.101 --pod-network-cidr=10.10.0.0/16
[init] using Kubernetes version: v1.11.2
[preflight] running pre-flight checks
        [WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
I0907 23:18:18.083378   27894 kernel_validator.go:81] Validating kernel version
I0907 23:18:18.083561   27894 kernel_validator.go:96] Validating kernel config
        [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.06.1-ce. Max validated version: 17.03
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [docker101.fen9.li kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.200.101]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [docker101.fen9.li localhost] and IPs [127.0.0.1 ::1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [docker101.fen9.li localhost] and IPs [192.168.200.101 127.0.0.1 ::1]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 58.007160 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.11" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node docker101.fen9.li as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node docker101.fen9.li as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "docker101.fen9.li" as an annotation
[bootstraptoken] using token: eam0t9.lb01oj8vt7ms4cja
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node as root:

  kubeadm join 192.168.200.101:6443 --token eam0t9.lb01oj8vt7ms4cja --discovery-token-ca-cert-hash sha256:e41523ead31df5ec4fa69b5616f502e29e9faf53a6dc3265e8f97376d30799cd

[root@docker101 ~]# exit

[fli@docker101 ~]$ 
...
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

* Ensure Kubernetes is up and running
```
[fli@docker101 ~]$ kubectl version
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:17:28Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:08:19Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
[fli@docker101 ~]$

[fli@docker101 ~]$ kubectl get nodes
NAME                STATUS    ROLES     AGE       VERSION
docker101.fen9.li   Ready     master    49m       v1.11.2
[fli@docker101 ~]$

[fli@docker101 ~]$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                        READY     STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-ps5vv                    1/1       Running   0          43m
kube-system   coredns-78fcdf6894-vfr75                    1/1       Running   0          43m
kube-system   etcd-docker101.fen9.li                      1/1       Running   0          42m
kube-system   kube-apiserver-docker101.fen9.li            1/1       Running   0          42m
kube-system   kube-controller-manager-docker101.fen9.li   1/1       Running   0          43m
kube-system   kube-flannel-ds-amd64-mh7sg                 1/1       Running   0          3m
kube-system   kube-proxy-dtq8f                            1/1       Running   0          43m
kube-system   kube-scheduler-docker101.fen9.li            1/1       Running   0          42m
[fli@docker101 ~]$
``` 

### On Kubernetes worker node
* Join cluster and ensure kubelet service is up
```
[fli@docker102 ~]$ sudo kubeadm join 192.168.200.101:6443 --token eam0t9.lb01oj8vt7ms4cja --discovery-token-ca-cert-hash sha256:e41523ead31df5ec4fa69b5616f502e29e9faf53a6dc3265e8f97376d30799cd
[sudo] password for fli:
[preflight] running pre-flight checks
        [WARNING RequiredIPVSKernelModulesAvailable]: the IPVS proxier will not be used, because the following required kernel modules are not loaded: [ip_vs_rr ip_vs_wrr ip_vs_sh ip_vs] or no builtin kernel ipvs support: map[ip_vs_sh:{} nf_conntrack_ipv4:{} ip_vs:{} ip_vs_rr:{} ip_vs_wrr:{}]
you can solve this problem with following methods:
 1. Run 'modprobe -- ' to load missing kernel modules;
2. Provide the missing builtin kernel ipvs support

I0908 09:19:49.513761   13032 kernel_validator.go:81] Validating kernel version
I0908 09:19:49.513952   13032 kernel_validator.go:96] Validating kernel config
        [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.06.1-ce. Max validated version: 17.03
[discovery] Trying to connect to API Server "192.168.200.101:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.200.101:6443"
[discovery] Requesting info from "https://192.168.200.101:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.200.101:6443"
[discovery] Successfully established connection with API Server "192.168.200.101:6443"
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.11" ConfigMap in the kube-system namespace
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[preflight] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "docker102.fen9.li" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
[fli@docker102 ~]$

[fli@docker102 ~]$ sudo systemctl is-active kubelet
[sudo] password for fli:
active
[fli@docker102 ~]$
```

* Double check on Kubernetes master node
```
[fli@docker101 ~]$ kubectl get nodes
NAME                STATUS    ROLES     AGE       VERSION
docker101.fen9.li   Ready     master    10h       v1.11.2
docker102.fen9.li   Ready     <none>    4m        v1.11.2
[fli@docker101 ~]$

[fli@docker101 ~]$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                        READY     STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-ps5vv                    1/1       Running   1          10h
kube-system   coredns-78fcdf6894-vfr75                    1/1       Running   1          10h
kube-system   etcd-docker101.fen9.li                      1/1       Running   1          10h
kube-system   kube-apiserver-docker101.fen9.li            1/1       Running   2          10h
kube-system   kube-controller-manager-docker101.fen9.li   1/1       Running   2          10h
kube-system   kube-flannel-ds-amd64-mh7sg                 1/1       Running   2          9h
kube-system   kube-flannel-ds-amd64-nl9dg                 1/1       Running   2          5m
kube-system   kube-proxy-dtq8f                            1/1       Running   1          10h
kube-system   kube-proxy-wzgv4                            1/1       Running   0          5m
kube-system   kube-scheduler-docker101.fen9.li            1/1       Running   1          10h
[fli@docker101 ~]$ 
```

## Where to go next
* Build more worker nodes and join the cluster
* Start to deploy application
