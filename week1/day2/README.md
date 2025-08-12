# Week 1, Day 2: Automated Sync and Configuration

---

## 🧠 Concept of the Day

The **`syncPolicy`** in an Argo CD `Application` spec controls how changes are applied from Git to your cluster. While the default is manual syncing, setting an **`automated`** policy is where the real power of GitOps shines.

An automated policy has two key sub-fields:
* **`prune: true`**: This tells Argo CD to **delete** resources from the cluster if they are removed from the Git repository. It keeps your cluster clean and free of orphaned resources.
* **`selfHeal: true`**: This is a powerful feature that enforces Git as the single source of truth. If Argo CD detects any "drift" – meaning a manual change was made directly in the cluster (e.g., using `kubectl edit`) – it will automatically revert that change to match the state defined in Git.

Additionally, **`syncOptions`** like **`CreateNamespace=true`** can be used to automatically create the target namespace if it doesn't already exist, simplifying the initial setup.

---

## 💼 Real-World Use Case

Imagine a fast-paced development environment that needs to reflect the `develop` branch instantly. An **automated sync policy** ensures that as soon as a developer merges a new feature, it gets deployed without any manual steps.

Furthermore, if a developer makes a temporary "hotfix" directly in the cluster using `kubectl` to debug an issue, the **`selfHeal`** feature will automatically overwrite that change on the next reconciliation loop (typically within 3 minutes). This prevents temporary fixes from becoming permanent, undocumented changes and strictly enforces that all long-term modifications must go through a proper Git workflow (e.g., a pull request).

---

## 💻 Code/Config Example

This YAML snippet is added to your `Application` manifest to enable automated sync, pruning, self-healing, and namespace creation.

```yaml
# In your app.yaml
spec:
  # ... source and destination sections ...

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
---
## 🛠️ Daily Task

Today's task is to enable automation and see self-healing in action. Copy the content from day1 to day2

1.  **Create a `ConfigMap`**: In your Git repo, inside the `week1/day2/` directory, create a new file named `configmap.yaml` with the following content:
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: nginx-config
    data:
      index.html: "<h1>Hello from GitOps!</h1>"
    ```

2.  **Update the Deployment**: Modify your `nginx.yaml` to mount this `ConfigMap` as the main `index.html` page. This replaces the default Nginx page.
    ```yaml
    # In nginx.yaml, add 'volumeMounts' to the container
    # and 'volumes' to the pod spec.
    spec:
      template:
        spec:
          containers:
          - name: nginx
            image: nginx:1.25
            ports:
            - containerPort: 80
            volumeMounts:
            - name: nginx-index
              mountPath: /usr/share/nginx/html
          volumes:
          - name: nginx-index
            configMap:
              name: nginx-config
    ```

3.  **Commit Your Changes**: Commit and push both new and modified files to your Git repository.

4.  **Enable Automation**: Modify your local `app.yaml` file (the one that defines the Argo CD Application) by adding the `syncPolicy` from the code example above. Apply this change to your cluster: `kubectl apply -f app.yaml`.

5.  **Observe and Test**:
    * Argo CD should automatically sync the changes. Port-forward to your Nginx pod and verify you see the "Hello from GitOps!" message.
    * Now, test self-healing! Manually delete the `ConfigMap` from your cluster: `kubectl delete configmap nginx-config -n frontend`.
    * Wait a few minutes and check the Argo CD UI. You'll see the application status become "OutOfSync." Shortly after, Argo CD's self-heal mechanism will kick in and re-create the `ConfigMap`, returning the application to a "Synced" and "Healthy" state.

---
## 🤔 Daily Self-Assessment

**Question**: If `selfHeal: true` is enabled, what happens when you use the command `kubectl scale deployment echo-frontend --replicas=5 -n frontend` on a deployment that is defined with only 1 replica in Git?

**Answer**: Argo CD will detect that the live state of the `Deployment` (5 replicas) has drifted from the desired state defined in Git (1 replica). During its next reconciliation, it will automatically scale the `Deployment` back down to 1 replica.
