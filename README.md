# Argo CD for a team — reference setup

Example GitOps repository for one team (`payments`) running two services
(`checkout-api`, `ledger-worker`) on Argo CD, following trunk-based development.
Everything is fictional (`git.example.com`, `registry.example.com`) but every
manifest validates: `kustomize build` for the overlays, `kubeconform` (with the
Argo CD CRD schemas) for the Argo resources.

## Layout

```
team-example/
├── CODEOWNERS                         prod overlays require release-manager review
├── apps/                              Kubernetes manifests (kustomize)
│   ├── checkout-api/
│   │   ├── base/                      env-agnostic; image has NO tag
│   │   └── overlays/
│   │       ├── staging/               namespace, image tag, replicas, config, ingress host
│   │       └── prod/                  + HPA, PDB, bigger resources; tag lags staging
│   └── ledger-worker/                 second app so the ApplicationSets have something to discover
├── argocd/                            Argo CD resources (applied by the root app)
│   ├── bootstrap/root.yaml            app-of-apps, the only thing applied by hand
│   ├── projects/team-payments.yaml    AppProject: fences, RBAC, sync windows
│   ├── applicationsets/
│   │   ├── staging.yaml               apps/*/overlays/staging  -> <app>-staging
│   │   ├── prod.yaml                  apps/*/overlays/prod     -> <app>-prod
│   │   └── pr-previews.yaml           one env per labelled PR (Forgejo pull-request generator)
│   ├── repos/                         repository credentials (placeholders)
│   └── reference/                     NOT applied: hand-written Application, matrix ApplicationSet, OCI patterns
└── ci/                                Forgejo Actions workflows (would live in .forgejo/workflows/)
    ├── checkout-api.ci.yaml           build image -> bump staging -> PR for prod
    └── publish-oci.yaml               helm push / oras push to an OCI registry
```

Two repositories are assumed:

| Repo | Content | Branches |
|------|---------|----------|
| `payments/checkout-api` (code) | source, Dockerfile, `ci/checkout-api.ci.yaml` | `main` + short-lived PR branches |
| `payments/gitops` (this tree) | `apps/`, `argocd/`, `CODEOWNERS` | `main` only |

## Trunk-based development with Argo CD

The one rule: **environments are directories, not branches.** Every
`Application` — staging, prod, previews — points at `targetRevision: main`.
The only thing that differs between environments is the overlay directory.

```
code repo                      gitops repo (main)                         cluster
─────────                      ─────────────────                          ───────
push to main ──build──> image sha-3f9c2a1
                  │
                  └──CI commit──> apps/checkout-api/overlays/staging   ──ArgoCD──> payments-staging
                                  newTag: sha-3f9c2a1                              (auto-sync, ~3 min)

release manager dispatches "promote sha-3f9c2a1"
                  └──CI opens PR──> apps/checkout-api/overlays/prod
                                    newTag: sha-3f9c2a1
                     CODEOWNERS review + merge                           ──ArgoCD──> payments-prod
                                                                                    (auto-sync, sync window)
```

- **Build once, deploy many.** CI tags every build `sha-<7 char git sha>`.
  The same image moves staging → prod; nothing is rebuilt.
- **Promotion is a one-line diff** (`newTag` in the prod overlay). Rollback is
  `git revert`. Git history *is* the deployment history.
- **Staging is automatic, prod is reviewed.** `CODEOWNERS` makes the prod
  overlay require a release-manager approval; CI pushes staging bumps straight
  to `main`.
- **No merge conflicts between environments**, no cherry-picks, no
  "staging branch drifted from main".
- **Previews for PRs.** Short-lived branches still get a real environment:
  `pr-previews.yaml` builds `staging overlay + this PR's image` into
  `payments-pr-<n>` for PRs labelled `preview` against `main`.

## Kustomize overlay conventions

| Concern | Where | How |
|---------|-------|-----|
| Image version | overlay `images:` | immutable `sha-…` tag; base image has no tag so it can't be deployed by accident |
| Replica count | overlay `replicas:` | prod value is the initial value only; HPA owns it afterwards |
| Config | `configMapGenerator` base + `behavior: merge` in overlay | hash suffix → config change rolls the Deployment |
| Ingress host | JSON patch in overlay | base host is a placeholder |
| Prod-only resources | `hpa.yaml`, `pdb.yaml`, `resources-patch.yaml` | listed in the prod kustomization only |
| Namespace | overlay `namespace:` | matches the Application's `destination.namespace` |
| Labels | `labels:` (not deprecated `commonLabels`) | `environment` added per overlay without touching selectors |

Render and inspect any overlay locally exactly as Argo CD will:

```sh
kustomize build apps/checkout-api/overlays/prod
kustomize build apps/checkout-api/overlays/prod | kubeconform -strict -summary
```

## Argo CD objects

### AppProject (`argocd/projects/team-payments.yaml`)

One project per team. It is the security boundary:

- `sourceRepos` — only the team's gitops repo and OCI registry.
- `destinations` — only namespaces matching `payments-*` (covers staging, prod, previews).
- `clusterResourceWhitelist` — only `Namespace` (needed for `CreateNamespace=true`); nothing else cluster-scoped.
- `namespaceResourceBlacklist` — `ResourceQuota`, `LimitRange`, `NetworkPolicy` stay with the platform team.
- `roles` — `developer` can sync `*-staging` and `*-pr-*`; `release-manager` can do everything. Groups map to SSO.
- `syncWindows` — deny prod syncs Fri 16:00 → Mon 06:00 UTC; `manualSync: true` lets a release manager hotfix.
- `orphanedResources.warn` — surfaces things living in the namespace that git doesn't know about.

### ApplicationSets (`argocd/applicationsets/`)

`staging.yaml` and `prod.yaml` use the **git directory generator**:

```yaml
generators:
  - git:
      repoURL: https://git.example.com/payments/gitops.git
      revision: main
      directories:
        - path: apps/*/overlays/staging
template:
  metadata:
    name: '{{index .path.segments 1}}-staging'   # apps/<name>/overlays/staging -> <name>-staging
  spec:
    source:
      path: '{{.path.path}}'
```

Adding a service = adding `apps/<name>/overlays/staging/`. No Argo CD change.
Deleting the directory removes the Application (staging) or leaves it for a
human to delete (prod, `applicationsSync: create-update`).

Why one ApplicationSet per environment instead of a matrix:

| | per-env sets (used) | matrix (`reference/all-envs-matrix.applicationset.yaml`) |
|-|-|-|
| Apps without every env | fine — only existing dirs match | breaks unless every app has every overlay |
| Prod-only policy | plain YAML in `prod.yaml` | hidden in `templatePatch` conditionals |
| Duplication | template repeated per env | DRY |
| Blast radius of an edit | one env | all envs |

Prod-specific knobs worth calling out:

- `applicationsSync: create-update` + `preserveResourcesOnDeletion: true` — the generator never deletes prod.
- `ignoreApplicationDifferences: [/spec/syncPolicy]` — on-call can disable auto-sync on one app during an incident without the ApplicationSet fighting back.
- `ignoreDifferences: /spec/replicas` + `RespectIgnoreDifferences=true` — HPA owns replicas.
- `allowEmpty: false` — refuse to sync an app down to zero resources.

`pr-previews.yaml` uses the **pull-request generator** (`gitea` provider — Forgejo
speaks the same API). It reads PRs from the *code* repo but deploys the *gitops*
repo's staging overlay with three render-time overrides (`kustomize.namespace`,
`kustomize.images`, `kustomize.patches` for the host). So a preview is "staging
plus this image", which makes the comparison honest.

### Sync policy (all environments)

| Option | Why |
|--------|-----|
| `automated.prune: true` | what git deletes, the cluster deletes |
| `automated.selfHeal: true` | `kubectl edit` is reverted; git is the only way in |
| `CreateNamespace=true` | namespace per env created on first sync |
| `PruneLast=true` | create/update first, prune at the end → no gaps mid-sync |
| `ApplyOutOfSyncOnly=true` | cheaper syncs for large apps |
| `ServerSideApply=true` | no `last-applied` annotation limit, clean field ownership with HPA/controllers |
| `retry` with backoff | transient webhook / CRD-ordering failures heal themselves |
| finalizer `resources-finalizer.argocd.argoproj.io` | deleting the Application cascades to resources |

### Root app (`argocd/bootstrap/root.yaml`)

```sh
kubectl apply -n argocd -f argocd/bootstrap/root.yaml
```

App-of-apps over `argocd/` with an explicit include list
(`projects/*.yaml,applicationsets/*.yaml`). `reference/` is documentation,
`repos/` secrets come from a secret manager, `bootstrap/` is the root itself.
The `AppProject` carries `sync-wave: "-10"` so it exists before the
ApplicationSets that reference it.

## OCI

Argo CD can pull from an OCI registry instead of (or in addition to) git.
Examples in `argocd/reference/oci-sources.yaml`, credentials in `argocd/repos/`,
producer side in `ci/publish-oci.yaml`.

| Pattern | Application | Repo credential | Version |
|---------|-------------|-----------------|---------|
| Helm chart, Helm-repo style | `repoURL: registry.example.com/payments/charts` (no `oci://`), `chart:`, `targetRevision: <semver>` | `type: helm`, `enableOCI: "true"` | ≥ 2.3 |
| Helm chart, native OCI | `repoURL: oci://…/charts/checkout-api`, `path: .`, `targetRevision: <semver>` | `type: oci` | ≥ 3.1 |
| Kustomize dir / manifests | `repoURL: oci://…/gitops`, `targetRevision: sha-… or sha256:…`, `path: apps/…/overlays/staging` | `type: oci` | ≥ 3.1 |

Rules for artifacts Argo CD will accept: **one layer**, media type
`application/vnd.oci.image.layer.v1.tar+gzip` (or the Helm chart type).
`oras push <ref> .` from inside a directory does this for you; for a tarball set
the media type explicitly. Standard `org.opencontainers.image.*` annotations are
shown in the UI — always stamp `…image.revision` with the git SHA.

Credential matching: a secret labelled `repo-creds` is a URL-prefix template
(one secret for `oci://registry.example.com/payments/*`); `repository` needs the
exact URL.

When OCI is worth it:

- Supply chain: artifacts are immutable, content-addressed, signable with
  cosign; a registry/admission policy can enforce "only signed artifacts reach prod".
- Argo CD needs registry credentials only — no git access from the cluster,
  cheaper pulls for many clusters or air-gapped setups.
- Clean split of ownership: chart published by one team to OCI, environment
  values kept in the consuming team's git repo (multi-source, `$values` ref).

Cost: you lose "git blame on the deployed thing" unless CI records the git SHA
in the artifact annotations, and you add a registry to the critical path.

## Validation used here

```sh
# overlays render and are valid k8s
for d in apps/*/overlays/*; do kustomize build "$d" | kubeconform -strict -summary; done

# Argo CD resources validate against the CRD schemas
kubeconform -strict -summary -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  argocd/projects argocd/applicationsets argocd/bootstrap argocd/reference argocd/repos
```

## Known gaps / things to decide for a real team

- Namespaces created by `CreateNamespace=true` are not tracked, so preview
  namespaces survive PR close. Ship a `Namespace` resource in the manifests, or
  add a Kyverno/cron cleanup on `environment=preview`.
- Root app uses project `default`; a real setup gets a locked-down `bootstrap` project.
- Repo credentials are plaintext placeholders; use SealedSecrets / External Secrets.
- CI write-back (`kustomize edit set image` + commit) vs Argo CD Image Updater:
  this tree uses CI write-back because it keeps prod promotion as a reviewed PR.
- Webhooks from Forgejo to Argo CD remove the polling delay for both git and
  pull-request generators.
- Multi-cluster: add cluster secrets, extend `destinations`, and either
  parameterise `destination.server` per env or keep one ApplicationSet per cluster.
