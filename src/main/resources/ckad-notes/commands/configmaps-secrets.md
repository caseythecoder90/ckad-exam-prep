# ConfigMaps & Secrets

## ConfigMap

```bash
k create configmap app-config --from-literal=ENV=prod $do > cm.yaml
k create configmap app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=debug $do > cm.yaml
k create configmap app-config --from-file=config.properties $do > cm.yaml
k create configmap app-config --from-file=app.conf=./local-file.conf $do > cm.yaml   # rename key
k create configmap app-config --from-env-file=app.env $do > cm.yaml                   # one key per line in file
```

## Secret — generic

Values are base64-encoded for you when you use `--from-literal` / `--from-file`.

```bash
k create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t $do > secret.yaml

k create secret generic tls-files --from-file=cert.pem --from-file=key.pem $do > secret.yaml
```

## Secret — TLS shorthand

```bash
k create secret tls my-tls --cert=path/to/cert --key=path/to/key $do > secret.yaml
```

## Secret — image pull (Docker registry)

```bash
k create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=alice \
  --docker-password=p4ss \
  --docker-email=a@b.c
```

## Consume in a pod

Three shapes; recall the fields with `explain` instead of memorizing them:

```bash
k explain pod.spec.containers.envFrom          # whole map/secret -> configMapRef / secretRef {name}
k explain pod.spec.containers.env.valueFrom    # one key          -> configMapKeyRef / secretKeyRef {name, key}
k explain pod.spec.volumes.configMap           # as files         -> configMap {name}
k explain pod.spec.volumes.secret              # as files         -> secret {secretName}  <- secretName, not name
```

Bulk env injection has an imperative shortcut (no YAML editing):

```bash
k set env deployment/web --from=configmap/app-config     # all keys as env vars
k set env deployment/web --from=secret/db-creds
k set env deployment/web --from=configmap/app-config --prefix=CFG_
```

Single-key `valueFrom` and volume mounts have no imperative flag — generate the
pod (`k run ... $do > pod.yaml`) and add the block. Docs (open on the exam):
"Configure a Pod to Use a ConfigMap" / "Managing Secrets using kubectl".

## Inspect

```bash
k get cm
k get cm app-config -o yaml
k describe cm app-config

k get secret
k get secret db-creds -o yaml
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d    # decode one value
```

## Edit

```bash
k edit cm app-config
k edit secret db-creds       # values shown base64-encoded — encode replacements before saving
```

## Delete

```bash
k delete cm app-config
k delete secret db-creds
```

## Gotchas

- A ConfigMap or Secret update is **not** automatically picked up by pods that mounted it as env vars (env is evaluated at pod start). Volume-mounted ConfigMaps/Secrets *do* update, but with kubelet sync delay.
- Forcing a refresh: `k rollout restart deployment/<name>`.
- Don't put real secrets in `--from-literal` on a shared machine — the value lands in shell history.

## See also

- `imperative.md` — full create-secret/create-configmap variants
- `deployments.md` — using `k set env --from=configmap/...`
