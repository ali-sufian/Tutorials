# Kubernetes Production Cluster Setup on RHEL 9 (VMware, On-Prem, Proxy)

## Cluster Overview

- **Cluster Type**: Kubernetes (vanilla)
- **Nodes**: 3 Master + 7 Worker
- **Platform**: VMware vSphere
- **OS**: RHEL 9
- **Networking**: On-prem, HTTP proxy with URL whitelisting
- **Container Runtime**: containerd
- **CNI**: Calico
- **Deployment Tool**: kubeadm

---

## Proxy Whitelist (Wildcard Style)

| Purpose             | Wildcard URL                                                |
| ------------------- | ----------------------------------------------------------- |
| Kubernetes packages | `*.k8s.io`, `*.googleapis.com`, `*.githubusercontent.com`   |
| GPG keys            | `*.google.com`, `*.gstatic.com`, `*.googleapis.com`         |
| Calico              | `*.projectcalico.org`, `*.githubusercontent.com`            |
| containerd, runc    | `*.github.com`, `*.githubusercontent.com`                   |
| Registries          | `*.docker.io`, `*.gcr.io`, `*.quay.io`, `*.registry.k8s.io` |
| RHEL Repos          | `*.redhat.com`, `*.rhsm.redhat.com`, `*.cdn.redhat.com`     |
| Helm                | `*.get.helm.sh`, `*.helm.sh`                                |

---

## System Preparation (All Nodes)

```bash
sudo hostnamectl set-hostname <node-name>
sudo dnf install -y vim git curl wget bash-completion net-tools iptables iproute
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Enable Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Sysctl Settings

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## Install containerd

```bash
sudo dnf install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Then:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Install kubeadm, kubelet, kubectl

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg \
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## Configure HAProxy Load Balancer (Control Plane)

Install HAProxy:

```bash
sudo dnf install -y haproxy
```

Create configuration:

```bash
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<EOF
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <MASTER1-IP>:6443 check
    server master2 <MASTER2-IP>:6443 check
    server master3 <MASTER3-IP>:6443 check
EOF
```

Enable and start:

```bash
sudo systemctl enable --now haproxy
```

Use this as `--control-plane-endpoint` for `kubeadm init`.

---

## Configure HAProxy Load Balancer (Ingress / Application Traffic)

For ingress traffic:

```bash
frontend ingress-frontend
    bind *:80
    bind *:443
    mode tcp
    default_backend ingress-backend

backend ingress-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server worker1 <WORKER1-IP>:<NODEPORT> check
    server worker2 <WORKER2-IP>:<NODEPORT> check
    server worker3 <WORKER3-IP>:<NODEPORT> check
    # Add more workers as needed
```

> Replace `<NODEPORT>` with your ingress controller's exposed NodePort.

---

## Initialize Master Node

```bash
sudo kubeadm init \
  --control-plane-endpoint "<haproxy-ip>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Install Calico

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

---

## Join Additional Nodes

### Master Nodes

```bash
kubeadm join <haproxy-ip>:6443 --control-plane --token ... --discovery-token-ca-cert-hash sha256:...
```

### Worker Nodes

```bash
kubeadm join <haproxy-ip>:6443 --token ... --discovery-token-ca-cert-hash sha256:...
```

---

## Post-Install

```bash
kubectl get nodes
kubectl get pods -A
```

---

## Optional Hardening

- etcd backup/disaster recovery
- RBAC policies
- Audit logging
- PodSecurity Standards
- Ingress controller (NGINX)
- Central logging (EFK or Loki)
- Monitoring (Prometheus + Grafana)

---

## Notes

- You can mirror container images using `ctr`, `skopeo`, or a local registry.
- Use HAProxy or Keepalived for HA of the control-plane endpoint if needed.
- RHEL 9 requires valid subscription for all packages — ensure RHSM is configured.
- For L7 load balancing, prefer MetalLB + NGINX ingress instead of raw NodePort.
- Always protect HAProxy and ingress with firewall or API gateway rules.
