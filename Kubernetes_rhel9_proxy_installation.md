
# Kubernetes Air-Gapped Installation with kubeadm on RHEL 9.5

This guide provides a step-by-step process to set up a production-grade, air-gapped Kubernetes cluster using `kubeadm`, `kubelet`, and `kubectl` on RHEL 9.5. It includes setting up a local container image registry, downloading required images, and initializing the cluster without internet access.

---

## Prerequisites

- RHEL 9.5 installed on all nodes (minimum 3 masters, 3 workers).
- `kubeadm`, `kubelet`, `kubectl` installed offline on all nodes.
- `containerd` is installed and running on all nodes.
- Internet access (proxy or direct) is available on at least one helper machine.
- Cluster nodes can communicate over the required ports.
- NTP is configured and synchronized on all nodes.

---

## Step 1: Set Up a Local Container Registry

On one of your nodes (e.g., `node1`), run:

```bash
podman run -d -p 5000:5000 --restart=always --name registry   -v /opt/registry:/var/lib/registry   registry:2
```

Verify the registry is accessible:

```bash
curl http://localhost:5000/v2/_catalog
```

> Optionally, configure `/etc/hosts` on other nodes to resolve the registry hostname.

---

## Step 2: Download Kubernetes Images on an Internet-Connected Machine

Run the following on a machine with internet:

```bash
export K8S_VER="v1.29.0"

IMAGES=(
  kube-apiserver:$K8S_VER
  kube-controller-manager:$K8S_VER
  kube-scheduler:$K8S_VER
  kube-proxy:$K8S_VER
  pause:3.9
  etcd:3.5.10-0
  coredns/coredns:v1.11.1
)

for img in "${IMAGES[@]}"; do
  docker pull registry.k8s.io/${img}
  docker tag registry.k8s.io/${img} localhost:5000/${img}
  docker push localhost:5000/${img}
done
```

---

## Step 3: Export and Transfer Images to Air-Gapped Nodes

Save images as tarballs:

```bash
for img in "${IMAGES[@]}"; do
  docker save -o $(basename ${img}).tar localhost:5000/${img}
done
```

Transfer these `.tar` files to each air-gapped node and import:

```bash
for tar in *.tar; do
  podman load -i $tar
done
```

---

## Step 4: Prepare kubeadm Config File

Create a file named `kubeadm-config.yaml` on the control plane node:

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "<CONTROL_PLANE_IP>"
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
imageRepository: "localhost:5000"
networking:
  podSubnet: "10.244.0.0/16"
```

Replace `<CONTROL_PLANE_IP>` with the IP of your master node.

---

## Step 5: Initialize the Cluster

On the control plane node, run:

```bash
kubeadm init --config kubeadm-config.yaml --upload-certs
```

After success, configure kubectl:

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 6: Join Worker Nodes

On each worker node, import the images as in Step 3. Then, run the `kubeadm join` command provided by `kubeadm init`.

Example:

```bash
kubeadm join <CONTROL_PLANE_IP>:6443 --token <token>   --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Step 7: Install Pod Network Add-on (Flannel)

1. On a machine with internet, download the Flannel YAML:

   ```bash
   wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
   ```

2. Modify the YAML to use local image registry if needed, then transfer to the control plane node.

3. Apply the manifest:

   ```bash
   kubectl apply -f kube-flannel.yml
   ```

---

## Step 8: Verify the Cluster

Check nodes:

```bash
kubectl get nodes
```

Check pods:

```bash
kubectl get pods -A
```

---

## Notes

- You can also use Harbor or Nexus as a registry instead of Podman/Docker registry.
- The registry hostname (e.g., `localhost:5000`) must be resolvable and trusted by all cluster nodes.
- Add the local registry as an insecure registry in `containerd` config if needed.
- Make sure to configure firewall rules and SELinux if applicable.

---

## References

- [Kubernetes Official kubeadm Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubeadm Config API Reference](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/)
- [Podman Registry Setup](https://docs.podman.io/en/latest/markdown/podman-container.html)
- [Air-Gapped Kubernetes GitHub Example](https://github.com/kairen/kubeadm-image-repo)




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
