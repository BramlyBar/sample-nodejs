# Helm Chart for Sample Node.js App

This document explains the steps, decisions, and features included in the Helm chart for deploying the Sample Node.js application to Kubernetes.

## 1. Workload Type: Deployment vs. StatefulSet

The workload is deployed using a **Deployment** rather than a StatefulSet. 

**Reasoning:**
This Node.js application is completely **stateless**. It does not rely on local persistent storage, require strict ordering of pod creation/termination, or need a stable network identity (which are the primary use cases for StatefulSets). A Deployment provides the best fit, allowing for easy horizontal scaling, seamless rolling updates, and automatic replacement of failed instances without state-related overhead.

## 2. Helm Chart Features

The chart was built to maximize the use of Kubernetes features based on the application's underlying code (`app.js`). Here are the key components included:

### Probes for Health Checking
Configured in the `deployment.yaml`, these probes use the endpoints explicitly defined in the Node.js application:
- **Readiness Probe:** Checks the `/ready` endpoint. Kubernetes uses this to ensure the app only receives external traffic when it is fully initialized and ready to handle it.
- **Liveness Probe:** Checks the `/live` endpoint. Kubernetes uses this to detect if the container hangs or crashes, automatically restarting it if the probe fails.

### Resource Requests and Limits
Added to ensure the cluster schedules the pods properly and to prevent a single pod from consuming excessive memory or CPU and starving other apps. 
- **Requests:** `100m` CPU / `128Mi` RAM (Guaranteed resources for the pod).
- **Limits:** `250m` CPU / `256Mi` RAM (Maximum resources the pod is allowed to use).
*(These values are easily configurable in `values.yaml`)*.

### Networking
- **Service:** Creates a `ClusterIP` service to expose port `8080` internally within the Kubernetes cluster, allowing other services to communicate with it.
- **Ingress:** An Ingress template is included and enabled by default in `values.yaml` for `sample-nodejs.local`. This makes it easy to route external HTTP/HTTPS traffic directly into the application.

### Configuration Management
- **ConfigMap:** A `ConfigMap` template (`configmap.yaml`) was added to manage environment variables gracefully. It feeds the `PORT` variable from `values.yaml` directly into the pod, which the Node.js app reads via `process.env.PORT`.
