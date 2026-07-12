# Helm

Helm is a package manager: a **chart** (templated manifests) + **values** (config) → a **release** (an installed instance). One chart can back many independent releases. Value precedence: `--set` > `-f values.yaml` > chart default `values.yaml`.

## Repos & search

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                          # refresh index (do after add / before install)
helm repo list
helm repo remove bitnami
helm search hub wordpress                 # Artifact Hub (public, no repo needed)
helm search repo wordpress                # only locally-added repos
```

## Release lifecycle

```bash
helm install <release> <chart>                        # e.g. helm install wordpress bitnami/wordpress
helm install <release> <chart> --set key=value --set replicaCount=5
helm install <release> <chart> -f my-values.yaml
helm install <release> ./local-chart                  # from a local directory
helm upgrade <release> <chart> --set replicaCount=5
helm rollback <release> <revision>                     # e.g. helm rollback wordpress 1
helm uninstall <release>                               # removes all objects in the release
```

## Inspect

```bash
helm list                                 # releases in current namespace
helm list -A                              # all namespaces (--all-namespaces)
helm status <release>
helm history <release>                    # revisions (for rollback targets)
helm show values <chart>                  # print the chart's default values.yaml (what's overridable)
helm show chart <chart>                   # print Chart.yaml
```

## Render locally / fetch (no install)

```bash
helm template <release> <chart>           # render manifests to stdout — no cluster changes
helm template <release> <chart> --debug
helm install <release> <chart> --dry-run  # preview an install
helm pull <chart>                         # download the .tgz
helm pull --untar <chart>                 # download + extract to ./<chart>/
helm version
```

## Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
sudo snap install helm --classic          # or: brew install helm / choco install kubernetes-helm
```

Helm 3 has no Tiller and no `helm init` (that was Helm 2 — ignore).

## See also

- `10-helm-fund/01-helm-intro-and-installation.md`, `02-helm-concepts.md`
- `kustomize.md` — the template-free alternative
