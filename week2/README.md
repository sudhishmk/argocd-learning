## ✨ Weekly Review: Week 2 ✨

This week, you leveled up from managing single applications to handling entire fleets. You mastered **ApplicationSets** with Git, Cluster, and Matrix generators. You learned how to implement safer, progressive delivery with **Argo Rollouts**, and tackled day-2 operational challenges by customizing diffs, managing resource pruning, and teaching Argo CD to understand your custom resources with Lua health checks.

TRY out below project by yourself

### Weekly Project: Onboard a New Service

Your goal is to onboard a new "backend" service and manage both it and the existing frontend service across two environments using a single `ApplicationSet`.

1.  **Create a Backend App Config**: In a new directory, `week2/apps/echo-backend`, create a Kustomize structure (`base` and `overlays/dev`) for a new application. For the `base`, use a simple backend image like `hashicorp/http-echo` in a `Deployment` manifest.
2.  **Refactor the App Dirs**: Reorganize your Git repo. Instead of `week2/apps/frontend-dev` and `frontend-staging`, create a structure like this:
    * `week2/final-project/frontend/dev`
    * `week2/final-project/frontend/staging`
    * `week2/final-project/backend/dev`
    * `week2/final-project/backend/staging`
3.  **Update the ApplicationSet**: Modify your `frontend-appset.yaml` to use a Matrix Generator.
    * **Generator A (Git)** should find the applications ( `frontend`, `backend`). Path: `week2/final-project/*`.
    * **Generator B (List)** should define the environments (`dev`, `staging`).
    * The template should generate applications named `{{path.basename}}-{{env}}` (e.g., `frontend-dev`, `backend-staging`).
4.  **Deploy and Verify**: Apply the new `ApplicationSet`. You should see four Argo CD `Applications` get created, deploying both services to both environments.
