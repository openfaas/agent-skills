# Community Edition

Use this workflow only after confirming that Community Edition fits the user's licensing and usage. CE is intended for personal use; commercial evaluation is time-limited. See [Deploy OpenFaaS CE to Kubernetes](https://docs.openfaas.com/deployment/kubernetes/).

## Install

CE uses controller mode and does not require a Pro license. Use a minimal override file only when the user requests non-default values:

```bash
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas
```

If overrides are required, write only those overrides and add `-f <values-file>`. Do not copy the complete upstream `values.yaml`; image versions and defaults are maintained by the chart.

Authenticate with Basic Auth as described in [pro-cli.md](pro-cli.md). The base `faas-cli` is used for CE; do not install the Pro plugin solely for CE.

## Constraints

- CE uses the legacy controller mode. The chart may install `Function` and `Profile` CRD objects for compatibility, but the OpenFaaS operator and supported CRD-based function management require Standard or For Enterprises.
- CE is not the appropriate default for an unspecified commercial or production workload.
- For local development, the official documentation may recommend arkade, but use Helm when following this skill.

References:

- [CE Kubernetes deployment](https://docs.openfaas.com/deployment/kubernetes/)
- [faas-netes and Helm chart](https://github.com/openfaas/faas-netes)
- [Chart values](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values.yaml)
