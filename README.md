# Kubernetes
![alt K8S][id]

[id]: https://github.com/d93y70123123/Kubernetes/blob/master/kubernetes-logo.png "123"  
Kubernetes（常簡稱為K8s），是用於自動部署、擴展和管理容器化（containerized）應用程式的開源系統。主要提供「跨主機集群的自動部署、擴展以及運行應用程式容器的平台」。  
# K8S的架構
Master:
Node:  
# 安裝K8S
### 安裝注意事項
* CPU至少兩個
* 記憶體至少2G
* SWAP必須禁用
* cluster內的網路必須接通
***
1. 新增容器庫
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
```
2. 將selinux設置成permissive  
  查看selinux狀態 : getenforce
3. yum update
4. 安裝 kubelet kubeadm kubectl
```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
5. 啟動kubelet
```
systemctl start kubelet
systemctl enable kubelet
```
6. 確保網路不會被iptables略過
```
vim /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

sysctl --system
```
*這邊因為還沒初始化的關係，kubelet會啟動失敗  

* 重開機吧 reboot

# 建立叢集
## Master建立  
1. Master節點初始化
```
kubeadm init --pod-network-cidr=10.244.0.0/16
..
....
.....
初始化過程中若出現下面這段文字，代表初始化成功
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
2. 先記下要加入叢集的的金鑰
```
如果忘記，可以用下面的方法查
kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
3. 為Master節點建立幾個檔案
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
4. 為所有的POD建立可以互相溝通的網路
```
這邊使用flannel的CNI
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml
```  
## Node建立


### 參考資料 ###
* K8S建立：https://kubernetes.io/docs/setup/independent/install-kubeadm/  

