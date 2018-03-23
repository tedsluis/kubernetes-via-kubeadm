# kubernetes-via-kubeadm
How to install a Kubernetes master and node on a single Fedora host.  
  
By default it is not possible (and not recommanded) to run applications on `master` node(s). In case you have only one host you could deploy a Kubernetes cluster on one host via `kubeadm`.  
Basicly this means that all the components needs to be installed on the same host and that the `master` should be configured `schedulable`.  
Down here you find the steps to configure a Fedora hosts as a single cluster hosts.  
  
## Prerequisites  
* Fedora 27  
* Docker 1.13 installed  
  
## Install the kubeadm, kubectl and kubelet packages  
```
[root@nuc ~]# dnf install kubernetes-kubeadm kubernetes kubernetes-client
[root@nuc ~]# systemctl enable kubelet && systemctl start kubelet
[root@nuc ~]# setenforce 0
```
note: `SELINUX` is not yet support with `kubeadm`, so turn it off!  
  
## Open firewall ports for Kubernetes. 
On the master(s):  
```
for PORT in 6443 2379 2380 10250 10251 10252 10255; do
	firewall-cmd --permanent --add-port=${PORT}/tcp --permanent
done
firewall-cmd --reload
firewall-cmd --list-ports
```  
On the node(s): 
```
for PORT in 10250 10255 30000-32767; do
	firewall-cmd --permanent --add-port=${PORT}/tcp --permanent
done
firewall-cmd --reload
firewall-cmd --list-ports
``` 
note: Since we have only one host that we use as `master` and `node` we apply both commands to our host.  
   
## Disable swapping: 
```
swapoff -a
```
It is not supported by `kubelet`.
  
## Create the cluster
```
[root@nuc ~]# kubeadm init
[init] Using Kubernetes version: v1.9.6
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
	[WARNING FileExisting-crictl]: crictl not found in system path
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [nuc.bachstraat20 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.11.100]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 49.003056 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node nuc.bachstraat20 as master by adding a label and a taint
[markmaster] Master nuc.bachstraat20 tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: 0b0c9a.4b51e2cad20a8e7f
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 0b0c9a.4b51e2cad20a8e7f 192.168.11.100:6443 --discovery-token-ca-cert-hash sha256:731387581a19e69a3a520abf6c634f89f4bbf971bde2ed99c9736b5f1585e8d7
```
Read on, you're not yet there...  
  
## Fix kubelet error: network plugin is not ready: cni config uninitialized
Check the node status:  
```
[root@nuc ~]# kubectl describe nodes nuc.bachstraat20 

<skip some lines>
CreationTimestamp:  Fri, 23 Mar 2018 16:01:21 +0100
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  OutOfDisk        False   Fri, 23 Mar 2018 16:27:04 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletHasSufficientDisk     kubelet has sufficient disk space available
  MemoryPressure   False   Fri, 23 Mar 2018 16:27:04 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Fri, 23 Mar 2018 16:27:04 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  Ready            False   Fri, 23 Mar 2018 16:27:04 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```
Remove the `remove $KUBELET_NETWORK_ARGS` environment entry from `/etc/systemd/system/kubelet.service.d/kubeadm.conf` and restart `kubelet`.  
```
vi /etc/systemd/system/kubelet.service.d/kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet.service
```
Now `kubelet' will report itself `ready`:    
```
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  OutOfDisk        False   Fri, 23 Mar 2018 17:36:14 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletHasSufficientDisk     kubelet has sufficient disk space available
  MemoryPressure   False   Fri, 23 Mar 2018 17:36:14 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Fri, 23 Mar 2018 17:36:14 +0100   Fri, 23 Mar 2018 16:01:12 +0100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  Ready            True    Fri, 23 Mar 2018 17:36:14 +0100   Fri, 23 Mar 2018 16:49:59 +0100   KubeletReady                 kubelet is posting ready status
```
      
## Make master node being able to schedule pods
By default it is not possible (and not very wise) to schedule pods on the master hosts, but if you just have only one host, you can make it `schedulable`.  
Remove `taints`, `effect:NoSchedule` and `key: node-role.kubernetes.io/master` from master node config.
```
[root@nuc ~]# kubectl edit node nuc.bachstraat20
spec:
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
```
Now the host is ready to schedule pods:  
```
[root@nuc ~]#  kubectl get nodes
NAME               STATUS    ROLES     AGE       VERSION
nuc.bachstraat20   Ready     master    1h        v1.9.1
```
   
## Redeploy a new cluster
```
[root@nuc ~]#  kubeadm reset
[root@nuc ~]#  kubeadm init
```
  
## Documentation  
* [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)  
* [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)  

