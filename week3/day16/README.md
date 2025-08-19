# Week 3, Day 16: SSO Integration with Dex

---
## 🧠 Concept of the Day

Managing local users and passwords in a tool like Argo CD doesn't scale and is a security risk. The standard enterprise approach is to use **Single Sign-On (SSO)**. Argo CD integrates with SSO providers through an identity broker called **Dex**, which is bundled with Argo CD.

The process uses an open standard called **OIDC (OpenID Connect)**. The flow looks like this:
1.  You click "Login" in Argo CD.
2.  Argo CD redirects you to Dex.
3.  Dex redirects you to your company's Identity Provider (IdP), like Okta, Google, or GitHub.
4.  You log in with your normal company credentials.
5.  The IdP sends your identity and group memberships (e.g., `sre-team`, `app-developers`) back to Dex.
6.  Dex gives Argo CD a signed token, and you're logged in.

This delegates user management and authentication to your central identity system.

---
## 💼 Real-World Use Case

An organization wants all its developers to log into Argo CD using their corporate Google Workspace accounts. The administrator configures the `argocd-cm` `ConfigMap` to use Google as an OIDC provider via Dex.

They then update the `argocd-rbac-cm` `ConfigMap` to map the `developers@company.com` Google Group to a read-only role in Argo CD. Now, any developer can log in seamlessly with their Google account, and their permissions are automatically determined by their group membership.

---
## 💻 Code/Config Example

This YAML shows how to configure the `argocd-cm` `ConfigMap` to enable login via GitHub. This is the primary configuration step for setting up an SSO connector.

```yaml
# In the argocd-cm ConfigMap
data:
  url: [https://argocd.your-domain.com](https://argocd.your-domain.com)
  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: YOUR_GITHUB_CLIENT_ID
        clientSecret: YOUR_GITHUB_CLIENT_SECRET
        orgs:
        - name: YourGitHubOrganization
```

### 🛠️ Daily Task

Today, you'll configure Argo CD to allow users to log in with their GitHub accounts. This is a multi-step process but demonstrates the full SSO workflow. You don't need any app for today's exercise, but I'm just copying yesterday's task if you want to test.

1.  **Create a GitHub OAuth App**:
    * Go to your GitHub profile -> Settings -> Developer settings -> OAuth Apps -> "New OAuth App".
    * **Application name**: `Argo CD Tutorial` (or anything you like).
    * **Homepage URL**: `https://<your-argocd-url>/` (e.g., `https://localhost:8080/`).
    * **Authorization callback URL**: `https://<your-argocd-url>/api/dex/callback`. This is the most important field.
    * Click "Register application". On the next page, generate a **New client secret**.
    * **Save the `Client ID` and the `Client Secret`**.

2.  **Configure Dex (`argocd-cm`)**: Patch your `argocd-cm` `ConfigMap` to add the Dex connector for GitHub. **Replace the placeholder values with your actual ID, Secret, and organization name.**
    ```bash
    kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"dex.config":"connectors:\n- type: github\n  id: github\n  name: GitHub\n  config:\n    clientID: YOUR_CLIENT_ID\n    clientSecret: YOUR_CLIENT_SECRET\n    orgs:\n    - name: YOUR_GITHUB_ORG\n"}}'
    ```

3.  **Configure Permissions (`argocd-rbac-cm`)**: Now, we'll map a GitHub team to an Argo CD role. Let's give a team named `admins` in your org full admin rights.
    * Edit the `argocd-rbac-cm` `ConfigMap`: `kubectl edit configmap argocd-rbac-cm -n argocd`.
    * Add the following line to the `policy.csv` key. This grants the `admin` role to any user who is a member of the `admins` team within your GitHub organization.
        ```csv
        # In the policy.csv data block
        g, YOUR_GITHUB_ORG:admins, role:admin
        ```

4.  **Log In via GitHub**:
    * Restart the `argocd-dex-server` and `argocd-server` pods to ensure the new configuration is loaded.
    * Log out of the Argo CD UI. You should now see a new button: **LOGIN VIA GITHUB**.
    * Click it and follow the GitHub authorization flow. If you are a member of the `admins` team in your org, you should be logged in with full admin privileges.

---
### 🤔 Daily Self-Assessment

**Question**: Which Argo CD `ConfigMap` is used to define SSO connectors (like GitHub or Google), and which `ConfigMap` is used to map the groups from those providers to actual permissions?

**Answer**: The `argocd-cm` `ConfigMap` is used to configure the Dex SSO connectors. The `argocd-rbac-cm` `ConfigMap` is used to map the user's groups to Argo CD roles (permissions).
    
    

