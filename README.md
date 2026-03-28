# Nephio Workload Deployment — OAI-RAN & SD-Core on Non-Kind Clusters

Deploy OAI 5G RAN (without OAI Core) and Aether SD-Core on real Kubernetes clusters using Nephio's intent-based automation.

## Architecture Overview

```
                    ┌──────────────────────────────────────────┐
                    │        Nephio Management Cluster         │
                    │     Porch · IPAM · VLAN · Controllers    │
                    │                                          │
                    │  ┌─────────────────────────────────────┐ │
                    │  │  nephio-workload-blueprints (Git)   │ │
                    │  │  Single upstream repo with ALL      │ │
                    │  │  packages: baseline, networking,    │ │
                    │  │  OAI-RAN, SD-Core                   │ │
                    │  └─────────────────────────────────────┘ │
                    └──────────┬────────────────┬──────────────┘
                               │                │
                    ┌──────────▼───┐    ┌───────▼───────────┐
                    │ Cluster: ran │    │ Cluster: core     │
                    │              │    │                   │
                    │ OAI-RAN      │    │ SD-Core 5G        │
                    │  • CUCP      │    │  • AMF  • SMF     │
                    │  • CUUP      │    │  • UPF  • NRF     │
                    │  • DU        │    │  • UDR  • UDM     │
                    │  • Operators │    │  • AUSF • PCF     │
                    │              │    │  • NSSF • WebUI   │
                    │              |    │  • MongoDB, Kafka │
                    └──────────────┘    └───────────────────┘
```

## Prerequisites

### 1. Nephio Management Cluster

A running Nephio management cluster with the following controllers installed:

| Component | Purpose |
|---|---|
| Porch | Package orchestration (kpt-as-a-service) |
| Config Sync | GitOps sync to workload clusters |
| IPAM Controller | Automatic IP address allocation |
| VLAN Controller | Automatic VLAN ID allocation |
| NAD Controller | Network Attachment Definition generation |
| Cluster API (CAPI) | Cluster lifecycle management |

### 2. Workload Clusters

Two Kubernetes clusters registered with the management cluster via CAPI:

| Cluster | Purpose | Provisioning |
|---|---|---|
| `ran` | OAI-RAN deployment | BYOH, kubeadm, or any CAPI provider |
| `core` | SD-Core deployment | BYOH, kubeadm, or any CAPI provider |

**Node requirements for each workload cluster:**

- Kubernetes v1.28+ with kubeadm
- A network interface available for data plane traffic (e.g., `ens3`, `eno49`, `eth1`)
- `containerd` as container runtime
- Calico CNI (or any primary CNI)
- Internet access for pulling container images

### 3. Git Repositories

| Repository | Type | Content |
|---|---|---|
| `nephio-workload-blueprints` | Upstream (Package) | All blueprint packages |
| `ran` | Downstream (Deployment) | Rendered packages for the RAN cluster |
| `core` | Downstream (Deployment) | Rendered packages for the Core cluster |

Create the downstream repos as empty Git repositories on GitHub (or your Git provider) before starting deployment.

---

## Repository Structure

```
nephio-deployment-mgmt/
├── README.md                              ← This file
├── clusterconfig/
│   ├── clustercontext.yaml                ← ClusterContext CRs (ran + core)
│   ├── workloadcluster.yaml               ← WorkloadCluster CRs (master interface + CNI)
│   ├── rootsync-ran.yaml                  ← RootSync CR for ran cluster (GitOps)
│   └── rootsync-core.yaml                 ← RootSync CR for core cluster (GitOps)
├── repositories/
│   ├── upstream-repos.yaml                ← Blueprint repo registration
│   └── downstream-repos.yaml             ← Per-cluster deployment repos
├── prerequisites/
│   ├── networks.yaml                      ← VPC Networks (cluster-name labels pre-baked)
│   ├── ipprefixes-ran.yaml                ← IP allocations for ran cluster
│   ├── ipprefixes-core.yaml               ← IP allocations for core cluster + DNN pool
│   └── vlanindex.yaml                     ← VLAN allocators for both clusters
└── packagevariants/
    ├── baseline/baseline-pv.yaml          ← Namespaces, StorageClass, PodSecurity
    ├── addons/addons-pv.yaml              ← Metrics server, local-path provisioner
    ├── networking/networking-pv.yaml       ← Multus CNI + VLAN auto-init DaemonSet
    ├── oai-ran/oai-ran-pv.yaml            ← CP Operators + RAN Operator + CUCP/CUUP/DU
    └── sdcore/sdcore-pv.yaml              ← CP + Operator + UPF + AMF + SMF
```

---

## Configuration — What You Need to Customize

Before deploying, update the following values to match your environment:

### 1. WorkloadCluster — Master Interface

Edit `clusterconfig/workloadcluster.yaml` and set the `masterInterface` to your node's data plane network interface:

```yaml
spec:
  masterInterface: ens3   # ← Change to your node's interface (e.g., eno49, eth1)
```

> **How to find it:** Run `ip addr show` on the workload node. Pick an interface that is UP and has link-layer connectivity. Use a dedicated data plane interface if available (not the management interface).

### 2. Repositories — Git URLs

Edit `repositories/upstream-repos.yaml` and `repositories/downstream-repos.yaml`:

```yaml
# upstream-repos.yaml
git:
  repo: https://github.com/YOUR-ORG/nephio-workload-blueprints.git  # ← Your blueprint repo

# downstream-repos.yaml  
git:
  repo: https://github.com/YOUR-ORG/nephio-ran.git    # ← Your RAN downstream repo
  repo: https://github.com/YOUR-ORG/nephio-core.git   # ← Your Core downstream repo
```

### 3. VLAN Init — Master Interface

Edit `packagevariants/networking/networking-pv.yaml` and set the `master-interface` for each cluster:

```yaml
# vlan-init-ran
configMap:
  master-interface: ens3           # ← Same interface as WorkloadCluster for ran

# vlan-init-core
configMap:
  master-interface: ens3           # ← Same interface as WorkloadCluster for core
```

### 4. RootSync — Downstream Git URLs

Edit `clusterconfig/rootsync-ran.yaml` and `clusterconfig/rootsync-core.yaml`:

```yaml
# rootsync-ran.yaml
spec:
  git:
    repo: https://github.com/YOUR-ORG/nephio-ran.git  # ← Same as downstream-repos.yaml

# rootsync-core.yaml
spec:
  git:
    repo: https://github.com/YOUR-ORG/nephio-core.git  # ← Same as downstream-repos.yaml
```

### 5. Cluster Names (Optional)

The default cluster names are `ran` and `core`. If you use different names, update:
- All `name:` fields in `clusterconfig/`
- All `cluster-name:` values in PackageVariant configMaps
- All `nephio.org/cluster-name:` labels in `prerequisites/`
- All `injectors.name:` and `downstream.repo:` in PackageVariants

---

## Deployment Guide

### Phase 0: Config Sync + RootSync (Workload Clusters)

Install Config Sync on **each workload cluster** to enable the GitOps pipeline. Config Sync watches the downstream Git repo and applies rendered packages to the cluster.

```bash
# Install Config Sync on ran cluster
kubectl apply -f https://github.com/GoogleContainerTools/kpt-config-sync/releases/latest/download/config-sync-manifest.yaml \
  --kubeconfig ran.kubeconfig

# Install Config Sync on core cluster
kubectl apply -f https://github.com/GoogleContainerTools/kpt-config-sync/releases/latest/download/config-sync-manifest.yaml \
  --kubeconfig core.kubeconfig
```

**Wait for Config Sync pods to be Running:**

```bash
kubectl --kubeconfig ran.kubeconfig get pods -n config-management-system
kubectl --kubeconfig core.kubeconfig get pods -n config-management-system
```

Then create RootSync CRs to connect each cluster to its downstream Git repo:

```bash
# Create RootSync on ran cluster
kubectl apply -f clusterconfig/rootsync-ran.yaml --kubeconfig ran.kubeconfig

# Create RootSync on core cluster
kubectl apply -f clusterconfig/rootsync-core.yaml --kubeconfig core.kubeconfig
```

**Verify sync status:**

```bash
kubectl --kubeconfig ran.kubeconfig get rootsync -n config-management-system
kubectl --kubeconfig core.kubeconfig get rootsync -n config-management-system
```

---

### Phase 1: Prerequisites (Management Cluster)

Apply the foundation resources on the **management cluster**:

```bash
# Step 1: Register cluster config
kubectl apply -f clusterconfig/clustercontext.yaml
kubectl apply -f clusterconfig/workloadcluster.yaml

# Step 2: Register Git repositories
kubectl apply -f repositories/upstream-repos.yaml
kubectl apply -f repositories/downstream-repos.yaml

# Step 3: Create network infrastructure
kubectl apply -f prerequisites/networks.yaml
kubectl apply -f prerequisites/ipprefixes-ran.yaml
kubectl apply -f prerequisites/ipprefixes-core.yaml
kubectl apply -f prerequisites/vlanindex.yaml
```

**Verify:**

```bash
kubectl get workloadclusters
kubectl get repositories.config.porch.kpt.dev
kubectl get networks.infra.nephio.org
kubectl get ipprefixes
kubectl get vlanindex
```

All resources should show `True` / `Ready`.

---

### Phase 2: Cluster Foundation

Deploy baseline configuration to **both** clusters:

```bash
# Namespaces, StorageClass, PodSecurity
kubectl apply -f packagevariants/baseline/baseline-pv.yaml
```

**Wait for Published:**

```bash
kubectl get packagerevisions | grep cluster-baseline
# Both ran and core should show "Published"
```

Then deploy platform addons:

```bash
# Metrics server, local-path provisioner
kubectl apply -f packagevariants/addons/addons-pv.yaml
```

**Wait for Published:**

```bash
kubectl get packagerevisions | grep platform-addons
```

---

### Phase 3: Networking + Automatic VLAN Setup

This is the key step — deploys Multus CNI AND the VLAN initialization DaemonSet:

```bash
kubectl apply -f packagevariants/networking/networking-pv.yaml
```

This deploys to **both** clusters:

| Package | What it does |
|---|---|
| `multus-cni` | Enables multi-network pod attachments (macvlan) |
| `vlan-init` | DaemonSet that auto-creates VLAN subinterfaces on the master interface |

**Wait for Published:**

```bash
kubectl get packagerevisions | grep -E "multus|vlan"
```

**Verify VLANs on each workload node:**

```bash
# On ran node
ssh user@ran-node "ip link show | grep <master-interface>"
# Should show: ens3.2, ens3.3, ens3.4, ... ens3.10

# On core node
ssh user@core-node "ip link show | grep <master-interface>"
# Should show: ens3.2, ens3.3, ens3.4, ... ens3.10
```

> **Note:** The VLAN DaemonSet creates VLAN subinterfaces 2-10 by default. It also monitors every 60 seconds and recreates any missing VLANs, ensuring self-healing. You do NOT need to SSH and manually create VLANs.

---

### Phase 4: OAI-RAN Deployment (ran cluster)

Deploy OAI 5G RAN **without** OAI Core components:

```bash
kubectl apply -f packagevariants/oai-ran/oai-ran-pv.yaml
```

This deploys 6 PackageVariants in the following order:

| # | Package | Purpose |
|---|---|---|
| 1 | `oai-cp-operators` | Installs `NFDeployment` CRD (required by RAN NFs) |
| 2 | `oai-ran-operator` | Operator that watches NFDeployment CRs and creates pods |
| 3 | `oai-ran-network` | Network Attachment Definitions for E1, F1, N2, N3 interfaces |
| 4 | `oai-ran-cucp` | CU Control Plane (gNB-CU-CP) |
| 5 | `oai-ran-cuup` | CU User Plane (gNB-CU-UP) |
| 6 | `oai-ran-du` | Distributed Unit (gNB-DU) |

**Wait and verify:**

```bash
# Check PackageVariant status
kubectl get packagerevisions | grep oai

# Check pods on ran cluster
kubectl --kubeconfig ran.kubeconfig get pods -A
```

**Expected result — all Running:**

```
oai-cn-operators    oai-amf-operator-xxx    1/1   Running
oai-cn-operators    oai-nrf-operator-xxx    1/1   Running
oai-cn-operators    ...                     1/1   Running
oai-ran-operators   oai-ran-operator-xxx    1/1   Running
oai-ran-cucp        oai-cu-cp-xxx           1/1   Running
oai-ran-cuup        oai-cu-up-xxx           1/1   Running
oai-ran-du          oai-du-xxx              1/1   Running
```

> **Note:** The `oai-cn-operators` namespace contains OAI Core **operators only** (for CRDs) — no actual core network functions (AMF, SMF, etc.) are deployed.

---

### Phase 5: SD-Core Deployment (core cluster)

Deploy Aether SD-Core 5G:

```bash
kubectl apply -f packagevariants/sdcore/sdcore-pv.yaml
```

This deploys 5 PackageVariants:

| # | Package | Purpose |
|---|---|---|
| 1 | `sdcore5g-cp` | Control Plane: NRF, UDR, UDM, AUSF, PCF, NSSF, WebUI, MongoDB, Kafka |
| 2 | `sdcore5g-operator` | SD-Core operator |
| 3 | `sdcore5g-upf` | User Plane Function (**must be Published before AMF/SMF**) |
| 4 | `sdcore5g-amf` | Access and Mobility Management Function |
| 5 | `sdcore5g-smf` | Session Management Function |

> **Important:** AMF and SMF have `Dependency` CRs on UPF. They will remain in Draft until UPF is Published. This ordering is handled automatically by Porch.

**Wait and verify:**

```bash
# Check PackageVariant status
kubectl get packagerevisions | grep sdcore

# Check pods on core cluster
kubectl --kubeconfig core.kubeconfig get pods -A
```

**Expected result — all Running:**

```
sdcore5g-cp     amf-sdcore-xxx         1/1   Running
sdcore5g-cp     smf-sdcore-xxx         1/1   Running
sdcore5g-cp     nrf-xxx                1/1   Running
sdcore5g-cp     udr-xxx                1/1   Running
sdcore5g-cp     udm-xxx                1/1   Running
sdcore5g-cp     ausf-xxx               1/1   Running
sdcore5g-cp     pcf-xxx                1/1   Running
sdcore5g-cp     nssf-xxx               1/1   Running
sdcore5g-cp     webui-xxx              1/1   Running
sdcore5g-cp     mongodb-xxx            1/1   Running
sdcore5g-cp     kafka-xxx              1/1   Running
sdcore5g-upf    upf-sdcore-0           5/5   Running
sdcore5g        sdcore5g-operator-xxx  2/2   Running
```

---

## Networking Details

### Interface Mapping

Nephio's IPAM and VLAN allocators automatically assign IPs and VLAN IDs. The following table shows the network interfaces used:

**OAI-RAN (ran cluster):**

| Interface | Network | VLAN | Used By |
|---|---|---|---|
| E1 | `vpc-cu-e1` | Auto-assigned | CUCP ↔ CUUP |
| F1 | `vpc-cudu-f1` | Auto-assigned | CU ↔ DU |
| N2 | `vpc-ran` | Auto-assigned | CUCP → AMF (control plane) |
| N3 | `vpc-internet` | Auto-assigned | CUUP → UPF (user plane) |

**SD-Core (core cluster):**

| Interface | Network | VLAN | Used By |
|---|---|---|---|
| N2 | `vpc-ran` | Auto-assigned | AMF |
| N3 | `vpc-ran` | Auto-assigned | UPF |
| N4 | `vpc-internal` | Auto-assigned | SMF ↔ UPF |
| N6 | `vpc-internet` | Auto-assigned | UPF → internet |

### How VLANs Work

```
1. WorkloadCluster CR defines masterInterface (e.g., ens3)
2. Nephio VLAN allocator assigns VLAN IDs (starting from 2)
3. nad-fn generates NADs with master: ens3.2, ens3.3, etc.
4. vlan-init DaemonSet pre-creates these VLAN subinterfaces
5. macvlan CNI attaches pods to the VLAN interfaces
```

### VLAN Init DaemonSet

The `vlan-init` package deploys a privileged DaemonSet that:

- Loads the `8021q` kernel module
- Creates VLAN subinterfaces (default: IDs 2-10) on the master interface
- Monitors every 60 seconds and recreates missing VLANs (self-healing)
- Runs as an init container (fast setup) + monitor container (ongoing)

Configure via the PackageVariant:

```yaml
configMap:
  master-interface: ens3              # Your node's data plane interface
  vlan-range: "2 3 4 5 6 7 8 9 10"   # VLAN IDs to create (default)
```

---

## Troubleshooting

### PackageVariant stuck in Draft/Error

```bash
# Check PackageVariant conditions
kubectl describe packagevariant <name>

# Common fix: check if upstream repo is registered and accessible
kubectl get repositories.config.porch.kpt.dev
```

### "no available routes" IPAM Error

This means the Network CR is missing `cluster-name` labels:

```bash
kubectl get networks.infra.nephio.org <vpc-name> -o yaml | grep cluster-name
```

> This should not happen with the new deployment — all `cluster-name` labels are pre-baked in `networks.yaml`.

### "Link not found" on Pod Creation

The macvlan CNI cannot find the VLAN subinterface on the node:

```bash
# Check if VLANs exist on the workload node
ssh user@node "ip link show | grep <master-interface>"

# Check vlan-init DaemonSet
kubectl --kubeconfig <cluster>.kubeconfig get pods -n kube-system | grep vlan
kubectl --kubeconfig <cluster>.kubeconfig logs -n kube-system <vlan-init-pod> -c vlan-setup
```

### "apply-setters" Image Pull Error

All kpt function images must use `ghcr.io/kptdev/krm-functions-catalog/`:

```bash
# Verify Kptfiles use correct image
grep -r "gcr.io/kpt-fn" nephio-workload-blueprints/
# Should return NO results

grep -r "ghcr.io/kptdev/krm-functions-catalog" nephio-workload-blueprints/
# Should return all Kptfile references
```

### UPF Stuck — AMF/SMF Won't Start

AMF and SMF have `Dependency` CRs on UPF. Check UPF status first:

```bash
kubectl get packagerevisions | grep upf
# Must show "Published" before AMF/SMF can proceed
```

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Single upstream repo | All packages in one place — simpler management, no external dependencies |
| Pre-baked Network CR labels | Eliminates manual `kubectl patch` commands after deployment |
| VLAN init DaemonSet | Automates VLAN creation — no SSH to nodes required |
| `ghcr.io` function images | GCR deprecated unauthenticated pulls — GHCR mirrors are publicly accessible |
| OAI-RAN without Core | RAN only needs `oai-cp-operators` for CRDs, not actual core NFs |
| Separate clusters | RAN and Core on different clusters — matches real-world 5G topology |

---

## Tested On

- **Management cluster:** Ubuntu 22.04, Kubernetes v1.32, Nephio R3
- **Workload clusters:** Ubuntu 22.04, Kubernetes v1.32, kubeadm, BYOH provider
- **OAI-RAN:** v2.3.0 (CUCP, CUUP, DU)
- **SD-Core:** Aether SD-Core 5G

---

## License

This deployment configuration is provided as-is for the Nephio community. Refer to the individual upstream projects for their respective licenses:

- [Nephio](https://github.com/nephio-project)
- [OpenAirInterface](https://gitlab.eurecom.fr/oai/openairinterface5g)
- [Aether SD-Core](https://github.com/omec-project)
