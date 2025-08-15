# Week 2, Day 11: Progressive Delivery with Argo Rollouts (Revised)

---
## 🧠 Concept of the Day

While Argo CD is excellent at synchronizing your desired state from Git, it doesn't natively handle advanced deployment strategies like **Canary** or **Blue-Green** releases. For that, we use another tool from the Argo family: **Argo Rollouts**.

Argo Rollouts is a separate Kubernetes controller that provides a new Custom Resource called `Rollout`. A `Rollout` resource looks almost identical to a standard `Deployment` but includes a `strategy` block where you can define progressive delivery steps.

Argo CD's role is to sync the `Rollout` manifest from Git. Once that's applied, the Argo Rollouts controller takes over, executing the canary or blue-green strategy. Argo CD understands the health of `Rollout` resources and will show the application as `Progressing` until the rollout is fully complete and promoted.

---
## 💼 Real-World Use Case

You're releasing a new version of a critical payment processing API. A standard rolling update is too risky as it could impact all users if there's a bug.

Instead, you use a `Rollout` object to perform a canary release. The strategy is defined to:
1.  Deploy the new version and send 5% of live traffic to it.
2.  Pause for 10 minutes while automated analysis checks Prometheus for an increase in error rates.
3.  If the analysis passes, gradually increase traffic to 100% over 20 minutes.
4.  If the analysis fails at any point, automatically roll back to the stable version.

This significantly reduces the risk (blast radius) of a bad release.

---
## 💻 Code/Config Example

This `Rollout` manifest replaces a standard `Deployment`. It defines a simple canary strategy that pauses after sending 20% of traffic to the new version, requiring manual promotion to continue.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicas: 5
  selector: # ... selector ...
  template: # ... pod template ...
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {} # Pauses indefinitely until manually promoted
```

### 🛠️ Daily Task

Today, you'll convert your `echo-frontend` `Deployment` into a `Rollout` and correctly update your Kustomize overlay to match.

1.  **Install Argo Rollouts Controller**: This is a one-time setup. The controller must be running in your cluster.
    ```bash
    kubectl create namespace argo-rollouts
    kubectl apply -n argo-rollouts -f [https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml](https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml)
    ```

2.  **Repo Setup**: Copy your work from Day 8 to a new directory for today's task.
    ```bash
    cp -r week2/day8/ week2/day11/
    ```

3.  **Convert Base Resource to a Rollout**: In your new `week2/day11/app/frontend-dev/base/`, rename `deployment.yaml` to `rollout.yaml` and change its content to be a `Rollout` resource. Also update the `kustomization.yaml` in the `base` to point to the new filename.
    ```yaml

    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: echo-frontend
    spec:
      replicas: 1 # This will be patched by Kustomize
      selector:
        matchLabels:
          app: echo-frontend
      template:
        metadata:
          labels:
            app: echo-frontend
        spec:
          containers:
          - name: nginx
            image: nginx:1.25
            ports:
            - containerPort: 80
      strategy:
        canary:
          steps:
          - setWeight: 25
          - pause: { duration: 30s }
    ```

4.  **Update Kustomize Overlay (The Fix!)**: Your overlay is still trying to patch a `Deployment`. You must update it to patch the `Rollout`. Go into `week2/day11/app/frontend-dev/overlays/dev/`.
    * Rename `deployment-patch.yaml` to `rollout-patch.yaml` and update its content:
        ```yaml
        # week2/day11/app/frontend-dev/overlays/dev/rollout-patch.yaml
        apiVersion: argoproj.io/v1alpha1
        kind: Rollout
        metadata:
          name: echo-frontend
        spec:
          replicas: 1 # Replica count for dev
        ```
    * Update the `kustomization.yaml` to use the new patch:
        ```yaml
        # week2/day11/app/frontend-dev/overlays/dev/kustomization.yaml
        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization
        resources:
        - ../../../base
        patches:
        - path: rollout-patch.yaml
        ```
    * **Important**: Make the same changes for the `frontend-staging` overlay directory.

5.  **Update ApplicationSet and Sync**:
    * Modify your local `frontend-appset.yaml` (from Day 8) to point its `path` to the new `week2/day11/app/*` directory.
    * Apply the change: `kubectl apply -f frontend-appset.yaml`.
    * Commit and push your new `week2/day11` directory to Git.

6.  **Trigger and Observe the Rollout**:
    * To trigger a new rollout, change the image tag in `week2/day11/appsfrontend-dev/base/rollout.yaml` (e.g., to `nginx:1.24`). Commit and push.
    * Watch in the Argo CD UI as your applications become `Progressing` and the canary strategy is executed.

---
### 🤔 Daily Self-Assessment

**Question**: When you use Argo CD to manage an Argo Rollouts `Rollout` object, which controller is actually responsible for pausing the deployment, creating new `ReplicaSets`, and shifting traffic?

**Answer**: The **Argo Rollouts controller**. Argo CD's job is just to apply the `Rollout` manifest from Git; the specialized Rollouts controller then reads that resource and executes the progressive delivery strategy.
