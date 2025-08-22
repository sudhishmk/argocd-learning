# Week 3, Day 17: Managing Secrets with HashiCorp Vault

---
## 🧠 Concept of the Day

Storing plain-text secrets (like API keys or passwords) in Git is a major security anti-pattern. A common and secure pattern is to store secrets in a dedicated manager like **HashiCorp Vault** and have Argo CD fetch them dynamically at deploy time.

This is achieved using a **Config Management Plugin (CMP)**. You configure Argo CD's repository server with a plugin that understands how to talk to Vault. In your Git repository, your YAML files contain special **placeholders** instead of real secret values.

When Argo CD syncs the application, the plugin intercepts the process. It finds the placeholders, connects to Vault to retrieve the actual secret data, and injects it into the manifests. The final manifest with the real secret is sent to the Kubernetes API, but the secret itself is never stored in Git or in Argo CD's own database.

---
## 💼 Real-World Use Case

A microservice needs credentials to connect to a message broker. These credentials are centrally managed in Vault. A developer creates a Kubernetes `Secret` manifest in Git, but for the `username` and `password` fields, they use Vault placeholders like `<path:secret/data/rabbitmq/creds#username>`. When Argo CD deploys the application, its Vault plugin replaces these placeholders with the real, live credentials from Vault, making them available to the application pod securely.

---
## 💻 Code/Config Example

This is what a `Secret` manifest containing Vault placeholders looks like in a Git repository. The plugin will replace the values inside the `<...>` placeholders.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  # These are not the real secrets, just pointers to them in Vault.
  DB_USER: <path:secret/data/webapp/db#user>
  DB_PASSWORD: <path:secret/data/webapp/db#password>
```
### 🛠️ Daily Task

Today, you'll configure Argo CD with a Vault plugin and create a manifest with placeholders. The installation steps are specific to the OpenShift GitOps Operator.

1.  **Install the Vault Plugin (The Operator Way)**:

    * Create custom image from Dockerfile and push it to your internal rep.
    ```yaml
    # Start from the Red Hat Universal Base Image
    FROM registry.access.redhat.com/ubi9/ubi:9.6-1755678605

    # Switch to the root user temporarily to install tools
    USER root

    # Install wget and tar, then download and install Helm v2
    RUN dnf install -y wget tar gzip && \
        mkdir /custom-tools && \
        wget -qO /custom-tools/argocd-vault-plugin https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.6.0/argocd-vault-plugin_1.6.0_linux_amd64 && \
        chmod +x /custom-tools/argocd-vault-plugin && \
        dnf clean all

    # IMPORTANT: Switch back to a non-root user to run securely
    USER 1001
```
    * Edit your `ArgoCD` Custom Resource to add the plugin as a sidecar.
        ```bash
        kubectl edit argocd openshift-gitops -n openshift-gitops
        ```
    * Add the following `repo` section to the `spec`. This correctly defines both the sidecar container and the volume it needs.
        ```yaml
        spec:
          # ... other configurations ...
          repo:
            sidecarContainers:
              - args:
                  - |-
                    wget -O argocd-vault-plugin https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.6.0/argocd-vault-plugin_1.6.0_linux_amd64
                    chmod +x argocd-vault-plugin &&\ mv argocd-vault-plugin /custom-tools/
                command:
                  - sh
                  - '-c'
                image: 'quay.io/rhn-support-sudnair/wget:1.0'
                name: custom-tools
                volumeMounts:
                  - mountPath: /custom-tools
                    name: custom-tools
            volumes:
              - emptyDir: {}
                name: custom-tools
        ```
    * Save the file. The Operator will automatically restart the `argocd-repo-server` pod with the plugin correctly installed.
    * **Note**: If your cluster is in a restricted network, you may need to mirror this `quay.io` image to your internal registry and update the `image:` field above.

2.  **Repo Setup**: Copy your work from Day 15 to a new directory.
    ```bash
    cp -r week3/day15/ week3/day17/
    ```

3.  **Create a Placeholder Secret**: In your new `week3/day17/apps/frontend-dev/base/` directory, create `app-secret.yaml`.
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: my-app-secret
    stringData:
      API_KEY: <path:secret/data/frontend/keys#api-key>
    ```

4.  **Update Kustomization**: Add this new secret to your `week3/day17/apps/frontend-dev/base/kustomization.yaml`.

5.  **Configure ApplicationSet to use the Plugin**: Modify your `frontend-appset.yaml`. Add a `plugin` block to the `template.spec.source` and update the path.
    ```yaml
    # In frontend-appset.yaml
    # ...
      template:
        spec:
          source:
            # ... repoURL, targetRevision ...
            path: 'week3/day17/apps/{{path.basename}}' # Update path
            plugin:
              name: argocd-vault-plugin
          # ... rest of the spec
    ```

6.  **Commit, Apply, and Observe**:
    * Commit and push your changes to Git.
    * Apply your updated `frontend-appset.yaml`.
    * In the Argo CD UI, `Refresh` and `Sync`. The sync will likely fail because it cannot connect to a real Vault instance, which is expected. The key is that the `repo-server` pod is running with the sidecar and attempting to render the manifests using the plugin.

---
### 🤔 Daily Self-Assessment

**Question**: Which Argo CD component is responsible for communicating with Vault to replace the secret placeholders?

**Answer**: The **`argocd-repo-server`**. The plugin runs alongside it, so the repo server is the component that needs network access and credentials to connect to Vault.
