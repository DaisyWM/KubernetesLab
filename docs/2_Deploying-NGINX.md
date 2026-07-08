# Kubernetes Application Deployment with NGINX

## Overview

With the Kubernetes cluster fully operational, the next step was to deploy a containerized application.

For this exercise, I deployed **NGINX** because it is lightweight, widely used, and ideal for learning the fundamentals of application deployment in Kubernetes.

This exercise demonstrates the complete deployment workflow, including creating a Deployment, managing replicas, exposing the application through a Service, and verifying that the workload is running successfully inside the cluster.

---

# Objectives

By the end of this exercise, I had successfully:

- Created an NGINX Deployment
- Verified that Kubernetes created the required Pods
- Scaled the application to multiple replicas
- Exposed the application using a NodePort Service
- Accessed the application from a web browser
- Verified that Kubernetes was successfully managing the application

---

# Understanding Deployments

A Deployment is the recommended Kubernetes resource for managing stateless applications.

Rather than creating Pods manually, a Deployment continuously ensures that the desired number of replicas are running. If a Pod fails, Kubernetes automatically creates a replacement to maintain the application's desired state.

Deployments also simplify rolling updates, rollbacks, and scaling operations, making them the preferred way to run production workloads.

---

# Creating the Deployment

I created an NGINX Deployment using the following command:

```bash
kubectl create deployment nginx-test \
    --image=nginx:latest
```

Once the Deployment was created, Kubernetes automatically generated a ReplicaSet, which in turn created the required Pods.

The Deployment controller continuously monitors the application's desired state, ensuring that the specified number of replicas remain available even if individual Pods fail.

---

# Verifying the Deployment

Before exposing the application, I verified that the Deployment, ReplicaSet, and Pods had all been created successfully.

```bash
kubectl get deployments

kubectl get replicasets

kubectl get pods -o wide
```

The Pods successfully reached the **Running** state, confirming that:

- Kubernetes scheduled the Pods onto worker nodes.
- The NGINX container image was pulled successfully.
- The kubelet started each container without errors.

---

# Scaling the Application

One of Kubernetes' biggest advantages is how easily applications can be scaled.

Instead of manually creating additional Pods, I increased the number of replicas managed by the Deployment.

```bash
kubectl scale deployment nginx-test \
    --replicas=2
```

After scaling, Kubernetes automatically created an additional Pod to satisfy the desired replica count.

```bash
kubectl get pods -o wide
```

Both Pods continued serving the same application while being managed by a single Deployment.

---

# Exposing the Application

Pods receive internal IP addresses that are only accessible within the Kubernetes cluster.

To make the application accessible externally, I exposed the Deployment using a **NodePort Service**.

```bash
kubectl expose deployment nginx-test \
    --type=NodePort \
    --port=80
```

---

# Verifying the Service

I confirmed that the Service had been created successfully.

```bash
kubectl get svc
```

The NodePort Service assigned a port within Kubernetes' default NodePort range (30000–32767), allowing the application to be accessed from outside the cluster.

---

# Inspecting the Deployment

Finally, I inspected all Kubernetes resources associated with the application.

```bash
kubectl get all
```

I also viewed the Deployment details to confirm that the desired replica count matched the running state.

```bash
kubectl describe deployment nginx-test
```

---

# Deployment Verification

After creating the Deployment and exposing it through a NodePort Service, I verified that all Kubernetes resources had been created successfully.

The output below confirms that the Deployment, ReplicaSet, Pods, and Service were all healthy.

![Deployment Verification](../images/nginx-deployment-verification.png)

---

# Accessing the Application

Once the Service was available, I accessed the application from a web browser using the worker node's IP address and the assigned NodePort.

For this lab, the application was available at:

```text
http://172.25.152.65:32019
```

Opening the URL displayed the default NGINX welcome page, confirming that:

- The Deployment was running successfully.
- The NodePort Service was forwarding traffic correctly.
- The application was accessible from outside the Kubernetes cluster.

![NGINX Welcome Page](../images/nginx-browser.png)

---

# Challenges Encountered

This deployment completed successfully without any major issues.

The verification steps helped reinforce how Kubernetes automatically schedules Pods, maintains the desired replica count, and keeps workloads available even as changes are made to the Deployment.

---

# Outcome

This exercise marked my first successful application deployment within the Kubernetes cluster.

By the end of the exercise, Kubernetes had successfully:

- Created and managed the required Pods.
- Maintained the desired replica count.
- Exposed the application through a NodePort Service.
- Made the application accessible from outside the cluster.

This demonstrated how Kubernetes simplifies application deployment while providing built-in capabilities for scaling, self-healing, and service discovery.

---

# Key Takeaways

This exercise strengthened my understanding of several fundamental Kubernetes concepts:

- Deployments manage the lifecycle of applications.
- ReplicaSets maintain the desired number of running Pods.
- Pods are the smallest deployable units in Kubernetes.
- Services provide stable networking and access to applications.
- Applications can be scaled with a single command.
- Kubernetes continuously maintains the desired state by automatically replacing failed Pods.

This deployment provided the foundation for the next stages of the project, including deploying the Kubernetes Dashboard, configuring monitoring with Prometheus and Grafana, setting up a Harbor private registry, and deploying private container images from Harbor.