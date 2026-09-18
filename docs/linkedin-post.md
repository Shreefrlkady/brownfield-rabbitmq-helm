# LinkedIn post

Draft accompanying this repo. Use the copy button on the block below — it is 2,992 characters, just under the 3,000 limit.

Attach both images, diagram first: `docs/adoption.png`, then `docs/card.png`.

```text
How do you put a running RabbitMQ cluster under Helm without restarting it?

The setup: RabbitMQ on two GKE clusters, installed years ago the imperative way. kubectl apply on one side. A backup restore on the other. No chart, no git history, no source of truth.

"Just use the Bitnami chart" doesn't work here. Third-party charts for these operators exist, and they're fine — for a fresh install. They deploy their operator, their way. None of them can render what is already running on your cluster, byte for byte.

That was the entire requirement. So I wrote my own. Three charts, with the live cluster as the only spec.

But step one wasn't writing YAML. It was forensics.

Kubernetes remembers how you installed things.

Objects applied with kubectl carry a last-applied-configuration annotation — a verbatim copy of the manifest someone ran. Objects restored from a backup carry none of that. They carry matched backup/restore UID labels instead.

Two different installation stories, both readable from the live cluster with read-only kubectl get.

That audit became the spec — not the old local files, which were stale and out of sync with what was actually running.

Everything got templated by hand from that evidence. Nothing forked, no subchart wrapped. One toggle reproduces two different upstream manifest layouts, because the clusters had drifted a version apart and both had to render byte-exact. The CRDs are the one thing vendored verbatim — they're API definitions, and they ship as their own chart because Helm won't upgrade CRDs bundled as a subchart.

Then the part that changed how I think about charts.

helm upgrade --install --take-ownership adopts existing objects into a release instead of recreating them. But adoption is only safe if your chart renders exactly what is already live. Byte for byte.

Which means the chart ships config that looks like garbage:

→ an empty resources block that overrides nothing
→ a cloud annotation that does nothing on that service type
→ an inert load balancer setting

Every one of those is live somewhere. Delete them and the render changes. The operator reconciles the StatefulSet. Brokers restart.

A "cleanup" becomes an outage.

So the rule for the whole repo became one line: the first apply must be a no-op. Render, diff against live, classify every line before touching anything.

The audit also surfaced what nobody sees until the config gets written down: gaps. A 3-node quorum running pause_minority with no PodDisruptionBudget. One node drain can evict two of three pods and pause the cluster.

The chart got a PDB template. Shipped disabled by default. Because adopting infrastructure and silently changing its behaviour are two different pull requests.

Writing a chart for something new is easy.
Writing a chart that proves it changes nothing is the actual engineering.

Ever adopted live infra into IaC? Curious whether you went adopt-in-place or rebuild-and-cut-over.

#Kubernetes #Helm #RabbitMQ #DevOps #GitOps
```

## First comment

Post the repo link as your own first comment rather than in the body — LinkedIn suppresses reach on posts with external links, and the body has no room left.

```text
Charts and the full write-up — the audit method, the zero-diff rules, and what was deliberately left out:
github.com/Shreefrlkady/brownfield-rabbitmq-helm
```
