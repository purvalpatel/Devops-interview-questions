## Design a production Kubernetes platform for 500–1000 bare-metal servers. Explain every major component.

For 500–1000 bare-metal servers, 
- I would not design Kubernetes as one giant flat cluster.
- I would design a multi-cluster platform with centralized management, standardized node provisioning, strong observability, GitOps, automated upgrades, and separate pools for general compute, GPU, storage, and specialized workloads.

A good production architecture looks like this:
```
                         ┌─────────────────────────────┐
                         │       Developers / Users     │
                         └──────────────┬──────────────┘
                                        │
                              Git / CI / API / Portal
                                        │
                                        ▼
                    ┌───────────────────────────────────┐
                    │       Platform Management          │
                    │                                   │
                    │ GitLab/GitHub   Argo CD           │
                    │ Vault            Backstage         │
                    │ Prometheus       Grafana            │
                    └───────────────┬───────────────────┘
                                    │
              ┌─────────────────────┼──────────────────────┐
              │                     │                      │
              ▼                     ▼                      ▼
       ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
       │ K8s Cluster │       │ K8s Cluster │       │ K8s Cluster │
       │ Production  │       │ GPU / AI    │       │ Platform    │
       │             │       │             │       │ Services    │
       └──────┬──────┘       └──────┬──────┘       └──────┬──────┘
              │                     │                      │
              ▼                     ▼                      ▼
        100–### 300 nodes          GPU nodes              Infra nodes
```
The exact cluster count depends on workload isolation, failure domains, compliance, and upgrade requirements.

### 1. First principle: don't build one 1000-node cluster by default

Kubernetes can technically handle very large clusters, but operationally I would prefer something like:
```
Cluster 1: Production applications       150 nodes
Cluster 2: Production applications       150 nodes
Cluster ### 3: Data/Stateful workloads       100 nodes
Cluster 4: GPU/AI                        100 nodes
Cluster 5: GPU/AI                        100 nodes
Cluster 6: Dev/Test                      100 nodes
Cluster 7: Platform services              50 nodes
```

#### Why?

- Failure isolation

If one cluster has a serious control-plane or networking problem, you don't lose the entire platform.

- Upgrade isolation

You can upgrade:
```
Cluster A → Kubernetes 1.### 34
Cluster B → Kubernetes 1.### 3### 3
```
and progressively migrate workloads.

- Workload isolation

You don't necessarily want:
```
Kafka
LLM inference
CI runners
customer applications
GPU training
monitoring
```
all competing inside the same scheduling domain.

- Blast-radius reduction

A bad:
```
DaemonSet
CNI change
NetworkPolicy
CRD
Admission webhook
Operator
```
shouldn't impact your entire infrastructure.

### 2. Bare-metal provisioning

This is one of the most important components.

At 1000 servers, you cannot manually install Linux.

I'd build:
```
        PXE / iPXE
            │
            ▼
     Provisioning system
            │
       ┌────┴────┐
       │         │
    BMC/IPMI   DHCP
       │         │
       └────┬────┘
            ▼
       OS installation
            │
            ▼
      Configuration
            │
            ▼
       Kubernetes
```
Possible technologies:
```
PXE/iPXE
MAAS
Foreman
OpenStack Ironic
Tinkerbell
Ansible
Terraform
vendor BMC APIs
```
For a large bare-metal environment, I'd strongly consider MAAS/Ironic/Tinkerbell + Ansible or an equivalent automated provisioning stack.

### ### 3. Hardware inventory

You need a source of truth for every machine.

For example:
```
Server ID
Rack
Row
Datacenter
BMC IP
NIC MAC
CPU
RAM
GPU
Disk
NIC speed
OS
Kubernetes cluster
Kubernetes version
Node role
```
Something like:
```
Server
 ├── hardware inventory
 ├── rack information
 ├── network information
 ├── OS state
 └── Kubernetes state
```
This becomes extremely important for 500–1000 machines.

### 4. Operating system

I would standardize the OS as much as possible.

For example:
```
Ubuntu / RHEL / Rocky / Flatcar
```
depending on organizational requirements.

Every node should have:
```
Kernel
container runtime
NTP/chrony
systemd
iptables/nftables
CSI prerequisites
GPU drivers where required
monitoring agent
logging agent
security agent
```
configured automatically.

### 5. Configuration management

Use Ansible or an equivalent system.

Don't SSH into 1000 servers manually.

Example:
```
Git
 │
 ▼
Ansible
 │
 ├── OS configuration
 ├── kernel parameters
 ├── users
 ├── SSH
 ├── sysctl
 ├── container runtime
 ├── GPU drivers
 ├── monitoring
 └── Kubernetes prerequisites
```
Everything should be reproducible.

### 6. Container runtime

Kubernetes should use a standard CRI runtime.

For example:
```
containerd
```
Architecture:
```
Kubernetes
     │
     ▼
   kubelet
     │
     ▼
 containerd
     │
     ▼
   runc
     │
     ▼
 Containers
```
I would avoid maintaining different runtimes unless there's a concrete requirement.

### 7. Kubernetes control plane

Each production cluster should have multiple control-plane nodes.

For example:

              Load Balancer
                   │
        ┌──────────┼──────────┐
        │          │          │
     Master-1   Master-2   Master-### 3
        │          │          │
        └──────────┼──────────┘
                   │
                 etcd

Typically:

### 3 control-plane nodes

for smaller clusters, potentially:

5 control-plane nodes

for larger/high-criticality clusters.

I would not use an even number such as 4 for etcd.

### 8. etcd

etcd stores Kubernetes state:
```
Deployments
Pods
Services
Secrets
ConfigMaps
CRDs
Nodes
RBAC
```
For production:

### 3 or 5 members

and ideally:

Control-plane failure domain ≠ single rack

For example:
```
Rack A → etcd-1
Rack B → etcd-2
Rack C → etcd-### 3
```
You also need:
```
automated backups
backup verification
restore testing
monitoring
disk latency monitoring
```
An etcd backup that has never been restored is not a proven backup.

### 9. API Server load balancing

At scale:
```
kubectl
  │
  ▼
VIP / Load Balancer
  │
  ├── API-Server-1
  ├── API-Server-2
  └── API-Server-### 3
```
For bare metal, possibilities include:
```
HAProxy
Keepalived
F5
hardware load balancer
MetalLB in appropriate designs
```
I like having a stable:

k8s-api.example.com:644### 3

endpoint.

### 10. CNI / networking

This is one of the biggest design decisions.

I'd evaluate:
```
Cilium
Calico
```
For a modern large cluster, Cilium is particularly attractive because it gives:
```
CNI
NetworkPolicy
eBPF
Service networking
Observability
Load balancing
```
Architecture:
```
Pod
 │
 ▼
CNI
 │
 ▼
Node network
 │
 ▼
Leaf/Spine network
```
### 11. Physical network

At 500–1000 bare-metal servers, networking should normally use a leaf-spine architecture.
```
              Spine
          /     |     \
       Leaf    Leaf    Leaf
       /|\     /|\     /|\
      / | \   / | \   / | \
   Servers  Servers  Servers
```
Avoid creating a huge Layer-2 domain unnecessarily.

Typical separation:
```
Management network
Storage network
Kubernetes/node network
Application network
BMC/IPMI network
GPU/RDMA network
```
depending on workload.

### 12. Storage

This needs careful design.

Do not assume Kubernetes itself provides storage.

You need:
```
Kubernetes
     │
     ▼
    CSI
     │
 ┌───┼──────────────┐
 │   │              │
NFS NetApp       Ceph
```
Possible backend:
```
NetApp Trident
Ceph/Rook
SAN
NFS
local NVMe
cloud-like storage appliance
```
Different workloads need different storage classes.

For example:
```
fast-local-nvme
fast-network
general-purpose
database
archive
```
### 13. Local storage

For workloads such as:
```
AI
ML
HPC
temporary processing
high-performance caches
```
local NVMe can be valuable.

But remember:

> local storage ≠ highly available storage

If the node dies:
```
Node dies
   ↓
Local NVMe unavailable
```
So local storage should only be used when the application architecture understands that failure mode.

### 14. GPU architecture

For your environment, I'd create dedicated GPU node pools.

GPU Cluster
```

 ┌────────────────────────────┐
 │ NVIDIA GPU nodes           │
 │                            │
 │ H100 / H200 / RTX / etc.   │
 └────────────────────────────┘


 ┌────────────────────────────┐
 │ AMD GPU nodes              │
 │                            │
 │ MI### 300 / MI-series          │
 └────────────────────────────┘
```
Node labels:
```
gpu.vendor: nvidia
gpu.type: h100
gpu.memory: 80gb
```
and:
```
gpu.vendor: amd
gpu.type: mi### 300
```
Then use:
```
NVIDIA GPU Operator/device plugin
AMD GPU device plugin
MIG where appropriate
HAMi where fractional GPU scheduling is appropriate
```

### 15. GPU scheduling

A workload might request:
```
resources:
  limits:
    nvidia.com/gpu: 2
```
Scheduler then determines:

- Which node?
- Which GPUs?
- Is there enough capacity?
- Are topology constraints satisfied?

For advanced AI workloads, you may additionally need:
```
GPU topology
NUMA topology
PCIe topology
NVLink
RDMA
GPU memory
MIG
```
This becomes much more important for multi-GPU training.

### 16. RDMA / high-performance networking

For GPU/HPC workloads:
```
GPU
 │
PCIe
 │
NIC
 │
RDMA
 │
High-speed fabric
 │
Other GPU nodes
```
You may need:
```
RDMA
RoCE
InfiniBand
SR-IOV
Multus
NVIDIA Network Operator
```
For distributed training, this can be as important as the GPU itself.

### 17. Node pools

Don't put every machine into one pool.

For example:
```
                    Kubernetes
                        │
       ┌────────────────┼─────────────────┐
       │                │                 │
   General           GPU               Storage
   nodes             nodes              nodes
       │                │                 │
  CPU workloads    AI/ML workloads    DB/storage
```
Use:
```
labels
taints
tolerations
affinity
topology spread
```
Example:
```
gpu=true:NoSchedule
```
means ordinary workloads won't accidentally consume expensive GPU nodes.

### 18. Scheduling

Kubernetes scheduler handles basic placement.

But at this scale, scheduling policies become important.

Use:
```
node affinity
pod affinity
pod anti-affinity
topology spread constraints
taints/tolerations
priority classes
resource requests/limits
PDBs
```
For example:
```
Replica 1 → Rack A
Replica 2 → Rack B
Replica ### 3 → Rack C
```
instead of:
```
Replica 1 → Rack A
Replica 2 → Rack A
Replica ### 3 → Rack A
```

### 19. Ingress

You need an ingress/load-balancing layer.

Typical architecture:
```
Internet / Internal users
          │
          ▼
    Load Balancer
          │
          ▼
      Ingress
          │
          ▼
       Service
          │
          ▼
        Pods
```
Possible technologies:
```
NGINX
HAProxy
Envoy
Traefik
Gateway API implementations
```
For a new platform, I'd seriously consider Gateway API rather than designing everything around the older Ingress model.

### 20. DNS

You need highly available DNS.

Inside Kubernetes:

CoreDNS

Outside:

Corporate DNS

Architecture:
```
Application
    │
    ▼
Service DNS
    │
    ▼
CoreDNS
```
At this scale, CoreDNS itself needs:
```
replicas
resource limits
monitoring
autoscaling
NodeLocal DNSCache where appropriate
```
### 21. Service discovery

Kubernetes provides:
```
service-name.namespace.svc.cluster.local
```
For external service discovery, integrate with your organization's DNS/service discovery system.

### 22. Identity and RBAC

Do not give developers:

cluster-admin

Use:
```
OIDC
   ↓
Identity provider
   ↓
Groups
   ↓
Kubernetes RBAC
```
For example:
```
team-a
  ↓
namespace-a
  ↓
read/write

team-b
  ↓
namespace-b
  ↓
read/write
```

Platform administrators have broader permissions.

### 23. Secrets

Do not put production credentials directly into Git.

I'd use something like:
```
Vault
  │
  ▼
External Secrets
  │
  ▼
Kubernetes Secret
```
or an equivalent secrets-management system.

Secrets include:
```
database passwords
API keys
TLS certificates
cloud credentials
registry credentials
```

24. Image registry

You need an internal registry.

For example:

Developer
   ↓
CI
   ↓
Image build
   ↓
Internal Registry
   ↓
Kubernetes

The registry should support:

vulnerability scanning
image signing
retention policies
replication
access control

Examples include Harbor and enterprise registries.

### 25. CI/CD

For your background, I'd design:

GitLab/GitHub
      │
      ▼
   CI Pipeline
      │
      ├── Unit tests
      ├── Security scan
      ├── Image build
      ├── Image scan
      └── Push registry
             │
             ▼
           GitOps
             │
             ▼
           Argo CD
             │
             ▼
        Kubernetes

The important separation is:

CI builds artifacts.

GitOps deploys artifacts.

### 26. GitOps

I would use:

Argo CD

or Flux.

Repository structure could look like:

platform/
├── clusters/
│   ├── prod-01/
│   ├── prod-02/
│   ├── gpu-01/
│   └── dev-01/
│
├── infrastructure/
│   ├── ingress/
│   ├── monitoring/
│   ├── storage/
│   └── security/
│
└── applications/
    ├── app-a/
    ├── app-b/
    └── app-c/

Now your cluster becomes reproducible.

### 27. Monitoring

I would use:

Prometheus
    │
    ├── Kubernetes metrics
    ├── Node metrics
    ├── Application metrics
    ├── GPU metrics
    └── Storage metrics
          │
          ▼
       Grafana

For node monitoring:

node_exporter

For Kubernetes:

kube-state-metrics

For NVIDIA:

DCGM exporter

For AMD:

ROCm-compatible exporters/metrics.

### 28. Logging

Centralized logging is mandatory.

Pods
 │
 ▼
Fluent Bit / Vector
 │
 ▼
Kafka / directly
 │
 ▼
Elasticsearch / OpenSearch / Loki
 │
 ▼
Grafana / Kibana

Don't depend on individual nodes' local logs.

If a node dies:

Node gone
→ logs should still exist

### 29. Distributed tracing

For microservices:

Application
   │
   ▼
OpenTelemetry
   │
   ▼
Collector
   │
   ▼
Tempo / Jaeger / compatible backend

Now you can trace:

Request
 ↓
Ingress
 ↓
Service A
 ↓
Service B
 ↓
Database

### ### 30. Alerting

Prometheus:

Prometheus
    │
    ▼
Alertmanager
    │
    ├── Slack
    ├── Email
    ├── PagerDuty
    └── Incident system

Don't alert on everything.

Use SLO-based alerts.

Examples:

API availability
Pod crash rate
Node failure
GPU ECC errors
GPU temperature
etcd latency
etcd space
CNI failures
PVC failures
control-plane latency
### ### 31. Security

I would have multiple layers.

                    Security
                       │
       ┌───────────────┼────────────────┐
       │               │                │
   Identity        Network           Runtime
       │               │                │
    OIDC/RBAC     NetworkPolicy      Seccomp
    Vault         Cilium             AppArmor
    MFA           Firewall           SELinux

Also:

image scanning
SBOM
image signing
admission policies
Pod Security Standards
vulnerability scanning
CIS benchmarks
audit logs
### 32. Admission control

Before a workload enters the cluster:

kubectl apply
      │
      ▼
API Server
      │
      ▼
Admission
      │
      ├── Policy
      ├── Security
      ├── Image verification
      ├── Resource limits
      └── Namespace rules
      │
      ▼
Scheduler

Tools could include:

Kyverno
Gatekeeper
ValidatingAdmissionPolicy
### 33. Resource governance

At 1000 servers, resource abuse becomes a serious problem.

Use:

ResourceQuota
LimitRange
PriorityClass
Requests
Limits
PDB

For example:

Team A
CPU quota = 20,000
Memory quota = 100 TB
GPU quota = ### 32

This prevents one team from consuming the entire platform.

### 34. GPU capacity automation

Coming back to your previous question, this platform should also automate GPU capacity.

                 GPU workloads
                       │
                       ▼
              Queue / application
                       │
                       ▼
                Prometheus
                       │
                       ▼
                 KEDA/HPA
                       │
                       ▼
                 GPU replicas
                       │
                       ▼
                 Kubernetes
                       │
                       ▼
                GPU scheduler

And at the cluster level:

GPU capacity exhausted
          │
          ▼
Pending GPU workload
          │
          ▼
Provisioning system
          │
          ▼
New bare-metal GPU node
          │
          ▼
GPU Operator/device plugin
          │
          ▼
Node becomes Ready
          │
          ▼
Workload scheduled
### ### 35. Autoscaling bare metal

This is more complicated than cloud Kubernetes.

Cloud:

Pod pending
 ↓
Cloud API
 ↓
New VM
 ↓
Node joins

Bare metal:

Pod pending
 ↓
Need GPU/CPU capacity
 ↓
Find available physical server
 ↓
BMC/IPMI
 ↓
PXE
 ↓
OS installation
 ↓
Ansible
 ↓
containerd
 ↓
kubeadm/join
 ↓
Node labels
 ↓
Node Ready
 ↓
Pod scheduled

That's why the provisioning platform is a critical part of your architecture.

### 36. Kubernetes upgrades

Never upgrade 1000 nodes simultaneously.

Use:

Cluster 1
  ↓
Canary


Cluster 2
  ↓
Canary


Cluster ### 3
  ↓
Progressive rollout

For nodes:

Cordon
   ↓
Drain
   ↓
Upgrade
   ↓
Validate
   ↓
Uncordon

Automate this with controlled concurrency.

### 37. Disaster recovery

You need DR at several layers.

Kubernetes state
etcd backup

or a supported Kubernetes backup mechanism such as Velero for application resources.

Application data

Handled by the storage/database backup strategy.

GitOps

Your desired state is already in Git.

Infrastructure

Provisioning/configuration should be reproducible.

The goal is:

Hardware failure
       ↓
Provision replacement
       ↓
Install OS
       ↓
Join cluster
       ↓
Argo CD
       ↓
Applications return
### 38. Multi-datacenter architecture

If these 1000 servers span multiple datacenters, I would not automatically stretch one Kubernetes cluster across all sites.

Prefer:

             Global Layer
                  │
       ┌──────────┼──────────┐
       │          │          │
     DC-1       DC-2       DC-### 3
       │          │          │
   Cluster(s) Cluster(s) Cluster(s)

This dramatically reduces failure-domain complexity.

### 39. Platform API / self-service

At this scale, developers shouldn't need to understand all of Kubernetes.

Give them:

Developer Portal
       │
       ▼
"Create application"
       │
       ├── CPU
       ├── Memory
       ├── GPU
       ├── Storage
       ├── Environment
       └── Region/DC
       │
       ▼
GitOps
       │
       ▼
Kubernetes

Backstage can be useful here.

40. The final architecture

Putting everything together:

                         USERS
                           │
                    Developer Portal
                           │
             ┌─────────────┴─────────────┐
             │                           │
           Git                        CI/CD
             │                           │
             │                      Build/Test
             │                           │
             │                       Registry
             │                           │
             └──────────────┬────────────┘
                            │
                         GitOps
                        Argo CD
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
   K8s PROD              K8s GPU              K8s DEV
   Cluster(s)            Cluster(s)            Cluster(s)
       │                    │                    │
       │                    │                    │
 ┌─────┼──────┐       ┌─────┼──────┐       ┌────┼─────┐
 │     │      │       │     │      │       │    │     │
CPU  Storage  App    NVIDIA AMD   RDMA    CPU  GPU   App
 │     │      │       │     │      │       │    │     │
 └─────┴──────┘       └─────┴──────┘       └────┴─────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                       Observability
                            │
            ┌───────────────┼────────────────┐
            │               │                │
        Prometheus       Grafana           Loki
            │                                │
        Alertmanager                    Logs/Traces
            │
            ▼
        On-call/SRE




                  BARE-METAL PLATFORM
                           │
             ┌─────────────┼─────────────┐
             │             │             │
           DHCP           PXE           BMC
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                     OS Provisioning
                           │
                         Ansible
                           │
                           ▼
                       Kubernetes
                           │
                           ▼
                   500–1000 Servers
