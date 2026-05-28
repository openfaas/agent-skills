---
name: openfaas-function-dev
description: Develops and troubleshoots OpenFaaS serverless functions in Python, Node.js, or Go using faas-cli. Use when scaffolding from templates, writing handlers, configuring stack.yaml, adding dependencies or secrets, building images, deploying to OpenFaaS, iterating locally with local-run, scheduling functions on a cron timer, wiring functions to event-sources (Kafka, Postgres, SQS, SNS, RabbitMQ, Pub/Sub, Cron), adding a custom readiness endpoint for handlers that initialize state (model load, cache warm-up, connection pools), or diagnosing function issues like image-pull errors, missing secrets, timeouts, empty bodies, slow start-up, or stalled Function CRs.
---

# OpenFaaS Function Development

## Pre-flight checks

Before generating any function code, verify the toolchain is available and configured.

1. Check `faas-cli` is installed: `faas-cli version`. If missing, install with `arkade get faas-cli` or `curl -sSL https://cli.openfaas.com | sudo sh`.
2. Inspect the output of `faas-cli version` for the gateway's orchestration provider. If it reports a Kubernetes server version, the cluster is Kubernetes-backed and `kubectl` is available as a break-glass option (see [Use kubectl only as a break glass](#use-kubectl-only-as-a-break-glass)).
3. Confirm Docker is running: `docker info`.
4. If deploying to a cluster, expect `OPENFAAS_URL` and `OPENFAAS_PREFIX` to be set (the latter is the image registry prefix used by `faas-cli new`, e.g. `ghcr.io/acme` or `ttl.sh/<user>`).

## Core workflow

The standard inner loop is:

```bash
faas-cli new --lang <template> <fn-name>     # scaffold
# edit ./<fn-name>/handler.<ext>
faas-cli local-run --build <fn-name>          # build + run locally with Docker (preferred)
# OR full cycle:
faas-cli up -f <fn-name>.yml                  # build + push + deploy
```

Prefer `faas-cli local-run --build` for local iteration. It builds the image and runs it in a single command, so a separate `faas-cli build` step is unnecessary. Only run `faas-cli build` on its own when you specifically need the image without running it (e.g. to push, inspect, or hand to another tool).

See [Testing with `local-run`](#testing-with-local-run) below for the local loop and [Iterating fast](#iterating-fast) for live-reload options.

## Testing with `local-run`

`faas-cli local-run --build` builds the function and starts it as a Docker container that listens on `http://127.0.0.1:8080` in a single step — there is no need to run `faas-cli build` first. Without `--build`, `local-run` will only run an existing image. **It is a long-running, foreground process** — it does not return until you stop it (Ctrl+C). Do not pipe input to it or expect it to exit on its own.

The workflow is two steps:

1. Start the function in a separate process — either in another terminal, in a tmux pane, or as a background process:

   ```bash
   faas-cli local-run --build my-fn &           # background, same shell
   # or run in another terminal:
   faas-cli local-run --build my-fn
   ```

2. Once you see `Listening on port: 8080`, invoke the function from elsewhere:

   ```bash
   curl -i http://127.0.0.1:8080 -d "hello"
   ```

When done, stop the container with Ctrl+C (foreground) or `kill %1` (background).

If port 8080 is already in use, override it with `--port`:

```bash
faas-cli local-run --build my-fn --port 3001
```

If `stack.yaml` has multiple functions, pass the function name as an argument: `faas-cli local-run --build <fn-name>`.

## Choosing a template

List available templates: `faas-cli template store list`. Recommended defaults:

| Language | Template | Notes |
|---|---|---|
| Python | `python3-http` | Flask-based HTTP. Default. |
| Python (native deps) | `python3-http-debian` | Debian base, supports `build_options` like `libpq`. |
| Node.js | `node24` | Express-based, async/await handler. |
| Go | `golang-middleware` | Uses `http.HandleFunc` from stdlib, supports modules and vendoring. |

Pull a template explicitly with `faas-cli template store pull <name>` if not auto-fetched.

## Scaffolding a function

```bash
export OPENFAAS_PREFIX=ttl.sh/<user>          # or your registry prefix
faas-cli new --lang python3-http hello-python
```

This creates:
- `hello-python/handler.py` (or `.js`, `.go`, etc.) — your code
- `hello-python/requirements.txt` (or `package.json`, `go.mod`) — dependencies
- `hello-python.yml` — the stack file

To append additional functions to the same stack file, use `--append`:

```bash
faas-cli new --lang node24 echo --append hello-python.yml
```

## Handler shapes per language

See [reference/handlers.md](reference/handlers.md) for complete handler templates and examples per language (Python, Node, Go).

Quick reference:

**Python (`python3-http`)** — return a dict with `statusCode`, `body`, optional `headers`. `event` exposes `body`, `headers`, `method`, `query`, `path`.

**Node.js (`node24`)** — `module.exports = async (event, context) => context.status(200).succeed(result)`. Async/await supported.

**Go (`golang-middleware`)** — `func Handle(w http.ResponseWriter, r *http.Request)` — write to `w`, read from `r`. Always `defer r.Body.Close()` if reading.

## stack.yaml essentials

Authoritative reference: [reference/stack-yaml.md](reference/stack-yaml.md). Common fields:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  my-fn:
    lang: python3-http
    handler: ./my-fn
    image: ${REGISTRY:-ttl.sh/myuser}/my-fn:${TAG:-latest}
    environment:
      LOG_LEVEL: info
    secrets:
      - my-fn-token
    build_options:
      - libpq                   # named bundle from template.yml
    build_args:
      ADDITIONAL_PACKAGE: "jq"  # extra apk/apt packages
    limits:
      memory: 128Mi
      cpu: 100m
    requests:
      memory: 64Mi
      cpu: 50m
    annotations:
      topic: kafka.payments-received
configuration:
  templates:
    - name: python3-http
      source: https://github.com/openfaas/python-flask-template
```

Notes:
- `image` supports `${VAR:-default}` envsubst — keep registry overridable.
- `handler` is always a folder, never a filename.
- Function name must be a DNS label (lowercase alphanumeric and `-`, max 63 chars).
- Pin templates by appending `#tag-or-branch` to the `source` URL for reproducible builds.
- `configuration.copy` lists project sub-paths to inject into every function's handler folder (for shared `./common`, `./models`, etc.).

## Dependencies

- **Python**: list packages in `<fn>/requirements.txt`. For native libs (e.g. `psycopg2`), use `python3-http-debian` and add `build_options: [libpq]` or `build_args: { ADDITIONAL_PACKAGE: "libpq-dev gcc python3-dev" }`.
- **Node**: `cd <fn> && npm install --save <pkg>`. Tests in `<fn>/test/` run automatically during `faas-cli build`.
- **Go**: `cd <fn> && go get <module>`. For private modules, set `GOPRIVATE=...`, run `go mod vendor`, commit `vendor/`. For sub-packages, add `replace handler/function => ./` to `go.mod`.

## Secrets

Always read secrets from files, never environment variables:

1. Create the secret: `faas-cli secret create my-token --from-file my-token.txt`. Prefer `faas-cli` for routine work; only fall back to `kubectl create secret` as a break glass when `faas-cli` cannot do the job and `faas-cli version` confirms a Kubernetes server (see [Use kubectl only as a break glass](#use-kubectl-only-as-a-break-glass)).
2. Bind in `stack.yaml`:
   ```yaml
   secrets:
     - my-token
   ```
3. Read at runtime from `/var/openfaas/secrets/my-token`.

For `local-run`, place secret files in a `.secrets/` directory **at the project root, next to `stack.yaml`** — for example `./.secrets/my-token`. `faas-cli local-run` bind-mounts this top-level directory into the running container at `/var/openfaas/secrets/`.

**Never create `.secrets/` inside a function's handler folder** (e.g. `./my-fn/.secrets/`). The handler folder is copied into the image during `faas-cli build`, which would bake the secret values directly into the published container image and leak them to anyone who can pull the image. Always keep `.secrets/` outside every handler folder, and add `.secrets/` to `.gitignore` and any `.dockerignore` files.

Update with `faas-cli secret update`. Re-read the file on each invocation (or use `fsnotify`) since the kubelet rotates the mounted file.

## Build/deploy commands

| Command | Purpose |
|---|---|
| `faas-cli build -f stack.yml` | Build image into local Docker library (rarely needed on its own — prefer `local-run --build` for local testing or `up` for deploy) |
| `faas-cli push -f stack.yml` | Push image to registry |
| `faas-cli deploy -f stack.yml` | Deploy to cluster |
| `faas-cli up -f stack.yml` | Build + push + deploy in one step |
| `faas-cli local-run --build [name]` | Build + run as a local Docker container on `:8080` (preferred local loop; omit `--build` only if the image is already built) |
| `faas-cli publish --platforms linux/arm64,linux/amd64` | Multi-arch build + push (use instead of build/push for ARM) |
| `faas-cli build --shrinkwrap` | Emit a `./build/<fn>/Dockerfile` for use with external builders |
| `faas-cli generate -f stack.yml` | Convert stack.yml to Kubernetes CRD YAML for GitOps |
| `faas-cli list -v` / `faas-cli describe <fn>` | Inspect deployed functions |
| `faas-cli logs <fn>` | Tail function logs |
| `echo "data" \| faas-cli invoke <fn>` | Invoke a deployed function |

Do not pass `--skip-push` when deploying to a Kubernetes cluster — the cluster pulls from a registry, so skipping the push leaves it with a missing or stale image. Only use `--skip-push` for faasd on the same host or when you have loaded the image into the cluster yourself (`kind load docker-image`, etc.).

## Always advance the image tag on deploy

**Critical:** Kubernetes will not pull a new image if the tag has not changed. Depending on the cluster's `imagePullPolicy` (commonly `IfNotPresent`), pushing a new image to the same tag — including `:latest` — can leave the old image running in the function pod. Every deploy must use a new, unique image tag.

Two ways to ensure tags advance:

1. **Let the CLI generate a unique tag** by passing `--tag` to `build`, `push`, `publish`, `deploy`, or `up`:

   | Value | Tag suffix | When to use |
   |---|---|---|
   | `--tag=digest` | content hash of the handler folder | Rebuilds and watch loops; new tag only when code actually changes. Recommended for `--watch`. |
   | `--tag=sha` | short Git commit SHA | CI/CD on a clean working tree. |
   | `--tag=branch` | branch + short SHA | Preview deploys per branch. |
   | `--tag=describe` | output of `git describe` | Release builds with annotated tags. |

   The generated value is appended to whatever tag is in `stack.yaml` (or `latest` if none). Example:

   ```bash
   faas-cli up -f stack.yml --tag=digest
   # builds and deploys image: ghcr.io/acme/my-fn:latest-abc123…
   ```

2. **Bump the tag manually in `stack.yaml`** using `envsubst`:

   ```yaml
   image: ghcr.io/acme/my-fn:${TAG:-0.1.0}
   ```

   ```bash
   TAG=0.1.1 faas-cli up
   ```

Avoid relying on `:latest` for cluster deploys. Reserve `:latest` for `local-run` only. The `--tag` flag combines with environment-variable substitution, so a CI pipeline can set a base version in `stack.yaml` and append `--tag=sha` for traceability.

## Iterating fast

- Use `faas-cli local-run --watch` for the tightest loop on a single function (no cluster needed).
- Use `faas-cli up --watch --tag=digest` when functions need cluster services (other functions, gateway). The `--tag=digest` is required here so each save produces a unique tag the cluster will actually pull.
- Use `ttl.sh/<user>` as registry for throwaway images during prototyping. **Warning: ttl.sh is a public, anonymous registry** — anyone who guesses the image path can pull it. Never use it for proprietary or customer code, secrets baked into images, or anything you would not publish openly. For private workloads use a private registry (GHCR private, ECR, GCR, Docker Hub private repo, Harbor, etc.) and run `faas-cli registry-login` to authenticate.

## Readiness checks for initialization

If the handler does any work before it can serve requests — loading an ML
model, warming a cache, downloading data, opening a DB / SDK client —
**add a custom readiness check**. Without one, the first request after
scale-from-zero (or after a new Pod starts) hits a Pod that is up but not
yet able to handle work, returning 500 / timeout.

Two pieces:

1. Handler exposes a path (conventionally `/ready`) that returns **200**
   when init has finished and **500** while it is still running. Run init
   off the request path (Python module load or `threading.Thread`, Node
   top-level async IIFE, Go `init()` + goroutine setting an
   `atomic.Value`).
2. `stack.yaml` points OpenFaaS at that path:

   ```yaml
   functions:
     my-fn:
       annotations:
         com.openfaas.ready.http.path: /ready
         com.openfaas.ready.http.initialDelaySeconds: 2
         com.openfaas.ready.http.periodSeconds: 2
   ```

For long warm-ups (large model load, big cache), raise
`initialDelaySeconds` so the first probe fires after init is expected to
complete (e.g. `60` for a 60-second load) rather than probing on a tight
loop.

Liveness (`com.openfaas.health.http.*`) is separate and rarely needed —
the watchdog ships a built-in liveness probe. Use readiness for "alive
but not yet able to take traffic"; use a custom liveness only when the
process should actually be killed and restarted on some condition.

For per-language handler patterns (Python / Node / Go), the full
annotation list, and the `max_inflight` + `/_/ready` + `ready_path`
watchdog combinator for hard concurrency limits, read
[reference/health-readiness.md](reference/health-readiness.md).

## Triggers and scheduling

Functions are invoked over HTTP by default. To run them on a schedule or from an external event-source, use an official OpenFaaS **event-connector** (Cron, Kafka, Postgres, SQS, SNS, Pub/Sub, RabbitMQ) by adding a `topic:` annotation in `stack.yaml`. Do not build polling or subscription SDKs into the handler.

**For scheduled / cron-style invocations, use the OpenFaaS cron-connector — never a Kubernetes `CronJob`.** The cron-connector is portable across all providers (Kubernetes CE/Pro, OpenFaaS Edge/faasd). When the user needs a schedule, read [reference/cron-schedule.md](reference/cron-schedule.md) for the annotation pattern and rules.

For the full list of available triggers and the generic connector wiring pattern, read [reference/triggers.md](reference/triggers.md).

## Helper functions from the store

The OpenFaaS function store ships small pre-built functions that are
useful while developing, testing, and debugging your own functions. Deploy
them on demand with `faas-cli store deploy <name>` (`faas-cli store list`
to browse). They live in the cluster, so they share the function's
network and DNS context.

Common picks:

- `printer` — pretty-prints incoming requests (headers + body) into its
  logs. Use as a chained target or async `X-Callback-Url`, then
  `faas-cli logs printer -t` to inspect what arrived.
- `chaos` — returns canned status codes / delays / bodies via `/set`.
  Drive error paths, retry / timeout handling without touching your code.
- `env` — dumps the function's env vars; verify `environment:` and
  watchdog config were applied.
- `sleep` — sleeps a configurable duration; reproduce timeouts.
- `curl` / `nslookup` — debug in-cluster connectivity and DNS from a
  function's network context.

Remove these when you're done — do not leave debug helpers running in
production. For full patterns and examples, read
[reference/store-helpers.md](reference/store-helpers.md).

## Use kubectl only as a break glass

Drive function development, deployment, and inspection through `faas-cli`. Do not reach for `kubectl` for routine tasks like deploying, listing, invoking, scaling, fetching logs, or managing secrets — `faas-cli` covers these consistently across providers (Kubernetes, faasd, etc.).

Only use `kubectl` when **both** of these are true:

1. `faas-cli version` reports a Kubernetes server (the gateway is running on Kubernetes). On non-Kubernetes providers like faasd, `kubectl` is not applicable — do not use it.
2. The task is a genuine break-glass or deeper-debugging scenario that `faas-cli` cannot handle, for example:
   - Inspecting pod-level state: `kubectl -n openfaas-fn describe pod <fn>-<hash>`, `kubectl -n openfaas-fn get events`.
   - Diagnosing image pull, scheduling, or CrashLoopBackOff issues that `faas-cli logs` and `faas-cli describe` cannot explain.
   - Examining the OpenFaaS control plane in `openfaas` namespace (gateway, queue-worker, operator).
   - Checking node, RBAC, NetworkPolicy, or admission-controller behaviour interfering with a function.

When you do use `kubectl`, prefer read-only commands (`get`, `describe`, `logs`, `events`). Avoid mutating function resources (`apply`, `edit`, `delete`, `patch`) — make those changes through `stack.yaml` and `faas-cli` so the source of truth stays consistent. Note your reason for breaking glass in the response so the user understands why `faas-cli` was insufficient.

## Troubleshooting

When a function misbehaves, start with `faas-cli` and only fall back to
`kubectl` per [Use kubectl only as a break glass](#use-kubectl-only-as-a-break-glass):

```bash
faas-cli list -v                   # replicas, image, invocations
faas-cli describe <fn>              # current image/env/secrets/status
faas-cli logs <fn> --tail 200       # recent function logs
echo "ping" | faas-cli invoke <fn>  # exercise end-to-end
```

**Scope: function configuration only.** This skill is for diagnosing and
changing **functions** — `stack.yaml`, handler code, function env/secrets,
function-level timeouts. Some fixes below require changing the OpenFaaS
**platform** itself (gateway, queue-worker, operator, NATS, Prometheus,
Helm values, ingress / LB). **Do not patch, edit, or `helm upgrade`
OpenFaaS components.** Surface the suspected platform-level change to the
user — what component, what setting, why — and let them apply it (or
involve their cluster operator).

For deeper, shareable cluster-wide diagnostics use the `diag` plugin.
`faas-cli diag` is a break-glass tool at the same level as `kubectl` — it
talks directly to the Kubernetes API (pods, events, Function CRs,
Prometheus, stern-style logs) and needs a kubeconfig with access to the
`openfaas` namespace and any function namespaces. Reach for it only when the
first-line `faas-cli` commands above are not enough:

```bash
faas-cli plugin get diag
faas-cli diag config simple
faas-cli diag -d 5m diag.yaml       # produces ./run/<date>/index.html + tar.gz
```

Common function-level symptoms and the first thing to check:

| Symptom | First check |
|---|---|
| Code change not visible after deploy | Image tag did not advance — redeploy with `faas-cli up --tag=digest`. See [Always advance the image tag on deploy](#always-advance-the-image-tag-on-deploy). |
| `0/1` pods, `ImagePullBackOff`, `CrashLoopBackOff` | `faas-cli logs <fn>`; if empty, break-glass `kubectl describe -n openfaas-fn deploy/<fn>` and `kubectl get events -n openfaas-fn`. Usual causes: missing secret, private registry without pull secret, handler crash on boot, OOMKilled. |
| Function CR stuck in `stalled` status | Fix the `.Spec` issue (missing secret, bad limits) and let the operator reconcile, or force a retry by bumping `com.openfaas.uid` annotation on the Function CR. |
| Function name rejected | CRD limits names to 63 chars — shorten and use a `namespace:`. |
| Timeouts | Check function logs first; then verify function-level `read_timeout`/`write_timeout`/`exec_timeout` in `stack.yaml`. Gateway timeouts and cloud LB idle timeouts may also need bumping — flag these to the user; do not patch them yourself. Use `http://gateway.openfaas:8080` for in-cluster calls. |
| Empty / nil request body in handler | Caller missing `Content-Type` header. For legacy/WSGI upstreams set `http_buffer_req_body: true`. |
| Slow start-up flapping readiness, or 500s on first request after scale-from-zero | Add a custom `/ready` endpoint in the handler and wire it via the `com.openfaas.ready.http.path` annotation; bump `com.openfaas.ready.http.initialDelaySeconds` for long warm-ups. See [Readiness checks for initialization](#readiness-checks-for-initialization). |
| Want to test without deploying | `faas-cli local-run --build <fn>` (preferred) or `faas-cli build` + `docker run -v $(pwd)/.secrets:/var/openfaas/secrets ...`. |
| JSON / structured logs are wrapped in a prefix | Set `prefix_logs: false` on the function. |

For step-by-step procedures, break-glass `kubectl` snippets, and links to the
upstream docs, read [reference/troubleshooting.md](reference/troubleshooting.md).

## Verification checklist

After scaffolding/editing:
1. `faas-cli local-run --build <fn>` starts (this also covers the build step); `curl http://127.0.0.1:8080` returns expected response. Only run `faas-cli build -f stack.yml` separately if you need the image without running it.
2. Handler returns proper status codes and `Content-Type` headers when returning JSON.
3. Secrets are read from `/var/openfaas/secrets/<name>`, not env vars. Any local `.secrets/` directory lives next to `stack.yaml`, never inside a handler folder (which would bake secrets into the image).
4. `image:` field includes a registry prefix (not bare `<fn>:latest`) before pushing.
5. When deploying to a cluster, the image tag has advanced since the previous deploy — either via `--tag=digest`/`--tag=sha` or by bumping the tag in `stack.yaml`. Never re-deploy with an unchanged tag.
6. If the handler does any pre-request initialization (model load, cache warm-up, DB pool, downloading data), a custom readiness endpoint is wired up via `com.openfaas.ready.http.path` and the handler returns 500 until init finishes. Verify by scaling to zero and invoking — the first request must succeed, not 500. See [Readiness checks for initialization](#readiness-checks-for-initialization).

## When to load deeper references

- For full handler examples per language → read [reference/handlers.md](reference/handlers.md).
- For per-language readiness handler examples (Python / Node / Go), the full `com.openfaas.ready.http.*` / `com.openfaas.health.http.*` annotation list, and combining `max_inflight` with `/_/ready` + `ready_path` → read [reference/health-readiness.md](reference/health-readiness.md).
- For the complete stack.yaml schema and advanced fields → read [reference/stack-yaml.md](reference/stack-yaml.md).
- For scheduling a function on a cron timer (annotation pattern, expression syntax, disable rules) → read [reference/cron-schedule.md](reference/cron-schedule.md).
- For the full list of official event triggers/connectors (Kafka, Postgres, SQS, SNS, Pub/Sub, RabbitMQ) and the generic `topic:` wiring pattern → read [reference/triggers.md](reference/triggers.md).
- For patterns using store functions (`printer`, `chaos`, `env`, etc.) during development, testing, and debugging → read [reference/store-helpers.md](reference/store-helpers.md).
- For step-by-step troubleshooting of function issues (didn't start, timeouts, stalled CR, empty body, slow start-up, etc.) → read [reference/troubleshooting.md](reference/troubleshooting.md).
- Official docs: https://docs.openfaas.com/languages/overview/, troubleshooting: https://docs.openfaas.com/deployment/troubleshooting/, and blog: https://www.openfaas.com/blog/.
