# Microservices and Existing Images (no template)

How to run a service you already have — FastAPI, Express, Sinatra, ASP.NET —
on OpenFaaS, without a language template. Two routes: implement the workload
definition yourself, or wrap the process in of-watchdog.

Upstream: <https://docs.openfaas.com/reference/workloads/>

## The workload definition

Any container OpenFaaS runs must:

* serve HTTP traffic on TCP port **8080**
* implement **`/_/health`** (liveness) and **`/_/ready`** (readiness)
* be stateless and assume ephemeral storage (`/tmp` only when read-only)
* have a name of 63 characters or fewer

It should also shut down gracefully on `SIGTERM`. That is not listed in the
workload definition, but the watchdog provides it and Kubernetes relies on it
to drain cleanly.

Templates satisfy all of this via the watchdog. Without one, the app owns it.

## Route A — no watchdog

Use `lang: dockerfile` and let the app serve 8080 directly. Nothing
OpenFaaS-specific goes in the image.

```yaml
functions:
  my-service:
    lang: dockerfile
    handler: ./my-service
    image: ${REGISTRY:-ghcr.io/acme}/my-service:${TAG:-0.1.0}
```

The app must answer `/_/health` and `/_/ready` itself:

```python
@app.get("/_/health")           # process is up; restart it if this fails
async def health():
    return {"status": "alive"}

@app.get("/_/ready")            # up, but not yet able to take traffic
async def ready():
    return JSONResponse({"ready": _ready}, status_code=200 if _ready else 503)
```

**Most watchdog environment variables do nothing on this route.**
`read_timeout`, `exec_timeout`, `max_inflight`, `prefix_logs` and
`upstream_url` are all read by the watchdog process, which is not in the
image. Setting them in `stack.yaml` is silently inert — timeouts and
concurrency limiting become the app server's job (uvicorn, Express, etc.).
`max_inflight`-based readiness (`/_/ready` combining) is likewise unavailable.

`write_timeout` is the exception. On OpenFaaS Pro the operator reads it off
the function to set the Pod's `terminationGracePeriodSeconds`, so it still
changes drain behaviour on scale-down and scale-to-zero even with no watchdog
in the image. See <https://docs.openfaas.com/reference/workloads/>.

Gateway-level timeouts still apply, so a slow function can still be cut off
upstream even though the function-level values are ignored.

## Route B — wrap the process in of-watchdog

Recommended for stateless microservices: you keep your framework and image,
and get the watchdog's consistent timeouts, request logging, graceful shutdown
and concurrency limiting back. The watchdog owns 8080 and `/_/health` +
`/_/ready`, and proxies every other path to your process on loopback.

```dockerfile
FROM ghcr.io/openfaas/of-watchdog:0.11.7 AS watchdog
FROM python:3.13-slim-bookworm

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

# ... install the app as normal, listening on 127.0.0.1:5000 ...

ENV fprocess="python main.py"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:5000"

CMD ["fwatchdog"]
```

Bind the app to `127.0.0.1`, not `0.0.0.0` — the watchdog is its only client.

Requests to `/_/health` and `/_/ready` are answered by the watchdog and never
reach the app, so do not implement them upstream. A *custom* readiness check
still works: expose your own path (conventionally `/ready`) and point
`com.openfaas.ready.http.path` at it, or set `ready_path` and probe `/_/ready`
to combine it with `max_inflight`. See
[health-readiness.md](health-readiness.md).

## Deploying a pre-built image you cannot rebuild

If the image already serves 8080 and has a health endpoint, skip the build
entirely:

```yaml
functions:
  kubesec:
    image: docker.io/stefanprodan/kubesec:v2.1
    skip_build: true
    annotations:
      com.openfaas.ready.http.path: /_/ready
      com.openfaas.ready.http.initialDelaySeconds: "30"
```

Note `skip_build: true` still needs a `lang:` for `faas-cli build`/`up` to
skip it cleanly — use `lang: dockerfile` alongside it, or deploy with
`faas-cli deploy` rather than `up`. The `com.openfaas.ready.http.*`
annotations are an OpenFaaS Pro feature and are ignored on CE.

If it listens on some other port and you cannot change the code, use Route B —
set `upstream_url` to that port and add the watchdog via a two-line Dockerfile
`FROM` the existing image.

## Choosing between them

| | No watchdog | of-watchdog |
|---|---|---|
| Image contents | just your app | app + `/usr/bin/fwatchdog` |
| `/_/health`, `/_/ready` | you implement | provided |
| `read_timeout` / `exec_timeout` | ignored | honoured |
| `write_timeout` | ignored by the app, but still sets `terminationGracePeriodSeconds` on Pro | honoured, and sets the grace period |
| `max_inflight` concurrency cap | unavailable | available |
| Graceful shutdown | your server's job | provided |
| Request logging format | your server's | consistent with all other functions |

Prefer of-watchdog unless you specifically want a dependency-free image.
