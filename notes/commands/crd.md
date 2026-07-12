# Custom Resources (CRDs)

A CRD registers a new resource type with the API server. The **resource** (data in etcd) and its **controller** (acts on it) are separate; an **operator** packages both. Only CRD usage is CKAD-examinable — controllers/operators are conceptual.

## Create & use

```bash
k create -f flightticket.yml                     # BEFORE the CRD: "no matches for kind FlightTicket"
k create -f flightticket-custom-definition.yml   # register the CRD (defines the type)
k create -f flightticket.yml                      # now the custom resource instance is accepted
```

## Discover & inspect

```bash
k api-resources | grep flight            # confirm it registered (group, namespaced, kind)
k get flightticket                        # list by singular name
k get ft                                  # by shortName (if defined)
k get flighttickets                       # plural also works
k delete -f flightticket.yml
```

(`k get crd` lists CRDs themselves; the notes discover via `api-resources` + `get <kind>`.)

## Operators (install as one unit)

```bash
k create -f flight-operator.yaml          # bundles CRDs + controller Deployment + SA/Role/RoleBinding
```

Operators are commonly installed via Helm (e.g. `helm install cert-manager jetstack/cert-manager`) or OLM/OperatorHub.

## See also

- `09-security/12-custom-resource-definitions.md`, `13-custom-controllers.md`, `14-operator-framework.md`
- `authorization.md` — `api-resources` / `api-versions` discovery
