# Week 1, Day 6: Running Tasks with Sync Hooks

---
## 🧠 Concept of the Day

While Sync Waves control the order of your declarative manifests, **Sync Hooks** are for running *imperative* tasks that don't fit the declarative model. Think of them as triggers for `Jobs` or `Pods` at specific points in a sync operation.

The main hook types are:
* **`PreSync`**: Runs *before* Argo CD applies any changes. Perfect for database backups or putting an app in maintenance mode.
* **`Sync`**: Runs *during* the sync phase, alongside other resources (and can be controlled by sync waves).
* **`PostSync`**: Runs *after* the sync is successful and all resources are healthy. Ideal for smoke tests, integration tests, or notifying external systems.

You can also control what happens to the hook resource after it runs using a **hook deletion policy**, such as `argocd.argoproj.io/hook-delete-policy: HookSucceeded`.

---
## 💼 Real-World Use Case

After a new version of a critical API is deployed and becomes healthy, an SRE wants to automatically run a `Job` that executes a small collection of smoke tests against the new API's `/health` and `/metrics` endpoints. By creating a `Job` with a `PostSync` hook, they ensure these tests run automatically after every successful deployment, providing immediate feedback and confidence in the release.

---
## 💻 Code/Config Example

This `Job` is configured to run after a successful sync and will be automatically deleted once it completes successfully.

```yaml
# week1/day6/app/base/preflight-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: smoke-test-
  annotations:
    # This is a PostSync hook
    argocd.argoproj.io/hook: PostSync
    # Delete the Job pod after it succeeds
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      # Pod-level security context to satisfy policy requirements
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: curl-test
        image: docker.io/curlimages/curl:7.72.0@sha256:bd5bbd35f89b867c1dccbc84b8be52f3f74dea20b46c5fe0db3780e040afcb6f" #tag: 7.72.0
        # This command will try to curl the service. Replace 'echo-frontend' if your service is named differently.
        command: ["sh", "-c", "echo 'Running smoke test...'; sleep 5; curl http://echo-frontend; echo 'Test complete!'"]
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - ALL # Drop all linux capabilities
      restartPolicy: OnFailure
  backoffLimit: 1
  ```
  ### 🛠️ Daily Task

Today, you'll add a `PostSync` hook to your Kustomize application to run a simple smoke test.

1.  **Copy Previous Work**: In your Git repository, make a copy of your work from Day 5 to create a new directory for today's task.
    ```bash
    cp -r week1/day5/ week1/day6/
    ```

2.  **Update Application Path**: Modify your local `app.yaml` file to point to the new `day7` directory. Apply this change so Argo CD starts watching the new path.
    ```yaml
    # In your local app.yaml
    spec:
      source:
        # Change day5 to day6
        path: 'week1/day6/overlays/dev' 
    #...
    ```

3.  **Create the Hook Manifest**: In your `week1/day6/app/base` directory, create a new file named `smoke-test-hook.yaml`. This `Job` will try to curl the Nginx service. *Note: For this to work, you need a `Service` for your deployment.* If you don't have one, create a simple `service.yaml` in your `base` as well.

    ```yaml
    # week1/day6/app/base/smoke-test-hook.yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      generateName: smoke-test-
      annotations:
        # This is a PostSync hook
        argocd.argoproj.io/hook: PostSync
        # Delete the Job pod after it succeeds
        argocd.argoproj.io/hook-delete-policy: HookSucceeded
    spec:
      template:
        spec:
          containers:
          - name: curl-test
            image: curlimages/curl
            # This command will try to curl the service. Replace 'echo-frontend' if your service is named differently.
            command: ["sh", "-c", "echo 'Running smoke test...'; sleep 5; curl http://echo-frontend; echo 'Test complete!'"]
          restartPolicy: Never
      backoffLimit: 1
    ```
    ```yaml
    # week1/day6/app/base/service.yaml (if you don't already have one)
    apiVersion: v1
    kind: Service
    metadata:
      name: echo-frontend
    spec:
      selector:
        app: echo-frontend
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    ```

4.  **Update Kustomization**: Add both the `service.yaml` and `smoke-test-hook.yaml` to your `week1/day6/app/base/kustomization.yaml` file.

    ```yaml
    # week1/day6/app/base/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - deployment.yaml
    - preflight-job.yaml
    - service.yaml         # Add this
    - smoke-test-hook.yaml # Add this
    ```

5.  **Commit and Sync**:
    * Commit and push your changes to Git.
    * In the Argo CD UI, `Refresh` and `Sync` your application.
    * Watch the sync process. After the `Deployment` and `Service` are healthy, you will see a new `Job` created by the hook. It will run its course, and because of the delete policy, it will disappear from the UI once it's finished successfully.

---
### 🤔 Daily Self-Assessment

**Question**: What's the key difference between a `Job` in a `sync-wave` and a `Job` used as a `Sync Hook`?

**Answer**: A `Job` in a sync wave is part of the **declarative desired state**; Argo CD considers it a managed resource of the application. A `Job` used as a Sync Hook is a **transient, imperative task** associated with the sync *operation* itself and is not considered part of the application's final desired state.
  
