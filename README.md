# Build a 2-nodes Kubernetes Cluster
A demo project on how to build Kubernetes cluster and deploy a wordpress app. 

## Prepare nodes

* Nodes design

| Hostname | IP Address | Note|
| ---- | ---- | ---- |
| kube31.fen9.li | 192.168.200.31 |	master node |
| kube32.fen9.li | 192.168.200.32 |	worker node |

* Ensure 2 nodes can ping each other

### Setup docker on 2 nodes
* Install docker and start/enable docker service by running following commands
```
wget https://download.docker.com/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
systemctl start docker
systemctl enable docker
```

### Prepare Kubernetes installation
* Disable selinux
```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

* Disable firewalld
```
systemctl stop firewalld
systemctl disable firewalld
```

* Enable net bridge 
```
modprobe br_netfilter
echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.d/bridge-nf-call.conf
echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.d/bridge-nf-call.conf

[root@localhost ~]# sysctl -p /etc/sysctl.d/bridge-nf-call.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
[root@localhost ~]#
```

* Disable swap
```
swapoff -a
sed -i '/swap/d' /etc/fstab
```

### Install Kubernetes

* Create kubernetes repo
```
cat <<EOT > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOT
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
systemctl enable kubelet
systemctl reboot
```

## Setup Kubernetes cluster
### On Kubernetes master node
* Initiate cluster
```
kubeadm init --apiserver-advertise-address=192.168.200.31 --pod-network-cidr=10.10.0.0/16
...
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

* Ensure Kubernetes is up and running
```
[fli@docker31 ~]$ kubectl version
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:17:28Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:08:19Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
[fli@docker31 ~]$

[fli@docker31 ~]$ kubectl get nodes
NAME                STATUS    ROLES     AGE       VERSION
kube1.fen9.li   Ready     master    49m       v1.11.2
[fli@docker31 ~]$

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
