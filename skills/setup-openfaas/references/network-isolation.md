# Optional function-to-platform network isolation

Use only when the user explicitly requests this add-on. For a newly provisioned cluster, check enforcement prerequisites before installing OpenFaaS. Finish normal OpenFaaS installation and authenticated verification before applying the isolation policy. Keep the policy in a separate manifest alongside the release configuration; it is not a required Helm value or a default production setting.

## Scope and limits

The bundled [policy](../assets/network-isolation.yaml) restricts ingress to **all pods in the OpenFaaS installation namespace**, including gateway, provider/operator, dashboard, NATS, Prometheus, and IAM components. It allows all ports from namespaces whose `openfaas` label is not `"1"`, including namespaces without the label. It excludes all pods in labeled function namespaces.

This intentionally blocks a function calling the internal gateway, including gateway-mediated synchronous and asynchronous function chaining. It does not add function egress restrictions or function ingress restrictions, so DNS, internet egress, and direct function-to-function traffic remain subject to existing policies. This is not tenant-to-tenant isolation.

A function can still reach a public gateway through an allowed ingress or tunnel pod: that proxy's namespace identity replaces the function's identity. Authentication and authorization remain necessary. Do not claim this policy prevents every function invocation or public access to platform services.

## CNI and policy preflight

Warn before applying: **Kubernetes can accept a NetworkPolicy even when nothing enforces it. Identify the CNI and active policy engine, then verify traffic; a successful apply is not proof of isolation.** See the [Kubernetes NetworkPolicy requirements](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

| Network setup | Decision |
|---|---|
| Cilium, Calico, or another implementation enforcing standard NetworkPolicy | Keep the provider; check policy enforcement is enabled and all relevant pods are managed. No provider-specific policy CRDs are needed. |
| Stock K3s with Flannel and its embedded kube-router policy controller enabled | Can enforce this standard policy without a CNI swap. The controller is embedded in K3s, so absence of a kube-router DaemonSet is not evidence it is missing. |
| Flannel alone, disabled policy engine, or unknown/unsupported implementation | Do not report isolation as active. Establish a supported enforcer first. For K3s, read the separate [CNI migration notes](k3s-cni-migration.md); do not automatically replace networking. |

K3s supplies its own [NetworkPolicy controller](https://docs.k3s.io/networking/networking-services#network-policy-controller); Flannel itself does not enforce the policy. Check effective K3s server arguments, configuration files/drop-ins, and service environment for `disable-network-policy`, plus controller logs. Avoid dumping environment files containing cluster tokens. CNI names alone do not establish enforcement.

An enabled controller or startup message does not establish successful rule synchronization. Inspect recent K3s logs for `Aborting sync` and `iptables-restore` failures. Errors involving NFLOG, including `unknown option "--nflog-group"`, warrant checking both the userspace NFLOG extension and running kernel support (`CONFIG_NETFILTER_NETLINK_LOG`, `CONFIG_NETFILTER_XT_TARGET_NFLOG`, and corresponding modules when configured as loadable). Missing kernel support can prevent policy synchronization even when the controller starts successfully.

Do not infer an nftables incompatibility from that error alone or switch system iptables alternatives as the first fix. If comparing backends, use their explicit binaries with `iptables-restore --test` on a minimal NFLOG rule first; changing backends cannot supply missing kernel support. Resolve the capability gap or plan a supported CNI before proceeding. Traffic tests remain required even when prerequisite checks pass.

Inspect the target context and existing topology:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl -n kube-system get daemonsets,pods -o wide
kubectl get namespaces --show-labels
kubectl get networkpolicy -A
kubectl -n openfaas get pods,services -o wide
```

Also inspect provider-specific policies if their CRDs exist. Read the specifications of policies selecting installation pods: standard NetworkPolicies combine allowed traffic additively, so another ingress policy may reopen access. Do not silently remove existing policies. A separate default-deny object is unnecessary for this manifest and cannot override another allow rule.

Enumerate actual function namespaces from the installation/operator configuration and workloads, not only `kubectl get ns -l openfaas=1`: an unlabeled function namespace would be missed and allowed. Label each confirmed function namespace before applying the policy or deploying new functions:

```bash
kubectl label namespace <function-namespace> openfaas=1 --overwrite
```

Record prior labels before changing them and inspect any other policies that use this label. Do not label the installation namespace `openfaas=1`. Namespace labels and policy changes must remain controlled by trusted administrators; a tenant able to remove the label can escape this boundary.

## Check compatibility before applying

- Locate ingress controllers and tunnel clients. A separate inlets client pod in an allowed namespace can still forward to the gateway. Direct external clients are not selected by a namespace selector; a direct LoadBalancer path may be blocked. Host-network, NodePort, and source-NAT behavior varies with routing and provider. Verify each actual exposure path rather than assuming external access survives.
- Identify functions that call `gateway.openfaas:8080`, other installation services, or platform-hosted authentication endpoints. Explain these intentional breaks before applying. If the requested behavior depends on such calls, resolve the design before enabling this policy.
- For watchdog JWT authentication, check where the deployed image fetches OIDC discovery and JWKS. Internal gateway discovery is blocked and can prevent new function replicas from starting. A healthy old replica may only have cached keys; validate a fresh start or scale-from-zero and key refresh behavior.
- A reachable public discovery URL is viable only if the deployed watchdog version supports configuring it and the advertised JWKS URI is also reachable. Verify the image's supported settings and URL format before proposing an override. Do not silently rebuild functions, disable JWT checks, or add broad policy exceptions. A discovery proxy inside the isolated namespace is blocked too.
- Check other workloads sharing the installation namespace, including optional builders/registries. `podSelector: {}` covers them as well. Tailor a different policy only when the user requests a different boundary.

## Prepare and apply

Copy [assets/network-isolation.yaml](../assets/network-isolation.yaml) to the deployment's maintained configuration directory and adapt `metadata.namespace` if the installation uses a custom namespace. Keep `podSelector: {}` for namespace-wide isolation.

For an existing deployment, retain the installed policy's name and management method, even when changing its pod selector. Update that object in place rather than leaving duplicate policies behind, and save its prior manifest for rollback. For GitOps-managed policy, make the change through its owning repository.

With `network-isolation.yaml` denoting the prepared local manifest:

```bash
kubectl apply --dry-run=server -f network-isolation.yaml
kubectl diff -f network-isolation.yaml
kubectl apply -f network-isolation.yaml
kubectl -n openfaas describe networkpolicy openfaas-allow-non-function-namespaces
```

Adapt namespace and policy name consistently. `kubectl diff` exit status 1 means differences exist, not that validation failed. Record the apply time for post-change checks. Do not change the CNI, Helm release, or function images as a side effect of applying this manifest.

## Verify the intended boundary

Capture a baseline before applying, then repeat with **new connections** after enforcement converges. Use current Service/pod/node addresses from the cluster. Test from an actual function pod in every function namespace, including across nodes where applicable; workstation requests or port-forwards cannot prove this boundary. Prefer existing workloads with the needed tools; if a disposable probe is necessary, keep it in the namespace being tested and remove it afterwards.

| Source and destination | Expected after applying |
|---|---|
| Each function namespace → internal gateway Service and pod IP, ports 8080/8081/8082 | Connection blocked, including provider and metrics ports |
| Each function namespace → dashboard, NATS, Prometheus, OIDC plugin, Signet, and other installation pods on their actual ports | Connection blocked; check both Service and representative pod IP paths |
| Function → exposed gateway NodePort/node IP or other external Service path | Test explicitly; NAT/host behavior can differ. Report any path that still reaches it. |
| Intended external client → actual gateway NodePort, ingress, or tunnel | Verify the configured access path. A different Service's NodePort or a node-local request is not equivalent. |
| Allowed installation/ingress/tunnel pod → gateway and other required platform services | Still reachable |
| Client → authenticated `faas-cli list`, synchronous and asynchronous invocation | Succeeds through the intended access path; async needs queue-worker completion status, not only HTTP 202 or an invocation count |
| Prometheus → gateway/provider metrics (commonly 8082 and 8081) | Targets healthy with fresh post-apply scrape times |
| Public dashboard and IdP/Signet | Required flows still work; endpoint health alone does not verify browser SSO |
| Function → DNS, internet HTTPS, and direct function in another namespace | Baseline behavior retained |
| Function → public gateway through ingress/tunnel | May remain reachable; document the proxy bypass and check the intended authentication |
| JWT-enabled function starting a new replica | Discovery/JWKS reachable through the chosen route and authentication still enforced |

Example for an existing function with `curl` (substitute the real namespace and deployment):

```bash
kubectl -n <function-namespace> exec deploy/<function> -- \
  curl -sS --connect-timeout 3 --max-time 5 http://gateway.openfaas:8080/healthz
```

A timeout or policy rejection must be distinguished from DNS failure or an unhealthy server by checking the same endpoint from an allowed source. HTTP 401/403 means the network connection succeeded and is not evidence of a network deny. For non-HTTP ports such as NATS 4222, use a bounded TCP probe rather than HTTP response status. Inspect provider drop diagnostics if results are ambiguous.

Report a pass, fail, not-applicable, or not-tested result for each matrix row, with the actual source and destination. If access was tested only through an administrative port-forward, say so; it does not prove ingress or NodePort access survives the policy. Do not describe isolation as fully verified while applicable paths remain untested. Include CNI/enforcer, namespaces/labels, policy name and retained file, and exposure limits. Re-run relevant checks after label, policy, CNI, or exposure changes.

## Rollback

If newly created, remove only this add-on policy:

```bash
kubectl -n openfaas delete networkpolicy openfaas-allow-non-function-namespaces
```

If updating a pre-existing policy, restore its saved manifest through the owning management workflow instead. Re-test previous connectivity. Removal restores unrestricted ingress only if no other selecting policies remain. Do not delete unrelated policies, undo the CNI, or remove function-discovery labels as a policy rollback.
