# 🚀 KubeEnterprise – Production-Grade Kubernetes Platform on AWS (On-Prem Style)

## Overview

KubeEnterprise is a production-oriented Kubernetes platform built on AWS EC2 using kubeadm (v1.33).  
The project simulates an on-premise Kubernetes architecture in a cloud environment and demonstrates real-world DevOps practices including secure public exposure, workload governance, observability, disaster recovery, and version-controlled deployments.

The cluster consists of one control-plane node and two worker nodes deployed on Ubuntu 22.04 with containerd runtime and Calico CNI networking.

---

## Infrastructure & Architecture

The Kubernetes cluster was provisioned manually using kubeadm to simulate real-world platform engineering practices rather than relying on managed services.

### Cluster Topology

- 1 Control Plane (API Server, etcd, Scheduler, Controller Manager)
- 2 Worker Nodes (Application workloads)
- Calico for Pod networking
- NodePort-based edge exposure
- EC2 Public IP used directly (no MetalLB or AWS LoadBalancer)

### Security Group Configuration

Master:
- 6443 (Kubernetes API)
- 2379–2380 (etcd)
- 10250 (kubelet)
- 22 (SSH)

Workers:
- 30000–32767 (NodePort range)
- 80 / 443 (Public ingress)
- 10250 (kubelet)
- 22 (SSH)

---

## Public Application Exposure

Applications are exposed externally using NGINX Ingress Controller deployed in bare-metal mode.

Traffic flow:

User → EC2 Public IP → NodePort (30080/30443) → Ingress-NGINX → ClusterIP Service → Pod

This approach eliminates the need for MetalLB and avoids AWS LoadBalancer service type, making it cost-efficient and portable.

TLS certificates are provisioned automatically using cert-manager with Let’s Encrypt ACME challenge.

This ensures:

- Encrypted HTTPS communication
- Automated certificate renewal
- Production-style edge exposure

---

## Governance & Access Control

The platform enforces strong workload governance using native Kubernetes mechanisms.

### Namespaces

Separate namespaces were created for development, staging, and production environments to ensure logical isolation and environment-level control.

### RBAC

Role-Based Access Control was implemented to enforce least-privilege access.

Examples:
- Developer read-only roles in development namespace
- Deployment privileges scoped to specific environments
- Restricted cluster-admin access

This prevents accidental or unauthorized resource modification.

### Resource Management

ResourceQuota and LimitRange policies were applied per namespace to:

- Prevent resource exhaustion
- Enforce CPU and memory boundaries
- Control pod count limits

---

## Configuration & Secrets Management

Configuration was externalized using:

- ConfigMaps for environment-specific settings
- Secrets for sensitive data (database credentials, tokens)

This decouples application configuration from container images and improves security posture.

---

## Persistent Storage

PersistentVolumeClaims backed by AWS EBS were implemented for stateful workloads.

This ensures:

- Durable storage
- Pod restarts without data loss
- Stateful application support

---

## Reliability & Health Management

Application reliability was enhanced using:

- Liveness Probes (automatic restart on failure)
- Readiness Probes (traffic only routed to healthy pods)

This guarantees high availability and zero-downtime deployments.

---

## Horizontal Scalability

Horizontal Pod Autoscaler (HPA) was configured using metrics-server.

Scaling policy:
- Minimum replicas: 2
- Maximum replicas: 6
- CPU threshold: 50%

This enables automatic scaling under load and resource optimization during low traffic.

---

## Observability

Full-stack observability was implemented.

### Monitoring

Prometheus and Grafana (via Helm) were deployed to monitor:

- Node-level metrics
- Pod-level resource usage
- Kubernetes object states
- Autoscaling behavior

Grafana dashboards provide real-time infrastructure visibility and capacity planning insights.

---

## Centralized Logging

ELK Stack was deployed for centralized log aggregation.

Components:
- Elasticsearch (log storage)
- Kibana (visualization)
- Filebeat (DaemonSet log shipping)

This allows:

- Cluster-wide log search
- Incident troubleshooting
- Application debugging

---

## Disaster Recovery

etcd snapshot-based backup strategy was implemented to protect cluster state.

Features:
- Periodic etcd snapshots
- Documented restore procedure
- Recovery validation testing

This ensures control-plane resilience and disaster recovery preparedness.

---

## Release Management

Helm was used as the Kubernetes package manager for application lifecycle management.

Capabilities demonstrated:

- Parameterized deployments using values files
- Atomic upgrades
- Version tracking
- Instant rollback capability

This ensures safe production releases and controlled change management.

---

## Key Technical Capabilities Demonstrated

- kubeadm-based Kubernetes deployment
- EC2 public IP ingress exposure without MetalLB
- TLS automation with cert-manager
- Namespace isolation & RBAC enforcement
- ResourceQuota & LimitRange governance
- ConfigMap & Secret management
- Persistent storage integration
- Health probes & autoscaling
- Prometheus + Grafana monitoring
- ELK centralized logging
- etcd backup & restore strategy
- Helm-based release lifecycle management

---

## Conclusion

KubeEnterprise demonstrates practical Kubernetes platform engineering skills aligned with real-world production environments.  

The project integrates infrastructure design, security governance, workload scalability, observability, disaster recovery, and release management into a cohesive and operational Kubernetes ecosystem.

This implementation reflects end-to-end DevOps capabilities required for managing production-grade Kubernetes environments.
