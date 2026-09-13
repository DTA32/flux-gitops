# data

Postgres, one instance per environment, each reconciled by its own Flux Kustomization.

| Path | Namespace | Storage | Kustomization |
|---|---|---|---|
| `production/postgres-postgis/` | `prod` | hostPath PV at `/mnt/data/postgres-postgis`, 10Gi | `data-production` |
| `development/postgres-postgis/` | `dev` | dynamic `local-path` claim, 5Gi | `data-development` |

**Why not inside `apps/`:** a bad reconcile or a stray `prune` under `apps` must never be able to
reach a database. Each `apps-*` Kustomization `dependsOn` its `data-*`, so nothing that needs
postgres is applied before it is Ready.

**Why two Kustomizations:** both run with `wait: true`. Combined, a dev postgres that cannot
schedule would mark the whole thing NotReady and hold `apps-production` back.

## production

- **Pinned to one node.** The PV is a hostPath with no node affinity, so the Deployment carries
  `nodeSelector: node_type: high-availability`. Create the directory on that node before the
  first reconcile. Without the pin, a rescheduled pod could start on another node against an empty
  directory — which looks exactly like a wiped database.
- **`storageClassName: manual`** makes the claim bind this hand-made PV instead of provisioning a
  new, empty `local-path` volume.
- **Empty on a new cluster.** Restore from R2 — see the [root README](../README.md#6-restore-databases).
- **Backed up nightly** by `apps/production/postgres-backup-r2`, one object per database plus
  roles.
- **Secret:** `postgres-postgis-secret` — `POSTGRES_PASSWORD`, `PGDATA`.

## development

Disposable by design: no hand-made directory and no node pin. `local-path` provisions wherever the
pod lands, and its reclaim policy is `Delete`, so `kubectl delete pvc` really discards the data.

It starts empty. Create the role and database that `bandung-coffeeshop-be-secret` in `dev`
expects:

    kubectl exec -n dev deploy/postgres-postgis -- \
      psql -U postgres -c "CREATE ROLE <user> LOGIN PASSWORD '<password>';"
    kubectl exec -n dev deploy/postgres-postgis -- \
      psql -U postgres -c "CREATE DATABASE <db> OWNER <user>;"

**Secret:** `postgres-postgis-secret` in `dev` — `POSTGRES_PASSWORD`, `PGDATA`.

## Notes

- Both instances answer as `postgres-postgis-service` inside their own namespace, which is why the
  dev and prod ConfigMaps can share the same `DB_HOST`.
- **The image is bumped by hand.** `imresamu/postgis` is upstream and not under image automation.
  Upgrade the backup image's Postgres client alongside it — `pg_dump` refuses to dump a server
  newer than itself. See
  [homeserver/k8s/postgres-backup-r2](https://github.com/DTA32/homeserver/tree/main/k8s/postgres-backup-r2).
