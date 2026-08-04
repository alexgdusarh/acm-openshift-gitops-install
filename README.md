# ACM OpenShift GitOps Operator - Multi-Cluster Pull Install

Installs the **OpenShift GitOps** operator on RHACM-managed clusters, targeted by
**ManagedClusterSet** (`dev`, `prod`), with the operator manifests versioned
**one branch per environment**:

- `dev`  branch → `manifests/openshift-gitops-operator/` → applied to clusters in the `dev`  ManagedClusterSet
- `prod` branch → `manifests/openshift-gitops-operator/` → applied to clusters in the `prod` ManagedClusterSet

The `main` branch holds only the **hub-side bootstrap resources** (ManagedClusterSets,
bindings, Placements, Channels, Subscriptions, Applications). ACM's Subscription
controller reads the Channel's Git repo, checks out the branch pinned by the
`apps.open-cluster-management.io/git-branch` annotation, and applies whatever is
under `manifests/openshift-gitops-operator/` to every cluster selected by the
matching Placement/ManagedClusterSet.

## Architecture

```
                    ┌───────────────────────────┐
                    │        ACM Hub             │
                    │  main branch resources     │
                    │  ManagedClusterSet: dev     │───▶ clusters labeled clusterset=dev  ─▶ pulls "dev"  branch
                    │  ManagedClusterSet: prod    │───▶ clusters labeled clusterset=prod ─▶ pulls "prod" branch
                    └───────────────────────────┘
```

Each Subscription's Placement resolves against its ManagedClusterSet, so adding a
cluster to a set is the only step needed to bring it under management — no new
YAML required per cluster.

## Repo layout

```
main (this branch)
├── namespaces/                         # dev-apps / prod-apps hub namespaces
├── clustersets/                        # ManagedClusterSet + ManagedClusterSetBinding
├── placements/                         # Placement per environment
├── channels/                           # Git Channel per environment
├── subscriptions/                      # Subscription per environment (pins branch+path)
└── applications/                       # app.k8s.io Application wrapper (console view)

dev  branch
└── manifests/openshift-gitops-operator/
    ├── namespace.yaml
    ├── operatorgroup.yaml
    ├── subscription.yaml               # channel: latest, Automatic approval
    └── kustomization.yaml

prod branch
└── manifests/openshift-gitops-operator/
    ├── namespace.yaml
    ├── operatorgroup.yaml
    ├── subscription.yaml               # channel: gitops-1.14 pinned, Manual approval
    └── kustomization.yaml
```

## One-time setup on the hub

1. Push this repo to GitHub (see below), then edit `channels/channel-*.yaml` and
   replace `<your-org>` with your real GitHub org/user.

2. Apply the hub bootstrap, in order:

   ```bash
   oc apply -f namespaces/
   oc apply -f clustersets/
   oc apply -f placements/
   oc apply -f channels/
   oc apply -f subscriptions/
   oc apply -f applications/
   ```

3. Put each managed cluster into the right set (this is the only per-cluster step):

   ```bash
   oc label managedcluster <dev-cluster-name>  cluster.open-cluster-management.io/clusterset=dev  --overwrite
   oc label managedcluster <prod-cluster-name> cluster.open-cluster-management.io/clusterset=prod --overwrite
   ```

   (Requires the caller to have the `open-cluster-management:clusterset-admin-<name>`
   role, or do this from the ACM console under Infrastructure → Clusters → cluster sets.)

4. ACM propagates the Subscription content to each matching cluster as a
   `ManifestWork`. Within a couple of minutes each managed cluster will have:
   - `openshift-gitops-operator` Namespace
   - an `OperatorGroup`
   - a `Subscription` to the `openshift-gitops-operator` package, which OLM
     resolves into the running GitOps operator + default `openshift-gitops` ArgoCD instance.

5. Check status from the hub:

   ```bash
   oc get subscriptions.apps.open-cluster-management.io -n dev-apps
   oc get subscriptions.apps.open-cluster-management.io -n prod-apps
   ```

   Or on a managed cluster directly:

   ```bash
   oc get csv -n openshift-gitops-operator
   ```

## Environment differences (dev vs prod)

| | dev branch | prod branch |
|---|---|---|
| OLM channel | `latest` | `gitops-1.14` (pinned) |
| Install plan approval | `Automatic` | `Manual` (requires an admin to approve upgrades) |

Adjust these per your own change-control policy — the point of the branch split is
that each environment's manifests can diverge (and be promoted dev → prod via a
normal PR/merge) without touching hub resources at all.

## Pushing to GitHub

```bash
gh repo create <your-org>/acm-openshift-gitops-install --private --source=. --remote=origin
git push -u origin --all
```

or, without the `gh` CLI:

```bash
git remote add origin https://github.com/<your-org>/acm-openshift-gitops-install.git
git push -u origin main dev prod
```
