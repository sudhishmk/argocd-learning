# Week 2, Day 14: Custom Resource Health Checks

---
## 🧠 Concept of the Day

Argo CD has built-in knowledge of the health of standard Kubernetes resources. It knows a `Deployment` is healthy when its replicas are available, and a `Job` is healthy when it completes successfully.

However, when you use Operators, you introduce Custom Resources (CRs). Argo CD doesn't natively understand the health of a `Prometheus` CR from the Prometheus Operator or a `Vault` CR from the Vault Operator.

To solve this, you can teach Argo CD how to assess the health of *any* resource by providing a **custom health check written in Lua**. These Lua scripts are configured in the `argocd-cm` `ConfigMap`. The script receives the full Kubernetes object as input and must return a status (`Healthy`, `Progressing`, `Degraded`, etc.) and an optional message.

---
## 💼 Real-World Use Case

A team is using the Strimzi operator to manage Kafka clusters. They deploy a `Kafka` custom resource. By default, Argo CD just shows this resource as `Synced`.

An SRE writes a custom health check for the `kafka.strimzi.io/Kafka` resource. The Lua script inspects the `status.listeners` and `status.conditions` fields of the CR. It returns a `Healthy` status only when all listeners are ready and the `Ready` condition is true. This makes Argo CD's health status a true reflection of the Kafka cluster's state, enabling safer automated deployments that depend on Kafka being ready.

---
## 💻 Code/Config Example

This YAML snippet shows how a Lua health check for a custom resource is added to the `argocd-cm` `ConfigMap`.

```yaml
# In the argocd-cm ConfigMap
data:
  resource.customizations.health.mycorp.com_CronJob: |
    hs = {}
    if obj.status ~= nil and obj.status.lastScheduleTime ~= nil then
      hs.status = "Healthy"
      hs.message = "Last run at: " .. obj.status.lastScheduleTime
      return hs
    end
    hs.status = "Progressing"
    hs.message = "Waiting for first run"
    return hs
```
### 🛠️ Daily Task

Today, you'll create a dummy Custom Resource and write a Lua health check to teach Argo CD how to understand its status.

1.  **Repo Setup**: Copy your work from Day 13 to a new directory.
    ```bash
    cp -r week2/day13/ week2/day14/
    ```

2.  **Update ApplicationSet Path**: Modify your `frontend-appset.yaml` to point to the new `week2/day14/app/*` directory. Commit the new folder to Git and apply the `ApplicationSet` change.

3.  **Create a Dummy CRD and CR**: In your new `week2/day14/app/frontend-dev/base/` directory, create two new files:
    * `crd.yaml`: Defines a simple custom resource.
        ```yaml
        apiVersion: apiextensions.ks.io/v1
        kind: CustomResourceDefinition
        metadata:
          name: dbbackups.mycorp.com
        spec:
          group: mycorp.com
          names:
            kind: DbBackup
            plural: dbbackups
            singular: dbbackup
          scope: Namespaced
          versions:
          - name: v1
            schema:
              openAPIV3Schema:
                type: object
                properties:
                  spec:
                    type: object
                    properties:
                      database: {type: string}
                  status:
                    type: object
                    properties:
                      phase: {type: string}
            served: true
            storage: true
        ```
    * `cr.yaml`: An instance of our new custom resource.
        ```yaml
        apiVersion: [mycorp.com/v1](https://mycorp.com/v1)
        kind: DbBackup
        metadata:
          name: my-first-backup
        spec:
          database: postgres
        status:
          phase: Succeeded
        ```

4.  **Update Kustomization**: Add the new CRD and CR to your `week2/day14/apps/frontend-dev/base/kustomization.yaml`.

5.  **Apply the Custom Health Check**: Add the Lua script to Argo CD's central `ConfigMap`. Run this `kubectl` command:
    ```bash
    kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"resource.customizations.health.mycorp.com_DbBackup":"hs = {}\nif obj.status ~= nil and obj.status.phase ~= nil then\n  if obj.status.phase == \"Succeeded\" then\n    hs.status = \"Healthy\"\n    hs.message = \"Backup Succeeded\"\n    return hs\n  end\n  if obj.status.phase == \"Failed\" then\n    hs.status = \"Degraded\"\n    hs.message = \"Backup Failed\"\n    return hs\n  end\n  hs.status = \"Progressing\"\n  hs.message = \"Backup is running\"\n  return hs\nend\nhs.status = \"Progressing\"\nhs.message = \"Waiting for status\"\nreturn hs\n"}}'
    ```

6.  **Commit and Observe**:
    * Commit and push your changes to Git.
    * In the Argo CD UI, `Refresh` and `Sync` your applications.
    * Find the new `DbBackup` resource. Its health status will be `Healthy` with the message "Backup Succeeded" because our Lua script successfully parsed the `status.phase` field.

---
### 🤔 Daily Self-Assessment

**Question**: Where are custom resource health checks configured, and in what programming language are they written?

**Answer**: They are configured in the `data` section of the `argocd-cm` `ConfigMap` and are written in **Lua**.

