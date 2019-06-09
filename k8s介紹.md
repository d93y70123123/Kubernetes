# Kubernetes
![alt K8S][id]

[id]: https://github.com/d93y70123123/Kubernetes/blob/master/kubernetes-logo.png "logo"  
Kubernetes（常簡稱為K8s），是用於自動部署、擴展和管理容器化（containerized）應用程式的開源系統。主要提供「跨主機集群的自動部署、擴展以及運行應用程式容器的平台」。  
# K8S的架構
![alt K8S-archtecture](https://github.com/d93y70123123/Kubernetes/blob/master/kubernetes-archtecture.jpg "archtecture")  
Master負責協調叢集，其中主要腳色:  
* API server : 所有的 K8s 操作都是透過 API Server。   
* etcd : 負責與 API Server 溝通。  
* k8s-schedule : 進行調度，分配最適合的 Node 來執行 Container。
* controller-manager : 負責所有的控制功能。  

Node負責執行應用程式，其中主要腳色:  
* kubelet : 安裝於每一個 Node 上，負責與 API Server 溝通。  
* kube-proxy : 提供給 kubectl 或 kubelet 進行 API Server 連線 。  
* container runtime : 就是容器啦，像是:docker。  

# 安裝K8S
### 安裝注意事項
* CPU至少兩個
* 記憶體至少2G
* SWAP必須禁用
* cluster內的網路必須接通
***
1. 新增容器庫
```bash
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
```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
5. 啟動kubelet
```bash
systemctl start kubelet
systemctl enable kubelet
```
6. 確保網路不會被iptables略過
```bash
vim /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

sysctl --system
```
*這邊因為還沒初始化的關係，kubelet會啟動失敗  

* 重開機吧 reboot

## 簡單介紹k8s的兩個主要指令
* kubeadm：建立節點必備，可以快速幫助你建立 Master 以及增加 Node
* kubectl：負責溝通整個叢集的工具，可以利用它管理以及建立 k8s 的最小單位 "pod"等功能。

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
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
4. 為所有的POD建立可以互相溝通的網路
```
*這邊使用flannel的CNI
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml
```  
## Node建立
安裝docker 
```bash
wget https://download.docker.com/linux/centos/docker-ce.repo -P /etc/yum.repos.d/
yum install docker-ce docker-ce-cli containerd.io
systetmctl start docker
systetmctl enable docker
```
重複安裝的步驟，可以將下面的指令段落做成腳本執行  
`*每個節點都要做` 
```bash
hostnamectl set-hostname k8s1
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

swapoff -a

sed -i 's/\/dev\/mapper\/centos-swap/#\/dev\/mapper\/centos-swap/g' /etc/fstab

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## 部屬環境
pod：最小單位，可用來創建和部署的單元。
pod可以單獨運行一個容器也可以運行多個容器，運行多個容器時共用 "IP" 以及 "儲存的資源"。  
![alt k8s](https://github.com/d93y70123123/Kubernetes/blob/master/module_03_nodes.svg "pods")  
pod的可以用指令建立或是yaml、json建立，這邊用yaml檔來建立pod：  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-kubernetes
  labels:
    app: webserver
spec:
  containers:
  - name: my-k8s
    image: nginx
```
1. 建立 pod
```bash
$ kubectl create -f myk8s.yaml
...建立成功後
pod/hello-kubernetes created
```
2. 確認 pod 有沒有活著
```bash
$ kubectl get pods

NAME               READY   STATUS    RESTARTS   AGE
hello-kubernetes   1/1     Running   0          32s
```
3. 讓外部與 pod 連線  
```
$ kubectl expose pods hello-kubernetes --type="NodePort" --port=80

service/hello-kubernetes exposed
```
4. 查看、指令對外開啟的 port
```
$ kubectl get service

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-kubernetes   NodePort    10.102.55.116   <none>        80:32136/TCP   74s
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP        8d

$ kubectl edit service hello-kubernetes
... 接著會進入編輯模式，跟 vim 大同小異
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2019-06-09T16:26:55Z"
  labels:
    app: webserver
  name: hello-kubernetes
  namespace: default
  resourceVersion: "972631"
  selfLink: /api/v1/namespaces/default/services/hello-kubernetes
  uid: 65506bf5-8ad3-11e9-bdb0-14dae957dc6b
spec:
  clusterIP: 10.102.55.116
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32136
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webserver
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}

```
<p style="color:red;">asdasd<p>

### 參考資料 ###
* K8S建立：https://kubernetes.io/docs/setup/independent/install-kubeadm/  

