# kubernetes-via-kubeadm
How to deploy a Kubernetes master and node on a single Fedora host via kubeadm.    
  
By default it is not possible (and not recommanded) to run applications on `master` node(s). In case you have only one host you could deploy a Kubernetes cluster on one host via `kubeadm`. Basicly this means that all the components needs to be installed on the same host and that the `master` should be configured `schedulable`.  
Down here you find the steps to configure a Fedora hosts as a single cluster hosts.  
  
## Prerequisites 
One host with:   
* Fedora 27 (or higher) installed 
* Docker 1.13 (or higher) installed  
  
## Install the kubeadm, kubectl and kubelet packages  
```
[root@nuc ~]# dnf install kubernetes-kubeadm kubernetes kubernetes-client
[root@nuc ~]# systemctl enable kubelet && systemctl start kubelet
[root@nuc ~]# setenforce 0
```
note: `SELINUX` is not yet support with `kubeadm`, so it must be turned off!  
  
## Open firewall ports for Kubernetes. 
Since we have only one host that we use as `master` and `node` we apply both commands to our host.  
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
   
## Disable swapping: 
```
swapoff -a
```
Swapping is not supported by `kubelet`.
  
## Create the cluster  
I choose to use `flannel` as network layer. This means:  
* kubeadm needs to know a network CIDR: `--pod-network-cidr=10.244.0.0/16`
* `net.bridge.bridge-nf-call-iptables` needs to be set to 1.

```
[root@nuc ~]# sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1

[root@nuc ~]# kubeadm init --pod-network-cidr=10.244.0.0/16
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
[apiclient] All control plane components are healthy after 25.502030 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node nuc.bachstraat20 as master by adding a label and a taint
[markmaster] Master nuc.bachstraat20 tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: e22eb5.9a9d3e3ce38eab32
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

  kubeadm join --token e22eb5.9a9d3e3ce38eab32 192.168.11.100:6443 --discovery-token-ca-cert-hash sha256:cb02a2c269453a25385106381da185e47e24f1254543b0c3cea0005af0cec1f1

```
Read on, you're not yet there... 

## Make sure kubectl works  
For regular users:  
```
[some-user@nuc ~]# mkdir -p $HOME/.kube
[some-user@nuc ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
``` 
Alternatively, if you are the root user, you could run this:  
```
[root@nuc ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
```  
Now `kubectl` works, but the node is `NotReady`:  
```
[root@nuc ~]# kubectl get nodes
NAME               STATUS     ROLES     AGE       VERSION
nuc.bachstraat20   NotReady   master    1m        v1.9.1
```
  
## Deploy a pod network  
In order to enable connectivity between pods, we deploy `flannel`:  
```
[root@nuc ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
clusterrole "flannel" created
clusterrolebinding "flannel" created
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
```
Now the node reports `ready`:  
```
[root@nuc ~]# kubectl get nodes
NAME               STATUS    ROLES     AGE       VERSION
nuc.bachstraat20   Ready     master    8m        v1.9.1
```
 
## Check the node
```
[root@nuc ~]# kubectl describe nodes
Name:               nuc.bachstraat20
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=nuc.bachstraat20
                    node-role.kubernetes.io/master=
Annotations:        flannel.alpha.coreos.com/backend-data={"VtepMAC":"1a:50:44:0f:e1:6b"}
                    flannel.alpha.coreos.com/backend-type=vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager=true
                    flannel.alpha.coreos.com/public-ip=192.168.11.100
                    node.alpha.kubernetes.io/ttl=0
                    volumes.kubernetes.io/controller-managed-attach-detach=true
Taints:             node-role.kubernetes.io/master:NoSchedule
CreationTimestamp:  Sat, 24 Mar 2018 08:20:46 +0100
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  OutOfDisk        False   Sat, 24 Mar 2018 08:30:47 +0100   Sat, 24 Mar 2018 08:20:38 +0100   KubeletHasSufficientDisk     kubelet has sufficient disk space available
  MemoryPressure   False   Sat, 24 Mar 2018 08:30:47 +0100   Sat, 24 Mar 2018 08:20:38 +0100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sat, 24 Mar 2018 08:30:47 +0100   Sat, 24 Mar 2018 08:20:38 +0100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  Ready            True    Sat, 24 Mar 2018 08:30:47 +0100   Sat, 24 Mar 2018 08:29:07 +0100   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.11.100
  Hostname:    nuc.bachstraat20
Capacity:
 cpu:     4
 memory:  16351108Ki
 pods:    110
Allocatable:
 cpu:     4
 memory:  16248708Ki
 pods:    110
System Info:
 Machine ID:                 ccb68e77d17e47e6b23fca3120ef55f4
 System UUID:                E820CB80-34D4-11E1-B35F-C03FD562BB77
 Boot ID:                    d0addd54-4b85-43d8-b3ec-da8d0362370e
 Kernel Version:             4.15.10-300.fc27.x86_64
 OS Image:                   Fedora 27 (Workstation Edition)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://1.13.1
 Kubelet Version:            v1.9.1
 Kube-Proxy Version:         v1.9.1
PodCIDR:                     10.244.0.0/24
ExternalID:                  nuc.bachstraat20
Non-terminated Pods:         (7 in total)
  Namespace                  Name                                        CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                                        ------------  ----------  ---------------  -------------
  kube-system                etcd-nuc.bachstraat20                       0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-apiserver-nuc.bachstraat20             250m (6%)     0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-controller-manager-nuc.bachstraat20    200m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-dns-6f4fd4bdf-5t7sr                    260m (6%)     0 (0%)      110Mi (0%)       170Mi (1%)
  kube-system                kube-flannel-ds-c59vn                       0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-proxy-9p9qk                            0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                kube-scheduler-nuc.bachstraat20             100m (2%)     0 (0%)      0 (0%)           0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  810m (20%)    0 (0%)      110Mi (0%)       170Mi (1%)
Events:
  Type    Reason                   Age                From                          Message
  ----    ------                   ----               ----                          -------
  Normal  Starting                 10m                kubelet, nuc.bachstraat20     Starting kubelet.
  Normal  NodeAllocatableEnforced  10m                kubelet, nuc.bachstraat20     Updated Node Allocatable limit across pods
  Normal  NodeHasSufficientDisk    10m (x9 over 10m)  kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeHasSufficientDisk
  Normal  NodeHasSufficientMemory  10m (x7 over 10m)  kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    10m (x7 over 10m)  kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeHasNoDiskPressure
  Normal  Starting                 9m                 kube-proxy, nuc.bachstraat20  Starting kube-proxy.
  Normal  NodeReady                1m                 kubelet, nuc.bachstraat20     Node nuc.bachstraat20 status is now: NodeReady
```
The node looks `okay`, but since it is a master, it reports `Taints: node-role.kubernetes.io/master:NoSchedule`. So it can't schedule any pods.    
      
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
[root@nuc ~]#  kubectl get nodes -o wide
[root@nuc ~]# kubectl get nodes -o wide
NAME               STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE                          KERNEL-VERSION            CONTAINER-RUNTIME
nuc.bachstraat20   Ready     master    17m       v1.9.1    <none>        Fedora 27 (Workstation Edition)   4.15.10-300.fc27.x86_64   docker://1.13.1
```
  
## Install Kubernetes Dashboard  
```
[root@nuc ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```
To make the dashboard accessible via a `node-port` change type in the `kubernetes-dashboard` service: ClusterIP to type: NodePort and save file: 
```
[root@nuc ~]#  kubectl -n kube-system edit service kubernetes-dashboard
service "kubernetes-dashboard" edited

[root@nuc ~]# kubectl get service -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   2h
kubernetes-dashboard   NodePort    10.104.204.179   <none>        443:30717/TCP   30m
```
The Dashboard has been exposed on port 31707 (HTTPS). It is accessible from a web browser at: https://nuc.bachstraat20:31707  
Continue below to create credentieels to login to the kubernetes-dashboard.
  
## Create an admin user  
To create a cluster-admin user, use these files: [admin-user.yaml](https://github.com/tedsluis/kubernetes-via-kubeadm/blob/master/admin-user.yaml) and [admin-user-clusterrolebinding.yaml](https://github.com/tedsluis/kubernetes-via-kubeadm/blob/master/admin-user-clusterrolebinding.yaml):  
```
[root@nuc kubernetes-via-kubeadm]# kubectl create -f admin-user.yaml -n kube-system
serviceaccount "admin-user" created
[root@nuc kubernetes-via-kubeadm]# kubectl create -f admin-user-clusterrolebinding.yaml
clusterrolebinding "admin-user" created
```
  
To get the token for this `admin-user`:  
```
[root@nuc kubernetes-via-kubeadm]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') | grep ^token: | sed 's/token:[ ]*//'
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW1oNzIyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwNWM0ZDZmZC0yZjYyLTExZTgtYTMxNi1jMDNmZDU2MmJiNzciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.butKxegADx3JQvKpn9Prf7RL_SoxaEyi_scYOvXurm4BAwEj8zfC9a7djqQ9mBtd5cQHlljvMb-3qFc6UPOzAwR8fc5khk-nAkH-5XeahpT8WsyxMcKxqLuyAg8gh4ZtMKvBPk9kOWDtyRBzAeGkisbLxr43ecKO71F5G8D7HR2UGSm-x4Pvhq0uqj8GyIcHw902Ti92BPuBRf-SyTl8uDCQJSDkS5Tru5w0p82borNuVXd1mmDwuI87ApQrqXTY9rbJ61m8iTr0kKJBqw5bHAUAhxwAVtVEKQNNKT6cxWp1FlhHbNkM9bhcj1qj8bN1QCMjPWlWKj7NkPbbBAJthQ
``` 
You can use the token to login to the kubernetes-dashboard.  
  
[![Kubernetes Dashboard](https://raw.githubusercontent.com/tedsluis/kubernetes-via-kubeadm/master/img/kubernetes-dashboard.gif)](https://raw.githubusercontent.com/tedsluis/kubernetes-via-kubeadm/master/img/kubernetes-dashboard.gif)
  
## Deploying Prometheus 

```
[root@nuc ~]# kubectl create namespace prometheus
namespace "prometheus" created

[root@nuc ~]# kubectl -n prometheus create -f prometheus-sa.yaml 
serviceaccount "prometheus" created

[root@nuc ~]# kubectl -n prometheus create -f prometheus-sa-clusterrolebinding.yaml
clusterrolebinding "prometheus" created

[root@nuc ~]# wget -O prometheus-config.yaml https://raw.githubusercontent.com/prometheus/prometheus/master/documentation/examples/prometheus-kubernetes.yml

[root@nuc ~]# kubectl create configmap prometheus-config --from-file=prometheus-config.yaml -n prometheus
configmap "prometheus-config" created

[root@nuc ~]# kubectl create -f prometheus-deployment.yaml -n prometheus
deployment "prometheus" created

[root@nuc ~]# kubectl -n prometheus logs prometheus-7c7b9585f9-5lczv
level=info ts=2018-03-25T15:25:47.849051693Z caller=main.go:220 msg="Starting Prometheus" version="(version=2.2.1, branch=HEAD, revision=bc6058c81272a8d938c05e75607371284236aadc)"
level=info ts=2018-03-25T15:25:47.849106663Z caller=main.go:221 build_context="(go=go1.10, user=root@149e5b3f0829, date=20180314-14:15:45)"
level=info ts=2018-03-25T15:25:47.84913756Z caller=main.go:222 host_details="(Linux 4.15.10-300.fc27.x86_64 #1 SMP Thu Mar 15 17:13:04 UTC 2018 x86_64 prometheus-7c7b9585f9-5lczv (none))"
level=info ts=2018-03-25T15:25:47.849165208Z caller=main.go:223 fd_limits="(soft=1048576, hard=1048576)"
level=info ts=2018-03-25T15:25:47.852738361Z caller=main.go:504 msg="Starting TSDB ..."
level=info ts=2018-03-25T15:25:47.852769276Z caller=web.go:382 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2018-03-25T15:25:47.859354423Z caller=main.go:514 msg="TSDB started"
level=info ts=2018-03-25T15:25:47.859412609Z caller=main.go:588 msg="Loading configuration file" filename=/var/lib/prometheus/prometheus-config.yaml
level=info ts=2018-03-25T15:25:47.861036136Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.861868278Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.862872662Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.863871661Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.865287076Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.86634838Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.867490787Z caller=kubernetes.go:191 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2018-03-25T15:25:47.868484361Z caller=main.go:491 msg="Server is ready to receive web requests."

[root@nuc ~]# kubectl -n prometheus create -f prometheus-service.yaml 
service "prometheus" created
```


  
## Tear down the cluster 
Perform these steps to desolve the cluster completly.  
```
[root@nuc ~]# kubectl drain nuc.bachstraat20 --delete-local-data --force --ignore-daemonsets
node "nuc.bachstraat20" cordoned
WARNING: Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: etcd-nuc.bachstraat20, kube-apiserver-nuc.bachstraat20, kube-controller-manager-nuc.bachstraat20, kube-scheduler-nuc.bachstraat20; Ignoring DaemonSet-managed pods: kube-proxy-5ksfm
pod "kube-dns-6f4fd4bdf-kcl6k" evicted
node "nuc.bachstraat20" drained
[root@nuc ~]# kubectl delete node nuc.bachstraat20
node "nuc.bachstraat20" deleted
[root@nuc ~]# kubeadm reset
[preflight] Running pre-flight checks.
[reset] Stopping the kubelet service.
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Removing kubernetes-managed containers.
[reset] Deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes /var/lib/etcd]
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
```
  
## Documentation  
* [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)  
* [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)  
* [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init)  
* [Flannel - kubeadm](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md)  
