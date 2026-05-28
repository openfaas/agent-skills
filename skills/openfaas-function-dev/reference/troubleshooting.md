# Function Troubleshooting Reference

Function-focused troubleshooting for OpenFaaS on Kubernetes. Drive every step
through `faas-cli` first. Reach for `kubectl` only as a break glass (see the
"Use kubectl only as a break glass" section in `SKILL.md`) and only when
`faas-cli version` confirms a Kubernetes server.

For non-function (gateway, queue-worker, operator, RBAC, control-plane) issues,
see the upstream guide: <https://docs.openfaas.com/deployment/troubleshooting/>.

## Scope: function configuration only

This reference is for diagnosing and changing **functions** — `stack.yaml`,
handler code, function env/secrets, function-level timeouts, image tags.

Several issues below have a root cause in the OpenFaaS **platform** itself
(gateway, queue-worker, operator, NATS, Prometheus, Helm values, ingress /
LoadBalancer, image-pull secrets in the function namespace). For those:

- **Do not** `kubectl edit`, `patch`, `apply`, `delete`, or `helm upgrade`
  OpenFaaS components or platform-level resources.
- **Do** clearly flag to the user: which component, which setting, current
  value (if known), recommended value, and why. Let the user (or their
  cluster operator) apply the change.
- Read-only inspection (`kubectl describe`, `kubectl logs`,
  `kubectl get … -o yaml`, `faas-cli diag`) is fine to gather evidence.

When in doubt, surface the finding rather than acting on it.

## First-line `faas-cli` diagnostics

Run these before anything else:

```bash
faas-cli list -v                   # see all functions, replicas, image, invocations
faas-cli describe <fn>             # status, URL, replicas, image, env, secrets
faas-cli logs <fn> --tail 200      # recent logs
faas-cli logs <fn> --follow        # stream
echo "ping" | faas-cli invoke <fn> # exercise the function end-to-end
```

If `describe` shows `replicas: 0` and the function is not on
[scale-to-zero](https://docs.openfaas.com/openfaas-pro/scale-to-zero/), or the
image / env / secrets do not match `stack.yaml`, the deploy did not land — see
[My function isn't updating](#my-function-isnt-updating).

## Bulk diagnostics with `faas-cli diag`

For deeper or shareable diagnostics, use the `diag` plugin. It collects
Function CR YAMLs, Kubernetes events, pod status, container logs (via stern),
and Prometheus metrics into a single tar.gz with an `index.html` report.

`faas-cli diag` is a break-glass tool at the same level as `kubectl`, not a
first-line `faas-cli` command. It talks directly to the Kubernetes API
(pods, events, Function CRs, Prometheus, stern-style logs) and needs a
kubeconfig with access to the `openfaas` namespace and any function
namespaces. Reach for it only when the gateway is on Kubernetes and the
first-line commands above did not give you enough to act on.

```bash
faas-cli plugin get diag
faas-cli diag config simple        # generate diag.yaml
faas-cli diag -d 5m diag.yaml      # collect for 5 minutes
# Output: ./run/<date>/index.html
```

Attach the tar.gz when asking for help.

## Debug helper functions from the store

The OpenFaaS function store ships several small functions that are very
useful while reproducing or isolating a function bug — `printer` to
inspect requests / async callbacks, `chaos` to drive error and timeout
paths, `env` to verify env / secrets injection, `curl` and `nslookup` to
debug in-cluster connectivity, and a few more.

These are equally valuable during normal development and testing, so they
are documented in their own reference: see
[reference/store-helpers.md](store-helpers.md).

## My function isn't updating

The most common cause is a re-pushed image with the same tag. Kubernetes will
not pull a new image if the tag has not changed (default
`imagePullPolicy: IfNotPresent`).

1. Check what's actually deployed:
   ```bash
   faas-cli describe <fn>
   ```
   Compare the `Image` field to what you just pushed.
2. Confirm the deploy succeeded (no auth/registry errors). Re-run with a fresh
   tag:
   ```bash
   faas-cli up -f stack.yml --tag=digest
   ```
   `--tag=digest` produces a new tag whenever the handler folder changes, so
   the cluster always pulls.
3. If the `Image` looks correct but the behaviour is stale, check pods are
   actually running the new image (break glass):
   ```bash
   kubectl get -n openfaas-fn deploy/<fn> -o yaml | grep image:
   kubectl get pods -n openfaas-fn -l faas_function=<fn> \
     -o jsonpath='{.items[*].spec.containers[*].image}'
   ```

See "Always advance the image tag on deploy" in `SKILL.md`.

## My function didn't start

`faas-cli describe <fn>` shows `Not Ready`, `0/1`, or restarts. Walk down this
ladder:

1. **Logs first** — most "didn't start" issues are application errors:
   ```bash
   faas-cli logs <fn>
   ```
2. **If logs are empty**, the container is not making it to a running state
   (image pull, missing secret, OOM, crash before logging). Break glass:
   ```bash
   kubectl get deploy -n openfaas-fn
   kubectl describe -n openfaas-fn deploy/<fn>
   kubectl get events -n openfaas-fn --sort-by=.metadata.creationTimestamp
   ```

Common root causes:

- **Missing secret** — `stack.yaml` lists `secrets: [foo]` but `foo` is not in
  the cluster. Create it: `faas-cli secret create foo --from-file foo.txt`.
- **Image pull error (`ErrImagePull` / `ImagePullBackOff`)** — private
  registry without an image-pull secret, typo in the image name, or wrong
  registry prefix. See
  <https://docs.openfaas.com/deployment/kubernetes/> for pull secrets.
- **Crash on boot** — error in handler code, missing dependency, bad
  `requirements.txt`/`go.mod`/`package.json`. Reproduce locally with
  `faas-cli local-run --build <fn>`.
- **OOMKilled** — function exceeded its `limits.memory`. Either raise the
  limit in `stack.yaml` or reduce memory usage.

## My Function CR is stalled (operator / CRD mode)

If you deploy a Function Custom Resource that fails validation (missing
secret, invalid `requests`/`limits`, parse error), `.Status` enters a
`stalled` condition. The operator backs off exponentially before retrying.

```bash
kubectl describe -n openfaas-fn function/<fn>
```

Look at the conditions / messages. After fixing the root cause:

- If you changed `.Spec`, the operator reconciles immediately.
- If `.Spec` is unchanged (e.g., you only created a missing secret), force a
  reconcile by bumping an annotation:
  ```bash
  kubectl annotate -n openfaas-fn function/<fn> \
    --overwrite com.openfaas.uid=$(uuidgen)
  ```

## Function name is too long

When using the Function CRD (OpenFaaS Standard / Enterprises), function names
are limited to **63 characters** because of Kubernetes label-selector limits.

Shorten the name and use a namespace for organisation:

```yaml
# Was: project-skunkworks-long-function-name
# Now:
functions:
  long-function-name:
    namespace: project-skunkworks
    ...
```

Namespaces require faasd or OpenFaaS for Enterprises.

## My function is timing out

Order of investigation:

1. **Function logs** — is the handler actually slow, hung, or erroring?
   ```bash
   faas-cli logs <fn> --follow
   ```
2. **Timeouts misconfigured** — every layer must allow the work to complete.
   Split by who owns the change:

   **Function-level (can be changed via `stack.yaml`):**
   - `read_timeout`, `write_timeout`, `exec_timeout` set as env on the
     function.

   **Platform-level (flag to the user, do not patch):**
   - Gateway `read_timeout`, `write_timeout`, `upstream_timeout`.
   - Cloud LoadBalancer idle timeout (often 60s on AWS NLB/ALB by default).
   - Service-mesh / ingress timeouts.

   When the function-level timeouts already allow the work to complete but
   one of the platform-level limits is shorter, report the specific
   component, current value (if visible via `kubectl get -o yaml` or Helm
   values), and the recommended value. See
   <https://docs.openfaas.com/tutorials/expanded-timeouts/>.
3. **Service mesh in use** (Linkerd, Istio) — the gateway needs
   `direct_functions: true` so it resolves the function by name and the
   mesh can load-balance. This is a gateway / Helm-values change — flag it,
   do not apply it yourself.
4. **Calling another function or the gateway from within a function** — use
   the in-cluster DNS name `http://gateway.openfaas:8080`, not the public
   URL. This is a function code change.

## My function gets a nil or empty body

The body is empty in the handler when the caller does not set a
`Content-Type` header. Send one:

```bash
curl -s -i http://127.0.0.1:8080/function/<fn> \
  -H "Content-Type: application/json" \
  --data-binary '{"id": 10}'
```

For legacy upstream HTTP servers that do not support chunked transfer
encoding (e.g., some WSGI stacks), set the of-watchdog env var to buffer the
request body:

```yaml
environment:
  http_buffer_req_body: true
```

## My function takes too long to start up

Symptoms: pods flap between `Ready` and `NotReady`, or the first request
after a scale-from-zero returns 500 / connection refused / times out.

Fix from the function side:

1. **Add a custom HTTP readiness endpoint** for any handler that does
   non-trivial init (model load, cache warm-up, opening a DB pool,
   downloading data). The handler returns `200` once init has finished
   and `500` while it is still running, then `stack.yaml` points
   OpenFaaS at it:

   ```yaml
   functions:
     my-fn:
       annotations:
         com.openfaas.ready.http.path: /ready
         com.openfaas.ready.http.initialDelaySeconds: 2
         com.openfaas.ready.http.periodSeconds: 2
   ```

2. **Extend the start-up window** for reliably slow but stable warm-ups
   by raising `com.openfaas.ready.http.initialDelaySeconds`, e.g. `60`
   for a 60-second model load.

For per-language handler patterns (Python, Node.js, Go) and the
`max_inflight` + `/_/ready` + `ready_path` combinator, see
[`health-readiness.md`](health-readiness.md).

## Testing a function without deploying it

For the normal local loop (`faas-cli local-run`, `--watch`, secrets via
`./.secrets/`), see "Testing with `local-run`" and "Iterating fast" in
`SKILL.md`. That is the right tool 99% of the time when reproducing a
function bug locally.

When you specifically need to bypass the watchdog (for example to confirm
whether an issue is in your handler vs. the watchdog/template, or to inject
arbitrary env vars / mounts that `local-run` does not expose), build the
image and run it directly with Docker:

```bash
faas-cli build -f stack.yml
docker run --rm -p 8081:8080 \
  -e some_var=value \
  -v $(pwd)/.secrets:/var/openfaas/secrets \
  <image>:latest
# invoke at http://127.0.0.1:8081
```

## Structured (JSON) logs

By default, lines from the function are wrapped in:

```
<RFC3339 Timestamp> <function name> (<container instance>) <msg>
```

To emit raw structured (e.g., JSON) lines, disable the prefix on the
function:

```yaml
environment:
  prefix_logs: false
```

See <https://docs.openfaas.com/cli/logs/>.

## Quick break-glass cheatsheet (Kubernetes only)

Use only when `faas-cli` cannot answer the question and the gateway is on
Kubernetes:

```bash
# Pod state, restarts, conditions
kubectl get pods -n openfaas-fn -l faas_function=<fn>
kubectl describe -n openfaas-fn pod/<fn>-<hash>

# Recent events in the function namespace
kubectl get events -n openfaas-fn --sort-by=.metadata.creationTimestamp

# Function CR status (operator / CRD mode)
kubectl describe -n openfaas-fn function/<fn>

# Raw deployment YAML, including the image actually scheduled
kubectl get -n openfaas-fn deploy/<fn> -o yaml
```

Stick to read-only verbs (`get`, `describe`, `logs`, `events`). Do not
`apply`, `edit`, `patch`, or `delete` function resources directly — make
those changes through `stack.yaml` and `faas-cli` so the source of truth
stays consistent.
