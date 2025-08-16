# Week 2, Day 13: Pruning and Resource Tracking

---
## 🧠 Concept of the Day

A core principle of GitOps is that your Git repository is the single source of truth. This means that if a manifest is *removed* from Git, the corresponding resource should be *removed* from the cluster. This cleanup process is called **Pruning**.

You enable this in Argo CD by setting `prune: true` in your `Application`'s `syncPolicy`.

For pruning to be safe, Argo CD needs to know exactly which resources it "owns." It does this via **Resource Tracking**. By default, Argo CD adds a tracking label (`app.kubernetes.io/instance: <your-app-name>`) to every resource it creates. When a sync occurs with pruning enabled, Argo CD will *only* delete resources that have its tracking label but are no longer present in the Git source. This prevents it from accidentally deleting resources managed by other tools or users.

---
## 💼 Real-World Use Case

A developer is refactoring a microservice. They decide to replace a `StatefulSet` with a simpler `Deployment`. In their pull request, they delete `statefulset.yaml` from the Git repository and add `deployment.yaml`.

Once the PR is merged, the automated sync policy kicks in. Argo CD sees that the `Deployment` is a new resource and creates it. It also sees that the `StatefulSet` (which has Argo CD's tracking label) exists in the cluster but no longer in Git. It safely prunes the `StatefulSet`, keeping the live cluster state perfectly synchronized with the Git repository.

---
## 💻 Code/Config Example

Enabling pruning is as simple as adding one line to the `syncPolicy`.

```yaml
# In an Application or ApplicationSet template
syncPolicy:
  automated:
    prune: true # <-- Enable pruning
    selfHeal: true
```
### 🛠️ Daily Task

Today, you'll see pruning in action by removing the pre-flight `Job` you created in an earlier lesson.

1.  **Repo Setup**: Copy your work from Day 12 to a new directory for today's task.
    ```bash
    cp -r week2/day12/ week2/day13/
    ```

2.  **Update ApplicationSet Path**: Modify your `frontend-appset.yaml` to point to the new `week2/day13/app/*` directory. Commit the new folder to Git and apply the `ApplicationSet` change.

3.  **Verify Pruning is Enabled**: Check your `frontend-appset.yaml`. The `template` section should already have `prune: true` in its `syncPolicy` from our earlier lessons. If not, add it now.

4.  **Remove the Resource from Git**: We will remove the `preflight-check` `Job` created on Day 5.
    * In your new `week2/day13/app/frontend-dev/base/` directory, **delete the `preflight-job.yaml` file**.
    * Edit the `kustomization.yaml` file in that same directory and **remove the line** that references `preflight-job.yaml`.
        ```yaml
        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - rollout.yaml # This was renamed from deployment.yaml
        # - preflight-job.yaml # <-- Delete this line
        - service.yaml
        - smoke-test-hook.yaml
        ```
    * Since `frontend-staging` shares this base, the change will affect it too.

5.  **Commit and Observe**:
    * Commit and push these changes to your Git repository.
    * In the Argo CD UI, `Refresh` both the `frontend-dev` and `frontend-staging` applications. They will now show as `OutOfSync`.
    * Click `Sync` on one of the applications. In the sync details screen, you will see the `Job` resource marked for **Pruning**.
    * After the sync completes, the `Job` will be gone from the cluster, and the application will be `Synced`.

---
### 🤔 Daily Self-Assessment

**Question**: If an Argo CD application is managing the `default` namespace with `prune: true`, what prevents it from deleting a `Pod` that you created manually with `kubectl run my-pod ...` in that same namespace?

**Answer**: The manually created `Pod` will not have the Argo CD tracking label (`app.kubernetes.io/instance: ...`). By default, Argo CD will only prune resources that it knows it owns via this label.

