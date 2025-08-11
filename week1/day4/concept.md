# Week 1, Day 4: Managing Environments with Kustomize

---
## 🧠 Concept of the Day

**Kustomize** is a powerful, template-free tool for customizing Kubernetes configurations that's built directly into `kubectl`. Argo CD has native support for it. The core idea is to have a common **`base`** directory containing your standard resource manifests (like a `Deployment` and `Service`).

Then, for each environment (e.g., `dev`, `staging`, `prod`), you create an **`overlay`** directory. This overlay doesn't copy the base manifests; instead, it contains a `kustomization.yaml` file that points to the `base` and applies specific **patches** or modifications. This lets you manage differences between environments (like replica counts, resource limits, or `ConfigMap` values) without duplicating all your YAML files.

---
## 💼 Real-World Use Case

A microservice needs 2 replicas and debug logging enabled in the `dev` environment, but 10 replicas and `info`-level logging in `prod`. Using Kustomize, the team defines the `Deployment` just once in the `base`. The `dev` overlay applies a patch to set replicas to 2 and update the logging `ConfigMap`, while the `prod` overlay applies a different patch for 10 replicas and the corresponding `ConfigMap` value. This keeps the configuration DRY (Don't Repeat Yourself).

---
## 💻 Code/Config Example

This YAML shows the contents of a `kustomization.yaml` file within an overlay. It declares that it uses the `base` resources and then applies a patch to modify the `Deployment`.

```yaml
# In overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patchesStrategicMerge:
- deployment-patch.yaml
```

### 🛠️ Daily Task

Today, you'll convert your application structure from Helm to Kustomize to manage a `dev` environment.

1.  **Create the Kustomize Structure**: In your `argocd-learning` Git repository, create a new directory structure:
    * `week1/day4/base`
    * `week1/day4/overlays/dev`

2.  **Populate the `base`**:
    * Move a simplified version of your Nginx `deployment.yaml` into the `base` directory.
    * Create a `kustomization.yaml` file inside the `base` directory to list the resources:
        ```yaml
        # week1/day4/base/kustomization.yaml
        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - deployment.yaml
        ```

3.  **Configure the `dev` Overlay**:
    * Create the `kustomization.yaml` file inside the `overlays/dev` directory. This tells Kustomize to use the `base` and apply a patch.
        ```yaml
        # week1/day4/overlays/dev/kustomization.yaml
        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - ../../base
        patchesStrategicMerge:
        - deployment-patch.yaml
        ```
    * Create the patch file, `deployment-patch.yaml`, in the same directory. This patch will set the replica count specifically for the `dev` environment.
        ```yaml
        # week1/day4/overlays/dev/deployment-patch.yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: echo-frontend # Must match the name in the base deployment.yaml
        spec:
          replicas: 1 # Dev environment only gets 1 replica
        ```

4.  **Update the Argo CD Application**: Modify your `app.yaml` file. Change the `spec.source.path` to point to the new Kustomize overlay and **remove the `helm` section entirely.**
    ```yaml
    # In your local app.yaml
    spec:
      source:
        repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
        path: 'week1/day4/overlays/dev'
        targetRevision: HEAD
      # ... rest of the file
    ```

5.  **Commit and Sync**:
    * Commit and push the new Kustomize directory structure to Git.
    * Apply your updated `app.yaml`.
    * Observe in the Argo CD UI as it detects the source type as Kustomize, builds the configuration, and deploys a single replica.

---
### 🤔 Daily Self-Assessment

**Question**: What happens in Kustomize if a `ConfigMap` named `my-config` is defined in the `base` and you also list a *different* `ConfigMap` with the same name (`my-config`) in the `resources` section of an overlay's `kustomization.yaml`?

**Answer**: Kustomize will throw an error because the same resource identifier (`ConfigMap/my-config`) is defined in two places. The correct way to modify a base `ConfigMap` is to use a `configMapGenerator` with a `behavior: merge` or `behavior: replace` policy in the overlay.
