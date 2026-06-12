# Cluster infrastructure (applied directly; Flux-adoptable later)

In-cluster LAN DNS stack: **local-path storage → MetalLB → Pi-hole**, plus the API
endpoint rename to `talos-cp.lan`. Applied with `kubectl`/`helm` against the bootstrapped
3-node Talos cluster. Manifests/values here are version-controlled so a future FluxCD
setup (master-plan Task 4) can adopt them.

## Order of operations

### 0. API endpoint rename (machine config)
`config/controlplane.yaml`: `cluster.controlPlane.endpoint: https://talos-cp.lan:6443`
plus `cluster.apiServer.certSANs: [talos-cp.lan, talos-cp, 192.168.31.20]` (IP kept so
kubectl can always reach the API by IP). Applied via `talosctl apply-config`.

### 1. Storage — local-path-provisioner
- Kubelet bind-mount for Talos' writable `/var/mnt` is set in `config/worker1.yaml`
  (`machine.kubelet.extraMounts`).
- Installed `rancher/local-path-provisioner` v0.0.31, then:
  - patched ConfigMap `local-path-config` `config.json` `nodePathMap` →
    `/var/mnt/local-path-provisioner` (Talos `/opt` is read-only),
  - labeled ns `local-path-storage` `pod-security.kubernetes.io/enforce=privileged`,
  - set `local-path` as the default StorageClass.

### 2. Load balancer — MetalLB (L2)
```sh
kubectl create namespace metallb-system
kubectl label ns metallb-system pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged pod-security.kubernetes.io/warn=privileged --overwrite
helm repo add metallb https://metallb.github.io/metallb && helm repo update
helm install metallb metallb/metallb -n metallb-system --wait
kubectl apply -f infra/metallb/pools.yaml      # pool 192.168.31.50-.59
```
**Reserve `192.168.31.50-.59` outside the router DHCP range.**

### 3. Pi-hole
```sh
kubectl create namespace pihole
kubectl label ns pihole pod-security.kubernetes.io/enforce=privileged --overwrite
kubectl -n pihole create secret generic pihole-admin --from-literal=password='<pw>'
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/ && helm repo update
helm install pihole mojo2600/pihole -n pihole -f infra/pihole/values.yaml
```
DNS+web on MetalLB VIP **192.168.31.53** (shared IP). Local record
`talos-cp.lan -> 192.168.31.20` is in `infra/pihole/values.yaml` (`dnsmasq.customDnsEntries`).

## Gotcha fixed during bring-up: duplicate Flannel VTEP MAC

After the earlier in-place node rename, all three nodes shared one Flannel VXLAN VTEP MAC
(`ae:74:07:24:c3:ca`) because, while they briefly collided on a single `talos` Node
object, they all wrote the same `flannel.alpha.coreos.com/backend-data` annotation. This
silently broke **cross-node** pod/service networking (MetalLB webhook timeouts were the
first symptom). Reboots didn't help because Flannel re-reads the MAC from the annotation.

**Fix (per node):** clear the flannel annotations, delete `flannel.1` (privileged
host-network pod), restart the node's Flannel pod → each node regenerates a unique VTEP
MAC. Verified distinct MACs + cross-node connectivity (`curl` to a LoadBalancer IP returns
200 from the LAN). This is a one-time repair; unique node names keep it from recurring.
