# helm

Helm charts deployed to my personal Kubernetes cluster.

Each top-level directory is an *umbrella chart*: a thin local chart whose
`Chart.yaml` declares the real upstream chart as a dependency, with all
configuration living in the sibling `values.yaml`. Nothing upstream is vendored
or forked, so Renovate can bump the pinned dependency version in `Chart.yaml`
and open a PR, and any local additions (extra ConfigMaps, dashboards, secrets)
can be dropped into `templates/` without diverging from upstream.

## Charts

| Chart | Upstream | Namespace | Purpose |
| --- | --- | --- | --- |
| `metrics-server` | [metrics-server](https://github.com/kubernetes-sigs/metrics-server) | `kube-system` | Resource metrics API backing `kubectl top` and the HorizontalPodAutoscaler |
| `external-dns` | [external-dns](https://github.com/kubernetes-sigs/external-dns) | `external-dns` | Syncs Ingress and Service hostnames into Cloudflare DNS |

## Prerequisites

- [Helm](https://helm.sh/) 3.14 or newer (the workflows pin v4)
- [Bun](https://bun.sh/) for the lint tooling and git hooks
- `kubectl` with a context pointing at the target cluster

```sh
bun install
```

## Local workflow

```sh
bun run helm:deps      # fetch upstream subcharts into <chart>/charts/
bun run helm:lint      # lint every chart
bun run helm:template  # render every chart, catching template errors
bun run check          # biome + helm lint, same gate as the pre-commit hook
```

`bun install` wires up husky, which runs `bun run check` before every commit and
validates the commit message against
[Conventional Commits](https://www.conventionalcommits.org/).

## Deploying

Each chart has a manually-triggered workflow under `.github/workflows/`. Run one
from the Actions tab and it will `helm upgrade --install` the chart against the
cluster, then roll back automatically if the release fails to become ready.

Deploys authenticate with a base64-encoded kubeconfig held in the `KUBECONFIG`
repository secret:

```sh
base64 -i ~/.kube/config | pbcopy
```

Paste that into **Settings → Secrets and variables → Actions → New repository
secret**, named `KUBECONFIG`. Prefer a kubeconfig for a dedicated deploy
ServiceAccount over your personal admin credentials.

To deploy by hand instead:

```sh
helm dependency update ./metrics-server
helm upgrade --install metrics-server ./metrics-server \
  --namespace kube-system \
  --values ./metrics-server/values.yaml \
  --wait
```

## Cloudflare credentials for external-dns

external-dns needs a Cloudflare API token with `Zone:Read` and `DNS:Edit` on the
zones it manages. Create the secret it reads before the first deploy:

```sh
kubectl create namespace external-dns
kubectl create secret generic external-dns-cloudflare \
  --namespace external-dns \
  --from-literal=apiToken=<token>
```

Then set `external-dns.domainFilters` in `external-dns/values.yaml` to the zones
you actually own, so external-dns never touches records outside them.

## Dependency updates

[Renovate](https://docs.renovatebot.com/) watches `Chart.yaml` dependency
versions, pinned image tags in `values.yaml`, and the GitHub Actions used by the
workflows. Configuration lives in `renovate.json`.

Renovate runs as the Mend-hosted GitHub App, so it must be installed on this
repository from <https://github.com/apps/renovate> before any PRs appear. Once
installed it opens a "Dependency Dashboard" issue listing everything it is
tracking.

Chart bumps arrive as PRs against `main`. The lint workflow re-resolves
dependencies and lints the chart on each PR, so a bump that breaks rendering
fails before merge. Merging a bump does not deploy it — run the chart's deploy
workflow afterwards.

## Adding a chart

1. Create `<name>/Chart.yaml` declaring the upstream chart under `dependencies`,
   with an exact pinned version.
2. Add `<name>/values.yaml`, keyed by the upstream chart's name.
3. Add `<name>/templates/NOTES.txt` describing what was installed.
4. Register the chart in the `helm:deps`, `helm:lint`, and `helm:template`
   scripts in `package.json`, and in the lint workflow's dependency step.
5. Copy an existing deploy workflow, adjusting the `env` block at the top.
