# Setting up Kubernetes cluster with kubeadm.
Simple step by step guide for setting up Kubernetes cluster with kubeadm consisiting of three master nodes, two worker nodes and one external load-balancer. Commands and brief descriptions suatable for the lates version of kubernetes (v1.25.3).

## Preparations of VMs.
Let's assume we have 5 virtual machines on the cloud (yandex cloud for instance) with full network connectivity between them and Ununtu 22.04 systems installed:
1. VM for load balancer:  
    - `lb-1` - `10.128.0.9`
2. Three VMs for master nodes:  
    - `k8s-mas-1` - `10.128.0.101`  
    - `k8s-mas-2` - `10.129.0.102`  
    - `k8s-mas-3` - `10.129.0.103`  
3. Two VMs for worker nodes:  
    - `k8s-work-1` - `10.128.0.110`  
    - `k8s-work-2` - `10.129.0.111`  

## 1. Install haproxy.
Do the following actions in `lb-1` terminal:
```
sudo apt-get update
sudo apt install -y haproxy
```

Supply config file of haproxy with the content provided in haproxy.cfg. Run you favourite editor (nano for example) and edit the content of haproxy.cfg:
```
sudo  nano /etc/haproxy/haproxy.cfg
```
Presumably you just need append these lines to your default haproxy.cfg file:

```
frontend kubernetes
        bind 10.129.0.9:6443
        mode tcp
        option tcplog
        default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
        mode tcp
        balance roundrobin
        option tcp-check
        server k8s-mas-1 10.129.0.101:6443 check fall 3 rise 2
        server k8s-mas-2 10.129.0.102:6443 check fall 3 rise 2
        server k8s-mas-3 10.129.0.103:6443 check fall 3 rise 2
```

Restart and enable haproxy then check the status of the service:
```
sudo systemctl restart haproxy
sudo systemctl enable haproxy
systemctl status haproxy.service
```

## 2. Prepare VM's.
The following step should be done on all systems exept load-balancer (`lb-1` in our case).

### 2.1 Disable Swap, load necessary kernel modules, set sysctl params required.
Update the apt package index:
```
sudo apt-get update
```

Disable swap and make it off-status permanent:
```
sudo swapoff -a && sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
free -m
```

Add additional kernel modules required:
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
Load modules into the kernel:
```
sudo modprobe overlay
sudo modprobe br_netfilter
```

Packets crossing a bridge are sent to iptables for processing because almost all K8s CNIs use iptables, set appropriate sysctl params: 
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Apply sysctl without rebooting:
```
sudo sysctl --system
```

### 2.2 Install docker.
Update the apt package index and install additional packages:
```
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
Add docker gpg key:
```
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
Setup repo:
```
echo \
 "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
 $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Now its time to install Docker engine itself. Run the following commands to install the latest version:
```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Enable docker and check its status:
```
sudo systemctl start docker && sudo systemctl enable docker
sudo systemctl status docker
```

Set Docker daemon to use systemd for the container’s cgroup managment:
```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Flush changes and restart docker:
```
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### 2.3 Build and install cri-dockerd.
Build and install cri-dockerd:
```
sudo -i
git clone https://github.com/Mirantis/cri-dockerd.git
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile

cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
exit
```

### 2.4 Install kubeadm, kubelet and kubectl.
Update the apt package index and install packages to use the Kubernetes repository:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
Download the public gpg key:
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
Add apt repo:
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update apt, install kubelet, kubeadm and kubectl, and  hold on their version:
```
sudo apt-get update
sudo apt-get install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 3.Initialize Kubernetes control-plane and install one of the pod network.
The following two steps should be performed on one of the master nodes, let it be `k8s-mas-1` system.

###  3.1 Run kubeadm with initial configuration manifest.
Run kubeadm with initial configuration manifest file provided in this repo. If you choose *k8s-conf.yaml* kubeadm will ask to convert it to the *apiVersion: v1beta3* format. If you choose *k8s-conf-new.yaml* no migration will be needed, btw *K8s-conf-new.yaml* file is a result of such migration done:
```
kubeadm config migrate --old-config k8s-conf.yaml --new-config k8s-conf-new.yaml
```

Just run the following command:
```
sudo kubeadm init --upload-certs --v=5 --config="k8s-conf.yaml"
```
or preferably:
```
sudo kubeadm init --upload-certs --v=5 --config="k8s-conf-new.yaml"
```

When kubeadm initialization completes you should recieve the following message:

> Your Kubernetes control-plane has initialized successfully!
> 
> To start using your cluster, you need to run the following as a regular user:
> 
>   mkdir -p $HOME/.kube
>   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
>   sudo chown $(id -u):$(id -g) $HOME/.kube/config
> 
> Alternatively, if you are the root user, you can run:
> 
>   export KUBECONFIG=/etc/kubernetes/admin.conf
> 
> You should now deploy a pod network to the cluster.
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>   https://kubernetes.io/docs/concepts/cluster-administration/addons/
> 
> You can now join any number of the control-plane node running the following command on each as root:
> 
>   kubeadm join 10.129.0.9:6443 --token *generated-token* \
>         --discovery-token-ca-cert-hash sha256:*generated-token-ca-cert-hash* \
>         --control-plane --certificate-key *generated-control-plane-certificate-key*
> 
> Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
> As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
> "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
> 
> Then you can join any number of worker nodes by running the following on each as root:
> 
> kubeadm join 10.129.0.9:6443 --token io8l1q.fphysnky4pmfp124 \
>         --discovery-token-ca-cert-hash
>         sha256:*generated-token-ca-cert-hash*

Please, save the command output, we will use it next step. Run next three step on the freshly initialized master node:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

###  3.2 Install pod network.
As you can see *kubeadm init* command output informs you to run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:https://kubernetes.io/docs/concepts/cluster-administration/addons/ Let's deploy **fannel** for simplicity.

Just remember for **flannel** we are using podCIDR 10.244.0.0/16 (look at the *k8s-conf.yaml* content), its a default value. For custom podCIDR (not 10.244.0.0/16) first download podnetwork.yaml manifest file and modify network parame.

Run:
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

More info on deploying flannel manually: https://github.com/flannel-io/flannel

## 4. Prepare supplied .yaml join files.
Modify the contents of provided *k8s-master-join.yaml* and *k8s-worker-join.yaml* files by replacing:
*generated-token*
*generated-token-ca-cert-hash*
*generated-control-plane-certificate-key*
with the values from 3.1 step (kubeadm initialization command output).

Then copy *k8s-master-join.yaml* to the master nodes `k8s-mas-2` and `k8s-mas-3` replacing *name* parameter with VM's actual names. Do the same for *k8s-worker-join.yaml*.

On the `k8s-mas-2` and `k8s-mas-3` master nodes run:
```
sudo kubeadm join --config k8s-master-join.yaml --v=5
```
On the worker nodes run:
```
sudo kubeadm join --config k8s-worker-join.yaml --v=5
```

Run the following command:
```
kubectl get nodes
```
*Kubectl* will list you the nodes of newly deployed K8s cluster. That's it. 