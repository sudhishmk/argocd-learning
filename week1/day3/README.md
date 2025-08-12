# Week 1, Day 3: Managing Applications with Helm

---
## 🧠 Concept of the Day

Argo CD has first-class, native support for **Helm**, one of the most popular package managers for Kubernetes. This means you don't need to manually run `helm template` and commit the output.

Instead, you can point an Argo CD `Application` directly to a Helm chart stored in a Git repository or even a public/private Helm chart repository. You can then specify your environment-specific overrides directly in the `Application` manifest's `spec.source.helm.values` block. Argo CD's repo-server handles the rendering process internally, combining the power of Helm's templating with Argo CD's continuous reconciliation.

---
## 💼 Real-World Use Case

A team needs to deploy a Redis cluster for caching. Instead of writing their own manifests, they use the official Bitnami chart. For their staging environment, they need to override the default values to disable persistence and set specific resource limits to save costs. They define an Argo CD `Application` that points to the Bitnami chart and injects their custom values, achieving a repeatable and maintainable deployment.

---
## 💻 Code/Config Example

This YAML shows how the `source` of an `Application` is configured to use a Helm chart from within your Git repository and override its default values.

```yaml
spec:
  source:
    repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
    path: 'week1/day3/app/echo-frontend' # The directory containing the Chart.yaml
    targetRevision: HEAD
    helm:
      values: |
        replicaCount: 2
        service:
          type: ClusterIP
          port: 80
```

## 🛠️ Daily Task

Today, you'll refactor your Nginx deployment into a proper Helm chart and manage it with Argo CD.

1.  **Create a New Directory & Chart**: In your `argocd-learning` Git repository, create a `week1/day3/` directory. Inside it, run `helm create echo-frontend` to scaffold a new chart.

2.  **Customize the Chart**:
    * Delete the default files in the `templates/` directory (like `serviceaccount.yaml`, `hpa.yaml`, `ingress.yaml`).
    * Modify `templates/deployment.yaml` to be a simple Nginx deployment. Crucially, parameterize the replica count. The file should look like this:
        ```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: nginx
        image: "public.ecr.aws/bitnami/nginx:1.25"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - ALL
        ports:
        - containerPort: 80
        ```
    * Update `values.yaml` to contain the `replicaCount` key:
        ```yaml
        replicaCount: 1
        ```

3.  **Update the Argo CD Application**: Modify your local `app.yaml` file to point to the new Helm chart and override the replica count. It should now look like this:
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: echo-frontend
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
        path: 'week1/day3/app/echo-frontend'
        targetRevision: HEAD
        helm:
          values: |
            replicaCount: 2
      destination:
        server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
        namespace: frontend
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    ```

4.  **Commit and Sync**:
    * Commit and push the new Helm chart structure to your Git repository.
    * Apply your updated `app.yaml` to your cluster.
    * Observe in the Argo CD UI as it renders the Helm chart with your override and deploys two replicas.

---
## 🤔 Daily Self-Assessment

**Question**: To deploy version `16.9.5` of the `redis` chart from the `https://charts.bitnami.com/bitnami` repository, what would the `spec.source` block look like?

**Answer**:
```yaml
source:
  repoURL: '[https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)'
  chart: 'redis'
  targetRevision: '16.9.5'
  helm:
    values: |
      auth:
        password: "your-secret-password"
        ```


