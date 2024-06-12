# kube-old
## How to Install Kubernetes Old Version by Binary Package

1. Set Your Enviroment
```
sudo timedatectl set-timezone Asia/Jakarta
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf 
overlay 
br_netfilter 
EOF

sudo modprobe overlay 
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf 
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1 
net.bridge.bridge-nf-call-ip6tables = 1 
EOF

sudo sysctl --system
```

2. Install Containerd Runtime
```
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

curl -LO https://dl.k8s.io/release/v1.17.1/bin/linux/amd64/kubectl
curl -LO https://dl.k8s.io/release/v1.17.1/bin/linux/amd64/kubeadm
curl -LO https://dl.k8s.io/release/v1.17.1/bin/linux/amd64/kubelet
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.26.0/crictl-v1.26.0-linux-amd64.tar.gz

tar xzvf crictl-v1.26.0-linux-amd64.tar.gz -C /usr/local/bin
chmod +x kube*
mv kube* /usr/local/bin
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
EOF
```

3. Build Kubeconfig
```
cat <<EOF | sudo tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "192.168.122.122:6443"  <-- Your IP or IP VRRP
networking:
    podSubnet: "192.168.1.0/24" <-- Freedom
EOF

kubeadm init --config=kubeadm-config.yaml --upload-certs --cri-socket="unix:///run/containerd/containerd.sock" --v=5

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

(Option) if failed pull image because you using proxy, pull manual with comamnd on bellow
```
ctr -n k8s.io i pull k8s.gcr.io/v2/coredns/manifests/1.6.5
ctr -n k8s.io i pull k8s.gcr.io/coredns:1.6.5
ctr -n k8s.io i pull k8s.gcr.io/etcd:3.4.3-0
ctr -n k8s.io i pull k8s.gcr.io/pause:3.1
ctr -n k8s.io i pull k8s.gcr.io/kube-proxy:v1.17.17
ctr -n k8s.io i pull k8s.gcr.io/kube-scheduler:v1.17.17
ctr -n k8s.io i pull k8s.gcr.io/kube-apiserver:v1.17.17
ctr -n k8s.io i pull k8s.gcr.io/kube-controller-manager:v1.17.17
```

4. Create Service Systemd
```
cat <<EOF | sudo tee /lib/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/home/

[Service]
ExecStart=/usr/local/bin/kubelet \
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml \
--container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock --resolv-conf=/run/systemd/resolve/resolv.conf
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now kubelet
```

5. Build CNI
```
kubectl apply -f https://docs.projectcalico.org/v3.17/manifests/calico.yaml
kubectl get pods -n kube-system
kubectl get all -n kube-system
```

## Node
after that you can 
* generate new token for new node joint
* set role to nodes master or worker

## Reference
* https://github.com/kubernetes-sigs/cri-tools/tags
* https://kubernetes.io/releases/download/
* https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.17.md#v1171
