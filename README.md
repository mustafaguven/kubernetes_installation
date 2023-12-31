# Kubernetes Instalation with Kubeadm

OS upgrade and reboot machine
``` bash
sudo apt update
sudo apt -y full-upgrade
# Reboot
sudo reboot
```

# Adim 1
```
sudo su -
curl https://raw.githubusercontent.com/mustafaguven/kubernetes_installation/main/k8s.sh | bash -
```

## Adim 1 (Detay)
``` bash
sudo apt install curl apt-transport-https -y
curl -fsSL  https://packages.cloud.google.com/apt/doc/apt-key.gpg|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/k8s.gpg
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

## List of package command
sudo apt list kubeadm -a

## Installation latest version command below
sudo apt install kubelet kubeadm kubectl -y

## Specific version installation
sudo apt install -y kubelet=1.25.1-00  kubeadm=1.25.1-00  kubectl=1.25.1-00 

## Check version
kubelet --version
kubectl version
kubeadm version

## hold 
sudo apt-mark hold kubelet kubeadm kubectl

```

# Adim 2
``` bash
sudo su -
curl https://raw.githubusercontent.com/mustafaguven/kubernetes_installation/main/swap-disable.sh | bash -
```


# Adim 2 (Detay)
``` bash
## Temporary
sudo swapoff -a 

## Permanently
sudo vim /etc/fstab
# swapfile -> adding in front of swapfile #

## Confirm setting is correct
sudo mount -a
free -h
```

# Adim 3
Enable **kernel modules** and **configure sysctl**
``` bash
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system

```

Install Container runtime (Master and Worker nodes)
``` bash
## Instalation containerd 

# Configure persistent loading of modules
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

# Load at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
#sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start service
sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd


# To use the systemd cgroup driver, set plugins.cri.systemd_cgroup = true 
cat /etc/containerd/config.toml | grep SystemdCgroup
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

```

 Initialize control plane (run on first master node)
 ``` bash
## br_netfilter module is loaded
lsmod | grep br_netfilter

## Enable kubelet service
sudo systemctl enable kubelet

## Pull container images. Only master node pull images
sudo kubeadm config images pull

## If you have multiple CRI sockets, please use --cri-socket to select one:
## Containerd
sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock

# Create cluster
sudo kubeadm init --control-plane-endpoint="192.168.0.250:6443" --upload-certs --apiserver-advertise-address=192.168.0.200 --pod-network-cidr=10.0.0.3/16
#sudo kubeadm init --control-plane-endpoint="kubernetes.dev.env.test:6443" --apiserver-advertise-address=192.168.1.7 --node-name k8s-master-1 --pod-network-cidr=192.168.0.0/24

# To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


# Referance
```
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd


```


https://raw.githubusercontent.com/projectcalico/calico/v3.25.2/manifests/calico.yaml 

https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml  




