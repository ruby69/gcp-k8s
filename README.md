# K8s on the GCP


## Create the VM instance
In the Google Cloud console, create and start 3VMs(Ubuntu 22.04 LTS Minimal).

### Pre-Setting
```
$ sudo apt-get update
$ sudo apt-get install -y vim tzdata apt-transport-https ca-certificates curl gpg gnupg lsb-release
```

### Turn off swap
```
$ sudo swapoff -a
$ sudo sed -i '/swap/s/^/#/' /etc/fstab
```

### Configure ip_forward
```
$ sudo vi /etc/sysctl.conf
net.ipv4.ip_forward=1 # uncomment

$ sudo sysctl -p
```


## Install Docker

### [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu)
Add Docker's official GPG key:
```
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add the repository to Apt sources:
```
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt-get update
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose
```


### [Install cri-dockerd](https://github.com/Mirantis/cri-dockerd)
The easiest way to install cri-dockerd is to use one of the pre-built binaries or packages from the [releases page](https://github.com/Mirantis/cri-dockerd/releases).
```
$ VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g'); echo $VER
$ wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
$ tar xvf cri-dockerd-${VER}.amd64.tgz
$ sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

$ sudo cri-dockerd --version

$ wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
$ wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
$ sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
$ sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

$ sudo systemctl daemon-reload
$ sudo systemctl enable cri-docker.service
$ sudo systemctl enable --now cri-docker.socket

$ sudo systemctl restart docker && sudo systemctl restart cri-docker
$ sudo systemctl status cri-docker.socket --no-pager 

$ sudo mkdir /etc/docker
$ cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

$ sudo systemctl restart docker && sudo systemctl restart cri-docker
$ sudo docker info | grep Cgroup

$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

$ sudo sysctl --system
```


## [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
These instructions are for Kubernetes 1.28:
```
$ sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo systemctl start kubelet && sudo systemctl enable kubelet

$ sudo apt-mark hold kubelet kubeadm kubectl
$ kubectl version --client=true
```


## [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

### Initializing control-plane node
To initialize the control-plane node run:
```
$ sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock
$ sudo kubeadm init \
        --ignore-preflight-errors=all \
	--pod-network-cidr=192.168.0.0/16 \
	--apiserver-advertise-address=10.178.0.11 \
	--cri-socket unix:///run/cri-dockerd.sock
...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
...

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.178.0.11:6443 --token <token_value> --discovery-token-ca-cert-hash <hash_value>

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

$ kubectl get nodes -o wide
$ kubectl get pod -A
```

If you lost the token:
```
$ kubeadm token create --print-join-command
kubeadm join 10.178.0.11:6443 --token <token_value> --discovery-token-ca-cert-hash <hash_value>
```

After login to gcloud, copy admin.conf file to other nodes:
```
$ sudo gcloud auth login
$ sudo gcloud config list
$ sudo gcloud compute scp /etc/kubernetes/admin.conf node-2:/etc/kubernetes/admin.conf
$ sudo gcloud compute scp /etc/kubernetes/admin.conf node-3:/etc/kubernetes/admin.conf
```

### Clean up the control plane
```
$ sudo systemctl stop kubelet
$ sudo kubeadm reset -f --cri-socket unix:///run/cri-dockerd.sock

$ sudo rm -rf ~/.kube
$ sudo rm -rf /root/.kube
$ sudo rm -rf /var/lib/etcd
$ sudo rm -rf /etc/kubernetes
```

### [Quickstart for Calico on K8s](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
Install Calico:
```
$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/tigera-operator.yaml -O
$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml -O
$ kubectl create -f tigera-operator.yaml
$ kubectl create -f custom-resources.yaml
$ watch kubectl get pods -n calico-system
```

Install calicoctl:
```
$ sudo curl -L "https://github.com/projectcalico/calico/releases/download/v3.26.4/calicoctl-linux-amd64" -o /usr/local/bin/calicoctl
$ sudo chmod +x /usr/local/bin/calicoctl

# CNI Type Check 
$ calicoctl get ippool -o wide

# BGP Protocol Check
$ sudo calicoctl node status

# Node Endpoint Check
$ calicoctl get workloadendpoint -A
```

Remove Calico:
```
$ kubectl delete -f custom-resources.yaml
$ kubectl delete -f tigera-operator.yaml

$ sudo rm -rf /var/run/calico/
$ sudo rm -rf /var/lib/calico/
$ sudo rm -rf /etc/cni/net.d/
$ sudo rm -rf /var/lib/cni/
$ sudo rm -rf /opt/cni
$ sudo reboot
```


## Joining nodes
```
$ sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock
$ sudo kubeadm join 10.178.0.11:6443 --token <token_value> --discovery-token-ca-cert-hash <hash_value>  --cri-socket unix:///run/cri-dockerd.sock
...

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

$ kubectl get nodes -o wide
```

To start using cluster as a regular user:
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```











