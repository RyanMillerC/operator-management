# operator-management

Manage operators via OLM with declarative versioning using manual upgrade mode with auto-approving InstallPlans based on Helm values.

## How it works

Each entry in `operators` produces a `Subscription` and, optionally, a `Namespace` and `OperatorGroup`. When `installPlanApproval: Manual` and `version` are set, an approval `Job` is also created that watches for the matching `InstallPlan` and approves it automatically.

ArgoCD sync waves ensure correct ordering:

| Wave | Resources |
|------|-----------|
| 0 | Namespace |
| 1 | OperatorGroup, ServiceAccount, Role, RoleBinding |
| 2 | Subscription, Job |

`Namespace`, `OperatorGroup`, and `Subscription` are annotated with `Prune=false` so removing an operator from values does **not** trigger ArgoCD to delete them on the next sync. Operator removal must be done manually.

## Version promotion workflow

This chart is intended for use across multiple environments (lab → nonprod → prod) with separate values files per environment. The `version` field is the GitOps approval signal — bumping it in a PR is how you promote an operator version.

1. Test the new version in lab by setting `version` in `values-lab.yaml`
2. Once validated, promote to nonprod via `values-nonprod.yaml`, then prod
3. On each sync, a new approval `Job` (named after the version) is created and approves the matching `InstallPlan`
4. Completed Jobs are auto-deleted after 10 minutes (`ttlSecondsAfterFinished`)

OLM does **not** support rollback. Reverting `version` in Git will not downgrade the operator.

## Values

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `<key>` | yes | | OLM package name; used as the Subscription name |
| `namespace` | yes | | Namespace for the Subscription and OperatorGroup |
| `channel` | yes | | OLM update channel (e.g. `stable`, `stable-v2`) |
| `source` | yes | | CatalogSource name (e.g. `redhat-operators`) |
| `sourceNamespace` | no | `openshift-marketplace` | Namespace of the CatalogSource |
| `installPlanApproval` | no | `Automatic` | `Automatic` or `Manual` |
| `version` | no | | Full CSV name to pin (e.g. `cluster-logging.v6.3.0`). Required when using `Manual` approval. |
| `scope` | no | `Cluster` | `Cluster` or `Namespaced` |
| `targetNamespaces` | no | `[namespace]` | Watched namespaces when `scope: Namespaced` |
| `createNamespace` | no | `false` | Create a Namespace resource |
| `createOperatorGroup` | no | `false` | Create an OperatorGroup. Do not set for `openshift-operators` — it ships with one already. |
| `config` | no | | `SubscriptionConfig` block passed verbatim to `spec.config` |
| `kubectlImage` | — | `registry.redhat.io/openshift4/ose-cli-rhel9:v4.21` | Image used by the InstallPlan approval Job. Override for non-OpenShift clusters or internal mirrors. |

## Examples

### Cluster-wide operator (automatic upgrades)

```yaml
operators:
  elasticsearch-operator:
    namespace: openshift-operators
    channel: stable
    source: redhat-operators
```

### Namespace-scoped operator with manual approval

```yaml
operators:
  cluster-logging:
    namespace: openshift-logging
    channel: stable
    source: redhat-operators
    installPlanApproval: Manual
    version: cluster-logging.v6.3.0
    createNamespace: true
    createOperatorGroup: true
```

### Manual approval with pinned version and custom config

```yaml
operators:
  advanced-cluster-management:
    namespace: open-cluster-management
    channel: release-2.10
    source: redhat-operators
    installPlanApproval: Manual
    version: advanced-cluster-management.v2.10.0
    createNamespace: true
    createOperatorGroup: true
    config:
      resources:
        requests:
          memory: 64Mi
          cpu: 250m
        limits:
          memory: 128Mi
```
