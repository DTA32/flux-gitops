# infrastructure

What every environment sits on: the application namespaces, the Gateway and TLS, and the
woodpecker route. Reconciled by the `infrastructure` Kustomization, which everything else
`dependsOn`, so these exist before any app tries to use them.

| Path | What |
|---|---|
| `namespaces.yaml` | the `prod` and `dev` namespaces |
| `envoy-gateway/namespace.yaml` | `envoy` — the Gateway, every HTTPRoute, and the TLS secrets |
| `envoy-gateway/gatewayclass.yaml` | GatewayClass for the Envoy Gateway controller |
| `envoy-gateway/gateway.yaml` | `main-gateway`: `:80` plus one `:443` listener per hostname |
| `envoy-gateway/cluster-issuer.yaml` | `letsencrypt-prod`, solving HTTP-01 through the Gateway |
| `envoy-gateway/http-redirect.yaml` | HTTP → HTTPS 301 on `:80` |
| `envoy-gateway/reference-grant.yaml` | lets HTTPRoutes in `envoy` reach Services in `prod` and `dev` |
| `woodpecker/namespace.yaml` | `woodpecker`, with Pod Security `privileged` for dind build pods |
| `woodpecker/envoy-httproute.yaml` | `ci.dta32.my.id` and the ReferenceGrant it needs |

## Requires

- **Envoy Gateway 1.9.0 and cert-manager v1.21.1**, installed with Helm, cert-manager with
  `config.enableGatewayAPI=true`. Commands are in the [root README](../README.md#2-cluster-add-ons).
  This directory only holds the resources those controllers consume, so Flux cannot bring them up
  by itself.
- **DNS for every listener hostname.** cert-manager's HTTP-01 challenge fails until the name
  resolves to the cluster.

## How a request gets in

    client ─> :443 listener for its hostname      (TLS terminated here, cert from cert-manager)
           ─> HTTPRoute in envoy                    (path match, optional rewrite / basic auth)
           ─> Service in prod or dev                (allowed by that namespace's ReferenceGrant)

`:80` only redirects to HTTPS and answers ACME challenges.

## Adding a hostname

1. A listener in `envoy-gateway/gateway.yaml`: a unique `name`, the `hostname`, and a
   `certificateRefs` Secret name such as `<name>-tls`. cert-manager creates that Secret.
2. The app's `envoy-httproute.yaml`, with `parentRefs[].sectionName` set to the listener name.
3. The DNS record.

## Notes

- `wait: true` on this Kustomization makes `data-*` and `apps-*` wait until the Gateway is
  actually programmed.
- Woodpecker itself is a Helm release outside Flux; only its namespace and route live here.
