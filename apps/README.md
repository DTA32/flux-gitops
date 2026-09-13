# apps

| Path           | Namespace | Kustomization      |
|----------------|-----------|--------------------|
| `production/`  | `prod`    | `apps-production`  |
| `development/` | `dev`     | `apps-development` |

Which of these images update automatically is listed in [images/README.md](../images/README.md).

## Each environment is ONE kustomize build

`production/` and `development/` each list `configmap` plus their app directories as `resources`,
and each is the path of its own Flux Kustomization.

That is required, not tidy. ConfigMaps are generated with a hash of their content in the name, and
kustomize rewrites `configMapKeyRef.name` / `configMapRef.name` to the hashed name only within a
**single** build. Split `configmap/` into its own Kustomization and every workload would keep
referencing a name that no longer exists — silently, until a pod restarts.

The hash is also what makes config changes roll out: edit an `.env`, the ConfigMap gets a new name,
the pod template changes, the workload restarts. Old generated ConfigMaps are pruned automatically.

The cost: a syntax error in one app fails its whole environment. Run
`kubectl kustomize apps/production` before pushing.

## Adding an app

1. `production/<app>/` with `deployment.yaml`, `service.yaml`, and a
   `kustomization.yaml` listing them.
2. `metadata.namespace: prod` on every workload resource — see below.
3. Config, if any: `production/configmap/<app>.env` plus an entry in
   `production/configmap/kustomization.yaml`.
4. Add `<app>` to `production/kustomization.yaml`.
5. If you want to expose publicly: A listener for its hostname in `infrastructure/envoy-gateway/gateway.yaml`, 
   `envoy-httproute.yaml` and include that in `kustomization.yaml`, and DNS.
6. To auto-update it: `images/production/<app>.yaml`, listed in that directory's
   `kustomization.yaml`, and a `# {"$imagepolicy": "flux-system:<app>"}` marker on the image line.
7. Its Secret, created by hand — never committed.

## Why every manifest names its namespace

Don't replace explicit namespaces with kustomize's `namespace:` field or Flux's `targetNamespace`.
Both would also stamp `prod` onto the `envoy-httproute.yaml` files, which must stay in `envoy`. And
a resource with no namespace doesn't fall back to `default`: Flux applies it into its own
Kustomization's namespace, `flux-system`.

The routes' `backendRefs` reach from `envoy` into `prod`/`dev`, which Gateway API only allows
because of the ReferenceGrants in `infrastructure/envoy-gateway/reference-grant.yaml`.
