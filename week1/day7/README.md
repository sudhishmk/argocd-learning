# Week 1, Day 7: Fine-Tuning Your Git Source

---
## 🧠 Concept of the Day

By default, Argo CD's repository server is smart enough to only process files it recognizes as Kubernetes manifests (like `.yaml` and `.json`). It automatically ignores other files like `README.md`, `.gitignore`, or source code files to prevent errors.

However, you can gain fine-grained control over this behavior using the `spec.source.directory` field in your `Application` manifest. This field allows you to:
* **`recurse`**: Process subdirectories within your specified `path`.
* **`include`**: A glob pattern specifying which files to *exclusively* include.
* **`exclude`**: A glob pattern specifying which files to explicitly ignore.

---
## 💼 Real-World Use Case

Your team maintains a "monorepo" for a microservice. This single Git repository contains the application's Go source code in `/src`, documentation in `/docs`, and Kubernetes manifests in `/deploy/k8s`.

To prevent Argo CD from trying to parse Go code or Markdown files, you configure the `Application` source to point to the `/deploy/k8s` path. Furthermore, you add an `exclude` pattern for `**/NOTES.txt` to ignore developer notes within the manifest directories.

---
## 💻 Code/Config Example

This example tells Argo CD to recursively search for manifests in a directory but to explicitly exclude any `README.md` files it finds.

```yaml
spec:
  source:
    repoURL: '...'
    path: 'deploy/'
    targetRevision: HEAD
    directory:
      recurse: true
      exclude: '**/README.md'
```
### 🛠️ Daily Task

1.  **Add a Non-Manifest File**: In your Git repository, go to the `week1/day7/app/base` directory and create a new file named `README.md`. Add some text like "Base manifests for the echo-frontend application."

2.  **Commit and Observe**:
    * Commit and push the `README.md` file.
    * Go to the Argo CD UI and `Refresh` your application. Notice that the application status remains `Synced`. Argo CD sees the new file but ignores it by default.

3.  **Explicitly Exclude a Pattern**: Now, let's practice using the `exclude` field. We'll add a rule to ignore any files intended for production.
    * Modify your local `app.yaml` to include the `directory` stanza with the new exclusion rule:
        ```yaml
        # In your local app.yaml
        spec:
          source:
            repoURL: '[https://github.com/YOUR_USERNAME/argocd-learning.git](https://github.com/YOUR_USERNAME/argocd-learning.git)'
            path: 'week1/day7/overlays/dev'
            targetRevision: HEAD
            directory:
              # Add this block to exclude production files
              exclude: '*.prod.yaml'
          # ... rest of the file
        ```
    * Apply this change to your cluster. You have now configured this `dev` application to explicitly ignore any files ending with `.prod.yaml`.
    
    * Add files that end with the `.prod.yaml` suffix to "week1/day7/app/overlays/dev" dir and commit and push the change.
    
    * Check if new YAML files are synced or not.

### 🤔 Daily Self-Assessment

**Question**: How would you configure an `Application` to *only* process files that end with the `.prod.yaml` suffix and ignore all other `.yaml` and `.json` files in its source path?

**Answer**: You would use the `include` property, as it exclusively includes files matching the pattern:
`directory: { include: '*.prod.yaml' }`
