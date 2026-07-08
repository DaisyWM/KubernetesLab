# Kubernetes Dashboard

## Overview

After deploying my first application, I wanted a more intuitive way to inspect and manage the Kubernetes cluster.

Although `kubectl` provides complete control through the command line, the Kubernetes Dashboard offers a web-based interface that makes it much easier to visualize workloads, monitor cluster resources, and inspect running applications.

In this exercise, I deployed the Kubernetes Dashboard, configured secure access using RBAC, and verified that I could successfully manage the cluster through the graphical interface.

---

# Objectives

By the end of this exercise, I had successfully:

- Deployed the Kubernetes Dashboard
- Verified that all dashboard components were running
- Created an administrator ServiceAccount
- Configured RBAC permissions
- Generated an authentication token
- Accessed the dashboard through a web browser
- Explored workloads and cluster resources

---

# Installing the Kubernetes Dashboard

The Kubernetes Dashboard was deployed using the official Kubernetes manifest.

This manifest automatically creates the required Deployments, Services, Pods, and supporting resources inside the `kubernetes-dashboard` namespace.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Once the deployment completed, I verified that all dashboard components were running successfully.

```bash
kubectl get pods -n kubernetes-dashboard
```

I waited until every Pod reached the **Running** state before proceeding.

The deployment included components such as:

- kubernetes-dashboard
- dashboard-metrics-scraper

---

# Creating an Administrator Account

The Kubernetes Dashboard requires authentication before users can access cluster resources.

For this lab, I created a dedicated ServiceAccount with **cluster-admin** privileges.

> **Note:** Granting cluster-admin permissions is convenient for learning environments but should not be used in production clusters.

First, I created the ServiceAccount.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Next, I created a ClusterRoleBinding to associate the ServiceAccount with the `cluster-admin` role.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Finally, I generated a login token for the ServiceAccount.

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

The generated token was copied and used to authenticate with the dashboard.

---

# Accessing the Dashboard

After deploying the Kubernetes Dashboard and configuring authentication, I accessed the web interface using the NodePort service.

For this lab, the dashboard was available at:

```text
https://172.25.152.66:31724
```

Because the dashboard uses a self-signed TLS certificate, the browser displayed a security warning. After accepting the warning, I authenticated using the administrator token generated during the previous step.

---

# Dashboard Login

The Kubernetes Dashboard login page loaded successfully, confirming that the deployment and networking configuration were correct.

Authentication was completed using the administrator ServiceAccount token.

![Dashboard Login](../images/dashboard-login.png)

---

# Exploring Cluster Resources

After signing in, I navigated through several areas of the dashboard to verify the health and status of the cluster.

The dashboard provided a graphical overview of:

- Running workloads
- CPU and memory utilization
- Pod health and restart counts
- Node placement
- Deployments
- ReplicaSets
- Services
- Namespaces

This provided a much clearer view of the cluster compared to using only command-line tools.

---

# Viewing Running Workloads

One of the first areas I inspected was the **Pods** view.

Here I confirmed that the existing application workloads, together with the Harbor-hosted deployment, were all running successfully.

![Kubernetes Dashboard - Pods](../images/dashboard-pods.png)

---

# Verifying the Dashboard Deployment

To confirm that the dashboard had been deployed correctly, I verified all resources in the `kubernetes-dashboard` namespace.

```bash
kubectl get all -n kubernetes-dashboard
```

This confirmed that all dashboard components were healthy and ready for use.

---

# Challenges Encountered

Installing the Kubernetes Dashboard was straightforward, but I quickly learned that deployment alone is not enough.

Without creating a ServiceAccount and assigning the appropriate RBAC permissions, it is not possible to authenticate successfully.

This exercise reinforced the importance of Kubernetes authentication and authorization mechanisms and highlighted the role that RBAC plays in securing cluster access.

---

# Outcome

By the end of this exercise, I had successfully deployed and configured the Kubernetes Dashboard for my cluster.

The dashboard provided a convenient graphical interface for inspecting workloads, monitoring resource usage, and managing Kubernetes objects without relying entirely on `kubectl`.

This significantly improved the visibility of the cluster and complemented the command-line tools used throughout the rest of the project.

---

# Key Takeaways

This exercise strengthened my understanding of several important Kubernetes concepts:

- Deploying applications using Kubernetes manifests
- Authenticating users with ServiceAccounts
- Managing permissions using RBAC
- Generating authentication tokens
- Monitoring workloads through the Kubernetes Dashboard
- Combining graphical tools with command-line management

The Kubernetes Dashboard became a valuable tool throughout the remainder of this project, particularly when validating Harbor deployments and monitoring workloads running inside the cluster.

---

# Optional Enhancement: Username/Password Authentication

## Why I Added This

By default, the Kubernetes Dashboard requires users to authenticate using a ServiceAccount token.

Although this works well, tokens generated using:

```bash
kubectl create token admin-user
```

expire after approximately one hour. This means a new token must be generated every time it expires, which becomes inconvenient during repeated testing.

To improve the login experience for my lab environment, I implemented an **NGINX reverse proxy** that performs HTTP Basic Authentication before forwarding authenticated requests to the Kubernetes Dashboard.

Instead of entering a Kubernetes token every time, I can simply log in with a username and password.

> **Note**
>
> This solution is intended for learning and home lab environments only.
>
> Every authenticated user receives the permissions of the `admin-user` ServiceAccount (`cluster-admin`), so this approach should **not** be used in production. Production environments should integrate the Dashboard with an identity provider such as OIDC or LDAP.

---

# Architecture

The authentication flow now looks like this:

```text
Browser
    │
Username + Password
    │
    ▼
NGINX Reverse Proxy
(Basic Authentication)
    │
Injects Kubernetes Bearer Token
    │
    ▼
Kubernetes Dashboard
    │
    ▼
Kubernetes API Server
```

The Dashboard itself still uses token authentication internally, but users no longer have to enter the token manually because the proxy injects it automatically.

---

# Creating a Long-Lived ServiceAccount Token

The first step was creating a non-expiring ServiceAccount token.

Unlike tokens generated using `kubectl create token`, this token is stored as a Kubernetes Secret and can be reused by the proxy.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-token
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
EOF
```

---

# Creating Username and Password Credentials

The proxy protects access using HTTP Basic Authentication.

I created an `htpasswd` file containing a username and password, then stored it as a Kubernetes Secret.

```bash
htpasswd -c auth admin

kubectl create secret generic dashboard-basic-auth \
  --from-file=auth \
  -n kubernetes-dashboard
```

---

# Deploying the Authentication Proxy

The authentication proxy is deployed using a single Kubernetes manifest containing:

- ConfigMap
- Deployment
- Service

The Deployment starts an NGINX container that:

- prompts users for a username and password
- validates the credentials using Basic Authentication
- injects the ServiceAccount token into every request
- forwards authenticated requests to the Kubernetes Dashboard

The complete manifest is included in this repository.

```bash
kubectl apply -f manifests/dashboard-basic-auth-proxy.yaml
```

---

# Verifying the Deployment

After deploying the proxy, I verified that it started successfully.

```bash
kubectl get pods -n kubernetes-dashboard | grep dashboard-proxy
```

I also confirmed that the ServiceAccount token had been injected correctly into the generated NGINX configuration.

```bash
kubectl exec -n kubernetes-dashboard \
deploy/dashboard-proxy -- \
cat /etc/nginx/conf.d/default.conf
```

The configuration should contain the ServiceAccount token instead of the `${TOKEN}` placeholder.

---

# Accessing the Dashboard

The proxy exposes the Dashboard through a NodePort Service.

```
http://<node-ip>:31800
```

Instead of the Kubernetes token login screen, the browser first prompts for:

- Username
- Password

After successful authentication, the Dashboard loads automatically without requiring the user to paste a Kubernetes token.

---

## Authentication Prompt

The browser prompts for HTTP Basic Authentication before forwarding requests to the Dashboard.


---

## Dashboard After Login

After entering the correct username and password, the Dashboard opens directly without displaying the Kubernetes token login page.

---

# Troubleshooting

If the proxy could not be accessed, I verified the following:

```bash
kubectl get svc dashboard-proxy -n kubernetes-dashboard
```

```bash
kubectl get pods -n kubernetes-dashboard
```

```bash
kubectl logs -n kubernetes-dashboard deploy/dashboard-proxy
```

```bash
kubectl get nodes -o wide
```

```bash
curl http://localhost:31800
```

These checks helped verify that the proxy was running correctly and reachable from the cluster.

---

# What I Learned

Implementing this authentication proxy helped me better understand how Kubernetes authentication works behind the scenes.

Rather than modifying the Dashboard itself, I placed an NGINX reverse proxy in front of it to handle authentication and automatically inject the required Kubernetes bearer token.

This exercise reinforced concepts around:

- Kubernetes ServiceAccounts
- Secrets
- RBAC
- Reverse proxies
- HTTP Basic Authentication
- NGINX configuration