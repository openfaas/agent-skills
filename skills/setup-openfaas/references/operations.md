# Operations and verification

Use these checks after installation and before declaring success. Adapt deployment names to the rendered chart instead of assuming every optional component is enabled.

## Verify the Helm release and workloads

```bash
helm status openfaas -n openfaas
helm get values openfaas -n openfaas
kubectl get pods,deployments,services -n openfaas
kubectl get events -n openfaas --sort-by=.lastTimestamp
```

Do not describe `CrashLoopBackOff` as expected success. Allow dependencies time to start, then investigate persistent restarts with `kubectl describe` and targeted container logs.

For Standard or For Enterprises, verify the operator CRDs and Pro workloads:

```bash
kubectl get crd functions.openfaas.com profiles.openfaas.com
kubectl get functions.openfaas.com -A
kubectl get deployments -n openfaas
```

Use the appropriate authentication flow in [pro-cli.md](pro-cli.md). That workflow retries login and an authenticated operation with a bounded deadline until the gateway is ready. After IAM authentication, use the same bounded `faas-cli list` retry pattern. Then record:

```bash
faas-cli version
faas-cli list
```

`faas-cli version` verifies the gateway/provider versions and `faas-cli list` verifies an authenticated API operation.

Do not improvise with `/ping` (it does not exist), unauthenticated `/system/*` requests (401 does not establish health), unbounded `kubectl get pods -w`, or rollout/Ready polling loops. If the authenticated operation misses its deadline, inspect a state snapshot immediately: Helm status, pod state, events, and targeted logs. For a crashing gateway, validate the license Secret as a normalized one-line JWT without printing it rather than only checking that the Secret exists; for `CreateContainerConfigError`, compare referenced Secret names with the pre-install checklist.

These checks are the default post-install verification boundary. Do not install a container build stack or deploy a test function merely to make routine verification more elaborate. When the user requests a function invocation or asynchronous-path test, prefer an existing suitable function or image. If a disposable function must be built or deployed, state why, keep every artifact inside the owned test environment, and remove the Function object, workloads, images, build files, background port-forwards, and any added build tooling when the test ends.

When enabled, verify the dashboard separately. Use its ingress when configured; otherwise manage the port-forward in the same shell and clean it up explicitly:

```bash
OPENFAAS_DASHBOARD_PF_LOG=$(mktemp)
kubectl port-forward -n openfaas svc/dashboard 8081:8080 >"$OPENFAAS_DASHBOARD_PF_LOG" 2>&1 &
OPENFAAS_DASHBOARD_PF_PID=$!
trap 'kill "$OPENFAAS_DASHBOARD_PF_PID" 2>/dev/null || true; wait "$OPENFAAS_DASHBOARD_PF_PID" 2>/dev/null || true; rm -f "$OPENFAAS_DASHBOARD_PF_LOG"' EXIT
OPENFAAS_DASHBOARD_READY=0
for OPENFAAS_ATTEMPT in $(seq 1 60); do
  if curl -fsS http://127.0.0.1:8081/ >/dev/null; then OPENFAAS_DASHBOARD_READY=1; break; fi
  sleep 2
done
test "$OPENFAAS_DASHBOARD_READY" -eq 1
kill "$OPENFAAS_DASHBOARD_PF_PID" 2>/dev/null || true
wait "$OPENFAAS_DASHBOARD_PF_PID" 2>/dev/null || true
rm -f "$OPENFAAS_DASHBOARD_PF_LOG"
trap - EXIT
unset OPENFAAS_DASHBOARD_PF_LOG OPENFAAS_DASHBOARD_PF_PID OPENFAAS_DASHBOARD_READY OPENFAAS_ATTEMPT
```

## Upgrade

Retain the exact custom values file. Before upgrading:

1. Read the current [Pro deployment and upgrade notes](https://docs.openfaas.com/deployment/pro/).
2. Update the Helm repository.
3. Compare current release values with the retained file.
4. Render the proposed release with `helm template`.
5. Use `helm upgrade --install`; do not uninstall first.
6. Repeat all relevant verification checks.

Changing from `clusterRole: false` to `true` may produce RBAC ownership conflicts. Remove only the exact conflicting objects Helm identifies and only after confirming they belong to this release.

For GitOps, pre-create stable Basic Auth credentials and set `generateBasicAuth: false`; otherwise the chart's pre-install hook can generate credentials that do not behave well under reconciliation. Follow the chart's current [GitOps and credential guidance](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md).

## Production decisions

Review [production guidance](https://docs.openfaas.com/architecture/production/) with the user. In particular, decide on:

- ingress, trusted TLS, and DNS
- persistent monitoring and Grafana dashboards
- external persistent NATS for critical asynchronous workloads
- backups and recovery for configuration and secrets
- resource requests/limits, availability, and node placement
- regular chart upgrades and support requirements
- air-gap/image mirroring when Internet egress is restricted

Do not silently expand a core installation into these adjacent systems. Report them as unresolved production-readiness items when they are not in scope.

## Troubleshooting

Start with Helm status, events, pod descriptions, and logs for the failing component. Check image pulls, license-secret presence, chart values, NATS readiness, webhook/CRD state, DNS, and network policy before changing resources. Consult the official [troubleshooting guide](https://docs.openfaas.com/deployment/troubleshooting/) and current chart documentation.

For diagnostics requested by the user or OpenFaaS support, prefer the supported `faas-cli diag` plugin:

```bash
faas-cli plugin get diag
faas-cli diag config simple
faas-cli diag -d 5m diag.yaml
```

Do not run diagnostics as a routine post-install check. The command collects Kubernetes resources, events, logs, and Prometheus data into an archive; tell the user where it was written and treat it as potentially sensitive before sharing it. Do not use or recommend the legacy standalone config-checker.
