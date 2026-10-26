# Handler Reference

Complete handler examples for the supported OpenFaaS templates: Python, Node.js, Go, Java.

## Python (`python3-http` / `python3-http-debian`)

The handler receives `event` and `context`:

- `event.body` — request body (string for text, parsed dict for JSON content-type)
- `event.headers` — dict of HTTP headers
- `event.method` — HTTP method
- `event.query` — query parameters
- `event.path` — request path
- `context.hostname` — function hostname

Response is a dict:

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {"message": "Hello from OpenFaaS!"},
        "headers": {
            "Content-Type": "application/json"
        }
    }
```

### Reading a secret

```python
def read_secret(name):
    with open(f"/var/openfaas/secrets/{name}") as f:
        return f.read().strip()

def handle(event, context):
    token = read_secret("python-auth-token")
    if "Authorization" not in event.headers:
        return {"statusCode": 401, "body": "Unauthorized"}
    if event.headers["Authorization"] != f"Bearer {token}":
        return {"statusCode": 401, "body": "Unauthorized"}
    return {"statusCode": 200, "body": "Access granted"}
```

### Reading a static file bundled with the function

```python
import os, json

def handle(event, context):
    current_dir = os.path.dirname(os.path.abspath(__file__))
    with open(os.path.join(current_dir, "data.json")) as f:
        data = json.load(f)
    return {
        "statusCode": 200,
        "body": json.dumps(data),
        "headers": {"Content-Type": "application/json"}
    }
```

### Recommended libraries

These packages are commonly used in Python OpenFaaS functions. Add them to `<fn>/requirements.txt`. Pin versions for reproducible builds.

| Use case | Package | Notes |
|---|---|---|
| Outbound HTTP | `requests` | Battle-tested sync client; the safe default. |
| Outbound HTTP (high concurrency) | `aiohttp` | Use when fanning out many concurrent calls from one invocation. |
| JSON / schema validation | `pydantic` v2 | De-facto standard for typed request/response models. |
| AWS (S3, SQS, etc.) | `boto3` | Use IRSA on EKS instead of static keys. `aioboto3` for async. |
| OpenAI / LLMs | `openai`, `anthropic` | Both are the official SDKs. Read API keys via `/var/openfaas/secrets/`. |
| GitHub API | `PyGithub` | Used in the official "github-stars" tutorial. |
| PostgreSQL | `psycopg[binary]` (psycopg 3) | Modern replacement for `psycopg2` — adds native async + connection pooling. Use `python3-http-debian` + `build_options: [libpq]` only if building from source. For async-only workloads, `asyncpg` is a faster lower-level alternative. |
| MySQL | `PyMySQL` (pure-Python) or `mysqlclient` (C extension, faster) | `PyMySQL` works on Alpine without build deps. |
| MongoDB | `pymongo` | |
| Redis | `redis` (a.k.a. `redis-py`) | Includes async support since v4 — no need for `aioredis`. |
| Kafka | `confluent-kafka` (sync) or `aiokafka` (async) | **Do not use `kafka-python`** — unmaintained since 2020. `confluent-kafka` needs the Debian variant. |
| Data processing | `pandas`, `numpy`, `pyarrow`, `polars` | `polars` is increasingly preferred for new code. Use `python3-http-debian`. |
| Image manipulation | `Pillow` | |
| Web scraping | `beautifulsoup4`, `lxml`, `selectolax` | |
| Browser automation | `playwright` | Requires the Debian template + extra system packages; image is large. |
| Templating | `Jinja2` | |
| Observability | `opentelemetry-distro`, `opentelemetry-exporter-otlp` | Zero-code instrumentation via `OTEL_*` env vars. |
| Unit testing | `pytest` | Add to `requirements.txt` and run locally; the build does not run pytest automatically. |

### Native deps (Postgres example)

`stack.yml`:

```yaml
functions:
  pgfn:
    lang: python3-http-debian
    handler: ./pgfn
    image: pgfn:latest
    build_options:
      - libpq
    secrets:
      - pg-conn
```

`requirements.txt`:

```
psycopg2==2.9.3
```

## Node.js (`node26`)

```js
'use strict'

module.exports = async (event, context) => {
  const result = {
    body: JSON.stringify(event.body),
    'content-type': event.headers['content-type']
  }
  return context.status(200).succeed(result)
}
```

### Recommended libraries

Common npm packages for Node.js OpenFaaS functions. Install with `cd <fn> && npm install --save <pkg>`.

| Use case | Package | Notes |
|---|---|---|
| Outbound HTTP | Built-in `fetch` | Native on Node 18+ (the `node26` template uses Node 26). No dependency to install. Reach for `undici` if you need a connection pool with more knobs, or `axios` for interceptors / progress events — but be aware that `axios` has had multiple HIGH-severity CVEs (CVE-2023-45857, CVE-2024-39338, CVE-2025-27152, CVE-2025-58754); pin and patch promptly. |
| JSON / schema validation | `zod` | De-facto standard for TypeScript-first schema validation. Use `ajv` for raw JSON Schema. |
| AWS SDK | `@aws-sdk/client-s3`, `@aws-sdk/client-sqs`, etc. | Use modular v3 clients to keep image size small. v2 (`aws-sdk`) is in maintenance mode — avoid for new code. |
| OpenAI / LLMs | `openai`, `@anthropic-ai/sdk` | Both are the official SDKs. Read API keys via `/var/openfaas/secrets/`. |
| PostgreSQL | `pg` (node-postgres) | The de-facto standard (~9M weekly downloads); used under the hood by Prisma, Drizzle, Knex. Use a singleton `Pool` outside the handler. `postgres` (postgres.js) is a faster modern alternative for direct (non-ORM) use. |
| MySQL | `mysql2` | Successor to `mysql`; supports prepared statements and promises. |
| MongoDB | `mongodb` | Official driver. |
| Redis | `redis` (node-redis, official) | Default for new projects. Use `ioredis` if you need BullMQ, advanced Cluster/Sentinel ergonomics, or are migrating an existing `ioredis` codebase. |
| Kafka | `@confluentinc/kafka-javascript` or `node-rdkafka` | **Do not use `kafkajs`** — Confluent's own deprecation matrix lists it as not maintained. The Confluent JS client wraps `librdkafka` and stays in step with broker releases. |
| PDF generation / browser automation | `puppeteer`, `playwright` | Use a custom Dockerfile template — Chromium is large. See the OpenFaaS PDF generation blog post. |
| Templating | `handlebars`, `ejs` | |
| Date / time | `date-fns` | Tree-shakable, immutable. Avoid `moment` (legacy, deprecated by its own maintainers). `luxon` is fine if you prefer an OO API. |
| Logging | `pino` | Fast JSON logger. Set `prefix_logs: false` for clean JSON output. |
| Observability | `@opentelemetry/api`, `@opentelemetry/auto-instrumentations-node` | Zero-code tracing — see the official Node template docs. |
| Unit testing | Built-in `node:test` + `node:assert` | Stable in Node 20+; no dependency needed. `vitest` is a good third-party choice if you want watch mode and richer matchers. Tests in `<fn>/test/` run automatically during `faas-cli build`. |

### With an npm dependency

```bash
cd http-req
npm install --save axios
```

```js
'use strict'
const axios = require('axios')

module.exports = async (event, context) => {
  if (!event.body.url) {
    return context.status(400).fail('Missing .url in request body')
  }
  try {
    const res = await axios({ method: 'GET', url: event.body.url, validateStatus: () => true })
    return context.status(res.status).succeed(res.data)
  } catch (e) {
    return context.status(500).fail(e)
  }
}
```

### Reading a secret

```js
'use strict'
const fs = require('fs').promises

const tokenSecretName = 'node-fn-token'

module.exports = async (event, context) => {
  const token = (await fs.readFile(`/var/openfaas/secrets/${tokenSecretName}`, 'utf8')).trim()
  if (!event.headers.authorization) return context.status(401).fail('Unauthorized')
  if (event.headers.authorization !== `Bearer ${token}`) return context.status(403).fail('Forbidden')
  return context.status(200).succeed('Authenticated')
}
```

### Unit tests

`package.json`:

```json
{
  "scripts": { "test": "mocha test/test.js" },
  "devDependencies": { "chai": "^4.2.0", "mocha": "^7.0.1" }
}
```

Tests under `<fn>/test/` are run automatically during `faas-cli build`. Failed tests fail the build.

## Go (`golang-middleware`)

```go
package function

import (
    "fmt"
    "io"
    "net/http"
)

func Handle(w http.ResponseWriter, r *http.Request) {
    var input []byte
    if r.Body != nil {
        defer r.Body.Close()
        body, _ := io.ReadAll(r.Body)
        input = body
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(fmt.Sprintf("Body: %s", string(input))))
}
```

### Recommended libraries

Common Go modules for OpenFaaS functions. Add with `cd <fn> && go get <module>`.

| Use case | Module | Notes |
|---|---|---|
| Outbound HTTP | `net/http` (stdlib) | Reuse a package-level `*http.Client` rather than creating one per request. |
| JSON | `encoding/json` (stdlib) | |
| Routing inside the handler | `net/http` (stdlib) | The template wires you straight into `http.Handler`. Go 1.22+ stdlib supports method + path patterns (`mux.HandleFunc("GET /items/{id}", ...)`); reach for `chi` only if you need middleware composition or sub-routers. |
| AWS SDK | `github.com/aws/aws-sdk-go-v2/...` | Modular v2 clients keep binaries small. v1 (`aws-sdk-go`) is end-of-life — avoid for new code. |
| OpenFaaS gateway / function management | `github.com/openfaas/go-sdk` | Includes the Function Builder API client. |
| OpenAI / LLMs | `github.com/openai/openai-go` (official), `github.com/anthropics/anthropic-sdk-go` (official) | The official OpenAI Go SDK reached v3 in 2026 — prefer it over the community `github.com/sashabaranov/go-openai`. Read API keys via `/var/openfaas/secrets/`. |
| PostgreSQL | `github.com/jackc/pgx/v5` | Pure-Go driver; no native deps. Use `pgxpool` for connection pooling. |
| MySQL | `github.com/go-sql-driver/mysql` | |
| MongoDB | `go.mongodb.org/mongo-driver/v2/mongo` | v2 was released January 2025 and is the current line — v1 is in maintenance only. |
| Redis | `github.com/redis/go-redis/v9` | Now the official client (the repo moved under `redis/` org). |
| Kafka | `github.com/twmb/franz-go` | Pure-Go, feature-complete, actively maintained, and the de-facto community recommendation. Avoid `github.com/segmentio/kafka-go` — Confluent's deprecation matrix flags it as not compliant for consumer apps and the project has had long stretches without releases. `github.com/confluentinc/confluent-kafka-go` is the official client but requires CGO and a `librdkafka` system package. |
| Crypto helpers | `golang.org/x/crypto/...` | E.g. `bcrypt` — used in the official template tutorial. |
| Shell / subprocess | `github.com/alexellis/go-execute/v2` | Used in the OpenFaaS sub-module example. |
| Structured logging | `log/slog` (stdlib) | The standard since Go 1.21; replaces `zap` / `logrus` for most needs. Set `prefix_logs: false` for clean JSON output. |
| Observability | `go.opentelemetry.io/otel`, `go.opentelemetry.io/otel/exporters/otlp/...` | Wrap the `http.Handler` with `otelhttp.NewHandler`. |
| Unit testing | `testing` (stdlib), `github.com/stretchr/testify` | `_test.go` files in the handler folder run during `faas-cli build`. |

### Adding dependencies

```bash
cd bcrypt
go get golang.org/x/crypto/bcrypt
```

`go.mod` is updated automatically.

### Sub-packages

If you create `handlers/handlers.go` with its own `go.mod`, add to the function's `go.mod`:

```
replace handler/function => ./
```

Or run: `go mod edit -replace handler/function=./`.

### Private modules

```bash
cd secret-fn
GOPRIVATE=github.com/acmecorp go get
GOPRIVATE=github.com/acmecorp go mod vendor
```

Commit the `vendor/` directory.

## Java (`java21-vertx`)

Use the [Java documentation](https://docs.openfaas.com/languages/java/) and
[official template](https://github.com/openfaas/templates/tree/master/template/java21-vertx)
when extending a Java function.

Edit `<fn>/src/main/java/com/openfaas/function/Handler.java`:

```java
package com.openfaas.function;

import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.web.RoutingContext;

public class Handler implements io.vertx.core.Handler<RoutingContext> {
    public Handler(Vertx vertx) {
        // Initialise reusable asynchronous clients here with the supplied Vertx.
    }

    @Override
    public void handle(RoutingContext context) {
        String name = context.queryParam("name").stream()
            .findFirst().orElse("World");
        JsonObject result = new JsonObject()
            .put("greeting", "Hello, " + name + "!")
            .put("method", context.request().method().name())
            .put("path", context.request().path())
            .put("body", context.body().asString());

        context.response()
            .setStatusCode(200)
            .putHeader("Content-Type", "application/json; charset=utf-8")
            .end(result.encode());
    }
}
```

The handler instance is shared across invocations. Keep request state local.
Prefer Vert.x `WebClient` and SQL clients; handle asynchronous failures and
check upstream HTTP status codes as well as transport errors.

For a configurable absolute upstream URL, use `client.getAbs(url)` (or
`requestAbs(method, url)`) to preserve the HTTP/HTTPS scheme, port, path, and
query. If constructing a request from separate URI components, set
`.ssl(true)` for HTTPS; selecting port 443 alone does not enable TLS.
When supporting both schemes, test HTTP and HTTPS upstreams with certificate
verification enabled.

### Dependencies and tests

Add entries to the existing `dependencies` block in `<fn>/build.gradle`:

```groovy
implementation 'io.vertx:vertx-web-client'
implementation 'io.vertx:vertx-pg-client'
```

The template supplies Maven Central and the Vert.x platform versions. For
other libraries use `implementation 'group:artifact:version'`; for a local
JAR in `<fn>/libs/`, use `implementation files('libs/helper.jar')`.

Put additional classes under `<fn>/src/main/java/` using package directories.
JUnit Jupiter is configured for tests in `<fn>/src/test/java/`, which run
during `faas-cli build` (also via `local-run --build`).

For static content, use `<fn>/src/main/resources/webroot/` with Vert.x
`StaticHandler`. For database pools, mounted secrets, and authentication
middleware, consult the linked Java documentation.

