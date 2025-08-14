# Week 2, Day 9: Scaling with ApplicationSets & the Cluster Generator

---
## 🧠 Concept of the Day

Yesterday you used the Git Generator to create applications based on a Git repository's structure. Today, we'll look at the **Cluster Generator**, which is your key to managing multi-cluster deployments.

Argo CD stores the connection details for every cluster it manages (including its own local cluster, often named `in-cluster`) as a Kubernetes `Secret` in its own namespace. The **Cluster Generator** works by scanning for these cluster `Secrets`. It generates a set of parameters for each cluster it finds, allowing you to stamp out an application for each one automatically.

The generated parameters include the cluster's `name` and its API `server` URL, which you can use in your `Application` template.

---
## 💼 Real-World Use Case

A platform engineering team needs to ensure that every single Kubernetes cluster in the organization has a standard set of "foundational" tools, such as a logging agent (Fluentd), a policy engine (Kyverno), and a monitoring namespace.

They create a single `ApplicationSet` using the Cluster Generator. This `ApplicationSet` automatically deploys these foundational tools to every registered cluster. When a new production cluster is added to Argo CD, it automatically gets these essential tools without any manual intervention, ensuring consistency and compliance across the entire fleet.

---
## 💻 Code/Config Example

This `ApplicationSet` targets every cluster known to Argo CD and deploys a `monitoring-agent` application to each one. It uses the `{{name}}` and `{{server}}` parameters from the generator in the application template.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
spec:
  generators:
  - clusters: {} # An empty selector targets ALL clusters
  template:
    metadata:
      # e.g., 'monitoring-agent-on-prod-us-east-1'
      name: 'monitoring-agent-on-{{name}}'
    spec:
      project: default
      source:
        repoURL: [https://github.com/my-org/cluster-addons.git](https://github.com/my-org/cluster-addons.git)
        targetRevision: HEAD
        path: 'fluentd-agent'
      destination:
        # These values come directly from the generator
        server: '{{server}}'
        namespace: 'monitoring'
```
### 🛠️ Daily Task

Today, you'll use the Cluster Generator to deploy a standard `monitoring` namespace to your cluster. Even with only one cluster, this demonstrates the powerful multi-cluster pattern.

1.  **Create Manifests in Git**:
    * In your `argocd-learning` repository, create a new directory: `week2/day9/cluster-resources`.
    * Inside this new directory, create a `namespace.yaml` file to define the namespace we want to deploy everywhere.
        ```yaml
        # week2/day9/cluster-resources/namespace.yaml
        apiVersion: v1
        kind: Namespace
        metadata:
          name: monitoring
          labels:
            team: platform-eng
        ```

2.  **Find Your Cluster Name**: Find the name of your local cluster as Argo CD sees it. The default for the cluster Argo CD is running in is `in-cluster`. You can verify this with the CLI:
    ```bash
    argocd cluster list
    ```

3.  **Create the ApplicationSet**: Create a new file locally named `namespace-appset.yaml`. This will deploy the `monitoring` namespace to all clusters it finds.
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: foundational-namespaces
      namespace: argocd
    spec:
      generators:
      - clusters: {} # Leaving this empty targets all clusters
      template:
        metadata:
          # This will create an app named, e.g., 'namespaces-for-in-cluster'
          name: 'namespaces-for-{{name}}'
        spec:
          project: default
          source:
            repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
            targetRevision: HEAD
            path: 'week2/day9/cluster-resources'
          destination:
            server: '{{server}}'
            # Note: For a namespace, the destination namespace doesn't matter,
            # but it's a required field. We use 'default'.
            namespace: 'default'
          syncPolicy:
            automated: { prune: true, selfHeal: true }
    ```

4.  **Commit and Apply**:
    * Commit and push your new `week2/day9` directory to Git.
    * Apply your `namespace-appset.yaml` to the cluster: `kubectl apply -f namespace-appset.yaml`.
    * Go to the Argo CD UI. You'll see a new application was generated for your cluster (e.g., `namespaces-for-in-cluster`). Check that the `monitoring` namespace was successfully created in your cluster.

---
### 🤔 Daily Self-Assessment

**Question**: How can you modify the Cluster Generator to target only the clusters that have been labeled for production? (e.g., a cluster `Secret` with the label `environment: production`)

**Answer**: You would add a `selector` to the generator:
`clusters: { selector: { matchLabels: { 'environment': 'production' } } }`

