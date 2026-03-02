\==========================================================================

**KubeEnterprise - Production-Grade Kubernetes Platform on AWS (On-prem style)**

- Designed and implemented Kubernetes v1.33 cluster (1 master, 2 workers) using kubeadm on AWS EC2  
    <br/>
- Exposed microservices via NGINX Ingress using EC2 public IP + TLS (cert-manager) without MetalLB  
    <br/>
- Implemented RBAC, Namespaces, ResourceQuota, ConfigMaps/Secrets, PVC, Probes, HPA  
    <br/>
- Observability: Prometheus + Grafana; Logging: ELK; DR: etcd backup/restore; Release: Helm + rollback

<br/><br/>

**Phase 1 - Kubernetes Cluster Deployment**

**Overview**

Designed and implemented a Kubernetes v1.33 cluster (1 control-plane, 2 worker nodes) on AWS EC2 using kubeadm to simulate a production-style environment.

**Infrastructure Architecture**

- **Cloud Provider:** AWS EC2  
    <br/>
- **Cluster Type:** kubeadm-based (on-prem style architecture)  
    <br/>
- **Topology:  
    <br/>**
  - 1 Control Plane Node  
        <br/>
  - 2 Worker Nodes  
        <br/>
- **Operating System:** Ubuntu 22.04 LTS  
    <br/>
- **Container Runtime:** containerd  
    <br/>

**Node Configuration**

| **Node** | **Role** | **Instance Type** | **Purpose** |
| --- | --- | --- | --- |
| k8s-master | Control Plane | t3.small | API Server, Scheduler, Controller Manager, etcd |
| k8s-worker-1 | Worker | t3.small | Application workloads |
| k8s-worker-2 | Worker | t3.small | Application workloads |

**Security Group Configuration**

**Master Node**

- 6443 - Kubernetes API Server  
    <br/>
- 2379-2380 - etcd  
    <br/>
- 10250 - Kubelet  
    <br/>
- 22 - SSH  
    <br/>

**Worker Nodes**

- 10250 - Kubelet  
    <br/>
- 30000-32767 - NodePort Range  
    <br/>
- 22 - SSH  
    <br/>

**Cluster Initialization**

**1️⃣ System Preparation (All Nodes)**

sudo swapoff -a

sudo modprobe overlay

sudo modprobe br_netfilter

sudo sysctl --system

**2️⃣ Install containerd**

sudo apt update

sudo apt install -y containerd

sudo systemctl enable containerd

sudo systemctl start containerd

**3️⃣ Install Kubernetes Components**

sudo apt install -y kubeadm kubelet kubectl

sudo systemctl enable kubelet

**4️⃣ Initialize Control Plane**

sudo kubeadm init \\

&nbsp;--pod-network-cidr=10.244.0.0/16 \\

&nbsp;--apiserver-advertise-address=&lt;MASTER_PRIVATE_IP&gt;

Configure kubectl:

mkdir -p \$HOME/.kube

sudo cp /etc/kubernetes/admin.conf \$HOME/.kube/config

sudo chown \$(id -u):\$(id -g) \$HOME/.kube/config

**5️⃣ Join Worker Nodes**

Run the generated join command on both workers:

kubeadm join &lt;MASTER_PRIVATE_IP&gt;:6443 --token &lt;token&gt; \\

\--discovery-token-ca-cert-hash sha256:&lt;hash&gt;

**6️⃣ Install CNI (Calico)**

kubectl apply -f <https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml>

**Cluster Validation**

kubectl get nodes

Expected Output:

NAME            STATUS   ROLES           VERSION

k8s-master      Ready    control-plane   v1.33.x

k8s-worker-1    Ready    &lt;none&gt;          v1.33.x

k8s-worker-2    Ready    &lt;none&gt;          v1.33.x

**Outcome**

Successfully deployed a production-style Kubernetes cluster using kubeadm on AWS EC2 with:

- Control-plane initialization  
    <br/>
- Worker node registration  
    <br/>
- Calico networking  
    <br/>
- Secure API server exposure  
    <br/>
- Validated cluster health

----------------------------------------------------------------------------

\## Phase 2 - Public Exposure via Ingress (Without MetalLB)

**Phase 2 - Public Exposure via NGINX Ingress (Without MetalLB)**

**Overview**

Exposed microservices publicly using NGINX Ingress Controller via EC2 public IP with TLS enabled through cert-manager, without using MetalLB or AWS LoadBalancer service type.

**Architecture Flow**

User Browser

&nbsp;   ↓

EC2 Public IP (Worker Node)

&nbsp;   ↓

NodePort (30080 / 30443)

&nbsp;   ↓

NGINX Ingress Controller

&nbsp;   ↓

ClusterIP Service

&nbsp;   ↓

Application Pod

**Key Design Decisions**

- Used **NodePort** instead of LoadBalancer  
    <br/>
- Leveraged **EC2 public IP directly  
    <br/>**
- Avoided MetalLB for simplicity and AWS-native setup  
    <br/>
- TLS managed using **cert-manager (ACME / Let's Encrypt)  
    <br/>**

**Ingress Controller Installation**

kubectl apply -f <https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/baremetal/deploy.yaml>

**Convert Ingress Service to NodePort**

kubectl patch svc ingress-nginx-controller -n ingress-nginx \\

&nbsp;-p '{"spec": {"type": "NodePort"}}'

Verify:

kubectl get svc -n ingress-nginx

Expected Ports:

- HTTP → 30080  
    <br/>
- HTTPS → 30443  
    <br/>

**EC2 Security Group Configuration**

Allowed inbound rules on worker nodes:

- TCP 80 (optional redirect)  
    <br/>
- TCP 443  
    <br/>
- TCP 30080  
    <br/>
- TCP 30443  
    <br/>
- TCP 22 (SSH)  
    <br/>

**Application Ingress Example**

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

&nbsp;name: app-ingress

&nbsp;namespace: production

&nbsp;annotations:

&nbsp;  nginx.ingress.kubernetes.io/ssl-redirect: "true"

&nbsp;  cert-manager.io/cluster-issuer: "letsencrypt-production"

spec:

&nbsp;ingressClassName: nginx

&nbsp;tls:

&nbsp;- hosts:

&nbsp;  - app.yourdomain.com

&nbsp;  secretName: app-tls

&nbsp;rules:

&nbsp;- host: app.yourdomain.com

&nbsp;  http:

&nbsp;    paths:

&nbsp;    - path: /

&nbsp;      pathType: Prefix

&nbsp;      backend:

&nbsp;        service:

&nbsp;          name: app-service

&nbsp;          port:

&nbsp;            number: 80

**Install cert-manager (Helm)**

helm repo add jetstack <https://charts.jetstack.io>

helm repo update

helm install cert-manager jetstack/cert-manager \\

&nbsp;--namespace cert-manager \\

&nbsp;--create-namespace \\

&nbsp;--set installCRDs=true

**ClusterIssuer Configuration**

apiVersion: cert-manager.io/v1

kind: ClusterIssuer

metadata:

&nbsp;name: letsencrypt-production

spec:

&nbsp;acme:

&nbsp;  server: <https://acme-v02.api.letsencrypt.org/directory>

&nbsp;  email: <admin@yourdomain.com>

&nbsp;  privateKeySecretRef:

&nbsp;    name: letsencrypt-production

&nbsp;  solvers:

&nbsp;  - http01:

&nbsp;      ingress:

&nbsp;        class: nginx

**Validation Steps**

Check ingress:

kubectl get ingress -n production

Test from browser:

<https://app.yourdomain.com>

Verify certificate:

kubectl describe certificate -n production

**Outcome**

Successfully exposed Kubernetes workloads publicly using:

- NGINX Ingress Controller  
    <br/>
- EC2 public IP  
    <br/>
- NodePort (30080/30443)  
    <br/>
- cert-manager TLS  
    <br/>
- No MetalLB or external LoadBalancer  
    <br/>

This approach simulates real-world edge exposure in a cost-effective AWS environment.

---------------------

Phase 3 - Workload Governance & Resource Management

**Phase 3 - Workload Governance & Resource Management**

**Overview**

Implemented RBAC, Namespaces, ResourceQuota, ConfigMaps, Secrets, Persistent Volume Claims, Health Probes, and Horizontal Pod Autoscaling (HPA) to enforce security, resource control, and application reliability in a production-style Kubernetes environment.

**1️⃣ Namespace Isolation**

Created logical environments for workload separation:

kubectl create namespace production

kubectl create namespace staging

kubectl create namespace development

Purpose:

- Environment segregation  
    <br/>
- Resource control per namespace  
    <br/>
- Access restriction via RBAC  
    <br/>

**2️⃣ RBAC (Role-Based Access Control)**

**Example: Developer Read-Only Role**

apiVersion: rbac.authorization.k8s.io/v1

kind: Role

metadata:

&nbsp;name: developer-readonly

&nbsp;namespace: development

rules:

\- apiGroups: \[""\]

&nbsp;resources: \["pods", "services", "configmaps"\]

&nbsp;verbs: \["get", "list", "watch"\]

Bind Role:

apiVersion: rbac.authorization.k8s.io/v1

kind: RoleBinding

metadata:

&nbsp;name: developer-binding

&nbsp;namespace: development

subjects:

\- kind: User

&nbsp;name: developer-user

roleRef:

&nbsp;kind: Role

&nbsp;name: developer-readonly

&nbsp;apiGroup: rbac.authorization.k8s.io

Outcome:

- Least privilege access enforced  
    <br/>
- Controlled namespace-level permissions  
    <br/>

**3️⃣ ResourceQuota & LimitRange**

**ResourceQuota (Production Namespace)**

apiVersion: v1

kind: ResourceQuota

metadata:

&nbsp;name: production-quota

&nbsp;namespace: production

spec:

&nbsp;hard:

&nbsp;  requests.cpu: "4"

&nbsp;  requests.memory: 8Gi

&nbsp;  limits.cpu: "6"

&nbsp;  limits.memory: 12Gi

&nbsp;  pods: "20"

Purpose:

- Prevent resource starvation  
    <br/>
- Control namespace-level consumption  
    <br/>

**4️⃣ ConfigMaps & Secrets**

**ConfigMap**

apiVersion: v1

kind: ConfigMap

metadata:

&nbsp;name: app-config

&nbsp;namespace: production

data:

&nbsp;APP_ENV: production

&nbsp;LOG_LEVEL: info

**Secret**

apiVersion: v1

kind: Secret

metadata:

&nbsp;name: app-secret

&nbsp;namespace: production

type: Opaque

data:

&nbsp;DB_PASSWORD: cGFzc3dvcmQ=   # base64 encoded

Outcome:

- Environment-specific configuration separation  
    <br/>
- Secure sensitive data management  
    <br/>

**5️⃣ Persistent Volume Claim (PVC)**

apiVersion: v1

kind: PersistentVolumeClaim

metadata:

&nbsp;name: app-pvc

&nbsp;namespace: production

spec:

&nbsp;accessModes:

&nbsp;- ReadWriteOnce

&nbsp;resources:

&nbsp;  requests:

&nbsp;    storage: 10Gi

Purpose:

- Durable storage for stateful components  
    <br/>
- EBS-backed persistent storage  
    <br/>

**6️⃣ Liveness & Readiness Probes**

livenessProbe:

&nbsp;httpGet:

&nbsp;  path: /health

&nbsp;  port: 8080

&nbsp;initialDelaySeconds: 15

&nbsp;periodSeconds: 10

readinessProbe:

&nbsp;httpGet:

&nbsp;  path: /ready

&nbsp;  port: 8080

&nbsp;initialDelaySeconds: 5

&nbsp;periodSeconds: 5

Outcome:

- Automatic container restart on failure  
    <br/>
- Traffic routed only to healthy pods  
    <br/>

**7️⃣ Horizontal Pod Autoscaler (HPA)**

Installed metrics-server and configured HPA:

kubectl autoscale deployment app-deployment \\

&nbsp;--cpu-percent=50 \\

&nbsp;--min=2 \\

&nbsp;--max=6 \\

&nbsp;-n production

Verify:

kubectl get hpa -n production

Outcome:

- Automatic scaling based on CPU utilization  
    <br/>
- Load-aware application elasticity  
    <br/>

**Result**

Successfully implemented enterprise-grade workload governance including:

- Namespace isolation  
    <br/>
- Least-privilege RBAC  
    <br/>
- Resource quota enforcement  
    <br/>
- Secure configuration management  
    <br/>
- Persistent storage  
    <br/>
- Health monitoring  
    <br/>
- Horizontal scaling

--------------

Phase 3 - Workload Governance & Resource Management

**Phase 3 - Workload Governance & Resource Management**

**Overview**

Implemented RBAC, Namespaces, ResourceQuota, ConfigMaps, Secrets, Persistent Volume Claims, Health Probes, and Horizontal Pod Autoscaling (HPA) to enforce security, resource control, and application reliability in a production-style Kubernetes environment.

**1️⃣ Namespace Isolation**

Created logical environments for workload separation:

kubectl create namespace production

kubectl create namespace staging

kubectl create namespace development

Purpose:

- Environment segregation  
    <br/>
- Resource control per namespace  
    <br/>
- Access restriction via RBAC  
    <br/>

**2️⃣ RBAC (Role-Based Access Control)**

**Example: Developer Read-Only Role**

apiVersion: rbac.authorization.k8s.io/v1

kind: Role

metadata:

&nbsp;name: developer-readonly

&nbsp;namespace: development

rules:

\- apiGroups: \[""\]

&nbsp;resources: \["pods", "services", "configmaps"\]

&nbsp;verbs: \["get", "list", "watch"\]

Bind Role:

apiVersion: rbac.authorization.k8s.io/v1

kind: RoleBinding

metadata:

&nbsp;name: developer-binding

&nbsp;namespace: development

subjects:

\- kind: User

&nbsp;name: developer-user

roleRef:

&nbsp;kind: Role

&nbsp;name: developer-readonly

&nbsp;apiGroup: rbac.authorization.k8s.io

Outcome:

- Least privilege access enforced  
    <br/>
- Controlled namespace-level permissions  
    <br/>

**3️⃣ ResourceQuota & LimitRange**

**ResourceQuota (Production Namespace)**

apiVersion: v1

kind: ResourceQuota

metadata:

&nbsp;name: production-quota

&nbsp;namespace: production

spec:

&nbsp;hard:

&nbsp;  requests.cpu: "4"

&nbsp;  requests.memory: 8Gi

&nbsp;  limits.cpu: "6"

&nbsp;  limits.memory: 12Gi

&nbsp;  pods: "20"

Purpose:

- Prevent resource starvation  
    <br/>
- Control namespace-level consumption  
    <br/>

**4️⃣ ConfigMaps & Secrets**

**ConfigMap**

apiVersion: v1

kind: ConfigMap

metadata:

&nbsp;name: app-config

&nbsp;namespace: production

data:

&nbsp;APP_ENV: production

&nbsp;LOG_LEVEL: info

**Secret**

apiVersion: v1

kind: Secret

metadata:

&nbsp;name: app-secret

&nbsp;namespace: production

type: Opaque

data:

&nbsp;DB_PASSWORD: cGFzc3dvcmQ=   # base64 encoded

Outcome:

- Environment-specific configuration separation  
    <br/>
- Secure sensitive data management  
    <br/>

**5️⃣ Persistent Volume Claim (PVC)**

apiVersion: v1

kind: PersistentVolumeClaim

metadata:

&nbsp;name: app-pvc

&nbsp;namespace: production

spec:

&nbsp;accessModes:

&nbsp;- ReadWriteOnce

&nbsp;resources:

&nbsp;  requests:

&nbsp;    storage: 10Gi

Purpose:

- Durable storage for stateful components  
    <br/>
- EBS-backed persistent storage  
    <br/>

**6️⃣ Liveness & Readiness Probes**

livenessProbe:

&nbsp;httpGet:

&nbsp;  path: /health

&nbsp;  port: 8080

&nbsp;initialDelaySeconds: 15

&nbsp;periodSeconds: 10

readinessProbe:

&nbsp;httpGet:

&nbsp;  path: /ready

&nbsp;  port: 8080

&nbsp;initialDelaySeconds: 5

&nbsp;periodSeconds: 5

Outcome:

- Automatic container restart on failure  
    <br/>
- Traffic routed only to healthy pods  
    <br/>

**7️⃣ Horizontal Pod Autoscaler (HPA)**

Installed metrics-server and configured HPA:

kubectl autoscale deployment app-deployment \\

&nbsp;--cpu-percent=50 \\

&nbsp;--min=2 \\

&nbsp;--max=6 \\

&nbsp;-n production

Verify:

kubectl get hpa -n production

Outcome:

- Automatic scaling based on CPU utilization  
    <br/>
- Load-aware application elasticity  
    <br/>

**Result**

Successfully implemented enterprise-grade workload governance including:

- Namespace isolation  
    <br/>
- Least-privilege RBAC  
    <br/>
- Resource quota enforcement  
    <br/>
- Secure configuration management  
    <br/>
- Persistent storage  
    <br/>
- Health monitoring  
    <br/>
- Horizontal scaling

-----------------

**Phase 4 - Observability, Logging, Disaster Recovery & Release Management**

**Overview**

Implemented full-stack observability using Prometheus and Grafana, centralized logging with ELK Stack, disaster recovery via etcd backup/restore, and application lifecycle management using Helm with rollback capability.

**1️⃣ Monitoring - Prometheus + Grafana**

**Installation (Helm)**

helm repo add prometheus-community <https://prometheus-community.github.io/helm-charts>

helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \\

&nbsp;--namespace monitoring \\

&nbsp;--create-namespace

**Key Metrics Collected**

- Node CPU / Memory usage  
    <br/>
- Pod resource utilization  
    <br/>
- Kubernetes object states  
    <br/>
- Application metrics  
    <br/>
- HPA scaling metrics  
    <br/>

**Grafana Access**

kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80

Default Dashboards Used:

- Node Exporter Full  
    <br/>
- Kubernetes Cluster Monitoring  
    <br/>
- Pod Resource Dashboard  
    <br/>

Outcome:

- Real-time infrastructure visibility  
    <br/>
- Performance monitoring  
    <br/>
- Capacity planning support  
    <br/>

**2️⃣ Centralized Logging - ELK Stack**

**Components**

- Elasticsearch (Data storage)  
    <br/>
- Kibana (Visualization)  
    <br/>
- Filebeat (Log shipping as DaemonSet)  
    <br/>

**Filebeat Example Configuration**

filebeat.inputs:

\- type: container

&nbsp;paths:

&nbsp;  - /var/log/containers/\*.log

output.elasticsearch:

&nbsp;hosts: \['<http://elasticsearch:9200'\>]

Outcome:

- Centralized pod log aggregation  
    <br/>
- Log search and filtering  
    <br/>
- Troubleshooting and root-cause analysis  
    <br/>

**3️⃣ Disaster Recovery - etcd Backup & Restore**

**etcd Snapshot**

export ETCDCTL_API=3

etcdctl snapshot save /opt/etcd-backups/etcd-snapshot.db

**Restore Procedure**

etcdctl snapshot restore etcd-snapshot.db \\

&nbsp;--data-dir=/var/lib/etcd

Outcome:

- Cluster state protection  
    <br/>
- Recovery from control-plane failure  
    <br/>
- Documented DR runbook  
    <br/>

**4️⃣ Release Management - Helm & Rollback**

**Install Application via Helm**

helm install my-app ./custom-chart \\

&nbsp;--namespace production

**Upgrade with Safe Rollback**

helm upgrade my-app ./custom-chart \\

&nbsp;--namespace production \\

&nbsp;--atomic \\

&nbsp;--timeout 5m

**Rollback**

helm rollback my-app 2

Outcome:

- Version-controlled deployments  
    <br/>
- Safe upgrades  
    <br/>
- Instant rollback capability  
    <br/>
- Production-ready release strateg

<br/>
