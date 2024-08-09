# Production-grade Kubernetes environment

## Requirements

1. Ubuntu 24.04 LTS, and K8s 1.30.3
2. Cloud agnostic terms, and architecture

## Approach

I always try to put emphasis on the usage of OSS tooling not only because of extensive experience with them
but also:

1. They prevent vendor lockin.
2. Maximum control

## Assumptions

1. Adequate SRE/DevOps/Platform team strength. The major downside of using OSS is that you need a lot of
expertise in running, and fine tuning them. Businesses must be ready to invest time, and money over this.
2.

## Layers

In terms of tooling, I would like to split the on premise infrastructure into 4 different layers:

### 1. Layer 0 (Physical Infrastructure)

This includes servers (either bare metal or VMs based on VMWare ESXi/Proxmox), Routers, Switches, iDRAC etc. For K8s
specifically, my recommendation is always:
a) 3 Controlplane nodes at minumum for HA. Master nodes will have lower specs compare to worker nodes because workloads won't
  schedule here.
b) 3 Worker nodes at minimum. Worker nodes' compute and storage can vary depending on the type of workloads. E.g ML
  related require GPU cards, virtualization support etc. **Such nodes should be tainted with specific K8s labels, and
  only tolerated by ML related Deployments**.
c) 3 separate nodes of DB for database. Management of stateful workloads on K8s clusters is difficult. Keeping
d) **ETCD will be part of Controlplane nodes, for simplicity's sake**.

### 2. Layer 1 (Host Setup)

This layer further devides into 2 parts:

a) Host OS (Ubuntu 22.04)
b) K8s 1.30.3 using Kubespary. Other options like K3s and RK2 are usually 2-3 versions behind the upstream K8s where Kube-
  spary always provides latest version first.
c) KeepAliveD for handling multi controlplane setup.

### 3. Core service setup

These include:

1. Nginx Ingress for handling ingress traffic.
2. Vault for secret management (Hashicorp Vault)
3. ArgoCD for Continous Deploment.
4. CertManager for generting & rotating TLS certificates.
5. Observability stack with Grafana (Metrics), and Elastic Stack (with Filebeat for logging).
6. Database (Postgres/MySQL) running outside of K8s cluster.
7. Velero for cluster (ETCD) backup.
8. MetalLB for Load Balancing.
9. Redis (non cluster modes) for caching within K8s.
10. CephFS distributed storage as CSI.

### 4. Workload Specific Services

These include application specific tooling like:

1. Apache Spark
2. Apache Airflow
3. NFS CSI.

Such services will be only deployed on specefic clusters, and/or specific worker nodes.

## Automation

Every step of boostraping K8s cluster will be automated using Ansible, Bash and Helm charts.
