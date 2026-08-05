# ACM OpenShift GitOps Operator - Governance Policy Install (push model)

Installs the **OpenShift GitOps** operator on RHACM-managed clusters using the
**Governance Policy framework** (push model), targeted by **ManagedClusterSet**
(`dev`, `prod`), with the policy content versioned **one branch per environment**.

## Why Policy instead of Subscription/Channel

| | Subscription (classic app push) | Policy (governance push) |
|---|---|---|
| Continuous drift correction | No — applies once per change | Yes — `remediationAction: enforce` keeps re-reconciling |
| Compliance visibility per cluster | Limited | Native compliance dashboard in ACM console |
| Templating | Manual ConfigMap/Secret overrides | Rich hub templates: `.ManagedClusterName`, `.ManagedClusterLabels`, `lookup` |
| Good fit for "bootstrap a cluster capability" | OK | Better — this is what Governance is designed for |

Policy hub-templates (`{{hub ... hub}}`) are resolved **on the hub**, per target
cluster, before the manifest is wrapped into a `ManifestWork` and pushed down. That
means every object we deploy can reference the destination cluster's own identity
without us hand-writing per-cluster YAML:

```
clusterName:   {{hub .ManagedClusterName hub}}
apiServerURL:  {{hub (index (lookup "cluster.open-cluster-management.io/v1" "ManagedCluster" "" .ManagedClusterName).spec.managedClusterClientConfigs 0).url hub}}
```

## Repo layout

```
main (this branch)                       # cluster-scoped, shared, environment-agnostic
├── clustersets/
│   ├── clusterset-dev.yaml              # ManagedClusterSet: dev
│   ├── clusterset-prod.yaml             # ManagedClusterSet: prod
│   ├── clustersetbinding-dev.yaml       # binds "dev"  set  into dev-apps ns
│   └── clustersetbinding-prod.yaml      # binds "prod" set  into prod-apps ns
└── namespaces/
    ├── namespace-dev-apps.yaml
    └── namespace-prod-apps.yaml

dev  branch                              # self-contained: Placement + Policy + Binding
└── policies/
    ├── placement-gitops-operator.yaml   # clusterSets: [dev]
    ├── policy-gitops-operator.yaml      # ConfigurationPolicy: NS + OperatorGroup + Subscription + templated ConfigMap
    ├── placementbinding-gitops-operator.yaml
    └── kustomization.yaml

prod branch                              # same shape, stricter posture
└── policies/  (same files, different values — see table below)
```

## Environment differences (dev vs prod)

| | dev branch | prod branch |
|---|---|---|
| Policy `remediationAction` | `enforce` (auto-applies) | `inform` (reports drift; you flip to `enforce` after review — or approve via `PolicyAutomation`) |
| OLM channel | `latest` | `gitops-1.14` (pinned) |
| Install plan approval | `Automatic` | `Manual` |
| Policy severity | `high` | `critical` |

Promote a change dev → prod the normal way: open a PR from `dev` into `prod`.

## One-time setup on the hub

These are true day-0 bootstrap steps — the hub's own namespaces, RBAC and Argo CD
config have to exist before Git-driven delivery of anything else can start.
**Prerequisite:** OpenShift GitOps is already installed on the hub itself (Layer 1's
Policy only installs it on the *managed* clusters).

1. Apply the shared, environment-agnostic resources from `main`:

   ```bash
   git checkout main
   oc apply -f namespaces/
   oc apply -f clustersets/
   oc apply -f gitopscluster/
   ```

2. Grant the hub's Argo CD permission to manage Policies/Placements, and enable the
   `PolicyGenerator` Kustomize plugin on it (edit the image tag in the file to match
   your RHACM version first):

   ```bash
   oc apply -f argocd-policy-cd/clusterrole-policy-admin.yaml
   oc apply -f argocd-policy-cd/clusterrolebinding-policy-admin.yaml
   oc patch argocd openshift-gitops -n openshift-gitops --type merge \
     --patch-file argocd-policy-cd/argocd-enable-policygenerator.yaml
   ```

3. Point Argo CD at each branch's `policies/` directory. From here on, the Policy
   that installs the operator is delivered **continuously** from Git — a merge to
   `dev`/`prod` is all it takes, no more manual `oc apply -k policies/`:

   ```bash
   oc apply -f argocd-policy-cd/application-policies-dev.yaml
   oc apply -f argocd-policy-cd/application-policies-prod.yaml
   ```

   `dev`'s Application auto-syncs (`prune: true`, `selfHeal: true`); `prod`'s has no
   `automated:` block, so someone runs `argocd app sync policies-gitops-operator-prod`
   (or clicks Sync in the console) after reviewing the diff — mirrors the
   `remediationAction: inform` already set on the generated prod Policy itself.

4. Put each managed cluster into the right set — the only per-cluster step:

   ```bash
   oc label managedcluster <dev-cluster-name>  cluster.open-cluster-management.io/clusterset=dev  --overwrite
   oc label managedcluster <prod-cluster-name> cluster.open-cluster-management.io/clusterset=prod --overwrite
   ```

5. Watch compliance:

   ```bash
   oc get policy -n dev-apps  policy-gitops-operator-dev  -o jsonpath='{.status.compliant}'
   oc get policy -n prod-apps policy-gitops-operator-prod -o jsonpath='{.status.compliant}'
   ```

   Each managed cluster ends up with, in the `openshift-gitops-operator` namespace:
   - the `Namespace`, `OperatorGroup`, `Subscription` (→ OLM installs the operator)
   - a `ConfigMap` named `cluster-gitops-info` containing that cluster's own name
     and API server URL, populated purely from ACM hub template variables — no
     per-cluster file in Git.

## Continuous delivery of the policies themselves

`policies/` in each branch is now driven by a `PolicyGenerator` (`policy-generator-config.yaml`
+ raw manifests under `manifests/`) instead of a hand-written `Policy`. The
`Application`s in `argocd-policy-cd/` (living on `main`, since they're hub-scoped)
point Argo CD at that Kustomize output per branch:

```
main
└── argocd-policy-cd/
    ├── clusterrole-policy-admin.yaml
    ├── clusterrolebinding-policy-admin.yaml
    ├── argocd-enable-policygenerator.yaml   # patch for the existing hub `argocd` CR
    ├── application-policies-dev.yaml        # watches dev  branch's policies/
    └── application-policies-prod.yaml       # watches prod branch's policies/

dev / prod
└── policies/
    ├── manifests/
    │   ├── namespace.yaml
    │   ├── operatorgroup.yaml
    │   ├── subscription.yaml
    │   └── configmap-cluster-info.yaml      # the hub-templated ConfigMap
    ├── placement-gitops-operator.yaml       # reused via placementPath, not regenerated
    ├── policy-generator-config.yaml
    └── kustomization.yaml
```

This Application only ever touches the hub (creating `Policy`/`Placement`/
`PlacementBinding` objects there) — it never needs GitOps running on the spokes, so
there's still no circular dependency, just one more link in the chain: Git → hub
Argo CD → `Policy` → managed cluster.

## Pushing to GitHub

```bash
git remote add origin https://github.com/alexgdusarh/acm-openshift-gitops-install.git
git push -u origin main dev prod
```

## Layer 2: pushing multiple Argo CD apps to every cluster in an environment

On top of the operator install (Layer 1, Policy-based), each branch now also carries
an **ApplicationSet push model** setup for deploying ordinary workloads:

```
main
└── gitopscluster/
    ├── managedclustersetbinding-gitops-dev.yaml
    ├── managedclustersetbinding-gitops-prod.yaml
    ├── placement-all-clusters.yaml   # every cluster, all environments
    └── gitopscluster.yaml            # registers them as Argo CD cluster secrets (push)

dev / prod (same shape in both)
├── apps/
│   ├── hello-world/                  # reusable app #1 (kustomize)
│   └── cluster-banner/               # reusable app #2 (kustomize)
└── appset/
    ├── placement-apps-<env>.yaml     # which clusters this env's apps go to
    └── applicationset-<env>.yaml     # matrix: apps/* x clusters -> N x M Applications
```

**Why push, and why this scales to "several Argo CD apps":** the `ApplicationSet`
uses a **matrix generator** — one axis is a `git` generator that auto-discovers every
folder under `apps/`, the other is a `clusterDecisionResource` generator that reads
the Placement's decisions. Add a new app = add a folder. Add a new cluster to the
`dev`/`prod` `ManagedClusterSet` = it automatically gets every existing app, no new
YAML. Both axes are native ACM/Argo CD variables:

```
name:   '{{name}}'     # the managed cluster's name
server: '{{server}}'   # the managed cluster's live API server URL
```

These come for free from the `clusterDecisionResource` generator — no hub templating
required for this layer (that's specifically a Policy feature, used in Layer 1).

`dev`'s ApplicationSet auto-syncs (`prune: true`, `selfHeal: true`); `prod`'s is
`automated: {}`-free so a human approves each sync from the Argo CD UI/CLI — same
promote-via-PR workflow as Layer 1, applied to the app layer too.

**Prerequisites for push specifically:**
- OpenShift GitOps operator installed on the **hub** itself (Layer 1's Policy only
  installs it on the *managed* clusters — the hub needs its own instance to run the
  Argo CD server that does the pushing).
- The `ManagedServiceAccount` add-on enabled, so the token used to push to each
  managed cluster rotates automatically.
- By default the hub's Argo CD application-controller service account needs
  cluster-admin on each managed cluster to apply arbitrary resources. If you don't
  want that, see "Creating a customized service account for Argo CD push model" in
  the RHACM GitOps docs — it lets you scope a `ManagedServiceAccount` +
  `ClusterPermission` to exactly the namespaces/verbs each app needs instead.

## Adding a third environment (e.g. staging)

1. `main`: add `ManagedClusterSet staging`, its `ManagedClusterSetBinding`s (dev-apps
   style + the `openshift-gitops` one), and add `- staging` to
   `gitopscluster/placement-all-clusters.yaml`.
2. Branch `dev` (or `prod`) into `staging`, adjust `policies/policy-gitops-operator.yaml`
   and `appset/placement-apps-staging.yaml` / `appset/applicationset-staging.yaml`
   `clusterSets:`/`revision:` to `staging`.
3. Label the clusters: `cluster.open-cluster-management.io/clusterset=staging`.

No changes needed anywhere else — `apps/` is reused as-is.

## `ocm-placement-generator` ConfigMap

Both `ApplicationSet`s reference `configMapRef: ocm-placement-generator`. This is
**not** auto-created by ACM/`GitOpsCluster` — `GitOpsCluster` only registers cluster
secrets; it has nothing to do with the `ApplicationSet` controller. The ConfigMap
is Argo CD's own "duck-typing" config for the `clusterDecisionResource` generator:
it tells the generator which GVK to read (`PlacementDecision`) and which status
field holds the resolved cluster list. You create it once per Argo CD instance —
`gitopscluster/configmap-ocm-placement-generator.yaml` on `main` does that — and
every `ApplicationSet` in that instance reuses the same name, distinguished only by
`labelSelector` picking a specific `Placement`.

It also needs RBAC: the `openshift-gitops-applicationset-controller` service
account has to be able to `list`/`watch` `PlacementDecisions`, which
`gitopscluster/clusterrole-placementdecision-reader.yaml` +
`clusterrolebinding-placementdecision-reader.yaml` grant. Without that RBAC you'll
see `cannot list resource "placementdecisions"` in the controller logs even with
the ConfigMap in place and `GitOpsCluster` reporting healthy.

Apply both, from `main`:

```bash
oc apply -f gitopscluster/
```

If an `ApplicationSet` still isn't generating `Application`s, check in this order:

```bash
oc get configmap ocm-placement-generator -n openshift-gitops
oc get placementdecisions -n openshift-gitops
oc logs -n openshift-gitops deploy/openshift-gitops-applicationset-controller | grep -i "placementdecision\|clusterdecision"
```
