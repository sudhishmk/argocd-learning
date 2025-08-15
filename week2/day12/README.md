# Week 2, Day 12: Customizing Diffs to Ignore Changes

---
## 🧠 Concept of the Day

Sometimes, you'll see an application in Argo CD marked as `OutOfSync` even though you haven't changed anything in Git. This often happens when in-cluster controllers, like admission webhooks, automatically modify resources *after* they're created. These modifications can include adding default values, injecting sidecar containers (e.g., for a service mesh), or adding labels.

Argo CD's **`ignoreDifferences`** feature solves this problem. By adding this block to your `Application` spec, you can provide a list of JSON Pointers that identify specific fields Argo CD should completely ignore when comparing the live state in the cluster to the desired state in Git. This allows you to eliminate "diff noise" from automated, cluster-side changes.

---
## 💼 Real-World Use Case

A team uses the Istio service mesh, which has a mutating admission webhook that automatically injects an `istio-proxy` sidecar container into every pod. In Git, their `Rollout` manifest only defines their main application container. In the cluster, the live pods have two containers.

This causes Argo CD to constantly report the application as `OutOfSync`. The team adds an `ignoreDifferences` rule to their `Application` manifest to ignore the `spec.template.spec.containers[1]` path, effectively telling Argo CD, "Don't worry about the second container in the pod; I know it's supposed to be there."

---
## 💻 Code/Config Example

This `Application` manifest is configured to ignore any changes made to the `replicas` field of a `Deployment`. This is useful if a Horizontal Pod Autoscaler (HPA) is managing the replica count.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
# ...
spec:
  # ...
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
    ```

### 🛠️ Daily Task

Today, you'll simulate an admission webhook adding a label to your `Rollout` and then use `ignoreDifferences` to make Argo CD accept this change.

1.  **Repo Setup**: Copy your work from Day 11 to a new directory for today's task.
    ```bash
    cp -r week2/day11/ week2/day12/
    ```

2.  **Update ApplicationSet Path**: Modify your local `frontend-appset.yaml` (from Day 8) to point its `path` to the new `week2/day12/apps/*` directory. Commit your new `week2/day12` folder to Git and apply the `ApplicationSet` change. 
    ```yaml
    # In frontend-appset.yaml
    spec:
      generators:
      - git:
          # ...
          directories:
          - path: week2/day12/apps/* # Update this path
      # ...
    ```

3.  **Simulate a Mutating Webhook**: Manually add a label to your `frontend-dev` `Rollout` resource directly in the cluster. This mimics what an automated controller might do. 
    ```bash
    kubectl label rollout echo-frontend -n frontend-dev mutated-by=webhook
    ```

4.  **Observe the `OutOfSync` State or Auto healing**: Go to the Argo CD UI and `Refresh` the `frontend-dev` application. You'll see it is now `OutOfSync` or would start self heal if "auto-sync" and "self-heal" enabled because it has detected a label that doesn't exist in your Git repository.

5.  **Apply the Fix**: Modify your `frontend-appset.yaml` again. Add an `ignoreDifferences` block to the `template` to tell Argo CD to ignore this specific label on all `Rollout` resources created by this `ApplicationSet`.
    ```yaml
    # In frontend-appset.yaml
    # ...
      template:
        metadata:
          name: '{{path.basename}}'
        spec:
          project: default
          # Add this ignoreDifferences block
          ignoreDifferences:
          - group: argoproj.io
            kind: Rollout
            jsonPointers:
            - /metadata/labels/mutated-by
          # ... rest of the spec
    ```

6.  **Observe the Fix**:
    * Apply the updated `frontend-appset.yaml`. Argo CD will re-configure the generated applications.
    * `Refresh` the `frontend-dev` application in the UI. It should now be `Synced`, even though the extra label is still on the live resource in the cluster. You've successfully told Argo CD to ignore this specific difference.

---
### 🤔 Daily Self-Assessment

**Question**: Besides a service mesh sidecar injector, what is another common Kubernetes controller that automatically modifies a workload's `spec` and would likely require you to use `ignoreDifferences` in a GitOps workflow?

**Answer**: A **Horizontal Pod Autoscaler (HPA)**. The HPA controller constantly updates the `spec.replicas` field of a `Deployment` or `Rollout` based on CPU/memory metrics, which would cause a constant `OutOfSync` condition if not ignored.
    
