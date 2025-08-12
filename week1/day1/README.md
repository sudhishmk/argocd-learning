You will need a Kubernetes cluster (like Minikube, k3d, or a cloud provider's offering) and a public Git repository (like GitHub or GitLab) to complete these tasks.

# Week 1: Laying the Foundation
## Day 1: Your First GitOps-managed Application

🧠 ** Concept of the Day **: The Application CRD is the core resource that tells Argo CD what to deploy, where to deploy it, and where to find its configuration. It forms the bond between your Git repository and your Kubernetes cluster.

💼 ** Real-World Use Case **: A developer needs to deploy a simple Nginx web server into the cluster for the first time. Instead of using kubectl apply -f, they define an Argo CD Application to create and manage it, ensuring its state is always tied to Git.

💻 ** Code/Config Example **:

** YAML **

### app.yaml
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: echo-frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/YOUR_USERNAME/argocd-learning.git'
    path: 'week1/day1'
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: frontend
```
### 🛠️ Daily Task:

- Create a new public Git repository named argocd-learning.
- Inside the repo, create a directory structure: week1/day1.
- In the day1 directory, create a file nginx.yaml with a simple Nginx Deployment.
- Create an app.yaml file locally (like the example above), pointing to your repository's path.
- Apply it to your cluster: kubectl apply -f app.yaml.
- Check the Argo CD UI to see your echo-frontend application and the Nginx pod it created.

🤔 Daily Self-Assessment: What are the three minimum required fields in the spec of an Application resource?

Answer: project, source, and destination.
