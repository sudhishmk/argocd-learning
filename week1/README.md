## ✨ Weekly Review: Week 1 ✨

This week, you built a powerful foundation. You started by deploying a single application and progressively added layers of control and sophistication. You mastered the core **`Application` CRD**, automated deployments with **sync policies**, and learned to manage configurations declaratively using both **Helm** and **Kustomize**. You finished the week by orchestrating complex deployments with ordered **Sync Waves** and imperative **Sync Hooks**.

TRY out below project by yourself

### Weekly Project: Building a Multi-Part Application

Your goal is to solidify the week's concepts by creating a complete two-part application environment using Kustomize.

1.  **Repo Cleanup & Consolidation**:
    * Create a new top-level directory in your Git repository named `week1-review`.
    * Copy the final state of your application from `week1/day6` into `week1-review/echo-frontend`. This directory should contain the Kustomize `base` and `overlays/dev` structure, complete with the sync-waved `Job` and the `PostSync` hook.

2.  **Create a Backend Application**:
    * Create a new Kustomize application structure in `week1-review/echo-backend`.
    * In its `base` directory, create a `deployment.yaml` for a simple backend service. Use an image like `hashicorp/http-echo` and expose port `5678`.
    * Add a `service.yaml` to expose the backend deployment.
    * Create a `kustomization.yaml` in the `base` and a simple overlay in `overlays/dev`.

3.  **Create the Argo CD Applications**:
    * Create two separate `Application` manifest files locally (do not commit them to the repo):
        * `frontend-app.yaml`: This should point to the `week1-review/echo-frontend/overlays/dev` path.
        * `backend-app.yaml`: This should point to the `week1-review/echo-backend/overlays/dev` path.
    * Make sure both applications have an automated `syncPolicy` with `prune: true` and `CreateNamespace=true`.

4.  **Deploy and Verify**:
    * Apply both `Application` manifests to your cluster.
    * In the Argo CD UI, verify that two separate applications (`echo-frontend` and `echo-backend`) are created, synced, and healthy. This setup is the perfect starting point for Week 2, where you'll learn to manage these applications with a single `ApplicationSet`.
