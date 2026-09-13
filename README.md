# flux-gitops — dta32-kube

Desired state for the dta32-kube k3s cluster, reconciled by [Flux](https://fluxcd.io).

**The flow**

    push to master (or dev)
      -> Woodpecker builds, pushes <sha> and <unix-ts>-<sha>
      -> ImageRepository lists tags (every 10m)
      -> ImagePolicy elects the highest <unix-ts>
      -> ImageUpdateAutomation commits the new tag into this repo (every 5m)
      -> Kustomization applies it

## Layout

| Path                                    | What                                                                | Flux Kustomization                    |
|-----------------------------------------|---------------------------------------------------------------------|---------------------------------------|
| `clusters/dta32-kube/`                  | Flux's entrypoint: Kustomizations, image automation, Discord alerts | `flux-system`                         |
| `infrastructure/`                       | `prod`/`dev` namespaces, Gateway API resources, woodpecker route    | `infrastructure`                      |
| `data/production/`, `data/development/` | postgres, one per environment                                       | `data-production`, `data-development` |
| `apps/production/`, `apps/development/` | workloads and their generated ConfigMaps                            | `apps-production`, `apps-development` |
| `images/`                               | `ImageRepository` + `ImagePolicy` for each automated image          | `images`                              |

    infrastructure ─┬─> data-production  ─> apps-production
                    └─> data-development ─> apps-development
    images            (independent; only reads registries)

Each directory's `README.md` has the detail.

## 1. Cluster prerequisites

- **Kubernetes with a LoadBalancer and a `local-path` StorageClass.** k3s ships both: ServiceLB
  exposes the Gateway on ports 80/443 of the nodes, and dev postgres provisions on `local-path`.
- **Node labels.** One always-on node must carry `node_type=high-availability` — the Flux
  controllers, prod postgres and the gotth-finance backup are pinned to it. `node_type=spot` is
  optional; job-scraper-mcp only prefers it.

      kubectl label node <always-on-node> node_type=high-availability
      kubectl label node <other-node> node_type=spot

- **Host directories** on the `high-availability` node, for the two hand-made hostPath volumes:

      sudo mkdir -p /mnt/data/postgres-postgis /mnt/data/gotth-finance-backups

- **Locally:** `kubectl`, `helm`, `flux` ≥ 2.9, `gh`, `jq`, `htpasswd`, and a GitHub PAT with
  `repo` scope.

## 2. Cluster add-ons

Flux does not install these; `infrastructure/` only holds the resources they consume.

    helm install eg oci://docker.io/envoyproxy/gateway-helm --version 1.9.0 \
      -n envoy-gateway-system --create-namespace

    helm repo add jetstack https://charts.jetstack.io && helm repo update
    helm install cert-manager jetstack/cert-manager --version v1.21.1 \
      -n cert-manager --create-namespace \
      --set crds.enabled=true --set config.enableGatewayAPI=true

`config.enableGatewayAPI=true` is not optional: the Let's Encrypt issuer solves HTTP-01 through
the Gateway. Woodpecker is also a Helm release, but it needs a namespace and a database from this
repo first — see [step 7](#7-woodpecker).

## 3. DNS

Point every Gateway hostname at the cluster before enabling `infrastructure`. cert-manager cannot
issue a certificate until the name resolves here. The authoritative list is the listeners in
`infrastructure/envoy-gateway/gateway.yaml`:

    bdgcafe.com  dev.bdgcafe.com  image.bdgcafe.com
    api.splitbill.dta32.my.id  api.dfgs.dta32.my.id  api.tiketin.dta32.my.id
    vern.dta32.my.id  kawan-ngonser.dta32.my.id  ci.dta32.my.id
    api.skrispi.mraditya.my.id  sinanaas.my.id  gotth-finance.sinanaas.my.id

## 4. Bootstrap Flux

First time only, create the repository:

    git init -b master && git add -A && git commit -m "flux-gitops: initial import"
    gh repo create DTA32/flux-gitops --public --source=. --push

Then, against the target cluster:

    flux check --pre
    export GITHUB_TOKEN=<PAT with repo scope>
    flux bootstrap github \
      --owner=DTA32 --repository=flux-gitops --branch=master \
      --path=clusters/dta32-kube --personal --private=false \
      --components-extra=image-reflector-controller,image-automation-controller \
      --read-write-key

- `--components-extra` adds the two image controllers, which a default install leaves out.
- `--read-write-key` lets image automation push commits. Without it everything looks healthy
  until the first tag bump fails on push.
- The PAT is only used here. The cluster keeps an SSH deploy key
  (`kubectl get secret flux-system -n flux-system`), never the token.
- Bootstrap commits `clusters/dta32-kube/flux-system/` to `master`, and image automation keeps
  committing there, so `git pull` before editing locally.

**Moving from another cluster?** Stop the old one first, or both will reconcile and both will
push image bumps:

    # against the OLD cluster
    flux suspend kustomization flux-system
    flux suspend image update flux-system

Bootstrapping the new cluster registers a second deploy key; delete the old one under the
repository's Settings → Deploy keys once the new cluster is healthy.

## 5. Bring it up

Everything except `images` is committed with `suspend: true` — the Kustomizations in
`clusters/dta32-kube/` and the `ImageUpdateAutomation`. **Enable them with a commit, not
`flux resume`.** These objects are defined in Git, so Flux re-applies `suspend: true` within a
reconcile and silently undoes a resume.

For each step, set `suspend: false` in the named file (`data.yaml` and `apps.yaml` hold two
objects each — edit the right one), then:

    git commit -am "flux: enable <name>" && git push
    flux reconcile source git flux-system
    flux get kustomizations

1. **Flux-level secrets** (the namespace exists after bootstrap): `flux-registry-creds` and,
   optionally, `discord-webhook`. See [Secrets reference](#secrets-reference).
2. **`infrastructure`** (`infrastructure.yaml`). Creates `envoy`, `prod`, `dev` and `woodpecker`,
   and the Gateway. Then create every secret for `envoy` and `prod`.
3. **`data-production`** (`data.yaml`). Then [restore the databases](#6-restore-databases).
4. **Woodpecker**, if you want CI on this cluster — [step 7](#7-woodpecker).
5. **`apps-production`** (`apps.yaml`).
6. **Image automation** (`image-automation.yaml`), once production is healthy.
7. **Development**, when you need it — [step 8](#8-development-environment).

## 6. Restore databases

Prod postgres starts empty. `postgres-backup-r2` dumps every database plus a `globals` file
(roles) to R2 nightly, and the restore job from
[homeserver/k8s/postgres-backup-r2](https://github.com/DTA32/homeserver/tree/main/k8s/postgres-backup-r2)
replays them. As published it targets the old layout, so change four fields in
`restore-job.yaml` first:

| Field                | Published                                            | Set to                                                                                  |
|----------------------|------------------------------------------------------|-----------------------------------------------------------------------------------------|
| `metadata.namespace` | `default`                                            | `prod`                                                                                  |
| `POSTGRES_HOST`      | `postgres-postgis-service.default.svc.cluster.local` | `postgres-postgis-service.prod.svc.cluster.local`                                       |
| container `image`    | `dta32/postgresql-backup-r2:latest`                  | the tag in `apps/production/postgres-backup-r2/cronjob.yaml` — `:latest` does not exist |
| `R2_PREFIX`          | `postgres-postgis`                                   | `postgres`, matching the CronJob, or it reads the wrong folder                          |

It reads `postgres-postgis-secret` and `postgres-backup-r2-secret` in `prod`. Restore `globals`
first so the roles exist, then each database:

    # POSTGRES_DATABASE=globals, BACKUP_FILE=latest
    kubectl apply -f restore-job.yaml
    kubectl logs -n prod job/postgres-restore -f
    kubectl delete job postgres-restore -n prod

    # then once per database: POSTGRES_DATABASE=<db>, BACKUP_FILE=latest, CREATE_DATABASE=yes

The `delete` between runs is required — a Job's pod template is immutable. Leaving `BACKUP_FILE`
empty lists the newest backups for that database instead of restoring.

MongoDB Atlas (job-scraper, kawan-ngonser, ofgs-api) and the R2 image bucket are external, so
nothing to restore there.

## 7. Woodpecker

Needs the `woodpecker` namespace (from `infrastructure`) and its database restored. Use
`values.yaml` and `secret.example.yaml` from
[homeserver/k8s/woodpecker](https://github.com/DTA32/homeserver/tree/main/k8s/woodpecker), with the
database URL pointing at `postgres-postgis-service.prod.svc.cluster.local`:

    kubectl apply -f secret.yaml        # woodpecker-secrets, in woodpecker
    helm install woodpecker oci://ghcr.io/woodpecker-ci/helm/woodpecker --version 3.7.1 \
      -n woodpecker -f values.yaml

## 8. Development environment

`dev.bdgcafe.com` runs bandung-coffeeshop behind basic auth, against its own empty postgres.

1. DNS for `dev.bdgcafe.com`, the `bdgcafe-dev-basic-auth` secret in `envoy`, and the two `dev`
   secrets.
2. Enable `data-development`, then create the role and database it expects — commands in
   [data/README.md](data/README.md#development).
3. The frontend image bakes `VITE_API_BASE_URL` in at build time. If it was built by hand, check
   it points at the dev API:

       docker run --rm --entrypoint sh <apps/development/bandung-coffeeshop-fe image> \
         -c 'grep -rl "dev.bdgcafe.com" / 2>/dev/null | head -3'   # no output = prod URL baked in

4. Enable `apps-development`.

## 9. Verify

    flux check
    flux get kustomizations
    kubectl get pods -n flux-system -o wide          # all on the high-availability node
    kubectl get gateway main-gateway -n envoy        # PROGRAMMED True
    kubectl get certificate -n envoy                 # READY True per hostname
    flux get image policy -A

A policy reports `no image found` until its repo has built once with the `<ts>-<sha>` scheme.
When one does, check the tag really has a timestamp prefix: `-3d895b7` with nothing before the
dash means `${CI_PIPELINE_STARTED}` did not resolve in Woodpecker, and nothing will deploy.

## Day-to-day

- **Deploy an automated image** (list in [images/README.md](images/README.md)): push to the app
  repo's default branch. Flux commits the bump within ~16 minutes; to hurry it:
  `flux reconcile image repository <name>` then `flux reconcile image update flux-system`.
- **Deploy a pinned image:** edit the tag in its manifest, commit, push.
- **Change config:** edit `apps/<env>/configmap/<app>.env`. The ConfigMap is regenerated under a
  new name, so the workload rolls on its own.
- **Pause everything now:** `flux suspend kustomization flux-system` first — otherwise it
  re-applies the children and undoes you — then suspend the child. To pause durably, commit
  `suspend: true`.
- **Add an app:** [apps/README.md](apps/README.md#adding-an-app).
- **Notifications:** with `discord-webhook` in place, `clusters/dta32-kube/notifications.yaml`
  posts Kustomization and image-update results, plus registry and Git errors. Check with
  `flux get alert-providers` and `flux get alerts`.

## Secrets reference

Key names come from what the manifests reference. Namespaces other than `flux-system` exist once
`infrastructure` has reconciled. Any database host inside a value must be the in-cluster postgres,
`postgres-postgis-service`.

| Namespace     | Secret                            | Keys                                                                                                                                                                                                                                                                                                                                                        |
|---------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `flux-system` | `flux-registry-creds`             | `.dockerconfigjson` — from `images/registry-creds.example.yaml`                                                                                                                                                                                                                                                                                             |
| `flux-system` | `discord-webhook`                 | `address` (optional)                                                                                                                                                                                                                                                                                                                                        |
| `envoy`       | `split-bill-be-basic-auth`        | `.htpasswd`                                                                                                                                                                                                                                                                                                                                                 |
| `envoy`       | `bdgcafe-dev-basic-auth`          | `.htpasswd` (dev only)                                                                                                                                                                                                                                                                                                                                      |
| `woodpecker`  | `woodpecker-secrets`              | `WOODPECKER_GITHUB_CLIENT` `WOODPECKER_GITHUB_SECRET` `WOODPECKER_DATABASE_DATASOURCE`                                                                                                                                                                                                                                                                      |
| `prod`        | `postgres-postgis-secret`         | `POSTGRES_PASSWORD` `PGDATA`                                                                                                                                                                                                                                                                                                                                |
| `prod`        | `postgres-backup-r2-secret`       | `R2_ACCOUNT_ID` `R2_ACCESS_KEY_ID` `R2_SECRET_ACCESS_KEY` `R2_BUCKET`                                                                                                                                                                                                                                                                                       |
| `prod`        | `bandung-coffeeshop-be-secret`    | `DB_USER` `DB_NAME` `DB_PASSWORD`                                                                                                                                                                                                                                                                                                                           |
| `prod`        | `bandung-coffeeshop-image-secret` | `R2_ACCOUNT_ID` `R2_ACCESS_KEY_ID` `R2_SECRET_ACCESS_KEY`                                                                                                                                                                                                                                                                                                   |
| `prod`        | `job-scraper-secret`              | `MONGO_URI` `DISCORD_WEBHOOK_URL`                                                                                                                                                                                                                                                                                                                           |
| `prod`        | `kawan-ngonser-be-secret`         | `MONGODB_URI`                                                                                                                                                                                                                                                                                                                                               |
| `prod`        | `ofgs-api-secret`                 | `SERVER_PORT` `API_PREFIX` `DATABASE_URL`                                                                                                                                                                                                                                                                                                                   |
| `prod`        | `sinan-gotth-finance-secret`      | `POSTGRES_USER` `POSTGRES_PASSWORD` `POSTGRES_DB` `ACCESS_TOKEN_PRIVATE_KEY` `ACCESS_TOKEN_PUBLIC_KEY` `REFRESH_TOKEN_PRIVATE_KEY` `REFRESH_TOKEN_PUBLIC_KEY` `SESSION_SECRET_KEY`                                                                                                                                                                          |
| `prod`        | `skripsi-secret`                  | `POSTGRES_URL`                                                                                                                                                                                                                                                                                                                                              |
| `prod`        | `split-bill-be-secret`            | `POSTGRES_USER` `POSTGRES_PASSWORD` `POSTGRES_DB` `GEMINI_API_KEY`                                                                                                                                                                                                                                                                                          |
| `prod`        | `tiketin-v2-be-secret`            | `APP_KEY` `DB_CONNECTION` `DB_HOST` `DB_PORT` `DB_DATABASE` `DB_USERNAME` `DB_PASSWORD` `DB_FOREIGN_KEYS`                                                                                                                                                                                                                                                   |
| `prod`        | `vern-secret`                     | `APP_KEY` `DB_CONNECTION` `DB_HOST` `DB_PORT` `DB_DATABASE` `DB_USERNAME` `DB_PASSWORD` `GOOGLE_CLIENT_ID` `GOOGLE_CLIENT_SECRET` `GOOGLE_REDIRECT_URL` `MAIL_MAILER` `MAIL_HOST` `MAIL_PORT` `MAIL_USERNAME` `MAIL_PASSWORD` `MAIL_FROM_ADDRESS` `MAIL_FROM_NAME` `MIDTRANS_CLIENT_KEY` `MIDTRANS_SERVER_KEY` `MIDTRANS_MERCHANT_ID` `MIDTRANS_PRODUCTION` |
| `dev`         | `postgres-postgis-secret`         | `POSTGRES_PASSWORD` `PGDATA`                                                                                                                                                                                                                                                                                                                                |
| `dev`         | `bandung-coffeeshop-be-secret`    | `DB_USER` `DB_NAME` `DB_PASSWORD`                                                                                                                                                                                                                                                                                                                           |

`PGDATA` is a subdirectory of the volume mount, for example `/var/lib/postgresql/data/pgdata`.

Plain secrets:

    kubectl create secret generic <name> -n <namespace> \
      --from-literal=KEY_1='<value>' --from-literal=KEY_2='<value>'

Basic auth:

    htpasswd -cbB /tmp/auth.htpasswd <user> '<password>'
    kubectl create secret generic <secret-name> -n envoy --from-file=.htpasswd=/tmp/auth.htpasswd
    rm /tmp/auth.htpasswd

Discord — create the webhook under Server Settings → Integrations → Webhooks; the webhook decides
the channel:

    kubectl create secret generic discord-webhook -n flux-system \
      --from-literal=address='https://discord.com/api/webhooks/<id>/<token>'

## Known limitations

- **Add-ons are outside Flux.** Envoy Gateway, cert-manager and Woodpecker are Helm CLI releases.
- **Secrets are manual.** SOPS or sealed-secrets would let them live in this repo safely.
- **Prod postgres is tied to one node** through its hostPath volume. Losing that node means
  restoring from R2.
- **Most images are pinned by hand.** Which ones auto-update is in [images/README.md](images/README.md).
