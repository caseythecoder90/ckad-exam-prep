---
section: 09-security
chapter: "09"
title: "Validating and Mutating Admission Controllers (Webhooks)"
course: "Mumshad Mannambeth – KodeKloud/Udemy CKAD"
examinable: true
companion_diagrams:
  - diagrams/09-admission-webhook-flow.png
related:
  - 08-admission-controllers.md   # built-in controllers, pipeline overview
  - 05-authorization.md
cka_cks_note: |
  CKAD: understand what webhooks are and the two-step setup (deploy server +
  create WebhookConfiguration). CKA: less focus here. CKS: deep — OPA
  Gatekeeper, image policy, Pod Security Admission, failure policies.
---

# 09 – Validating and Mutating Admission Controllers (Webhooks)

![Admission webhook request/response flow](diagrams/09-admission-webhook-flow.png)

---

## 1. The two admission controller types, precisely defined

The previous chapter introduced this distinction. Here it gets concrete:

**Mutating admission controller** – receives the object, can modify it, and
passes a (potentially changed) version downstream. The object that reaches
etcd may be different from what the user submitted.

Example: `DefaultStorageClass`. When a PVC arrives without a
`storageClassName`, this controller injects the cluster-default StorageClass
into the PVC spec before it's persisted. The user never specified it; the
controller added it.

**Validating admission controller** – receives the object (already post-
mutation), inspects it, and returns allow or deny. It cannot change the
object.

Example: `NamespaceLifecycle`. It checks whether the target namespace exists
and rejects the request if not. No modification — pure allow/deny.

**Some controllers do both** – they mutate first (as a mutating controller)
and then validate the result (as a validating controller). These are registered
for both phases.

### Execution order — why it matters

```
Mutating controllers  →  Validating controllers  →  etcd
```

Mutating runs first so that validators see the **final form** of the object.
If validators ran on the pre-mutated object, a `DefaultStorageClass` injection
would never be visible to a quota validator that checks storage class limits.
The ordering is not configurable — it's enforced by the apiserver pipeline.

---

## 2. The extensibility problem the webhooks solve

All built-in admission controllers are compiled into kube-apiserver. You can
enable or disable them, but you can't add new logic without modifying
Kubernetes source code and rebuilding.

Two built-in controllers solve this:

| Controller | Purpose |
|---|---|
| `MutatingAdmissionWebhook` | Calls out to an external HTTP server during the mutating phase |
| `ValidatingAdmissionWebhook` | Calls out to an external HTTP server during the validating phase |

These are generic call-out mechanisms. The external server contains your
custom logic. Kubernetes just handles the HTTP call and interprets the
response. The result is a clean extension point without forking k8s.

---

## 3. How the webhook call works

When a request reaches the webhook controller, it serializes the incoming
object into an **AdmissionReview** JSON object and POSTs it to your server.

### AdmissionReview request (what your server receives)

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "subResource": "scale",
    "requestKind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "requestResource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "userInfo": {
      "username": "admin",
      "uid": "014fbff9-a0af-11e8-a8c9-42010af003bb",
      "groups": ["system:authenticated", "system:masters"]
    },
    "object": { ... },      // the object being created/modified
    "oldObject": null,      // previous version (for UPDATE operations)
    "dryRun": false
  }
}
```

Key fields your webhook will read:
- `request.uid` – echo this back verbatim in your response
- `request.object` – the full object spec (pod, deployment, etc.)
- `request.userInfo` – who made the request
- `request.operation` – CREATE, UPDATE, DELETE, CONNECT

### AdmissionReview response (what your server returns)

**Validating response:**
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value_from_request.uid>",   // must match
    "allowed": true
  }
}
```

For a denial:
```json
{
  "response": {
    "uid": "<value_from_request.uid>",
    "allowed": false,
    "status": {
      "code": 403,
      "message": "You can't create objects with your own name"
    }
  }
}
```

**Mutating response** – same envelope but adds a JSONPatch:
```json
{
  "response": {
    "uid": "<value_from_request.uid>",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "<base64-encoded JSON patch>"
  }
}
```

The patch field is a base64-encoded array of [RFC 6902 JSON Patch](https://jsonpatch.com/)
operations. For example, to add a label `users: alice` to the object's metadata:

```python
patch = [{"op": "add", "path": "/metadata/labels/users", "value": user_name}]
# Encode as base64 before sending
"patch": base64.b64encode(json.dumps(patch).encode()).decode()
```

A mutating webhook that always allows but injects a label looks like this
(simplified Python/Flask):

```python
@app.route("/mutate", methods=["POST"])
def mutate():
    user_name = request.json["request"]["userInfo"]["name"]
    patch = [{"op": "add", "path": "/metadata/labels/users", "value": user_name}]
    return jsonify({
        "response": {
            "allowed": True,
            "uid": request.json["request"]["uid"],
            "patch": base64.b64encode(json.dumps(patch).encode()).decode(),
            "patchType": "JSONPatch",
        }
    })

@app.route("/validate", methods=["POST"])
def validate():
    object_name = request.json["request"]["object"]["metadata"]["name"]
    user_name   = request.json["request"]["userInfo"]["name"]
    allowed = object_name != user_name
    message = "You can't create objects with your own name" if not allowed else ""
    return jsonify({
        "response": {
            "allowed": allowed,
            "uid": request.json["request"]["uid"],
            "status": {"message": message},
        }
    })
```

The reference implementation from the Kubernetes repo is in Go:
`https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go`

> **Java note**: the webhook server is just an HTTP service that accepts
> JSON POST requests and returns JSON. Spring Boot + Jackson is a
> natural fit — define a `AdmissionReview` POJO, deserialize the incoming
> request, apply your logic, serialize the response. The k8s Go reference
> is for structure; your Spring Boot implementation would use the same JSON
> contract.

---

## 4. Two-step deployment

### Step 1 – Deploy the webhook server

The server runs anywhere that kube-apiserver can reach over HTTPS:

- **Inside the cluster** (preferred): deploy as a standard Deployment + Service.
  Easier to manage TLS with a cluster CA.
- **Outside the cluster**: deploy anywhere accessible, provide its URL in the
  webhook configuration.

TLS is mandatory — kube-apiserver will not call a plain HTTP endpoint.
Your server needs a TLS certificate, and the `caBundle` field in the
WebhookConfiguration must contain the CA that signed it (so apiserver can
verify the server).

### Step 2 – Create the WebhookConfiguration object

This is a cluster-scoped Kubernetes resource that tells the apiserver where
to call and for what operations.

**ValidatingWebhookConfiguration:**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
  - name: "pod-policy.example.com"
    clientConfig:
      # Option A: in-cluster service
      service:
        namespace: "webhook-namespace"
        name: "webhook-service"
      # Option B: external URL
      # url: "https://my-webhook.example.com/validate"
      caBundle: "Ci0tLS0t......tLS0K"   # base64 CA cert
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]             # only fires on pod creation
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail                 # Fail or Ignore — see gotchas
```

**MutatingWebhookConfiguration** uses the same structure but `kind:
MutatingWebhookConfiguration`. The server endpoints are typically different
paths on the same server (`/validate` vs `/mutate`).

### rules field — controlling when the webhook fires

The `rules` array is how you scope the webhook to only the operations and
resources you care about:

```yaml
rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
```

This is important for performance: you don't want a heavy external call on
every single API operation. Scope it tightly.

---

## 5. Full deployment picture

```
kube-apiserver
    │
    ├─ AuthN → AuthZ → Built-in admission controllers
    │
    ├─ MutatingAdmissionWebhook
    │       │  POST AdmissionReview (JSON)
    │       ▼
    │  Webhook Server (/mutate)
    │       │  returns AdmissionReview + JSONPatch
    │       ▼
    │  [object mutated in memory]
    │
    ├─ ValidatingAdmissionWebhook
    │       │  POST AdmissionReview (JSON)
    │       ▼
    │  Webhook Server (/validate)
    │       │  returns allowed: true/false
    │       ▼
    │  [rejected or allowed]
    │
    └─ etcd (if allowed)
```

The webhook server in-cluster topology:

```
Kubernetes cluster
┌────────────────────────────────────────────┐
│  webhook-namespace                         │
│  ┌──────────────────────┐                  │
│  │  webhook-deployment  │                  │
│  │  (Admission Webhook  │◄─────────────────┤── kube-apiserver
│  │   Server pod)        │                  │
│  └──────────┬───────────┘                  │
│             │                              │
│  webhook-service (ClusterIP)               │
└────────────────────────────────────────────┘
```

---

## 6. The Java angle — writing a webhook server in Spring Boot

Since you asked: here's the structure you'd use. The JSON contract is all
that matters; the language is irrelevant to Kubernetes.

```java
// AdmissionReview POJO (simplified)
@Data
public class AdmissionReview {
    private String apiVersion = "admission.k8s.io/v1";
    private String kind = "AdmissionReview";
    private AdmissionRequest request;
    private AdmissionResponse response;
}

@Data
public class AdmissionResponse {
    private String uid;       // must echo request.uid
    private boolean allowed;
    private String patchType; // "JSONPatch" for mutations
    private String patch;     // base64-encoded JSON patch array
    private Status status;
}

// Controller
@RestController
public class WebhookController {

    @PostMapping("/validate")
    public AdmissionReview validate(@RequestBody AdmissionReview review) {
        String objectName = extractName(review.getRequest());
        String userName   = review.getRequest().getUserInfo().getName();
        boolean allowed   = !objectName.equals(userName);

        AdmissionResponse resp = new AdmissionResponse();
        resp.setUid(review.getRequest().getUid());
        resp.setAllowed(allowed);
        if (!allowed) {
            resp.setStatus(new Status(403, "Can't create object with your own name"));
        }

        AdmissionReview response = new AdmissionReview();
        response.setResponse(resp);
        return response;
    }

    @PostMapping("/mutate")
    public AdmissionReview mutate(@RequestBody AdmissionReview review) {
        String userName = review.getRequest().getUserInfo().getName();

        // JSON Patch: add /metadata/labels/users = userName
        String patchJson = String.format(
            "[{\"op\":\"add\",\"path\":\"/metadata/labels/users\",\"value\":\"%s\"}]",
            userName
        );
        String patchB64 = Base64.getEncoder()
            .encodeToString(patchJson.getBytes(StandardCharsets.UTF_8));

        AdmissionResponse resp = new AdmissionResponse();
        resp.setUid(review.getRequest().getUid());
        resp.setAllowed(true);
        resp.setPatchType("JSONPatch");
        resp.setPatch(patchB64);

        AdmissionReview response = new AdmissionReview();
        response.setResponse(resp);
        return response;
    }
}
```

TLS wiring for Spring Boot:
```yaml
# application.yaml
server:
  port: 8443
  ssl:
    key-store: /tls/keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
```

Kubernetes secret mount:
```yaml
volumes:
  - name: tls
    secret:
      secretName: webhook-tls-secret
volumeMounts:
  - name: tls
    mountPath: /tls
    readOnly: true
```

This is a meaningful bridge to the Go learning goal: the webhook contract is
identical regardless of server language. Implement it in Spring Boot first to
understand the flow, then read the Go reference implementation to compare.

---

## 7. Exam-pattern gotchas

**Gotcha 1 – `failurePolicy` is a security-critical field**
```yaml
failurePolicy: Fail    # if the webhook is unreachable, reject the request
failurePolicy: Ignore  # if the webhook is unreachable, allow the request
```
`Fail` is the safe default for security-sensitive webhooks. `Ignore` means
a downed webhook server silently allows everything through — dangerous for
image policy enforcement.

**Gotcha 2 – uid echo is mandatory**
The response `uid` must exactly match the request `uid`. Kubernetes uses this
to correlate requests to responses. A mismatch or missing uid causes the
admission to fail.

**Gotcha 3 – patch must be base64-encoded**
The `patch` field in a mutating response is base64 of the JSON Patch array,
not the raw JSON. Sending raw JSON fails silently or causes a decode error.

**Gotcha 4 – TLS is non-negotiable**
kube-apiserver refuses plain HTTP webhook endpoints. If your webhook server
cert is signed by a custom CA, that CA must appear in `caBundle`; otherwise
the apiserver rejects the TLS handshake.

**Gotcha 5 – mutating runs before validating (again)**
This is worth repeating because exam questions can test it indirectly.
A `ValidatingWebhookConfiguration` sees the object after any
`MutatingWebhookConfiguration` has already transformed it. Ordering within
the same phase (multiple mutating webhooks) is not guaranteed.

**Gotcha 6 – `admissionReviewVersions` and `sideEffects` are required fields**
In Kubernetes ≥ 1.16, both `admissionReviewVersions` and `sideEffects` must
be present on every webhook. Missing them causes the
`WebhookConfiguration` object itself to be rejected by the API. Use:
```yaml
admissionReviewVersions: ["v1"]
sideEffects: None
```

---

## 8. JPMC context

JPMC's platform almost certainly runs OPA Gatekeeper or a similar webhook-
based policy engine. Gatekeeper is implemented on top of
`ValidatingAdmissionWebhook` and exposes policy via CRDs (ConstraintTemplates
+ Constraints) rather than requiring you to write raw webhook server code.

When you try to deploy a pod with `ubuntu:latest` and get a rejection like
"image must be pulled from internal registry", you're hitting a
`ValidatingAdmissionWebhook` backed by something like Gatekeeper or an
in-house policy server.

The `AdmissionReview` object flowing between apiserver and that policy
server looks exactly like what's in this chapter. The internal policy server
is almost certainly doing the same inspect-and-allow/deny logic, just at
enterprise scale and written against JPMC's internal image registry allow-list.

When you write your Spring Boot webhook server: the TLS certificate mount
pattern above is exactly how your existing Spring Boot deployments handle
secrets at JPMC. The muscle memory already exists.

---

## 9. TL;DR

- **Mutating controllers** change the object; **validating controllers**
  allow/deny it. Mutating always runs first.
- Built-in controllers are compiled into kube-apiserver. To add custom
  logic you use `MutatingAdmissionWebhook` and/or
  `ValidatingAdmissionWebhook`.
- The webhook call is a JSON HTTP POST: kube-apiserver sends an
  `AdmissionReview` request; your server returns an `AdmissionReview`
  response with `allowed: true/false` and, for mutations, a base64
  JSONPatch.
- Setup is two steps: (1) deploy the webhook server (Deployment + Service
  inside the cluster, or an external URL), (2) create a
  `ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration`
  pointing to it.
- The `rules` field in the WebhookConfiguration controls which operations
  and resources trigger the call — scope it tightly.
- `failurePolicy: Fail` is the safe default for security webhooks.
- `uid` must be echoed; `patch` must be base64; TLS is required; both
  `admissionReviewVersions` and `sideEffects` are required fields.

---

## Open threads

- [ ] **OPA Gatekeeper** (CKS scope): the production-scale policy engine
  built on `ValidatingAdmissionWebhook`. Uses Rego policy language +
  CRDs. Worth flagging for CKS study; you likely interact with it at JPMC.
- [ ] **Pod Security Admission** (PSA): replaced PodSecurityPolicy in 1.25.
  Operates as a built-in admission controller (not a webhook) enforcing
  `restricted`, `baseline`, `privileged` profiles at namespace level. CKS
  territory.
- [ ] **JSONPatch (RFC 6902)**: four operations —  `add`, `remove`,
  `replace`, `move`. `path` follows JSON Pointer syntax (RFC 6901).
  Worth a quick read alongside the Go chapter of learning since the Go
  standard library handles this natively.

## Resolved threads (from ch08)

- [x] **Mutating vs validating phases and ordering**: fully covered in
  sections 1 and 3 of this chapter.
- [x] **MutatingAdmissionWebhook + ValidatingAdmissionWebhook mechanism**:
  covered end-to-end in sections 2–5.
