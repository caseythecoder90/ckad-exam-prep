# Deployment Updates and Rollbacks

> **Section:** 05-pod-design
> **Course chapter:** 2 (Deployment Updates and Rollbacks)
> **Why this is in CKAD:** Deployments are the most-used workload object and rollouts/rollbacks are heavily tested. You must be able to update an image, watch a rollout, read revision history, and roll back - and explain the difference between the Recreate and RollingUpdate strategies.
> **Companion files:** `01-labels-selectors-annotations.md` (a Deployment finds its ReplicaSet/pods by selector, and `change-cause` is an annotation); `../04-observability/03-logging.md` (the ReplicaSet hash in pod names comes from exactly this mechanism)

---

## 1. Rollouts and revisions (the part worth getting straight)

The instructor's wording - "a new rollout creates a new ReplicaSet, recorded as a revision" - only clicks once you see the object hierarchy:

```
Deployment  ->  ReplicaSet  ->  Pods
```

A **Deployment does not manage pods directly.** It manages **ReplicaSets**, and each ReplicaSet manages the pods. This indirection is the whole trick behind rollouts and rollbacks.

- **Creating** a Deployment triggers the first **rollout**, which creates **ReplicaSet #1**, which creates the pods. That state is **Revision 1**.
- **Upgrading** (changing the pod template - most often the container image, e.g. `nginx:1.7.0` -> `nginx:1.7.1`) triggers a **new rollout**. The Deployment does **not** edit the existing ReplicaSet. It creates a **brand-new ReplicaSet #2** for the new template, scales it up, and scales the old one down. That new state is **Revision 2**.

The key insight (this is what was unclear): **the old ReplicaSet is not deleted - it is kept around at 0 replicas.** That retained ReplicaSet *is* the revision. So "Revision 2" is literally ReplicaSet #2. Rollback is then just "scale the old ReplicaSet back up and the new one down" - no rebuild, no re-pull, because the old ReplicaSet (with its old template and image) is still sitting there.

![How a Deployment rolls out and rolls back via ReplicaSets](./diagrams/02-deployment-rollout-rollback.png)

This also explains the hash-suffixed pod names from your logging question: each ReplicaSet's name carries a template hash (`myapp-deployment-67c749c58c`), and its pods inherit it (`...-67c749c58c-x7k2p`). When you see two ReplicaSets, one at 5 replicas and one at 0, you are looking at the current revision and the previous one.

## 2. Deployment strategies: Recreate vs RollingUpdate

The instructor frames the core problem: you have N replicas on the old version and want them on the new version. Two strategies:

### Recreate

Destroy **all** old pods first, then create all new ones. Simple, but there is a window where **zero** pods are running - the application is **down** during the switch (images 4-5: all old pods go down, then all new come up, with "Application down" in between).

```yaml
spec:
  strategy:
    type: Recreate
```

- **Downtime: yes** (full outage during the swap).
- Use when: the app cannot run two versions simultaneously (e.g. an incompatible schema migration, or a singleton that can't have two instances).
- **Not the default.**

### RollingUpdate (the default)

Take down old pods **a few at a time** and bring up new ones in their place, so the app stays available throughout (image 6, bottom row: old and new pods coexist, swapping gradually). No downtime, and old + new versions run side by side during the transition.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%      # how many below desired you can dip (default 25%)
      maxSurge: 25%            # how many above desired you can temporarily add (default 25%)
```

- **Downtime: no** (rolling, always some pods serving).
- `maxUnavailable` / `maxSurge` control the pace and whether you trade capacity for speed.
- **This is the default** - if you write no `strategy:`, you get RollingUpdate.

Your work situation maps directly: switching a Deployment from `Recreate` to `RollingUpdate` means upgrades stopped causing a brief outage and instead roll pod-by-pod with the old version still serving until the new pods are ready. The trade-off you took on: during a rolling update, **both versions are live at once**, so the app (and anything it talks to, like a DB schema) must tolerate mixed versions. That is the one real gotcha of switching off Recreate.

You can confirm which strategy a Deployment uses:

```bash
kubectl describe deployment myapp-deployment   # StrategyType: RollingUpdate (and RollingUpdateStrategy: 25% max unavailable, 25% max surge)
```

## 3. How an update happens under the hood

When you change the template on a Deployment that wants 5 replicas:

1. The Deployment creates a **new ReplicaSet** for the new template.
2. It scales the new ReplicaSet **up** while scaling the old ReplicaSet **down**, a step at a time (RollingUpdate), respecting `maxSurge`/`maxUnavailable`.
3. When the new ReplicaSet is at 5 and the old is at 0, the rollout is complete. The old ReplicaSet stays at 0 as the previous revision.

`kubectl get replicasets` during/after a rollout shows this directly - two ReplicaSets, the new one ramping to 5, the old one draining to 0 (images 9/11).

## 4. How to trigger an update

### Option A - edit the YAML, then apply (the GitOps / pipeline way)

Change the field (commonly the image) in the Deployment manifest and apply:

```bash
# deployment-definition.yml: spec.template.spec.containers[].image: nginx:1.7.1
kubectl apply -f deployment-definition.yml
```

This is almost certainly what your work pipeline does: your push builds a new image (new tag/digest), the pipeline updates the image field in the manifest, and `apply` reconciles the change - which the Deployment turns into a new rollout/revision. Worth confirming, but your hunch is right: the manifest changes, `apply` runs, a new ReplicaSet is born.

### Option B - `kubectl set image` (imperative, and the behavior caveat)

```bash
kubectl set image deployment/myapp-deployment nginx-container=nginx:1.9.1
```

This also triggers a rollout, but with **different behavior** the instructor flagged: it changes the image on the **live object only**. Your YAML file on disk now **disagrees** with the cluster. The next time someone runs `kubectl apply -f deployment-definition.yml` from the (unchanged) file, it will revert the image. So `set image` is fast for the exam and quick fixes, but in a GitOps/pipeline world it causes drift between Git and the cluster. Rule: use `apply` when the file is the source of truth; use `set image` only for throwaway/exam changes.

## 5. Watching, recording, and inspecting rollouts

### Status

```bash
kubectl rollout status deployment/myapp-deployment
# Waiting for rollout to finish: 5 of 10 updated replicas are available...
# deployment "myapp-deployment" successfully rolled out
```

### History and the `--record` flag

```bash
kubectl rollout history deployment/myapp-deployment
# REVISION   CHANGE-CAUSE
# 1          <none>
# 2          kubectl apply --filename=deployment-definition.yml --record=true
```

- `--record=true` (appended to the command that caused the change) stores that command as the revision's **CHANGE-CAUSE**, which is just the `kubernetes.io/change-cause` **annotation** (ties back to chapter 01). Without it, CHANGE-CAUSE shows `<none>` - harder to tell revisions apart later.
- Note (beyond the lecture): `--record` is **deprecated** in current kubectl. The modern equivalent is to set the annotation yourself: `kubectl annotate deployment/myapp-deployment kubernetes.io/change-cause="updated nginx to 1.9.1"`. Know `--record` for the exam; know the annotation for real life.

### Inspecting a specific revision with `--revision`

```bash
kubectl rollout history deployment/myapp-deployment --revision=1
# shows the pod template (image, labels, etc.) captured at revision 1
```

Use this to see *what* a revision actually contained before deciding whether to roll back to it.

## 6. Rollback

If a new version misbehaves, undo it:

```bash
kubectl rollout undo deployment/myapp-deployment
# rolls back to the immediately previous revision
```

Mechanically (images 10/11): the **current ReplicaSet scales down to 0** and the **previous ReplicaSet scales back up** to the desired count. Because the old ReplicaSet was retained, this is fast - no image re-pull, no rebuild.

Roll back to a **specific** revision instead of just the previous one:

```bash
kubectl rollout undo deployment/myapp-deployment --to-revision=1
```

`kubectl get replicasets` after a rollback shows the swap reversed: the old ReplicaSet back at 5, the new one at 0.

You can also pause/resume a rollout to batch several changes into one revision (beyond the lecture, occasionally on the exam):

```bash
kubectl rollout pause deployment/myapp-deployment
# make several changes... they don't roll out yet
kubectl rollout resume deployment/myapp-deployment   # one rollout for all of them
```

## 7. Command reference (the important ones)

```bash
# create / scaffold
kubectl create deployment myapp --image=nginx:1.7.0 --replicas=5
kubectl create deployment myapp --image=nginx $do > deploy.yaml   # $do = --dry-run=client -o yaml

# update
kubectl apply -f deployment-definition.yml                         # file is source of truth (preferred)
kubectl set image deployment/myapp nginx-container=nginx:1.9.1      # live-only; causes file drift
kubectl edit deployment myapp                                      # live edit (also drifts from file)
kubectl scale deployment myapp --replicas=8                        # scale (not a template change)

# watch / inspect
kubectl rollout status deployment/myapp                            # progress of current rollout
kubectl rollout history deployment/myapp                           # revisions + CHANGE-CAUSE
kubectl rollout history deployment/myapp --revision=2              # what's in revision 2
kubectl get replicasets                                            # see old(0)/new(N) ReplicaSets
kubectl describe deployment myapp                                  # StrategyType, surge/unavailable, events

# record a change-cause
kubectl annotate deployment/myapp kubernetes.io/change-cause="nginx 1.9.1"   # modern
# kubectl apply -f deploy.yml --record=true                        # legacy/deprecated equivalent

# roll back
kubectl rollout undo deployment/myapp                              # to previous revision
kubectl rollout undo deployment/myapp --to-revision=1             # to a specific revision

# pause / resume (batch changes into one rollout)
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp
```

## 8. Exam-pattern gotchas

- **Deployment -> ReplicaSet -> Pod.** A Deployment never edits a ReplicaSet's template; it makes a **new** ReplicaSet per revision and keeps the old at 0. That retained ReplicaSet is what makes rollback instant.
- **RollingUpdate is the default**; Recreate must be set explicitly and causes downtime. If a question says "no downtime," that's RollingUpdate; "can't run two versions at once," that's Recreate.
- **`set image` / `edit` cause file drift.** They change the live object, not your YAML. In a pipeline, the next `apply` from the unchanged file reverts them. Prefer `apply` when the file is source of truth.
- **`--record` is deprecated**; use the `kubernetes.io/change-cause` annotation. Empty CHANGE-CAUSE means nobody recorded it.
- **`--revision=N` inspects, `--to-revision=N` rolls back.** Don't confuse the two flags.
- **`scale` is not a rollout.** Changing replica count doesn't create a new revision; only template changes do.
- **RollingUpdate means mixed versions briefly coexist** - the app and its dependencies must tolerate that. This is the real-world risk of moving off Recreate.

## 9. TL;DR / takeaways

- A Deployment manages **ReplicaSets**, which manage pods. Each template change spawns a **new ReplicaSet = a new revision**; the old ReplicaSet is kept at 0 for instant rollback.
- **Recreate** = kill all, then create all (downtime, not default). **RollingUpdate** = swap pod-by-pod (no downtime, default, but old+new coexist briefly).
- Update by **`kubectl apply -f`** (file is source of truth - your pipeline's path) or **`kubectl set image`** (live-only, drifts from the file).
- Watch with `rollout status`; review with `rollout history` (+ `--revision=N` to inspect a revision); record intent via the `change-cause` annotation (`--record` is the deprecated way).
- Roll back with `rollout undo` (previous) or `--to-revision=N` (specific); under the hood it just rescales the retained old ReplicaSet.
- Your case: moving Recreate -> RollingUpdate removed the upgrade outage; the cost is tolerating two versions live at once during the roll.

---

### Open threads
- [ ] Confirm your pipeline's mechanism: image build -> manifest image field updated -> `apply` -> new ReplicaSet/revision. (Check whether it edits the file or uses `set image`.)
- [ ] **maxSurge / maxUnavailable** tuning deep-dive if a later chapter or killer.sh exercise covers blue-green / canary patterns.
- [ ] Tie `kubernetes.io/change-cause` back to the annotations section in `01-labels-selectors-annotations.md`.
