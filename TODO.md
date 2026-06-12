# Talos Homelab Implementation TODO

## ✅ Completed
- [x] **Project Workspace Initialized**: Created `~/Projects/Personal/talos-homelab/` directory.
- [x] **Task 1: Configuration Generation**:
    - [x] Generated and fixed `controlplane.yaml` — cluster name `talos-cluster`, static IP `192.168.31.20/24`, hostname `talos.cp.local`.
    - [x] Generated `worker1.yaml` (talos.w1.local / 192.168.31.21/24) and `worker2.yaml` (talos.w2.local / 192.168.31.22/24).
    - [x] Enabled SSH service in machine configurations for emergency access.
    - [x] Static IP, gateway, and nameservers configured in all node configs.
    - [x] `talosconfig` updated with context `talos-cluster` and endpoint `192.168.31.20`.
- [x] **Infrastructure Defined**:
    - Control Plane: `talos.cp.local` @ `192.168.31.20/24`
    - Worker 1: `talos.w1.local` @ `192.168.31.21/24`
    - Worker 2: `talos.w2.local` @ `192.168.31.22/24`

## ✅ Cluster Bootstrapped (2026-06-12)
- [x] **Task 2: Cluster Bootstrap**:
    - [x] CP config applied; etcd bootstrapped and healthy.
    - [x] Fetched kubeconfig. NOTE: cluster endpoint is `talos.cp.local` which does not resolve from the workstation — the kubeconfig cluster `server` was repointed to `https://192.168.31.20:6443` (the apiserver cert includes the CP IP SAN). Consider changing `cluster.controlPlane.endpoint` in `controlplane.yaml` to the IP for a self-resolving kubeconfig.
- [x] **Task 3: Worker Node Provisioning**:
    - [x] worker1 + worker2 configs applied; both joined.
    - [x] All 3 nodes `Ready`: `talos-cp`, `talos-w1`, `talos-w2`.
    - **Bug fixed during bootstrap:** hostnames were FQDNs (`talos.cp.local` etc.), which Talos truncates to the label `talos` — so all 3 nodes collided on a single Kubernetes node named `talos` (workers stuck looping `Failed to update lease "talos"`). Renamed hostnames to single labels `talos-cp` / `talos-w1` / `talos-w2`, re-applied (no reboot), and deleted the stale `talos` node object.
- CNI: Talos default **Flannel** (intentionally left commented out in `controlplane.yaml`). `kube-flannel`, `kube-proxy`, `coredns`, and control-plane static pods all Running.

## ✅ In-cluster DNS stack — MetalLB + Pi-hole (2026-06-12)
Manifests/values in `infra/` (see `infra/README.md`). Applied directly via kubectl/helm.
- [x] **API endpoint renamed** to `https://talos-cp.lan:6443` (`controlplane.yaml` endpoint + `apiServer.certSANs: [talos-cp.lan, talos-cp, 192.168.31.20]`). IP kept as SAN for fallback.
- [x] **Storage:** `local-path-provisioner` v0.0.31 (default SC), patched to `/var/mnt/local-path-provisioner`; `worker1.yaml` has kubelet `extraMounts` for `/var/mnt`.
- [x] **MetalLB** (L2), pool `192.168.31.50-.59` (`infra/metallb/pools.yaml`). ⚠️ Exclude this range from router DHCP.
- [x] **Pi-hole** (mojo2600 chart, v6) on MetalLB VIP **`192.168.31.53`** (DNS 53 tcp/udp + web share the IP). PVC on local-path, pinned to `talos-w1`. Admin password in secret `pihole/pihole-admin`. Local record `talos-cp.lan -> 192.168.31.20`. Verified: `dig @192.168.31.53 talos-cp.lan` and upstream both resolve from the Mac; web UI 302.
- **Networking bug fixed:** all 3 nodes shared one Flannel VXLAN VTEP MAC (legacy of the earlier `talos` node-name collision) → cross-node pod networking was broken (MetalLB webhook timeouts). Fixed by clearing each node's `flannel.alpha.coreos.com/backend-data` annotation + recreating `flannel.1` so each got a unique MAC. See `infra/README.md`.

### ⏳ Remaining DNS cutover (user-side) — IN PROGRESS, not yet effective
Attempted 2026-06-12: re-fetched kubeconfig to `talos-cp.lan` but the **Mac still resolves
via the router `192.168.31.1`** (DHCP not yet handing out Pi-hole, or lease not renewed), so
the name didn't resolve and kubeconfig was **reverted to the IP** `https://192.168.31.20:6443`
(works). Pi-hole answers correctly when queried directly (`dig @192.168.31.53 talos-cp.lan`).
- [ ] Reserve `192.168.31.50-.59` outside router DHCP; set DHCP **primary DNS = 192.168.31.53** + a **secondary fallback** (router/1.1.1.1).
- [ ] Make this Mac use Pi-hole: either renew DHCP after the router change, or set it directly — `sudo networksetup -setdnsservers Wi-Fi 192.168.31.53 1.1.1.1` (clear later with `... Wi-Fi empty`).
- [ ] Once `talos-cp.lan` resolves from the Mac: `talosctl --nodes 192.168.31.20 kubeconfig --force` to switch kubeconfig back to the name.
- Cluster nodes intentionally keep upstream DNS (192.168.31.1/8.8.8.8), NOT Pi-hole, to avoid a boot dependency loop.

## ✅ Power-loss recovery (2026-06-12)
Full AC power loss took all 3 nodes down. Recovered with no rebuild:
- `talos-cp` auto-powered-on; etcd recovered from disk on its own. The **two workers did NOT
  auto-power-on** (Dell OptiPlex, no IPMI) — needed a manual power-button press, then rejoined
  `Ready` automatically. Pi-hole/MetalLB/local-path rescheduled themselves; Pi-hole PVC
  reattached on `talos-w1`; Flannel VTEP-MAC fix persisted (no re-repair). Cross-node
  networking, VIP `192.168.31.53`, and DNS all verified back.
- [ ] **Harden:** set workers' BIOS **AC Power Recovery → On** (so the whole cluster
  auto-recovers next time).
- [ ] **Harden:** scheduled `talosctl etcd snapshot` (single CP is a SPOF).
- [ ] **Harden:** DHCP secondary DNS so LAN resolution survives a cluster/Pi-hole outage; UPS on the CP.

## ⏳ Pending
- [ ] **Task 4: GitOps Setup (FluxCD)**:
    - [ ] Install Flux CLI: `brew install fluxcd/tap/flux`
    - [ ] Bootstrap Flux onto cluster from GitHub repo.
    - [ ] Verify Flux controllers running in `flux-system` namespace.
- [ ] **Task 5: GitOps Application Deployment**:
    - [ ] Create GitHub manifests repo and seed with Nginx deployment.
    - [ ] Verify Flux syncs and Nginx is accessible at `http://192.168.31.20:30080`.
