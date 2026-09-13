# images

One `ImageRepository` + `ImagePolicy` pair for each image that updates automatically. Reconciled by
the `images` Kustomization, which only reads registries and so is never suspended.

| Path | |
|---|---|
| `production/` | Six enabled pairs, named after the app (`ofgs-api`, …) |
| `development/` | Two pairs, suffixed `-dev` — every policy lives in `flux-system`, so names must be unique |

- **ImageRepository** polls a registry every 10 minutes and caches its tags.
- **ImagePolicy** filters those tags and elects one.
- **ImageUpdateAutomation** (`clusters/dta32-kube/image-automation.yaml`) rewrites every image
  line carrying that policy's marker, commits as `fluxcdbot`, and pushes to `master`.

## Tag scheme

Every pipeline pushes two tags:

    ${CI_COMMIT_SHA:0:7}                          e.g. 3d895b7              for people and rollback
    ${CI_PIPELINE_STARTED}-${CI_COMMIT_SHA:0:7}   e.g. 1756180000-3d895b7   what Flux sorts on

and every policy uses the same filter:

    filterTags:
      pattern: '^(?P<ts>[0-9]+)-[0-9a-f]{7}$'
      extract: '$ts'
    policy:
      numerical:
        order: asc

A bare sha has no order Flux can use, which is why the timestamp is there. Pipelines publish only
from their default branch, so a feature branch can never win the election; the bandung repos
publish `master` to `/prod` and `dev` to `/dev`.

## Credentials

All eight repositories use `flux-registry-creds`. Three of the images are on Docker Hub, where
anonymous tag listing is rate limited, so the secret is not optional. Copy
`registry-creds.example.yaml`, fill it in, and apply it by hand; read-only scope is enough.

## Notes

- A policy reports `version list argument cannot be empty` until its repo has built once with
  this tag scheme — no tag matches the filter yet. It clears on the first such build.
- `flux get image policy -A` shows the newest elected tag for everything.
- End to end is up to ~16 minutes (registry scan, then automation, then Git poll). To hurry:
  `flux reconcile image repository <name>` then `flux reconcile image update flux-system`.
- A full marker rewrites the whole image reference, registry and tag — so a manifest can move to a
  new registry in the same commit that picks up its first matching tag.
