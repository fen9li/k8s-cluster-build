# Build Kubernetes Cluster and Deploy WordPress 
A demo project on how to build Kubernetes cluster and deploy a wordpress app. 

## Prepare nodes

* Nodes design

| Hostname | IP Address | Note|
| ---- | ---- | ---- |
| kube31.fen9.li | 192.168.200.31 |	master node |
| kube32.fen9.li | 192.168.200.32 |	worker node |

* Ensure all nodes in the cluster can ping each other

### Setup docker on all nodes
* Install docker and start/enable docker service by running following commands
```
wget https://download.docker.com/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
systemctl start docker
systemctl enable docker
```

### Prepare nodes for Kubernetes installation
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
yum install -y kubelet kubeadm kubectl
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
[fli@kube31 ~]$ kubectl version
Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:17:28Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.2", GitCommit:"bb9ffb1654d4a729bb4cec18ff088eacc153c239", GitTreeState:"clean", BuildDate:"2018-08-07T23:08:19Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
[fli@kube31 ~]$

[fli@kube31 ~]$ kubectl get nodes
NAME             STATUS    ROLES     AGE       VERSION
kube31.fen9.li   Ready     master    49m       v1.11.2
[fli@kube31 ~]$
``` 

### On Kubernetes worker node
* In case join token losted
```
[fli@kube31 k8s-cluster-build]$ kubeadm token create --print-join-command
kubeadm join 192.168.200.31:6443 --token 0a8h56.hf12gyt8b8n3kmyl --discovery-token-ca-cert-hash sha256:74a6c4df0b72ae505bcf9b8989b88bdbaf35b6e1ca7dd43d7bab81a76a9a5b82
[fli@kube31 k8s-cluster-build]$
```

* Join cluster and ensure kubelet service is up
```
sudo kubeadm join 192.168.200.31:6443 --token 0a8h56.hf12gyt8b8n3kmyl --discovery-token-ca-cert-hash sha256:74a6c4df0b72ae505bcf9b8989b88bdbaf35b6e1ca7dd43d7bab81a76a9a5b82

[fli@kube32 ~]$ sudo systemctl is-active kubelet
[sudo] password for fli:
active
[fli@kube32 ~]$
```

* Double check on Kubernetes master node
```
[fli@kube31 ~]$ kubectl get nodes
NAME             STATUS    ROLES     AGE       VERSION
kube31.fen9.li   Ready     master    10h       v1.11.2
kube32.fen9.li   Ready     <none>    4m        v1.11.2
[fli@kube31 ~]$

[fli@kube31 k8s-cluster-build]$ kubectl get all -n kube-system
NAME                                         READY   STATUS     RESTARTS   AGE
pod/coredns-576cbf47c7-x47gj                 1/1     Running    11         7d23h
pod/coredns-576cbf47c7-xv8wv                 1/1     Running    11         7d23h
pod/etcd-kube31.fen9.li                      1/1     Running    10         7d23h
pod/kube-apiserver-kube31.fen9.li            1/1     Running    15         7d23h
pod/kube-controller-manager-kube31.fen9.li   1/1     Running    14         7d23h
pod/kube-flannel-ds-amd64-l2jk5              1/1     Running    13         7d23h
pod/kube-flannel-ds-amd64-t7svm              1/1     Running    12         7d22h
pod/kube-proxy-4b52r                         1/1     Running    10         7d23h
pod/kube-proxy-f4lm6                         1/1     Running    10         7d22h
pod/kube-scheduler-kube31.fen9.li            1/1     Running    15         7d23h

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   7d23h

NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
daemonset.apps/kube-flannel-ds-amd64     2         2         2       3            2           beta.kubernetes.io/arch=amd64     7d23h
daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           beta.kubernetes.io/arch=arm       7d23h
daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           beta.kubernetes.io/arch=arm64     7d23h
daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           beta.kubernetes.io/arch=ppc64le   7d23h
daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           beta.kubernetes.io/arch=s390x     7d23h
daemonset.apps/kube-proxy                2         2         2       3            2           <none>                            7d23h

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   2         2         2            2           7d23h

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-576cbf47c7   2         2         2       7d23h
[fli@kube31 k8s-cluster-build]$
```

## Deploy WordPress on this k8s cluster
* Clone this repository to k8s master node
```
git clone -b develop git@github.com:fen9li/k8s-cluster-build.git
cd k8s-cluster-build/
```

* Tear down namespace 'wordpress' if it exists
```
kubectl delete ns wordpress
```

* Deploy the wordpress application to k8s cluster
```
kubectl create -f wp-namespace.yaml -f mariadb-deployment.yaml -f mariadb-service.yaml -f wordpress-deployment.yaml -f wordpress-service.yaml
```

* Check resources/objects in namespace wordpress

```
[fli@kube31 k8s-cluster-build]$ kubectl get all -n wordpress
NAME                             READY   STATUS    RESTARTS   AGE
pod/mysql-d7d5db6dd-wl8lm        1/1     Running   0          103s
pod/wordpress-748465d6cf-8gdv7   1/1     Running   0          103s

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/mysql       ClusterIP   10.107.247.169   <none>        3306/TCP       104s
service/wordpress   NodePort    10.101.166.40    <none>        80:31764/TCP   101s

NAME                        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql       1         1         1            1           104s
deployment.apps/wordpress   1         1         1            1           104s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-d7d5db6dd        1         1         1       104s
replicaset.apps/wordpress-748465d6cf   1         1         1       104s
[fli@kube31 k8s-cluster-build]$
```

## Check out this new wordpress installation 

> Test out below URL:

> http://192.168.200.32:31764

* ![Test out url 'http://192.168.200.32:31764'](images/k8s_01.png)

> Enter below and click 'Submit' button

> Database Name: wordpress

> Username: root

> Password: aStr0ngPassW0rd

> Database Host: mysql or mysql.wordpress.svc.cluster.local

* ![Enter below when prompt](images/k8s_02.png) 

> Once 'Run the installation' button can be clicked, it proves that the wordpress web pod can consume mysql service 

* ![wordpress web container can consume mysql service](images/k8s_03.png)

## Where to go next
* Build more worker nodes and join the cluster
