# Talos OS GitOps Homelab Master Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** A complete, production-grade (homelab style) GitOps-managed Kubernetes cluster running on Talos OS with full LAN connectivity and automated deployments.

**Architecture:** A 3-node cluster (1 Control Plane, 2 Workers) managed via Talos API (`talosctl`). State is synchronized from a Git repository using FluxCD.

**Tech Stack:** Talos OS, `talosctl`, `kubectl`, `fluxcd`, `mDNS/Avahi`.

---

### Task 1: Local Toolchain & Environment Prep

**Objective:** Prepare the workstation with the necessary CLI tools.

**Files:**
- None (System-wide)

**Step 1: Install Talos and Kubectl**
Run: 
```bash
curl -sL https://get.talos.dev | sh -s -- install --version v1.x.x
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin
```

**Step 2: Verify installation**
Run: `talosctl version` and `kubectl version --client`
Expected: Version output for both tools.

**Step 3: Create project workspace**
Run: `mkdir -p talos-homelab/config talos-homelab/flux-manifests`
Expected: Directory structure created.

---

### Task 2: Cryptographic Secrets & YAML Configuration

**Objective:** Generate the cluster credentials and node configurations.

**Files:**
- Create: `talos-homelab/config/secrets.yaml`
- Create: `talos-homelab/config/controlplane.yaml`
- Create: `talos-homelabb/config/worker.yaml`

**Step 1: Generate cluster configuration**
Run:
```bash
cd talos-homelab
talosctl gen config --cluster my-homelab --with-uid --host <CP_IP_OR_HOSTNAME>
```
*Note: Replace `<CP_IP_OR_HOSTNAME>` with your intended control plane address.*

**Step 2: Configure SSH for diagnostics**
Edit `talos-homelab/config/controlplane.yaml` and `talos-homelab/config/worker.yaml`.
Ensure the following is present under the `machine.service.ssh` section:
```yaml
service:
  ssh:
    enabled: true
```

**Step 3: Verify configuration**
Run: `grep "etcd" talos-homelab/config/controlplane.yaml`
Expected: Presence of etcd configuration.

---

### Task 3: OS Image Provisioning (Manual/Physical)

**Objective:** Prepare the installation media for the nodes.

**Files:**
- `talos-amd64.iso`

**Step 1: Download Talos ISO**
Run:
```bash
curl -L -o talos-amd64.iso https://github.com/siderolabs/talos/releases/latest/download/talos-amd64.iso
```
Expected: `talos-amd64.iso` downloaded successfully.

**Step 2: Flash Installation Media (User Action)**
- **Physical Nodes:** Flash `talos-amd64.iso` to a USB drive.
- **Virtual Machines:** Attach `talos-amd64.iso` to the VM's virtual optical drive.

**Step 3: Boot and Verify Network**
1. Boot node from media.
2. Monitor serial console/monitor.
3. Verify node has an IP via DHCP.
Run (from workstation): `ping <node-ip>`
Expected: Successful ICMP replies.

---

### Task 4: Talos Bootstrapping & K8s Initialization

**Objective:** Apply configurations to the running nodes and initialize the K8s API.

**Files:**
- None (Remote interaction)

**Step 1: Apply Control Plane Config**
Run: `talosctl apply --nodes <CP_IP> --config talos-homelab/config/controlplane.yaml`
Expected: `applied successfully`

**Step 2: Apply Worker Configs**
Run: 
```bash
talosctl apply --nodes <WK1_IP> --config talos-homelab/config/worker.yaml
talosctl apply --nodes <WK2_IP> --config talos-homelab/config/worker.yaml
```

**Step 3: Bootstrap the Cluster**
Run: `talosctl bootstrap --nodes <CP_IP>`
Expected: Etcd is initializing.

**Step 4: Verify Node Readiness**
Run: `talosctl kubectl --nodes <CP_IP> get nodes`
Expected: All 3 nodes status `Ready`.

**Step 5: Commit deployment log**
```bash
echo "Cluster bootstrapped successfully" >> deployment_log.txt
```

---

### Task 5: Network Discovery & Connectivity (mDNS)

**Objective:** Enable access via LAN URLs (`.local`).

**Files:**
- None

**Step 1: Configure talosctl for .local**
Run: `talosctl config endpoint <cp-hostname>.local --insecure`

**Step 2: Verify mDNS resolution**
Run: `ping <cp-hostname>.local`
Expected: Resolution to the correct IP.

**Step 3: Verify API availability via .local**
Run: `talosctl version --nodes <cp-hostname>.local --insecure`
Expected: Successful response.

---

### Task 6: GitOps Orchestration (FluxCD)

**Objective:** Install FluxCD to manage cluster state via Git.

**Files:**
- `talos-homelab/flux-manifests/` (Generated)

**Step 1: Install Flux via CLI**
Run:
```bash
flux bootstrap --version=latest \
  --repository=<your-github-repo> \
  --branch=main \
  --path=./clusters/homelab \
  --namespace=flux-system
```

**Step 2: Verify Flux controllers**
Run: `kubectl get pods -n flux-system`
Expected: `source-controller`, `kustomize-controller`, etc., are `Running`.

**Step 3: Test GitOps Synchronization**
1. Create `talos-homelab/flux-manifests/nginx.yaml` with a basic Nginx deployment.
2. Commit and push to Git.
3. Run: `kubectl get pods -A`
Expected: Nginx pod is deployed automatically.

---

### Task 7: Final Verification & Documentation

**Objective:** Ensure all components are healthy and documented.

**Step 1: Final Cluster Status Check**
Run: `kubectl get nodes` and `kubectl get pods -A`
Expected: Everything running smoothly.

**Step 2: Cleanup & Commit**
```bash
git add .
git commit -m "feat: full homelab cluster deployment complete"
```
```