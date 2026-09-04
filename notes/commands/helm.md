# Helm

Helm is a package manager: a **chart** (templated manifests) + **values** (config) → a **release** (an installed instance). One chart can back many independent releases. Value precedence: `--set` > `-f values.yaml` > chart default `values.yaml`. Releases are namespaced: `-n <ns>` on every command.

## Repos & search

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                          # refresh index (after add, before "newest version")
helm repo list
helm repo remove bitnami
helm search hub wordpress                 # Artifact Hub (public, no repo needed)
helm search repo wordpress                # only locally-added repos — newest version per chart
helm search repo wordpress --versions     # every version, newest first
helm show values bitnami/wordpress        # the chart's default values.yaml = what you can override
helm show values bitnami/wordpress --version 12.1.0
helm show chart bitnami/wordpress         # Chart.yaml (version vs appVersion)
```

## Install

```bash
helm install <release> <chart> -n <ns>                              # e.g. helm install wp bitnami/wordpress
helm install <release> <chart> -n <ns> --create-namespace           # helm does not create Namespaces otherwise
helm install <release> <chart> --version 12.1.0                     # CHART version (omit = newest)
helm install <release> <chart> --set key=value --set replicaCount=5
helm install <release> <chart> --set 'env[0].name=X,env[0].value=Y' # list entries
helm install <release> <chart> --set-string image.tag=1.27          # keep "1.27" a string
helm install <release> <chart> -f my-values.yaml
helm install <release> ./local-chart                                # from a directory or .tgz
helm install <release> <chart> --dry-run                            # preview, no release
helm install <release> <chart> --wait --timeout 60s                 # block until Pods ready
```

## Upgrade / rollback / uninstall

```bash
helm upgrade <release> <chart> -n <ns>                        # newest chart version, chart-default values + what you pass NOW
helm upgrade <release> <chart> --reuse-values --set k=v      # keep previous values, override one — without --reuse-values old --set values are lost
helm upgrade <release> <chart> -f values.yaml                # re-read an edited values file
helm upgrade --install <release> <chart>                     # install if missing
helm history <release> -n <ns>                               # revisions (for rollback targets)
helm rollback <release> <revision> -n <ns>                   # e.g. helm rollback wp 1
helm rollback <release> -n <ns>                              # no revision = one back
                                                             # a rollback CREATES a new revision; it does not rewind the counter
helm uninstall <release> -n <ns>                             # removes all objects in the release
helm uninstall <release> -n <ns> --keep-history
```

## Inspect

```bash
helm list -n <ns>                         # deployed + failed releases in one namespace
helm list -A                              # all namespaces (--all-namespaces)
helm list -a                              # all statuses: pending-install/upgrade/rollback, uninstalled (--keep-history)
helm list -A -a                           # everything, everywhere — "find the broken release"
helm list --pending  /  --failed  /  --uninstalled
helm status <release> -n <ns>
helm get values <release> -n <ns>         # user-supplied values (what it was installed with)
helm get values <release> --all           # merged with chart defaults
helm get values <release> --revision 3
helm get manifest <release> -n <ns>       # rendered YAML Helm applied
kubectl get secret -n <ns>                # sh.helm.release.v1.<release>.v<rev> = where Helm keeps release state
```

`deployed` means the API server accepted the manifests, not that Pods are healthy — `kubectl get pod` decides which revision "worked".

## Render locally / fetch (no install)

```bash
helm template <release> <chart> -n <ns> -f values.yaml --set k=v > out.yaml   # clean YAML, no release
kubectl apply -n <ns> -f out.yaml          # -n needed again: most charts render no metadata.namespace
helm template <release> <chart> --debug
helm pull <chart>                          # download the .tgz
helm pull <chart> --version 0.6.0 --untar -d ./charts   # download + extract to ./charts/<chart>/
helm lint ./charts/<chart>
helm version
```

## Package a chart / publish a repo

```bash
helm package ./mychart -d ./repo                          # ./repo/mychart-1.0.0.tgz (a gzipped tar of the directory)
helm repo index ./repo --url https://example.com/repo     # writes ./repo/index.yaml — that plus the .tgz files IS a repo
tar -tzf ./repo/mychart-1.0.0.tgz                         # list the packaged files
helm push mychart-1.0.0.tgz oci://ghcr.io/<org>/charts    # OCI registry instead of an index-based repo
```

Hub (artifacthub.io) = search over registered repos, hosts nothing. Repo = `index.yaml` + `.tgz` on an HTTP server. Chart = the directory/tarball. Release = an installed copy.

## Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
sudo snap install helm --classic          # or: brew install helm / choco install kubernetes-helm
```

Helm 3 has no Tiller and no `helm init` (that was Helm 2 — ignore). The exam node has Helm installed.

## See also

- `10-helm-fund/01-helm-intro-and-installation.md`, `02-helm-concepts.md`
- `10-helm-fund/03-helm-exam-patterns.md` — task patterns and the traps (`--reuse-values`, rollback revisions, `ls -a`)
- `kustomize.md` — the template-free alternative
