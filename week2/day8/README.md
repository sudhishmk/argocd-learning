# Week 2, Day 8: Scaling with ApplicationSets & the Git Generator

---
## 🧠 Concept of the Day

As you manage more applications, creating individual `Application` manifests for each one becomes repetitive. The **`ApplicationSet`** controller solves this by acting as a "factory" for `Application` resources.

You define a single `ApplicationSet` custom resource. It uses **Generators** to gather parameters (like a directory path or a cluster name) and a **template** to stamp out `Application` manifests using those parameters.

Today's focus is the **Git Generator**. It can scan a Git repository for directories or files that match a specific path pattern, using each match to generate a new `Application`.

---
## 💼 Real-World Use Case

A company has a monorepo for all its microservices, with each service's Kustomize configuration in its own directory (e.g., `/apps/users-api`, `/apps/billing-api`, `/apps/frontend`).

Instead of creating and maintaining dozens of separate `Application` manifests, the DevOps team creates a single `ApplicationSet`. The Git Generator is configured to scan for all directories matching `/apps/*`. When a new service directory is added to the repo, the `ApplicationSet` controller automatically detects it and creates a corresponding Argo CD `Application` for it, completely automating the onboarding of new services.

---
## 💻 Code/Config Example

This `ApplicationSet` uses the Git Generator to find all directories under the `apps/` path and generates an `Application` for each one, using the directory name (`{{path.basename}}`) for the application's name and namespace.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-apps
spec:
  generators:
  - git:
      repoURL: [https://github.com/my-org/my-apps.git](https://github.com/my-org/my-apps.git)
      revision: HEAD
      # Find all directories matching this pattern
      directories:
      - path: apps/*
  template:
    metadata:
      # Use the directory name as the application name, e.g., 'users-api'
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: [https://github.com/my-org/my-apps.git](https://github.com/my-org/my-apps.git)
        targetRevision: HEAD
        # The source path for the app is the directory that was found
        path: '{{path}}'
      destination:
        server: [https://kubernetes.default.svc](https://kubernetes.default.svc)
        # Deploy each app into a namespace matching its name
        namespace: '{{path.basename}}'
```
### 🛠️ Daily Task

Today you'll stop managing a single application and start using an `ApplicationSet` to manage applications for multiple environments.

1.  **Refactor Your Git Repo**:
    * In your `argocd-learning` repository, create a new directory for this week: `week2/day8/`.
    * Copy your work from day 7 into two new environment-specific directories:
        ```bash
        # Copy the entire kustomize structure
        cp -r week1/day7 week2/day8
        
        # Copy app directory seperatly to frontend-dev and frontend-staging
        cp -r week1/day7/app week2/day8/app/frontend-dev
        cp -r week1/day7/app week2/day8/app/frontend-staging
        ```
    * In the staging copy (`week2/day8/apps/frontend-staging/overlays/dev/deployment-patch.yaml`), change the replica count to `3` to make it different from dev. 

2.  **Delete Old Application**: Delete the single `echo-frontend` application you've been using so far. The `ApplicationSet` will now manage its lifecycle.
    ```bash
    kubectl delete app echo-frontend -n argocd
    ```

3.  **Create the ApplicationSet**: Create a new file locally named `frontend-appset.yaml`. This `ApplicationSet` will find your new environment directories and create an `Application` for each.
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: echo-frontend-appset
      namespace: argocd
    spec:
      generators:
      - git:
          repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
          revision: HEAD
          directories:
          - path: week2/day8/app/*
      template:
        metadata:
          name: '{{path.basename}}' # Will be 'frontend-dev', 'frontend-staging'
        spec:
          project: default
          source:
            repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
            targetRevision: HEAD
            path: '{{path}}/overlays/dev' 
          destination:
            server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
            namespace: '{{path.basename}}'
          syncPolicy:
            automated:
              prune: true
              selfHeal: true
            syncOptions:
            - CreateNamespace=true
    ```

4.  **Commit and Apply**:
    * Commit and push your new `week2` directory structure to Git.
    * Apply your `frontend-appset.yaml` to the cluster: `kubectl apply -f frontend-appset.yaml`.
    * Go to the Argo CD UI. You'll now see two new applications, `frontend-dev` and `frontend-staging`, both managed by the `ApplicationSet`.

---
### 🤔 Daily Self-Assessment

**Question**: What would happen if a developer created a new directory `week2/day8/apps/frontend-prod` in the Git repository and pushed the change?

**Answer**: The `ApplicationSet` controller would detect the new directory matching the `apps/*` pattern and automatically generate a new Argo CD `Application` named `frontend-prod`.

