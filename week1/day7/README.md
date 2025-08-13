# Week 1, Day 7: Kustomize and The Power of Explicit Inclusion

---
## 🧠 Concept of the Day

Today's core concept is understanding Argo CD's rendering modes. When you point an `Application` to a Git path, Argo CD checks for specific files to decide how to render the manifests.

* **Directory Mode**: If no `kustomization.yaml`, `Helm` chart, or other plugin is detected, Argo CD treats the path as a plain directory of YAMLs. In *this mode only*, it uses its own logic, including the `directory` stanza (`include`/`exclude`), to find manifests.
* **Kustomize Mode**: If Argo CD finds a `kustomization.yaml` file, it delegates all responsibility for finding and assembling manifests to the `kustomize` tool. The `kustomization.yaml` file becomes the *single source of truth* for which files are included. The `directory` stanza in the `Application` spec is ignored.

The Kustomize philosophy is **explicit inclusion**. You don't tell it what to ignore; you only tell it what to use.

---
## 💼 Real-World Use Case

A platform team maintains a Kustomize `base` for a standard application, which includes manifests for a `Deployment`, `Service`, `ServiceAccount`, `NetworkPolicy`, and `PodDisruptionBudget`.

When deploying to a lightweight `dev` environment, they only need the `Deployment` and `Service`. Instead of trying to exclude the other files, they simply create a `dev` overlay whose `kustomization.yaml` only lists the two required components in its `resources` list. This results in a minimal, clean deployment for the dev environment while reusing the base manifests.

---
## 💻 Code/Config Example

This `kustomization.yaml` file demonstrates explicit inclusion. Even if a file named `extra-config.yaml` exists in the same directory, it will be ignored because it is not listed in the `resources`.

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Only the files explicitly listed here will be processed by the kustomize tool.
resources:
- deployment.yaml
- service.yaml
```

### 🛠️ Daily Task

Today, you'll see the Kustomize principle of explicit inclusion in action by adding a new manifest that Argo CD will ignore.

1.  **Copy Previous Work**: In your Git repository, make a copy of your work from Day 4 to create a new directory for today's task.
    ```bash
    cp -r week1/day6/ week1/day7/
    ```

2.  **Update Application Path**: Modify your local `app.yaml` file to point to the new `day7` directory. Apply this change so Argo CD starts watching the new path.
    ```yaml
    # In your local app.yaml
    spec:
      source:
        # Change day6 to day7
        path: 'week1/day7/overlays/dev' 
    #...
    ```

3.  **Create an "Optional" Manifest**: Now, working inside the **new** `week1/day7/base` directory, create a new file named `optional-quota.yaml`. This will define a resource quota that we don't want in our `dev` environment.
    ```yaml
    # week1/day7/base/optional-quota.yaml
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: dev-quota
    spec:
      hard:
        pods: "5"
    ```

4.  **Do Not Update Kustomization**: This is the key step. **Do not** add `optional-quota.yaml` to the `resources` list in your **new** `week1/day7/base/kustomization.yaml` file.

5.  **Commit and Observe**:
    * Commit the new `week1/day7` directory to your Git repository.
    * Go to the Argo CD UI and `Refresh` your application.
    * **Observe**: You will see that the application remains `Synced`. Even though you added a new YAML file to the source directory, Argo CD doesn't see it as a change because the `kustomization.yaml` (the source of truth) was not updated to include it.

---
### 🤔 Daily Self-Assessment

**Question**: If you add a new manifest file named `debug-tools.yaml` to a Kustomize directory that Argo CD is managing, what is the one crucial step you must take for Argo CD to actually deploy it?

**Answer**: You must add the filename `debug-tools.yaml` to the `resources` list inside the `kustomization.yaml` file for that directory.


