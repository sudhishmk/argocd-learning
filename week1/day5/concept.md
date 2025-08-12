# Week 1, Day 5: Controlling Deployment Order with Sync Waves

---
## 🧠 Concept of the Day

By default, Argo CD applies manifests from Git in a non-deterministic order. For applications with dependencies, this can be a problem. **Sync Waves** solve this by letting you group resources into ordered phases.

You assign a resource to a wave using the `argocd.argoproj.io/sync-wave` annotation with an integer value. Lower numbers are synced first. Argo CD will apply all resources in a given wave (e.g., wave `-5`) and, crucially, **wait for them all to become healthy** before it proceeds to the next wave (e.g., wave `0`).

---
## 💼 Real-World Use Case

You're deploying an application that requires a database schema migration to be run before the application pods start. If the new pods start with the old schema, they'll immediately crash.

By placing the database migration `Job` in a lower sync wave (e.g., `sync-wave: "10"`) and the application `Deployment` in a higher wave (e.g., `sync-wave: "20"`), you guarantee the migration `Job` runs and completes successfully before Argo CD even attempts to apply the `Deployment` manifest.

---
## 💻 Code/Config Example

This example shows a `Job` that will be synced first, followed by a `Deployment` only after the `Job` is healthy.

```yaml
# A Job that must run before the Deployment
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "10"
# ... spec ...
---
# The main application Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-wave: "20"
# ... spec ...
```
### 🛠️ Daily Task

Today, you'll add a "pre-flight" check `Job` to your Kustomize application and use a sync wave to ensure it runs before the main deployment.

1.  **Create the Job Manifest**: In your `week1/day4/base` directory, create a new file named `preflight-job.yaml`. This `Job` will simulate a check that needs to run before your application starts.
    ```yaml
    # week1/day4/base/preflight-job.yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: preflight-check
    spec:
      template:
        spec:
          containers:
          - name: checker
            image: busybox:1.35
            command: ["sh", "-c", "echo 'Running pre-flight checks...'; sleep 15; echo 'Checks complete!'"]
          restartPolicy: Never
      backoffLimit: 1
    ```

2.  **Update Kustomization**: Add the new `Job` to your `week1/day4/base/kustomization.yaml` file.
    ```yaml
    # week1/day4/base/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - deployment.yaml
    - preflight-job.yaml # Add this line
    ```

3.  **Annotate Your Resources**:
    * Add a `sync-wave` annotation to your new `preflight-job.yaml` so it runs first.
        ```yaml
        # week1/day4/base/preflight-job.yaml
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: preflight-check
          annotations:
            argocd.argoproj.io/sync-wave: "-5"
        #...
        ```
    * Add a `sync-wave` annotation to your `deployment.yaml` in the same `base` directory so it runs second.
        ```yaml
        # week1/day4/base/deployment.yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: echo-frontend
          annotations:
            argocd.argoproj.io/sync-wave: "5"
        #...
        ```

4.  **Commit and Sync**:
    * Commit and push your changes to Git.
    * In the Argo CD UI, press `Refresh` on your application. It will show as `OutOfSync`.
    * Press `Sync`. Observe the resources carefully. You will see the `Job` get created and run first. The `Deployment` will wait until the `Job` completes successfully before it is synced.

---
### 🤔 Daily Self-Assessment

**Question**: Will a resource without a `sync-wave` annotation be deployed before or after a resource with `sync-wave: "-5"`?

**Answer**: It will be deployed **after**. Any resource without a `sync-wave` annotation is considered to be in Wave `0` by default.


