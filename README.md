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

1. Apply the shared, environment-agnostic resources from `main`:

   ```bash
   git checkout main
   oc apply -f namespaces/
   oc apply -f clustersets/
   ```

2. Apply each environment's policy bundle straight from its branch:

   ```bash
   git checkout dev
   oc apply -k policies/

   git checkout prod
   oc apply -k policies/
   ```

   (In practice you'd point an ACM `PolicyGenerator`/Argo CD Application at each
   branch so this reconciles continuously instead of a one-time `oc apply` — see
   "Continuous delivery of the policies themselves" below.)

3. Put each managed cluster into the right set — the only per-cluster step:

   ```bash
   oc label managedcluster <dev-cluster-name>  cluster.open-cluster-management.io/clusterset=dev  --overwrite
   oc label managedcluster <prod-cluster-name> cluster.open-cluster-management.io/clusterset=prod --overwrite
   ```

4. Watch compliance:

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

To make this actually GitOps-driven rather than a one-time `oc apply`, wrap
`policies/` in each branch with a `PolicyGenerator` and point an OpenShift GitOps
`Application` (running on the hub) at that branch. That Application only ever
touches the hub (creating `Policy`/`Placement`/`PlacementBinding` objects there) —
it never needs GitOps on the spokes, so there's no circular dependency.

## Pushing to GitHub

```bash
git remote add origin https://github.com/<your-org>/acm-openshift-gitops-install.git
git push -u origin main dev prod
```
