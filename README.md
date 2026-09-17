# brownfield-rabbitmq-helm

Helm charts for a RabbitMQ estate that was **already running** before the charts existed —
installed imperatively, with no chart and no manifest history behind it.

The interesting constraint is not "deploy RabbitMQ." It is: bring live, load-bearing
infrastructure under Helm and GitOps **without recreating a single object**, on brokers that
cannot be restarted at will.

> **These are adoption charts, not fresh-install charts.** If you want RabbitMQ on a new
> cluster, use the [Bitnami chart][bitnami] — it is good and it is maintained. Reach for this
> pattern when something is already running, you need it in git, and a restart is not on the
> table.

[bitnami]: https://github.com/bitnami/charts/tree/main/bitnami/rabbitmq-cluster-operator

## The problem

Two GKE clusters, each running two RabbitMQ clusters under the RabbitMQ Cluster Operator. One
environment had been installed with `kubectl apply -k`; the other had been materialised by a
backup restore. Neither had a chart, a release, or a trustworthy copy of the manifests that
produced it. Local source files existed but had drifted — hand-edited and re-applied over time,
never kept in sync with what was actually pushed.

Upstream ships these operators as raw YAML release artifacts rather than a chart of their own,
and third-party charts install *their* operator, *their* way, with *their* object names. None of
them can render what is already on the cluster. So the charts had to be written.

## Step one: establish provenance, not YAML

You cannot write a zero-diff chart against infrastructure whose installation method you are
guessing at. Kubernetes records more than people expect:

| Evidence on a live object | What it proves |
|---|---|
| `kubectl.kubernetes.io/last-applied-configuration` | Applied with `kubectl apply` — and the annotation *is* a verbatim copy of the manifest that was applied |
| `app.kubernetes.io/managed-by: kustomize` | Rendered through `kustomize build` before being applied |
| Matched backup/restore UID labels, no `last-applied-configuration` | Never applied here at all — materialised by a restore controller from a snapshot |
| No `helm.sh/release.v1` Secret in the namespace | Helm was never involved, whatever the directory names suggest |
| Container image tags and digests | The exact operator versions in play, without guessing |

That read-only survey — `kubectl get` only, no writes — became the specification the charts were
written against. **Live cluster state is the source of truth; older local files are a lead, not a
spec.**

## The charts

Install in this order. Each depends on the one before it.

| # | Chart | What it contains |
|---|-------|------------------|
| 1 | `rabbitmq-crds` | The `rabbitmq.com` CRDs, vendored verbatim. Separate chart on purpose — Helm never upgrades CRDs bundled as a subchart dependency. |
| 2 | `rabbitmq-cluster-operator` | cluster-operator + messaging-topology-operator, templated from the upstream release manifests. Excludes cert-manager objects by design. |
| 3 | `rabbitmq-messaging` | `RabbitmqCluster` CRs, the hand-authored internal proxy LoadBalancer, Shovel/Queue/Exchange topology, an optional PDB, and the shovel-URI ExternalSecrets. |

Per-environment values live beside each chart as `values-dev.yaml` / `values-prod.yaml`.
`rabbitmq-crds` has none — it vendors the CRDs verbatim and has nothing to configure.

### One toggle, two upstream layouts

The topology operator's 1.19.0 release restructured its manifest: object names gained a
`messaging-topology-` prefix, a metrics service, cert and metrics-auth RBAC appeared, and the
container gained `--leader-elect` / `--health-probe` args plus probes.

The two environments here sit on either side of that jump, and **both** have to render exactly
what is running on them. Rather than fork the chart, one value switches the whole layout:

```yaml
topologyOperator:
  naming:
    legacy: true   # pre-1.19.0 layout: bare names, no metrics objects, no args or probes
    # legacy: false  -> the 1.19.0 layout
```

## The prime directive: zero-diff

These charts wrap live infrastructure. **The first apply must be a no-op.**

That inverts a normal instinct. Several values exist *only* to reproduce something already on the
cluster, and they look like dead weight:

- an empty `resources: {}` inside a `RabbitmqCluster` override — it overrides nothing
- a `cloud.google.com/neg` annotation on a Service that no Ingress references and that has no NEGs
- a `cloud.google.com/load-balancer-type: Internal` annotation on a **ClusterIP** Service, where
  it has no effect at all

Delete any of them and the render changes. The cluster-operator reconciles the StatefulSet.
Brokers restart. The cleanup *is* the outage. Each one is marked `# Fidelity:` in the values
files, with the reason.

So: render, diff against live, and classify every line before applying anything.

```bash
helm template rabbitmq-messaging rabbitmq-messaging -n messaging \
  -f rabbitmq-messaging/values-dev.yaml | kubectl diff -f -
```

## Adopting a running estate

Order matters, and the lower environment goes first and soaks.

```bash
helm upgrade --install rabbitmq-crds rabbitmq-crds \
  -n rabbitmq-system --take-ownership

helm upgrade --install rabbitmq-operator rabbitmq-cluster-operator \
  -n rabbitmq-system -f rabbitmq-cluster-operator/values-dev.yaml --take-ownership

helm upgrade --install rabbitmq-messaging rabbitmq-messaging \
  -n messaging -f rabbitmq-messaging/values-dev.yaml --take-ownership
```

`--take-ownership` is what makes this adoption rather than installation: Helm takes over
existing objects instead of colliding with them. It is only safe because of the zero-diff work
above — ownership transfers, the rendered spec matches, and nothing reconciles.

> **Never `helm uninstall` these releases.** They wrap live RabbitMQ; uninstalling deletes it.
> To hand a release over to another controller, delete its `sh.helm.release.v1.*` Secret
> instead — that drops Helm's ownership record without touching the workload.

### Secrets: adopt those too

Each shovel's source/destination URIs come from Vault through external-secrets, one
`ExternalSecret` per live `*-shovel-uris` Secret, all reading one KV v2 path per environment.

The ordering constraint is easy to get wrong and expensive to get wrong:

1. Load the Vault path **before** installing, with values **identical** to the live Secret.
2. `creationPolicy: Owner` then adopts the existing Secret. Identical content makes adoption a
   no-op and the shovels never notice.
3. A failed lookup *after* adoption empties the Secret instead — which stops the shovels.

Note the Vault path carries no `secret/data/` prefix: the ClusterSecretStore already mounts at
`secret` with KV v2, so keys are relative to the mount.

## Deliberate omissions

| Not here | Why |
|---|---|
| cert-manager `Issuer` / `Certificate` | cert-manager is managed outside these charts. The topology-operator webhook's `caBundle` stays externally injected via `cert-manager.io/inject-ca-from`. |
| `PodDisruptionBudget`, enabled | The template exists; it ships `pdb.enabled: false`. See below. |
| Queue / Exchange CRs in production | `topology.declareQueues: false` there. See below. |
| Publishing / CI plumbing | Registry-specific and not portable. The one lesson worth carrying: treat chart publishes as **immutable**. Bump `version:` in `Chart.yaml` rather than overwriting a version that is already published — re-publishing the same version can change what a cluster renders with no git-visible signal. |

**On the PDB.** A multi-replica RabbitMQ running `cluster_partition_handling = pause_minority`
with no PodDisruptionBudget can be taken down by an ordinary node drain: nothing prevents
evicting a majority of pods at once, and the surviving minority pauses itself. That is worth
fixing — but adopting a chart must not silently change disruption behaviour on a running system.
The template is there; enabling it is a separate, reviewed change.

**On queue declaration.** Where an estate's queues were created imperatively — a migration
script driving the RabbitMQ HTTP API, or shovels declaring their own destination queues on first
message — there are no Queue/Exchange CRs behind them. Adopting hundreds of unreviewed queue
names into CRs is a decision of its own, not a side effect of installing a chart. Hence the
per-environment gate.

## Licensing

The CRDs under `rabbitmq-crds/templates/` are vendored verbatim from the upstream RabbitMQ
operators, both MPL-2.0. See [`rabbitmq-crds/NOTICE`](rabbitmq-crds/NOTICE).

## Verify locally

```bash
for chart in rabbitmq-crds rabbitmq-cluster-operator rabbitmq-messaging; do
  helm lint "$chart"
  for env in dev prod; do
    [ -f "$chart/values-$env.yaml" ] || continue
    helm template "$chart" "$chart" -f "$chart/values-$env.yaml" >/dev/null
  done
done
```
