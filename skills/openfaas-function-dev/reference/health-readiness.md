# Readiness (and Health) Checks for Functions

Function-side reference for custom readiness/liveness checks. The handler
contract, annotations, and per-language patterns — nothing about gateway
or Helm configuration.

Upstream: <https://docs.openfaas.com/reference/workloads/> ·
<https://www.openfaas.com/blog/health-and-readiness-for-functions/> ·
examples <https://github.com/alexellis/ready-fns>.

## When to add one

Add a custom **readiness** endpoint whenever the handler does work before
it can serve requests: loading an ML model, warming a cache, downloading
data, opening DB / SDK clients. Without one, the first request after a
fresh Pod (cold start, scale-from-zero, rolling update) lands on a Pod
that is up but not yet able to handle work.

Liveness (`com.openfaas.health.http.*`) is separate and rarely needed —
the watchdog already serves a built-in liveness probe. Use a custom
liveness only when the process should actually be killed and restarted.

## Annotations

Set on the function in `stack.yaml`:

| Annotation | Purpose |
|---|---|
| `com.openfaas.ready.http.path` | Path the probe calls (e.g. `/ready`). Use `/_/ready` to invoke the watchdog's combined check (see "max_inflight" below). |
| `com.openfaas.ready.http.initialDelaySeconds` | Delay after Pod start before the first probe. Raise for slow starters. |
| `com.openfaas.ready.http.periodSeconds` | Probe interval. |
| `com.openfaas.ready.http.timeoutSeconds` | Probe HTTP timeout. |
| `com.openfaas.ready.http.successThreshold` | Consecutive successes before Ready. |
| `com.openfaas.ready.http.failureThreshold` | Consecutive failures before NotReady. |

The matching `com.openfaas.health.http.*` set exists for custom liveness
(same fields, no `successThreshold`).

## Handler contract

1. Kick init off the request path at module load (Python / Node) or
   `init()` (Go), and track readiness in a variable.
2. Detect the readiness path in the handler. Return `200` once init has
   finished, `500` (or any non-2xx) while it is still running.
3. Fall through to the normal handler for every other path.

Use the same path string in the handler and the annotation. `/ready` is
conventional.

### Python (`python3-http`)

```python
import time, threading

_ready = False

def _initialize():
    global _ready
    # do real init here
    time.sleep(5)
    _ready = True

threading.Thread(target=_initialize, daemon=True).start()

def handle(event, context):
    if event.path == "/ready":
        return {"statusCode": 200 if _ready else 500,
                "body": "Ready" if _ready else "Not Ready"}
    return {"statusCode": 200, "body": "Hello from OpenFaaS!"}
```

Blocking init at module load (before the watchdog starts serving) also
works for short warm-ups — the Pod just stays NotReady until init
returns.

### Node.js (`node24`)

```js
'use strict'

const state = { ready: false }

;(async () => {
  // do real init here
  await new Promise(r => setTimeout(r, 5000))
  state.ready = true
})()

module.exports = async (event, context) => {
  if (event.path === '/ready') {
    return state.ready
      ? context.status(200).succeed({ status: 'ready' })
      : context.status(500).succeed({ error: 'not ready' })
  }
  return context.status(200).succeed({ message: 'Hello from OpenFaaS!' })
}
```

### Go (`golang-middleware`)

Use `sync/atomic` so the flag is safe to read from the request handler
while the init goroutine writes it.

```go
package function

import (
    "io"
    "net/http"
    "sync/atomic"
    "time"
)

var isReady atomic.Value

func init() {
    isReady.Store(false)
    go func() {
        // do real init here
        time.Sleep(5 * time.Second)
        isReady.Store(true)
    }()
}

func Handle(w http.ResponseWriter, r *http.Request) {
    if r.Body != nil {
        defer r.Body.Close()
    }
    if r.URL.Path == "/ready" {
        if !isReady.Load().(bool) {
            w.WriteHeader(http.StatusInternalServerError)
            w.Write([]byte("Not Ready"))
            return
        }
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Ready"))
        return
    }
    body, _ := io.ReadAll(r.Body)
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("Body: " + string(body)))
}
```

## Long warm-ups

For reliably slow but stable init (large ML models, big caches), raise
`initialDelaySeconds` so the first probe fires after init is expected to
complete — don't probe on a tight loop while the Pod is still loading:

```yaml
annotations:
  com.openfaas.ready.http.path: /ready
  com.openfaas.ready.http.initialDelaySeconds: 60   # don't probe for 60s
  com.openfaas.ready.http.periodSeconds: 5
```

## Combining custom readiness with `max_inflight`

The of-watchdog can enforce a hard per-Pod concurrency limit via the
`max_inflight` environment variable. To make Kubernetes route around
overloaded Pods *and* honour your own readiness check, route the probe
through the watchdog's combined endpoint:

```yaml
functions:
  my-fn:
    environment:
      max_inflight: 2          # watchdog concurrency cap
      ready_path: /custom-ready # your readiness path on the function
    annotations:
      com.openfaas.ready.http.path: /_/ready  # call the watchdog
      com.openfaas.ready.http.initialDelaySeconds: 2
      com.openfaas.ready.http.periodSeconds: 2
```

The watchdog returns "ready" only when both are true:

- `max_inflight` is not saturated, **and**
- the function's `ready_path` returns 2xx (skipped if `ready_path` is
  unset).

If `max_inflight` is exceeded, the watchdog returns NotReady regardless
of your custom check, so the Pod is taken out of rotation until inflight
drops.
