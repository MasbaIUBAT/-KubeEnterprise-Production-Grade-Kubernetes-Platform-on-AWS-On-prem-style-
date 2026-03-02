# -KubeEnterprise-Production-Grade-Kubernetes-Platform-on-AWS-On-prem-style-
Production-grade Kubernetes v1.33 cluster (1 control plane, 2 workers) built with kubeadm on AWS EC2 (on-prem style). Implemented NGINX Ingress + TLS, RBAC, HPA, PVC, Prometheus-Grafana, ELK logging, etcd backup/restore, and Helm-based releases


# Kubernetes Production-Grade Cluster Deployment (Phase 1)

![Kubernetes Version](https://img.shields.io/badge/kubernetes-v1.33-blue?style=flat-square&logo=kubernetes)
![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)
![Status](https://img.shields.io/badge/status-active-success?style=flat-square)

This repository contains the documentation and configuration steps for Phase 1 of our infrastructure rollout: the deployment of a production-style Kubernetes v1.33 cluster on AWS EC2 using `kubeadm`.

## 📌 Project Overview

The goal of this phase was to design and implement a highly available, "on-prem style" Kubernetes architecture within a cloud environment. By using `kubeadm` instead of managed services like EKS, we maintain full control over the control plane, container runtime, and networking stack.

## 🚀 Key Features

- **Standardized Topology**: 1 Control Plane and 2 Worker nodes for workload distribution.
- **Container Runtime**: Industry-standard `containerd` integration.
- **Networking**: Robust Pod networking via Calico CNI.
- **Security-First**: Granular AWS Security Group configurations for API and Kubelet isolation.
- **Modern Stack**: Built on Ubuntu 22.04 LTS and K8s v1.33.

---

## 🏗 Infrastructure Architecture

### Node Topology
| Node Name | Role | Instance Type | OS | Component Highlights |
| :--- | :--- | :--- | :--- | :--- |
| `k8s-master` | Control Plane | t3.small | Ubuntu 22.04 | API Server, etcd, Scheduler |
| `k8s-worker-1` | Worker | t3.small | Ubuntu 22.04 | Application Runtime |
| `k8s-worker-2` | Worker | t3.small | Ubuntu 22.04 | Application Runtime |

### Network Security Groups

#### Master Node (Inbound)
| Port Range | Protocol | Purpose |
| :--- | :--- | :--- |
| `6443` | TCP | Kubernetes API Server |
| `2379-2380` | TCP | etcd server client API |
| `10250` | TCP | Kubelet API |
| `22` | TCP | SSH Access |

#### Worker Nodes (Inbound)
| Port Range | Protocol | Purpose |
| :--- | :--- | :--- |
| `10250` | TCP | Kubelet API |
| `30000-32767` | TCP | NodePort Services |
| `22` | TCP | SSH Access |

---

## 🛠 Installation & Setup

### 1. System Preparation (All Nodes)
Run the following to configure the kernel modules and disable swap:

```bash
sudo swapoff -a
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2. Container Runtime Installation
Install and enable `containerd`:

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 3. Kubernetes Tooling
Install the core Kubernetes binaries:

```bash
sudo apt update && sudo apt install -y apt-transport-https curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 4. Control Plane Initialization
On the `k8s-master` node:

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_PRIVATE_IP>

# Setup local kubeconfig
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5. Join Worker Nodes
Execute the join command provided by the `kubeadm init` output on both worker nodes:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### 6. CNI Deployment (Calico)
Deploy the Calico network plugin from the master node:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

---

## 🔍 Validation

To verify that the cluster is operational and all nodes have joined successfully, run:

```bash
kubectl get nodes
```

**Expected Output:**
```text
NAME            STATUS   ROLES           AGE   VERSION
k8s-master      Ready    control-plane   10m   v1.33.x
k8s-worker-1    Ready    <none>          5m    v1.33.x
k8s-worker-2    Ready    <none>          5m    v1.33.x
```

## 🤝 Contributing
1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/AmazingFeature`).
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`).
4. Push to the branch (`git push origin feature/AmazingFeature`).
5. Open a Pull Request.

## 📄 License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

This repository documents the implementation of Phase 2 of the infrastructure rollout: Exposing microservices publicly using an NGINX Ingress Controller on AWS EC2. This architecture bypasses the need for MetalLB or the AWS LoadBalancer service type, utilizing a cost-effective NodePort strategy with automated TLS termination.

🏗️ Architecture Flow
The traffic flow bypasses cloud-native load balancers to reduce costs and complexity in specialized environments:

graph LR
    A[User Browser] -->|HTTPS:443| B[EC2 Public IP]
    B -->|NodePort:30443| C[NGINX Ingress Controller]
    C -->|Internal Routing| D[ClusterIP Service]
    D --> E[Application Pods]
🚀 Key Features
Cost Optimization: Eliminates AWS ELB/ALB costs by leveraging EC2 Public IPs.
Direct Exposure: Uses NodePort mapping for high-performance ingress routing.
Automated TLS: Full certificate lifecycle management via cert-manager and Let's Encrypt.
Production Ready: Includes SSL redirection and security group hardening.
🛠️ Installation
1. Deploy NGINX Ingress Controller
Install the bare-metal version of the NGINX Ingress controller:

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/baremetal/deploy.yaml
2. Configure NodePort Mapping
Patch the controller service to utilize specific static ports:

kubectl patch svc ingress-nginx-controller -n ingress-nginx \
 -p '{"spec": {"type": "NodePort", "ports": [{"name": "http", "port": 80, "nodePort": 30080}, {"name": "https", "port": 443, "nodePort": 30443}]}}'
3. Install cert-manager
Manage SSL/TLS certificates automatically via Helm:

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
 --namespace cert-manager \
 --create-namespace \
 --set installCRDs=true
⚙️ Configuration
EC2 Security Group Rules
Ensure the following ports are open on your Worker Node security groups to allow external traffic:

| Protocol | Port | Description | | :--- | :--- | :--- | | TCP | 80 | HTTP Redirect (Optional) | | TCP | 443 | HTTPS Traffic | | TCP | 30080 | NGINX HTTP NodePort | | TCP | 30443 | NGINX HTTPS NodePort | | TCP | 22 | SSH Management |

ClusterIssuer Setup
Create a ClusterIssuer to communicate with Let's Encrypt:

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourdomain.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
        ingress:
          class: nginx
🖥️ Usage
Deploying an Application Ingress
To expose your service, apply the following Ingress manifest. Replace app.yourdomain.com with your actual domain.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.yourdomain.com
    secretName: app-tls
  rules:
  - host: app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
🔍 Validation & Troubleshooting
Verify the status of your deployment using the following commands:

Check Ingress Status:

kubectl get ingress -n production
Verify TLS Certificate Issuance:

kubectl describe certificate -n production
Check Controller Service Ports:

kubectl get svc -n ingress-nginx
🤝 Contributing
Fork the repository.
Create a feature branch (git checkout -b feature/Governance).
Commit changes (git commit -m 'Add resource limits').
Push to the branch (git push origin feature/Governance).
Open a Pull Request.
