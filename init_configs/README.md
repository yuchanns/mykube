# Install Kubernetes with Kubeadm
## Common Section
This section shows what should be installed on **all hosts** (both master and nodes) that will join the cluster.
### Install Containerd
Let's make some preparation and install containerd.
```sh
# make some prepare
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# specific for Chinese users
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# make cache
sudo yum makecache fast
# configures before install containerd
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# install containerd
sudo yum install -y containerd.io
```
Then genrate config file and make some modifications.
```sh
# generate config file
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml

# vim to edit file
vim /etc/containerd/config.toml
# search and replace sandbox_image
[plugins."io.containerd.grpc.v1.cri"]
    sandbox_image="registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2"
# search and set to use systemd as cgroup
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
# search and set repository mirrors
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."*"]
    endpoint = ["https://[your mirror].mirror.aliyuncs.com"] # You can get your own mirror from [aliyun container service](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors).
# save vim edition
```
Finally set containerd auto start after launch and restart it.
```sh
sudo systemctl enable containerd
sudo systemctl restart containerd
# check if status is ok
sudo systemctl status containerd
```
### Install Tools of Kubernetes
```sh
# set the repository using aliyun kubernetes
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes Repository
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF
# search and decide the version to install
yum search kubelet kubeadm kubectl --showduplicates
# install specific version of kubernetes tools
sudo yum install -y kubelet-1.20.6 kubeadm-1.20.6 kubectl-1.20.6 --disableexcludes=kubernetes

# config kubelet cgroup
cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
EOF

# config CRI
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

# set containerd auto start after launch and start it
systemctl enable kubelet && systemctl start kubelet
```
### Stop Firewall and Disable Swap
```sh
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo setenforce 0

sudo sed -i '/swap/ s/^/#/' /etc/fstab && sudo swapoff -a && sysctl -w vm.swappiness=0
```
## Master Section
This section shows what to do on **master host**.

First, pull image from `init-config.yaml`.
```sh
sudo kubeadm config images pull --config=init-config.yaml
```
Then, init from `init-config.yaml`.
```sh
sudo kubeadm init --config=init-config.yaml
# Your Kubernetes control-plane has initialized successfully!
# 
# To start using your cluster, you need to run the following as a regular user:

#   mkdir -p $HOME/.kube
#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#   sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 
# Alternatively, if you are the root user, you can run:
# 
#   export KUBECONFIG=/etc/kubernetes/admin.conf
# 
# You should now deploy a pod network to the cluster.
# Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
#   https://kubernetes.io/docs/concepts/cluster-administration/addons/

# Then you can join any number of worker nodes by running the following on each as root:

# kubeadm join 10.9.11.4:6443 --token hx8f96.5yh392lco45qkd94 \
#     --discovery-token-ca-cert-hash sha256:cb21467e644298e460e0db036e0472bdce9d87bb5beba6e1453d9f3efa8ebe48
```
Notice that you get a token and a ip address to join, save them.

Run the following as a regular user:
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Installing weave for net policy:
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Now the master is finished.
## Node Section
This section shows what to do on **node hosts**.

Create a `join-config.yaml` with given token and ip address:
```sh
cat > join-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.9.11.4:6443
    token: hx8f96.5yh392lco45qkd94
    unsafeSkipCAVerification: true
  tlsBootstrapToken: hx8f96.5yh392lco45qkd94
EOF
```
Then apply to join the cluster:
```sh
sudo kubeadm join --config=join-config.yml
```

## Success
Wait minutes then check nodes and pods.
```sh
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k8s-master   Ready    control-plane,master   10h     v1.20.6
k8s-node1    Ready    <none>                 10h     v1.20.6
k8s-node2    Ready    <none>                 10h     v1.20.6
k8s-node3    Ready    <none>                 4h37m   v1.20.6
[root@k8s-master ~]# kubectl get pods --all-namespaces
NAMESPACE              NAME                                                  READY   STATUS    RESTARTS   AGE
kube-system            coredns-7f89b7bc75-75kp2                              1/1     Running   0          10h
kube-system            coredns-7f89b7bc75-kr9fx                              1/1     Running   0          10h
kube-system            etcd-k8s-master                                       1/1     Running   0          10h
kube-system            kube-apiserver-k8s-master                             1/1     Running   0          10h
kube-system            kube-controller-manager-k8s-master                    1/1     Running   0          10h
kube-system            kube-proxy-67fvq                                      1/1     Running   1          10h
kube-system            kube-proxy-974zd                                      1/1     Running   0          10h
kube-system            kube-proxy-d7fvl                                      1/1     Running   1          10h
kube-system            kube-proxy-nllvf                                      1/1     Running   1          4h39m
kube-system            kube-scheduler-k8s-master                             1/1     Running   0          10h
kube-system            weave-net-7ktvr                                       2/2     Running   2          10h
kube-system            weave-net-g7vrd                                       2/2     Running   3          10h
kube-system            weave-net-sjbpf                                       2/2     Running   1          10h
kube-system            weave-net-zv7mz                                       2/2     Running   2          4h39m
```
