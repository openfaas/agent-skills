# Triggers and event-connectors

Functions are most often invoked over HTTP, but OpenFaaS ships a family of **event-connectors** that map topics, queues, or subjects from external systems to functions via the standard `topic:` annotation — no SDKs or subscription code in the handler.

Prefer a connector over building polling or subscription logic into a function: connectors handle reconnects, scaling, and back-pressure, and the function stays a plain HTTP handler.

## Available triggers

| Trigger | Edition | Notes |
|---|---|---|
| Cron | CE + Pro | [cron-connector](https://github.com/openfaas/cron-connector). See [reference/cron-schedule.md](cron-schedule.md). |
| Apache Kafka | Pro | Subscribe to Kafka topics — https://docs.openfaas.com/openfaas-pro/kafka-events/ |
| Postgres | Pro | Trigger on insert/update/delete via Postgres WAL. |
| AWS SQS | Pro | Consume messages from SQS queues. |
| AWS SNS | Pro | Receive SNS notifications over HTTPS (needs a public gateway endpoint). |
| Google Cloud Pub/Sub | Pro | Subscribe to Pub/Sub topics. |
| RabbitMQ | Pro | Consume from RabbitMQ queues. |

## Wiring pattern

Same shape for every connector — only the `topic` value differs:

```yaml
functions:
  payments-received:
    lang: node26
    handler: ./payments-received
    image: ${REGISTRY:-ttl.sh/myuser}/payments-received:${TAG:-latest}
    annotations:
      topic: payments.received   # Kafka topic, SQS queue, SNS topic, etc.
```

## Reference

- https://docs.openfaas.com/reference/triggers/
