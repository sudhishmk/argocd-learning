# Week 3, Day 15: Layered Security - AppProjects and Kubernetes RBAC

---
## 🧠 Concept of the Day

Today's concept is **Layered Security**. When troubleshooting Argo CD permissions, you're dealing with two independent security layers that must both allow an operation to succeed.

1.  **Layer 1: Argo CD `AppProject` Rules**: This is Argo CD's internal security. The `AppProject` defines what an application is *allowed to ask for*. It governs source repos, destination namespaces, and the types of resources (`clusterResourceWhitelist`/`Blacklist`) an application can contain. An error like "...not permitted in project..." comes from this layer.

2.  **Layer 2: Kubernetes RBAC**: This is the cluster's main security system. It defines what the Argo CD controller's `ServiceAccount` is actually *allowed to do* in the cluster. An error like "...cannot create resource..." comes from this layer.

Think of it like this: The `AppProject` is the **company policy** that says you're allowed to request a keycard for the server room. Kubernetes RBAC is the **security guard** who checks the master list to see if you're actually authorized to be given one. Both must approve.

---
## 💼 Real-World Use Case

A platform team onboards a new application that uses a custom `RedisCluster` CRD. To do this securely, they perform two distinct actions:
1.  **As Cluster Admins (Layer 2)**, they create a `ClusterRole` and `ClusterRoleBinding` that grants the Argo CD controller's `ServiceAccount` the permission to create, update, and delete `RedisCluster` resources.
2.  **As Argo CD Admins (Layer 1)**, they create an `AppProject` for the new application. In this project, they add the `redis.redis.opstreelabs.in` API group to the `clusterResourceWhitelist` to allow the application to contain `RedisCluster` manifests.

With both layers configured, the application can now be deployed successfully and securely.

---
## 💻 Code/Config Example

A complete, proactive security setup requires configuring both layers.

```yaml
# Layer 2: A ClusterRole granting permissions on the custom resource
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-custom-resource-manager
rules:
- apiGroups: ["mycorp.com"]
  resources: ["dbbackups"]
  verbs: ["*"]
---
# Layer 1: An AppProject allowing the custom resource
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
spec:
  # ... destinations, sourceRepos ...
  clusterResourceWhitelist:
  - group: 'mycorp.com'
    kind: 'DbBackup'
```

### 🛠️ Daily Task

Today, you'll proactively configure both security layers to avoid the errors you discovered.

1.  **Repo Setup**: Copy your work from Day 14 to a new directory.
    ```bash
    cp -r week2/day14/ week3/day15/
    ```

2.  **Step A: Configure Kubernetes RBAC (Layer 2)**: First, as a cluster admin, you need to grant the Argo CD controller permission to manage `DbBackup` resources.
    * Create a file named `dbbackup-rbac.yaml`:
        ```yaml
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        metadata:
          name: argocd-dbbackup-manager-role
        rules:
        - apiGroups: ["mycorp.com"]
          resources: ["dbbackups"]
          verbs: ["*"]
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: argocd-dbbackup-manager-binding
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: argocd-dbbackup-manager-role
        subjects:
        - kind: ServiceAccount
          namespace: openshift-gitops
          name: openshift-gitops-argocd-application-controller
        ```
    * Apply it directly to your cluster: `kubectl apply -f dbbackup-rbac.yaml`.

3.  **Step B: Configure the AppProject (Layer 1)**: Now, create the `AppProject` with the correct whitelist from the start.
    * Create a file named `echo-project.yaml`:
        ```yaml
        apiVersion: argoproj.io/v1alpha1
        kind: AppProject
        metadata:
          name: echo-apps
          namespace: argocd
        spec:
          description: Project for the Echo applications
          sourceRepos:
          - '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
          destinations:
          - server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
            namespace: 'frontend-*'
          # Proactively whitelist the CRD and the CR
          clusterResourceWhitelist:
          - group: 'apiextensions.k8s.io'
            kind: 'CustomResourceDefinition'
          - group: 'mycorp.com'
            kind: 'DbBackup'
        ```
    * Apply it: `kubectl apply -f echo-project.yaml`.

4.  **Update the ApplicationSet**: Modify your `frontend-appset.yaml` to use the new project and point to the day 15 directory.
    ```yaml
    # In frontend-appset.yaml
    spec:
      generators:
      - git:
          directories:
          - path: week3/day15/apps/* # Update path
      template:
        spec:
          project: echo-apps # Assign to the new project
    ```
    * Apply the change: `kubectl apply -f frontend-appset.yaml`.

5.  **Commit and Sync**: Commit your `week3/day15` directory. The application should sync without any permission errors because you've correctly configured both security layers.

---
### 🤔 Daily Self-Assessment

**Question**: An application sync fails with the error message "resource Services is not permitted in project my-app". Which security layer is responsible for this error: the `AppProject` or Kubernetes RBAC?

**Answer**: The **`AppProject` (Layer 1)**. The error message explicitly mentions the project, indicating it's an internal Argo CD policy that's blocking the sync.
