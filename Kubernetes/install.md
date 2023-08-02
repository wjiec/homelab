Install Kubernetes
---------------------------

### 配置系统

```bash
timedatectl set-timezone Asia/Shanghai

apt update
apt install -y apt-transport-https ca-certificates curl

cat > /etc/sysctl.d/50-kubernetes.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

### 启用 ipvs

```bash
apt install -y ipvsadm ipset

# https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md
echo "ip_vs" >> /etc/modules
echo "ip_vs_rr" >> /etc/modules
echo "ip_vs_wrr" >> /etc/modules
echo "ip_vs_sh" >> /etc/modules
# use `nf_conntrack` instead of `nf_conntrack_ipv4` for Linux kernel 4.19 and later
echo "nf_conntrack" >> /etc/modules
# https://gist.github.com/iamcryptoki/ed6925ce95f047673e7709f23e0b9939
# for `net.bridge.bridge-nf-call-iptables`
echo "br_netfilter" >> /etc/modules
```

### 安装 containerd

```bash
# https://github.com/containerd/containerd/blob/main/docs/getting-started.md

tar Cxzvf /usr/local /mnt/nas/kubernetes/containerd-1.7.3-linux-amd64.tar.gz

mkdir -p /usr/local/lib/systemd/system
cp /mnt/nas/kubernetes/containerd.service /usr/local/lib/systemd/system/containerd.service

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

systemctl daemon-reload
systemctl enable --now containerd

# install runc
install -m 755 /mnt/nas/kubernetes/runc.amd64 /usr/local/sbin/runc

# install cni
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin /mnt/nas/kubernetes/cni-plugins-linux-amd64-v1.3.0.tgz
```



### 安装kubelet kubectl kubeadm

参考地址：

```bash
curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

apt update # apt list -a kubeadm
apt install -y kubeadm=1.26.7-00 kubelet=1.26.7-00 kubectl=1.26.7-00
apt-mark hold kubelet kubeadm kubectl
```



```yaml
kind: InitConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: solar-system.d9f60ca41a70953c
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
localAPIEndpoint:
  advertiseAddress: 172.16.2.6
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: mercury
  taints: null
---
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
apiServer:
  certSANs:
    - 172.16.2.6
    - mercury.solar
  extraArgs:
    bind-address: 0.0.0.0
    authorization-mode: Node,RBAC
    service-node-port-range: 30000-32767
  timeoutForControlPlane: 5m0s
certificatesDir: /etc/kubernetes/pki
clusterName: solar
controlPlaneEndpoint: 172.16.2.6:6443
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kubernetesVersion: 1.26.7
networking:
  dnsDomain: cluster.local
  podSubnet: 10.6.0.0/16
  serviceSubnet: 10.7.0.0/16
scheduler: {}
---
# https://pkg.go.dev/k8s.io/kube-proxy/config/v1alpha1#KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

### 安装完成之后

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

mkdir -p $HOME/.kube
cp -i /mnt/nas/kubernetes/solar/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装flannel

```bash
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged

helm repo add flannel https://flannel-io.github.io/flannel/
helm install flannel --set podCidr="10.6.0.0/16" --namespace kube-flannel flannel/flannel
```



### 