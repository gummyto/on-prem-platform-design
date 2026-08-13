# Infrastructure Architecture (Working Draft)

## Architecture Principles (Draft)

- Simplicity over complexity.
- Kubernetes First.
- Infrastructure as Code First.
- Open Source First.
- High Availability only where it adds real business value.
- Minimize operational overhead.
- Everything should be replaceable.
- Everything should be reproducible.

---

> Status: Working document (Iteration 1)

## Project Scope

Initial architecture for the company's on-prem infrastructure. The first phase targets **Development** and **Staging** environments. Production-grade optimizations will be introduced later where justified.

---


## Solution Technology Stack

- Proxmox VE
  - Proxmox unattended installation
  - Terraform / OpenTofu
  - Terragrunt
  - Ansible
  - cloud-init
- Kubernetes
  - kubeadm
  - Cilium CNI
    - Gateway API implementation
    - NetworkPolicy
    - Hubble (future evaluation)
  - kube-vip
    - Kubernetes API VIP
    - Service type LoadBalancer VIPs
  - cert-manager
  - ExternalDNS (conditional)
  - NFS CSI
  - External Secrets Operator
  - Metrics Server
  - Jenkins
    - Kubernetes ephemeral agents
    - JCasC
    - HashiCorp Vault plugin
  - GitLab CI/CD or GitHub Actions (candidate alternatives to Jenkins)
  - Argo CD
    - GitOps deployment reconciliation
    - Helm chart deployment/lifecycle management
  - JFrog Artifactory
  - Database operator (TBD)
  - Velero (optional)
- HashiCorp Vault
  - Kubernetes Auth
- Git
- Helm
- Observability / Monitoring
  - Prometheus — metrics collection and Prometheus-compatible scraping/query model
  - VictoriaMetrics — metrics storage / scalable Prometheus-compatible backend option
  - Grafana — dashboards and visualization across metrics/logging data sources
  - node_exporter — Linux host/VM OS metrics
  - IPMI exporter — bare-metal hardware/BMC metrics where supported
  - Proxmox VE API/exporter integration — Proxmox cluster/node/VM/storage metrics (implementation TBD)
- Alerting
  - Alertmanager — alert grouping, deduplication, routing and notification orchestration
  - PagerDuty — external incident/on-call notification and escalation integration
- Logging
  - OpenSearch — centralized log indexing, search and analytics option
  - Graylog — centralized log management/search option
  - Elasticsearch — centralized log indexing/search option
  - systemd-journald / rsyslog — Proxmox and Linux VM log sources/forwarding

## Table of Contents

- [Layer 1 – Hypervisor & Physical Infrastructure](#layer-1--hypervisor--physical-infrastructure)
  - [1. Architecture Goals](#1-architecture-goals)
  - [2. Proxmox Cluster](#2-proxmox-cluster)
  - [3. Networking](#3-networking)
  - [4. Storage](#4-storage)
- [Layer 2 – Virtual Machines & Infrastructure Services](#layer-2--virtual-machines--infrastructure-services)
  - [5. Virtual Machine Layout](#5-virtual-machine-layout)
- [Layer 3 – Kubernetes Platform & Operations](#layer-3--kubernetes-platform--operations)
  - [6. Kubernetes Platform Services](#6-kubernetes-platform-services)
  - [7. Backup & Recovery](#7-backup--recovery)
  - [8. Observability](#8-observability)
  - [9. Monitoring](#9-monitoring)
  - [10. Logging](#10-logging)
  - [11. Alerting](#11-alerting)
- [Layer 4 – Application Delivery & Runtime](#layer-4--application-delivery--runtime)
  - [12. GitOps](#12-gitops)
  - [13. CI/CD](#13-cicd)
  - [14. Release Process](#14-release-process)
  - [15. Artifact Registry](#15-artifact-registry)
  - [16. Databases](#16-databases)
  - [17. Security & Access Management](#17-security--access-management)
  - [18. Environment Strategy](#18-environment-strategy)
- [Layer 5 – AI Integration](#layer-5--ai-integration)
  - [19. AI Integration](#19-ai-integration)
- [Open Questions](#open-questions)

# Layer 1 – Hypervisor & Physical Infrastructure

## 1. Architecture Goals

- Unified cluster management
- High availability where appropriate
- Simple operations for Dev/Stage
- Easy future migration to Production
- Infrastructure as Code

---

## 2. Proxmox Cluster

### Decisions
- **5 physical servers**
- **Single Proxmox Cluster**
- **Symmetric cluster** (all nodes equal)
- No dedicated node roles initially
- Service specialization will happen on VM/Kubernetes level

### Open Questions
- Future hardware specialization if required

---

## 3. Networking

### 3.1 Management Network
**Decision**
- Dedicated management network
- Prefer dedicated NIC
- VLAN if hardware requires it

### 3.2 Corosync Network
**Decision**
- Dedicated network
- Prefer dedicated NIC
- Separate from VM/Storage traffic

**Open Questions**
- Number of NICs
- Datacenter networking capabilities

### 3.3 VM Network
**Decision**
- Single VM network initially
- VLAN segmentation later if needed

### 3.4 Storage Network
**Decision**
- Uses infrastructure network initially
- Future optimization possible

### 3.5 Live Migration
**Decision**
- No dedicated network for Dev/Stage
- Uses VM network initially

### 3.6 Backup Network
**Decision**
- No dedicated backup network for Dev/Stage

### 3.7 Network Addressing
Status: Deferred

---

## 4. Storage

### Option 1 (Current Preferred)

**Local NVMe + NFS**

Purpose:
- Maximum simplicity
- Fast deployment
- Easy maintenance

Storage Usage

Local NVMe
- Ephemeral Kubernetes volumes
- Stateless workloads
- Temporary storage
- Build cache
- CI cache

Shared Storage (NFS)
- Stateful workloads
- Persistent Volumes
- VM disks requiring HA
- Live Migration support

Advantages
- Very simple architecture
- Easy troubleshooting
- Low operational overhead

---

### Option 2 (Development Optimized)

**Local NVMe for both Ephemeral and Stateful workloads**

Purpose:
- Use local NVMe for both ephemeral and stateful workloads during Development.

Requirements
- Local disks configured as RAID1 (Mirror)
- Regular backups
- Recovery procedures documented
- Acceptable downtime for Dev/Stage

Advantages
- Excellent performance
- Very simple infrastructure
- No external storage dependency

Disadvantages
- No automatic HA
- Live Migration limitations
- Restore required after node failure

Suitable for
- Development
- Testing
- CI

---

### Future Evaluation

- Ceph
- ZFS Replication

# Layer 2 – Virtual Machines & Infrastructure Services


- iSCSI
- Enterprise Shared Storage

## 5. Virtual Machine Layout

### Design Principles

- Minimum number of VMs
- Kubernetes First
- Infrastructure services should run inside Kubernetes whenever practical

### Initial VM List

### 5.1 Infrastructure Bootstrap & Automation

Purpose:
- Make the Proxmox platform reproducible from bare metal to VM provisioning.

Initial approach:
- Proxmox unattended installation for bare-metal bootstrap.
- Ansible for host configuration and Proxmox cluster creation/join.
- Terraform/OpenTofu for VM provisioning and API-managed Proxmox resources.
- Cloud-init for minimal VM bootstrap.
- Ansible for full guest configuration.

Bootstrap flow:
Bare Metal → Proxmox unattended install → Ansible → Proxmox Cluster → Terraform/OpenTofu → Infrastructure Management VM → cloud-init → Ansible → Kubernetes / platform services

Open Questions:
- Exact unattended installation workflow and answer-file design.
- Final Terraform/OpenTofu provider choice.
- Which Proxmox settings remain intentionally manual, if any.

### 5.2 Infrastructure Management VM

Purpose:
- Reproducible bootstrap, recovery and manual administration environment.
- Normal day-to-day automation is executed by Jenkins running in Kubernetes.

Responsibilities:
- Bootstrap new infrastructure
- Break-glass administration (administration fallback)
- Disaster recovery
- Manual execution of infrastructure workflows when Jenkins is unavailable

Execution Model:
- Normal execution: Jenkins with ephemeral Kubernetes agents
- Fallback execution: Infrastructure Management VM
- Both execution environments use the same Infrastructure as Code repositories

Tooling:
- Terraform / OpenTofu
- Terragrunt
- Ansible
- kubectl
- Helm
- Git
- jq / yq
- Python
- Bash

Tool Version Management:
- Management VM and Jenkins Infrastructure Agents must use identical tool versions.
- Tool versions are managed and synchronized from Git as the single source of truth.
- Jenkins agent images are rebuilt when tool versions change.
- Infrastructure Management VM is updated through Ansible using the same version definitions.

Notes:
- Not HA
- Reliable backup required
- Infrastructure code must remain in Git

### 5.3 Vault VM

Purpose:
- Provide centralized secrets management early in the infrastructure lifecycle.
- Remain independent from both the Infrastructure Management VM and the Kubernetes cluster.

Decision:
- Deploy HashiCorp Vault on a dedicated VM outside Kubernetes.
- Do not run Vault inside the Infrastructure Management VM.
- Infrastructure Management VM contains Vault client/admin tooling only.
- Kubernetes, Jenkins and platform services consume secrets from Vault.
- For Dev/Stage, start with a single Vault VM with persistent storage and reliable backup.
- Evaluate a multi-node Vault cluster with integrated storage/Raft for Production.

Bootstrap order:
Proxmox → Infrastructure Management VM → Vault VM → Vault initialization/configuration → Kubernetes → Jenkins → Applications / platform services

Open Questions:
- Vault bootstrap credentials and unseal/auto-unseal strategy.
- Backup and restore procedure for Vault data.

### 5.4 Kubernetes Control Plane

Decision:
- 3 dedicated Kubernetes Control Plane VMs for HA and quorum.
- Use stacked etcd on the Control Plane nodes for Dev/Stage simplicity.
- Control Plane VMs do not host normal application workloads.
- Provision VMs with Terraform/OpenTofu/Terragrunt and configure/bootstrap Kubernetes with Ansible + kubeadm. Kubernetes versions and upgrade configuration are managed from Git; rolling upgrades are orchestrated through Jenkins/Ansible, with the Infrastructure Management VM available as the administration fallback.
- Use kube-vip for a stable highly available kube-apiserver VIP / controlPlaneEndpoint.
- Start with kube-vip Layer 2 / ARP mode for Dev/Stage; evaluate more advanced networking only if required later.
- etcd HA does not replace backups. Take periodic etcd snapshots from one control-plane node and store them outside the Control Plane VMs.
- A single valid etcd snapshot is sufficient for cluster-state disaster recovery; there is no need to back up every etcd member independently.
- kubeadm upgrade creates local etcd backups as part of the upgrade safety process, but these do not replace the regular etcd snapshot strategy.
- External etcd remains a future Production option.

Bootstrap order:
Terraform/OpenTofu/Terragrunt provisions 3 Control Plane VMs → cloud-init minimal bootstrap → Ansible configures OS, container runtime and Kubernetes prerequisites → kube-vip static pod provides the API VIP / controlPlaneEndpoint → kubeadm initializes the first Control Plane node with stacked etcd → CNI is installed → remaining Control Plane nodes join with kubeadm → cluster health and etcd quorum are validated → regular etcd snapshot automation is enabled
### 5.5 Kubernetes Workers

Worker nodes are organized into logical node groups by workload type (initially applications, platform and monitoring/observability).

Option A — Initial / Preferred:
Terraform / Terragrunt declaratively manages the worker groups and replica counts in Git. Proxmox provides the VMs; Ansible + kubeadm configure and join new workers to Kubernetes. Jenkins executes the provisioning workflow. Scaling is performed by changing the desired replica count in Git and applying the infrastructure code. This provides a simple declarative node-group lifecycle without automatic node autoscaling.

Initial example:
- applications workers: 3
- platform workers: 2
- monitoring workers: 2

Option B — Future / More Advanced:
Evaluate Cluster API Provider for Proxmox together with Kubernetes Cluster Autoscaler to provide cloud-like dynamic worker lifecycle and node autoscaling. This is intentionally deferred until the additional operational complexity is justified.

# Layer 3 – Kubernetes Platform & Operations




## 6. Kubernetes Platform Services
Installation & lifecycle model:
- Prefer official/community-supported Helm charts for Kubernetes platform services where available.
- Argo CD manages those Helm-based deployments declaratively from Git and remains the normal lifecycle/reconciliation mechanism.
- Direct Helm installation is reserved for initial bootstrap components that must exist before Argo CD is available, or for administration fallback/recovery.


### 6.1 Kubernetes Networking & Gateway

Decision:
- Use Cilium as the preferred Kubernetes CNI.
- Use Cilium as the preferred Kubernetes Gateway API implementation, avoiding a separate ingress controller initially.
- Gateway API is the standard Kubernetes routing API; Cilium provides the controller/data-plane implementation.
- Initial Cilium scope: pod networking, Kubernetes NetworkPolicy, and Gateway API.
- Do not enable Service Mesh initially. Evaluate Cilium Service Mesh capabilities later only if justified by microservice requirements such as mTLS, service identity, advanced east-west traffic management, or additional observability.
- Evaluate Hubble for network observability during the detailed Kubernetes design.

### 6.2 Kubernetes Load Balancer Services

Decision:
- Use kube-vip as the preferred bare-metal implementation for Kubernetes Service type LoadBalancer.
- Use the same kube-vip solution for both the Kubernetes API VIP and LoadBalancer service VIPs, while keeping their IP addresses and responsibilities logically separate.
- Cilium Gateway API implementation is responsible for L7 HTTP/HTTPS routing; kube-vip provides the network-level VIP for Gateway services.
- Start with Layer 2 / ARP mode for Dev/Stage to minimize operational complexity.
- MetalLB remains an alternative for future evaluation rather than the current preferred solution.
- Define dedicated address ranges for the Control Plane VIP and Kubernetes LoadBalancer service IP pool during detailed network design.

### 6.3 Certificate Management

Decision:
- Use cert-manager as a standard Kubernetes platform service for automated certificate issuance and renewal.
- Integrate cert-manager with Gateway API resources for TLS certificate lifecycle management.

Open Questions:
- Final certificate issuer strategy for Dev/Stage: public ACME/Let's Encrypt, Vault/private PKI, or a combination of both.

### 6.4 External DNS

Decision:
- ExternalDNS is planned but conditional on the selected DNS provider exposing a supported API/integration.
- If suitable DNS automation is not available, use a simple wildcard DNS record pointing the Dev/Stage domain to the kube-vip Gateway VIP.
- When enabled, ExternalDNS should derive records from Kubernetes Gateway API routes/services and keep them synchronized with the DNS provider.

Open Questions:
- Final DNS provider and ExternalDNS integration method.

### 6.5 Kubernetes Storage / CSI

Decision:
- Expose two storage profiles that align with the underlying VM/storage architecture.
- `nfs-stateful` StorageClass: shared persistent storage backed by NFS, intended as the default choice for stateful workloads that need node-independent persistence.
- `local-nvme` StorageClass / local ephemeral storage: high-performance node-local storage backed by worker-local NVMe.
- For purely ephemeral workloads such as caches, build data and temporary files, prefer local ephemeral storage (for example `emptyDir`) instead of persistent volumes where practical.
- In the Development-optimized storage option, mirrored local NVMe may also be used for selected stateful workloads, accepting node affinity, lack of transparent HA and restore requirements after node loss.
- NFS dynamic provisioning should use an NFS CSI implementation; exact backend and configuration are deferred to detailed design.
- Local persistent volumes should use `WaitForFirstConsumer`-style topology-aware scheduling where applicable.

Open Questions:
- Final reclaim policies (`Delete` vs `Retain`) per StorageClass/workload category.
- Local persistent volume provisioning implementation if dynamic local PV provisioning is required.
- Kubernetes PVC snapshot/backup policy and integration with the wider backup strategy.
- Final NFS backend and operational ownership.

### 6.6 Kubernetes Secrets Integration

Decision:
- HashiCorp Vault remains the single source of truth for application and platform secrets.
- Use External Secrets Operator as the standard Kubernetes integration layer for synchronizing secrets from Vault into Kubernetes Secrets.
- Use Vault Kubernetes Auth for workload/operator authentication where practical, avoiding long-lived static Vault tokens.
- Kubernetes workloads consume the generated Kubernetes Secrets through their normal deployment configuration.
- Jenkins credentials should use the HashiCorp Vault Jenkins plugin so Jenkins credential usage is backed by and synchronized with Vault rather than maintained independently in Jenkins.
- Vault policies and access boundaries should be mapped to Kubernetes namespaces, ServiceAccounts, Jenkins jobs/roles, and workload responsibilities.

Open Questions:
- Detailed Vault policy and namespace/ServiceAccount mapping.
- External Secrets refresh intervals and secret rotation behavior.
- Kubernetes Secrets encryption-at-rest configuration.
- Detailed Jenkins Vault authentication and credential mapping strategy.

### 6.7 Jenkins

Decision:
- Run a single Jenkins controller in Kubernetes for Dev/Stage; do not introduce Jenkins controller HA initially.
- Use ephemeral Kubernetes agents for build, deployment and infrastructure execution. Agents are created for jobs and removed after execution.
- Controller persistence: `JENKINS_HOME` must use persistent storage, preferably the `nfs-stateful` StorageClass, so controller state survives pod rescheduling/restarts.
- Jenkins Configuration as Code (JCasC): controller configuration, Kubernetes cloud/agents, security settings and plugin configuration must be declarative and version-controlled in Git.
- Pipeline as Code: keep `Jenkinsfile` definitions in application/infrastructure repositories rather than creating pipelines manually through the Jenkins UI.
- Credentials: Vault remains the source of truth and Jenkins uses the HashiCorp Vault plugin for credential access, as defined in the Kubernetes Secrets Integration section.
- Agent images: use versioned container images according to workload type, including infrastructure agents with Terraform/OpenTofu, Terragrunt, Ansible, kubectl and Helm, and application build agents for the required application technologies. Infrastructure agent tool versions remain synchronized with the Infrastructure Management VM.
- Controller execution policy: the Jenkins controller must not execute normal build/deployment jobs; workload execution runs on agents.
- Backup/restore: persistent Jenkins state is backed up, while Git/JCasC/Pipeline-as-Code remain the primary mechanism for reproducible controller recovery and rebuild.
- Upgrade/plugin strategy: Jenkins controller image version and plugin list/versions are managed declaratively from Git rather than through uncontrolled UI upgrades.
- Scheduling: controller affinity, priority and placement rules are deferred to detailed Kubernetes design.

### 6.8 Metrics Server

Decision:
- Deploy Metrics Server as a standard Kubernetes platform service.
- Metrics Server provides the resource metrics required for Horizontal Pod Autoscaler (HPA) based on CPU/memory utilization and for `kubectl top`.
- Custom/application metrics for advanced autoscaling are deferred to the Observability/Autoscaling detailed design.

## 7. Backup & Recovery

Scope:
- Backup and recovery must cover the full platform, not only Kubernetes.

Areas:
- Proxmox / VM layer: protect important VMs and platform configuration; evaluate Proxmox-native backup tooling during detailed design.
- Storage layer: back up NFS/shared storage and any selected stateful local-NVMe workloads according to workload criticality.
- Infrastructure services: include Vault VM, Infrastructure Management VM and Jenkins persistent state in the backup strategy.
- Kubernetes control plane: continue periodic etcd snapshots stored outside the Control Plane VMs.
- Kubernetes applications: evaluate Velero as optional Kubernetes-aware application/namespace backup and restore tooling, including persistent volume data protection where supported by the selected storage integration.
- Application data: use database-native or application-native backups where appropriate.

Principle:
- Kubernetes-aware backup tooling complements but does not replace etcd snapshots, VM backups, storage/backend-native backups or application-native backups.

## 8. Observability

Scope:
- Observability must cover the full infrastructure and application stack, not only Kubernetes.

Areas:
- Physical / Proxmox layer: host metrics, hardware health, CPU, memory, disk, network and cluster status.
- VM layer: operating-system metrics, filesystem usage, core services/processes and availability.
- Kubernetes layer: nodes, control plane, pods, deployments, resource usage and autoscaling-related metrics.
- Platform services: monitor Vault, Jenkins, Cilium, cert-manager, ExternalDNS and other critical platform components.
- Application layer: application metrics, distributed tracing where justified, and service-level health indicators.
- Centralized dashboards should aggregate observability signals from all layers where practical; logging and alerting are defined as dedicated cross-platform sections below.

Open Questions:
- Metrics stack candidates: Prometheus for collection/querying, with VictoriaMetrics evaluated as a Prometheus-compatible metrics storage/backend where longer retention or higher scale justifies it.
- Grafana is the preferred visualization layer for metrics and observability dashboards.
- Distributed tracing technology remains to be selected during detailed design.
- Data retention and storage strategy for observability data.

## 9. Monitoring

Scope:
- Monitoring must cover the full platform, not only Kubernetes.

Areas:
- Physical / Proxmox layer: host health, CPU, memory, disk, network, hardware and cluster status.
- VM layer: operating-system metrics, filesystem usage, service health and availability.
- Kubernetes layer: nodes, control plane, pods, workloads, resource usage and capacity.
- Platform services: monitor Vault, Jenkins, Cilium, cert-manager, ExternalDNS and other critical platform components.
- Application layer: service metrics, latency, errors, throughput and relevant service/business indicators.
- Prefer a centralized metrics and dashboard experience across infrastructure and applications where practical.

Technology direction:
- Prometheus for metrics collection and Kubernetes/infrastructure monitoring.
- Grafana for dashboards and visualization.
- VictoriaMetrics is an option for Prometheus-compatible metrics storage and longer-term/scalable retention if required.
- Bare metal and Linux VMs use node_exporter; hardware/BMC monitoring may use IPMI exporter.
- Proxmox metrics are integrated into Prometheus through the Proxmox API/exporter model; exact implementation remains TBD.

Open Questions:
- Final Prometheus-only vs Prometheus + VictoriaMetrics architecture.
- Metrics retention, storage sizing and long-term retention strategy.

## 10. Logging

Scope:
- Centralized logging must cover the full platform, not only Kubernetes.

Areas:
- Physical / Proxmox layer: collect relevant host, cluster and platform logs.
- VM layer: collect operating-system and critical service logs from Infrastructure Management VM, Vault VM and other standalone VMs.
- Kubernetes layer: collect node, control-plane, platform-service and workload/container logs.
- Application layer: centralize microservice application logs with consistent metadata such as environment, service, namespace and instance/pod identity.
- Prefer a shared logging backend and query/visualization experience across infrastructure and application layers where practical.

Technology direction:
- Evaluate OpenSearch, Graylog and Elasticsearch as centralized logging/search backends.
- Prefer a single shared log platform for VM, Kubernetes and application logs where practical.
- Proxmox and Linux VM logs originate from systemd-journald/rsyslog and are forwarded into the selected centralized logging backend.

Open Questions:
- Final logging backend and log collection/forwarding implementation.
- Retention, archival and storage sizing policy.
- Log parsing, labeling and sensitive-data handling standards.

## 11. Alerting

Scope:
- Alerting must cover the full platform and consume relevant signals from infrastructure, Kubernetes, applications and platform services.

Areas:
- Physical / Proxmox layer: hardware, host, storage, network and cluster-health alerts.
- VM layer: OS resource exhaustion, filesystem, service health and availability alerts.
- Kubernetes layer: node/control-plane health, workload failures, capacity and platform-service alerts.
- Application layer: service health, error-rate, latency and other application/SLO-oriented alerts where applicable.
- Centralize alert routing, deduplication and notification policy where practical.
- Avoid alerting on every raw signal; alerts should represent actionable conditions.

Technology direction:
- Use Alertmanager for alert grouping, deduplication, routing and notification orchestration for Prometheus-compatible alerts.
- Integrate PagerDuty for incident notification, escalation and on-call workflows where required.
- Use the same Alertmanager/PagerDuty path for Proxmox, bare-metal and VM alerts where practical.

Open Questions:
- Final notification integrations in addition to PagerDuty, if any.

# Layer 4 – Application Delivery & Runtime


- Severity model, routing, ownership and escalation policy.
- On-call model and environment-specific alert thresholds.
## 12. GitOps

Decision:
- Use Argo CD as the preferred Kubernetes GitOps/CD controller.
- Git is the source of truth for application deployment state.
- Keep deployment/GitOps configuration version-controlled and separate from normal imperative deployment operations.
- Argo CD reconciles Kubernetes with the desired state stored in Git.

## 13. CI/CD

### 13.1 Jenkins
- Responsible for CI and release automation: build, test, scan, package and artifact/image publication.
- Updates the desired deployment state in the GitOps repository instead of deploying applications directly with imperative Kubernetes/Helm commands.
- MCP Server Plugin can be evaluated for AI-assisted access to jobs, builds and logs.

### 13.2 GitLab CI/CD / GitHub Actions
- Evaluate GitLab CI/CD with Kubernetes Runners and GitHub Actions with self-hosted Kubernetes runners as potential replacements for Jenkins.
- Simpler initial setup and lower operational overhead: less controller/plugin lifecycle management compared with Jenkins.
- CI configuration lives directly with the repository and integrates naturally with merge requests/pull requests, permissions and the selected source-control platform.
- Kubernetes-based ephemeral runners preserve the preferred isolated, disposable job-execution model.
- Preferred candidate depends primarily on the final source-control platform: GitLab CI/CD for GitLab or GitHub Actions for GitHub.

### 13.3 Argo CD
- Responsible for GitOps-based continuous delivery and reconciliation of Kubernetes deployment state from Git.
- Detects GitOps changes produced by the CI/release workflow and reconciles them to Kubernetes.
- Argo CD MCP integration can be used for AI-assisted application, resource and synchronization workflows.

Open Questions:
- Final CI platform selection: Jenkins vs GitLab CI/CD vs GitHub Actions.

## 14. Release Process

Decision:
- Define an explicit image promotion strategy across environments.
- Follow the principle: build once, promote the same immutable artifact.
- A container image is built and published once, then the same image version/digest is promoted through Development → Staging → Production without rebuilding it per environment.
- Environment promotion is represented by a controlled change to the GitOps configuration (image tag/digest), which Argo CD then reconciles to the target environment.
- Prefer immutable version tags and, where appropriate, pin deployments by image digest to guarantee that the exact tested artifact is promoted.
- Detailed promotion gates, approvals, versioning/tagging and release workflow are deferred to the CI/CD detailed design.

## 15. Artifact Registry

Decision:
- Use self-managed JFrog Artifactory as the preferred central artifact repository.
- Host Artifactory in Kubernetes for Dev/Stage, initially as a single instance with persistent storage; evaluate HA for Production if required.
- Use Artifactory for Docker/OCI application images and Jenkins agent images, Helm charts and other build artifacts where appropriate.
- Artifact Registry is part of the software delivery layer and is introduced after the Kubernetes platform is available.
## 16. Databases

Decision:
- Run application databases in Kubernetes for Dev/Stage using a mature database operator rather than manually managed StatefulSets.
- The exact database operator is deferred to detailed design.
- Database persistence, backup and restore must integrate with the selected Kubernetes storage and wider backup strategy.
- VM-based databases remain an option for future Production requirements if stronger operational isolation, performance or storage constraints justify it.

## 17. Security & Access Management

Decision:
- Apply least-privilege access across Proxmox, Linux VMs, Kubernetes, Jenkins, Vault, Artifactory, monitoring and other platform services.
- Prefer centralized identity/SSO and role-based access where supported; keep administrative access separate from normal user/application access.
- Use VPN as the primary remote access path to the internal infrastructure network; management endpoints should not be exposed directly to the public Internet unless explicitly required.
- Secrets and privileged credentials remain managed through Vault.
- Detailed RBAC, identity provider, MFA, SSH access and network access policies are deferred to the security detailed design.

## 18. Environment Strategy

Decision:
- Start with a dedicated Development Kubernetes cluster.
- Staging is deferred until later; evaluate whether it should be an isolated namespace in the Development cluster or a separate Kubernetes cluster based on isolation, operational and resource requirements.
- Production environment topology will be designed separately when Production requirements are defined.






# Layer 5 – AI Integration

## 19. AI Integration

Purpose:
- Integrate AI assistants with internal infrastructure and platform systems through Model Context Protocol (MCP) servers to simplify operations, troubleshooting and day-to-day engineering workflows.

MCP integration candidates:
- Grafana — MCP integration for dashboards, metrics, logs and alerting workflows.
- Graylog — MCP integration for log search and operational investigation.
- Kubernetes — community MCP server implementations; final implementation and permission model TBD.
- Jira — Atlassian Rovo MCP Server.
- GitHub — GitHub MCP Server.
- GitLab — GitLab MCP Server (Beta).
- Argo CD — Argo Project Labs MCP server.
- Slack — Slack MCP Server.
- JFrog Artifactory — JFrog Platform MCP Server (Beta).


Principle:
- Start with read-only / least-privilege MCP access where possible and explicitly control any mutating operations.

# Open Questions

- Storage technology for Production
- Infrastructure as Code strategy
- VM placement strategy
- Resource allocation
- Kubernetes architecture


