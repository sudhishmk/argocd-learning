# Week 3, Day 18: Managing Secrets with Sealed Secrets

---
## 🧠 Concept of the Day

**Sealed Secrets** is a Kubernetes controller and toolset designed to solve one problem: how to safely store secrets in a public Git repository.

It works using asymmetric encryption with a public/private key pair that is unique to your cluster:
1.  **Controller:** A controller runs in your cluster. On startup, it generates a private key (which it keeps secret) and a public key (which it makes available to you).
2.  **`kubeseal` CLI:** You use a command-line tool called `kubeseal` on your local machine. You feed it a regular Kubernetes `Secret` manifest. `kubeseal` fetches the public key from the controller and uses it to encrypt your secret data.
3.  **`SealedSecret` CRD:** The output is a new Custom Resource of `kind: SealedSecret`. This manifest contains your encrypted data and is **safe to commit to Git**.
4.  **Decryption:** When Argo CD syncs the `SealedSecret` resource, the controller in the cluster recognizes it. Since only the controller has the private key, it's the only thing that can decrypt the data and create a standard Kubernetes `Secret` in the cluster.

---
## 💼 Real-World Use Case

A developer needs to add a database password for their application. They create a standard `Secret` YAML file on their laptop containing the password. They run `kubeseal` against this file, which generates a `sealed-secret-db.yaml` file. The original file with the plain-text password is never committed to Git.

The developer adds the `sealed-secret-db.yaml` to their Kustomize base, commits it, and creates a pull request. The encrypted secret can be safely reviewed by teammates. Once merged, Argo CD syncs the `SealedSecret`, and the controller in the cluster securely decrypts it, creating the final `Secret` resource for the application to use.

---
## 💻 Code/Config Example

You start with a normal, plain-text secret (DO NOT COMMIT THIS):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-api-key
  namespace: my-app
data:
  API_KEY: "c3VwZXJzZWNyZXR2YWx1ZQ==" # "supersecretvalue" base64 encoded
```

After running kubeseal, you get a SealedSecret that is safe to commit:

```
apiVersion: [bitnami.com/v1alpha1](https://bitnami.com/v1alpha1)
kind: SealedSecret
metadata:
  name: my-api-key
  namespace: my-app
spec:
  encryptedData:
    API_KEY: AgA...[long encrypted string]...==
  template:
    metadata:
      name: my-api-key
      namespace: my-app
      ```
### 🛠️ Daily Task

Today, you'll install Sealed Secrets and use it to encrypt and deploy a secret.

1.  **Install the Sealed Secrets Controller**: This is a one-time setup for your cluster.
    ```bash
    kubectl apply -f [https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.1/controller.yaml](https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.1/controller.yaml)
    ```

2.  **Install the `kubeseal` CLI**: Download the `kubeseal` binary for your OS from the [official releases page](https://github.com/bitnami-labs/sealed-secrets/releases) and place it in your PATH.

3.  **Fetch the Public Key**: The `kubeseal` CLI needs to get the public key from the controller. It can do this automatically. Run this command to fetch the certificate and save it locally. This command may take 10-20 seconds to run the first time.
    ```bash
    kubeseal --fetch-cert > pub-cert.pem
    ```

4.  **Create and Seal a Secret**:
    * First, create a regular, plain-text secret manifest locally. Name it `my-plain-secret.yaml`.
        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: frontend-secret
          namespace: frontend-dev # Important: namespace must match the destination
        stringData:
          API_KEY: "ThisIsMySuperSecretDevKey"
        ```
    * Now, use `kubeseal` to encrypt it. This reads your plain secret, encrypts it using the public key, and outputs the `SealedSecret` manifest.
        ```bash
        kubeseal --cert pub-cert.pem --format yaml < my-plain-secret.yaml > sealed-secret.yaml
        ```

5.  **Add Sealed Secret to Git**:
    * Copy the new `sealed-secret.yaml` file (the encrypted one) into your `week3/day15/apps/frontend-dev/base` directory. **Do not commit `my-plain-secret.yaml` or `pub-cert.pem`!**
    * Update the `kustomization.yaml` in that directory to include `sealed-secret.yaml` in its resources list.

6.  **Commit, Sync, and Verify**:
    * Commit and push your changes. Your Argo CD application (`frontend-dev`) will become `OutOfSync`.
    * `Sync` the application. You will see the `SealedSecret` resource get created.
    * A moment later, the Sealed Secrets controller will create the standard `Secret`. Verify this with `kubectl`:
        ```bash
        # You should see the 'SealedSecret' you deployed from Git
        kubectl get sealedsecret frontend-secret -n frontend-dev

        # You should see the normal 'Secret' created by the controller
        kubectl get secret frontend-secret -n frontend-dev -o yaml
        ```

---
### 🤔 Daily Self-Assessment

**Question**: In the Sealed Secrets model, which component holds the private key and is responsible for decrypting the `SealedSecret`?

**Answer**: The **Sealed Secrets controller** running inside the Kubernetes cluster.
