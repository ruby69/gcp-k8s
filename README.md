# K8s on the GCP


## Create the VM instance
In the Google Cloud console, create and start 3VMs(Ubuntu 22.04 LTS Minimal).

### Pre-Setting
```
$ sudo apt-get update
$ sudo apt-get install -y vim tzdata apt-transport-https ca-certificates curl gpg gnupg lsb-release
```

### turn off swap
```
$ sudo swapoff -a
$ sudo sed -i '/swap/s/^/#/' /etc/fstab
```

### configure ip_forward
```
$ sudo vi /etc/sysctl.conf
net.ipv4.ip_forward=1 # uncomment

$ sudo sysctl -p
```


## Install Docker

### Docker
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# docker-compose
$ sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ sudo groupadd docker
$ sudo gpasswd -a ${USER} docker
```

### CRI-Docker
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


## Install kubeadm, kubectl and kubelet 
```
$ sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo systemctl start kubelet && sudo systemctl enable kubelet

$ sudo apt-mark hold kubelet kubeadm kubectl
$ kubectl version --client=true
```


## Initialize Master

### Create single control-plane cluster
```
$ sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock
$ sudo kubeadm init --ignore-preflight-errors=all --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.178.0.11 --cri-socket /var/run/cri-dockerd.sock
  ->     kubeadm join 10.178.0.11:6443 --token <token_value> --discovery-token-ca-cert-hash <hash_value>

# check the token
$ kubeadm token create --print-join-command
```

```
# configuration kubeadm
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

$ kubectl get nodes -o wide
$ kubectl get pod -A
```

```
# copy admin.conf to other nodes.
$ sudo scp /etc/kubernetes/admin.conf worker@node02:/home/worker/admin.conf
$ sudo scp /etc/kubernetes/admin.conf worker@node03:/home/worker/admin.conf
```

### Initialize CNI(Calico)
```
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/tigera-operator.yaml
$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml -O
$ kubectl create -f custom-resources.yaml
$ watch kubectl get pods -n calico-system

# calicoctl
$ sudo curl -L "https://github.com/projectcalico/calico/releases/download/v3.23.3/calicoctl-linux-amd64" -o /usr/local/bin/calicoctl
$ sudo chmod +x /usr/local/bin/calicoctl
```


## Initialize Worker
```
$ sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock
$ sudo kubeadm join 10.178.0.11:6443 --token <token_value> --discovery-token-ca-cert-hash <hash_value>  --cri-socket unix:///run/cri-dockerd.sock
```

```
# configuration kubeadm
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

$ kubectl get nodes -o wide
$ kubectl get pod -A
```













