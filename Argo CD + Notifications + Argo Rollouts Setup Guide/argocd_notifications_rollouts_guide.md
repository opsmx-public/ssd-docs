# Argo CD + Notifications + Argo Rollouts Setup Guide

## Overview
This document describes end-to-end setup of Argo CD with:
- Ingress for UI access
- Email & Slack notifications
- Sample applications
- Argo Rollouts with Canary & Blue-Green deployments

All installations use `kubectl apply` (official manifests).

---

## 1️⃣ Create Namespace for Argo CD
```
kubectl create namespace argocd
```
Verify:
```
kubectl get ns argocd
```

---

## 2️⃣ Install Argo CD via kubectl apply
```
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
This command installs all required Argo CD components such as the API server, application controller, repo server, and related RBAC resources using the official Argo CD manifest.

Verify pods:
```
kubectl get pods -n argocd
```

---

## 3️⃣ Port Forward Argo CD (Temporary Access)
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Used for quick local access before setting up ingress.

Access:
```
https://localhost:8080
```
Get admin password:
```
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```
Argo CD generates a default admin password stored securely in a Kubernetes Secret.

---

## 4️⃣ Ingress YAML for Argo CD
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```
This Ingress exposes the Argo CD UI externally using NGINX. The backend protocol is set to HTTP because Argo CD serves HTTP internally even though the service port is 443.

Apply:
```
kubectl apply -f argocd-ingress.yaml
```

---

## 5️⃣ Configure Argo CD Notifications
Argo CD Notifications enable automatic alerts when application events such as sync success or failure occur.

### 5.1 Secret YAML
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  email-username: your-email@gmail.com
  email-password: your-app-password
  slack-token: xxx1234567890abcdef
```
This Secret securely stores sensitive credentials used by notification services. Slack uses a bot token, not a webhook URL.

Apply:
```
kubectl apply -f argocd-notifications-secret.yaml
```

### 5.2 ConfigMap YAML
Edit the ConfigMap in the cluster:
```
kubectl edit cm -n argocd argocd-notifications-cm
```
Add / update content like below:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.email.my_server: |
    host: smtp.gmail.com
    port: 587
    from: your-email@gmail.com
    username: $email-username
    password: $email-password

  service.slack.my_slack: |
    token: $slack-token

  template.app-sync-success: |
    email:
      subject: "ArgoCD Sync Succeeded"
      body: |
        Application {{.app.metadata.name}} synced successfully.
    slack:
      message: ":white_check_mark: {{.app.metadata.name}} synced successfully"

  template.app-sync-failed: |
    email:
      subject: "ArgoCD Sync Failed"
      body: |
        Sync failed for {{.app.metadata.name}}
    slack:
      message: ":x: Sync failed for {{.app.metadata.name}}"

  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase == 'Succeeded'
      send: [app-sync-success]

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error','Failed']
      send: [app-sync-failed]
```
Defines an email notification service. Credentials are dynamically loaded from the Secret using variable references.

Apply changes (if using local YAML):
```
kubectl apply -f argocd-notifications-cm.yaml
```
Restart notifications controller:
```
kubectl rollout restart deployment argocd-notifications-controller -n argocd
```

---

## 6️⃣ Create Sample Application with Notification Annotations
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.email: my_server
    notifications.argoproj.io/subscribe.on-sync-failed.slack: my_slack
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo.git
    targetRevision: main
    path: manifests/sample-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Apply:
```
kubectl apply -f sample-app.yaml
```

Optional: Add Annotations in UI:
- Open Argo CD UI
- Select the Application → Edit → Metadata → Annotations
- Add:
```
notifications.argoproj.io/subscribe.on-sync-succeeded.email = my_server
notifications.argoproj.io/subscribe.on-sync-failed.slack = my_slack
```
Save. GitOps annotations are preferred.

---

## 7️⃣ Create Namespace for Argo Rollouts
```
kubectl create namespace argo-rollouts
```

---

## 8️⃣ Install Argo Rollouts via kubectl apply
```
kubectl apply -n argo-rollouts \
  -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
```
Verify:
```
kubectl get pods -n argo-rollouts
```

---

## 9️⃣ Expose Argo Rollouts Dashboard
Install CLI:
```
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```
Verify:
```
kubectl argo rollouts version
```
Start Dashboard:
```
kubectl argo rollouts dashboard -n argo-rollouts --port 4000
```
Access:
```
http://localhost:4000
```

---

## 🔟 Sample Canary Rollout YAML
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: canary-app
  namespace: argo-rollouts
spec:
  replicas: 3
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: { duration: 30s }
      - setWeight: 50
      - pause: { duration: 60s }
      - setWeight: 100
  selector:
    matchLabels:
      app: canary-app
  template:
    metadata:
      labels:
        app: canary-app
    spec:
      containers:
      - name: app
        image: nginx:latest
```

---

## 1️⃣1️⃣ Sample Blue-Green Rollout YAML
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: bluegreen-app
  namespace: argo-rollouts
spec:
  replicas: 2
  strategy:
    blueGreen:
      activeService: bg-active
      previewService: bg-preview
      autoPromotionEnabled: false
  selector:
    matchLabels:
      app: bg-app
  template:
    metadata:
      labels:
        app: bg-app
    spec:
      containers:
      - name: app
        image: nginx:latest
```

