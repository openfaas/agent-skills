# Separate K3s CNI replacement notes

Read only when requested network isolation lacks a working policy enforcer, or the user explicitly requests CNI replacement. **Stock K3s with Flannel and a working embedded kube-router policy controller does not need a CNI swap.** Check the [enforcement preflight](network-isolation.md#cni-and-policy-preflight), including controller sync errors and NFLOG support. If the controller was disabled, investigate why and whether restoring it is appropriate before proposing replacement. Leave a healthy Cilium/Calico installation in place.

CNI replacement disrupts cluster networking and recreates pod sandboxes. Present a concrete migration/recovery plan and use existing authorization for that work; a request to install OpenFaaS or add a NetworkPolicy alone does not authorize taking down the cluster. The procedure below covers Flannel-to-Cilium replacement during a single-node K3s maintenance outage. Multi-node clusters require a topology-specific migration plan.

## Prepare the migration

1. Identify the effective K3s configuration on every server/agent, active CNI, policy engine, kube-proxy mode, pod/Service CIDRs, host routes, and K3s/Cilium versions. Inspect only relevant settings; service environment/configuration can contain tokens.
2. Retain configuration and workload manifests, current policies/labels, and a recoverable datastore/volume backup appropriate to the installation. Confirm node console access independent of pod networking and a maintenance window for the outage. Check restart-sensitive dependencies such as license expiry before restarting OpenFaaS.
3. Select a supported Cilium version and IPAM/routing settings using the [Cilium K3s guide](https://docs.cilium.io/en/stable/installation/k3s/) and its linked requirements. Account for the real pod CIDR and avoid host/Service network overlaps.
4. For multi-node or availability-sensitive clusters, prepare a provider-supported plan using [Cilium migration guidance](https://docs.cilium.io/en/stable/installation/k8s-install-migration/) or move workloads to a replacement cluster. Do not run the single-node teardown on all nodes as a rolling upgrade.

## Configure K3s for Cilium

Merge these settings into `/etc/rancher/k3s/config.yaml` on every K3s server, preserving unrelated settings and checking overrides from arguments/drop-ins:

```yaml
flannel-backend: none
disable-network-policy: true
```

The first disables Flannel; the second avoids competing policy engines. This procedure retains K3s kube-proxy and explicitly sets Cilium's `kubeProxyReplacement=false` below. K3s embeds kube-proxy rather than running a DaemonSet; Cilium CLI autodetection can mistake the missing DaemonSet for an absent proxy and enable replacement. If replacement is deliberately chosen instead, configure K3s `disable-kube-proxy: true` and Cilium's API-server connection settings using the [Cilium K3s guide](https://docs.cilium.io/en/stable/installation/k3s/). Do not silently accept a different mode from the migration plan.

Disabling the old controller leaves its rules behind; follow the [K3s controller cleanup guidance](https://docs.k3s.io/networking/networking-services#network-policy-controller) on affected nodes. If troubleshooting changed iptables alternatives, restore or deliberately retain that choice and inspect both legacy and nftables rulesets for obsolete rules. Cleanup through the currently selected backend alone may leave the other backend's rules active. Preserve unrelated firewall rules.

## Replace pod networking during single-node maintenance

A plain `systemctl restart k3s` can leave containerd and old pod sandboxes alive when the service uses `KillMode=process`. Cilium may then manage new pods while old pods retain Flannel networking. Inspect the unit and actual pod management state; do not infer a complete migration from the service becoming active.

For the planned single-node outage, K3s provides [k3s-killall.sh](https://docs.k3s.io/upgrades/killall) to stop containers and reset networking without deleting cluster data. It is not an uninstall command. Before using it, inspect for Cilium leftovers from an earlier attempt: K3s specifically warns that Cilium interfaces and rules need cleanup before killall/uninstall to avoid losing host connectivity. Follow [K3s custom-CNI cleanup](https://docs.k3s.io/networking/basic-network-options#custom-cni) for the installed configuration.

Once those preconditions and recovery access are in place:

```bash
sudo /usr/local/bin/k3s-killall.sh
```

Verify that the old K3s server has exited and port 6443 is free before starting it again. Avoid broad `pkill -f` patterns that can match the controlling shell. Start without tying up the session, then inspect with a bounded deadline:

```bash
sudo systemctl start --no-block k3s
sudo systemctl is-active k3s
sudo journalctl -u k3s --since=-5m --no-pager -n 80
```

The API must become reachable before installing Cilium; nodes/pods may remain unready until the CNI is installed. If startup misses the chosen deadline, inspect logs and networking instead of repeatedly restarting.

## Install and verify Cilium

Install the CLI using the upstream guide (or `arkade get cilium` when arkade is already available). Use the intended kubeconfig; `/etc/rancher/k3s/k3s.yaml` is usually root-only. Example from the node, substituting a validated version and pod CIDR:

```bash
sudo env KUBECONFIG=/etc/rancher/k3s/k3s.yaml "$HOME/.arkade/bin/cilium" install \
  --version <cilium-version> \
  --set kubeProxyReplacement=false \
  --set 'ipam.operator.clusterPoolIPv4PodCIDRList=<pod-cidr>'
sudo env KUBECONFIG=/etc/rancher/k3s/k3s.yaml "$HOME/.arkade/bin/cilium" status
sudo kubectl -n kube-system get configmap cilium-config \
  -o jsonpath='{.data.kube-proxy-replacement}{"\n"}'
sudo kubectl get nodes
sudo kubectl get pods -A -o wide
```

Confirm the effective Cilium proxy setting matches the plan (`false` here) and K3s's configuration. Coexistence is possible, but the two proxies maintain independent NAT state; changing modes can break connections. See [Cilium's coexistence guidance](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/).

The install command returning is not readiness. Use bounded status checks and investigate failures. Check the Cilium agent/operator health, CNI configuration, endpoint coverage for every non-host-network workload, and replacement of old Flannel sandboxes/interfaces. Pod IP ranges alone do not prove Cilium manages a pod, especially when retaining the original CIDR. During migration recovery, inspect Deployment/DaemonSet conditions and active pod status; completed Helm Job pods can display `0/1` and must not keep a readiness loop running. Report a missed deadline rather than continuing as if the check passed.

Verify DNS, Service routing, cross-node traffic when applicable, external access, and normal authenticated OpenFaaS operations before applying the optional [isolation policy](network-isolation.md). Then run its allowed/blocked traffic matrix. An already migrated, healthy cluster needs only policy work.

## Troubleshoot migration failures

- **Stale Cilium rules:** after a failed install/teardown, orphaned firewall/TPROXY rules can block loopback TCP to the API. If startup stalls at `Reconciling bootstrap data between datastore and disk`, port 6443 is listening but connections remain in `SYN-SENT`, or CA fetches time out, check API reachability, processes, interfaces, and firewall rules before attributing the failure to datastore corruption.
- **Firewall cleanup:** use documented provider-specific cleanup for the active firewall backend, preserving unrelated rules and console access. Flushing all nftables/iptables state removes unrelated firewall and remote-access protections.
- **Partial migration:** healthy Cilium agents with incomplete workload endpoint coverage and old Flannel sandboxes indicate an unfinished migration. Recreate the remaining old sandboxes through the chosen maintenance plan and verify coverage again.
- **Rollback:** deleting a NetworkPolicy does not restore the prior CNI. If migration fails, use the prepared recovery plan to restore the former CNI/configuration and recreate pod networking, or recover the cluster from backup. Do not simply re-enable Flannel alongside active Cilium.
