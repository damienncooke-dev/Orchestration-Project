# Kubernetes Fundamentals: Deploying the Docker Voting App on GKE

This project demonstrates foundational Kubernetes operational concepts such as deployment, service discovery, 
persistent storage, health monitoring, and troubleshooting using Docker's Example Voting App Microservice. 
The purpose of this project was to build and strengthen practical Site Reliability Engineering skills and show production-readiness thinking.

---

## Why This Project

GitLab Dedicated provisions and operates **isolated, single-tenant GitLab environments** for customers at scale. The team's core challenge is making many environments reliable, consistent, and automatable — using Kubernetes as the operational backbone.

This project demonstrates the foundational Kubernetes skills that underpin that work: deploying multi-service workloads, managing configuration separately from code, exposing services reliably, and understanding what to do when things go wrong.

---

## Application Stack (Pre-Built — Not Modified)
The YAML manifests (Pods, Services, Deployments) live in this repository. The application code is not the focus — the **Kubernetes operational layer** is.

**Time to complete:** ~2 days  
**Cloud provider:** Google Cloud Platform (GKE)  
**Application:** [Docker Example Voting App Microservice](https://github.com/dockersamples/example-voting-app)

| Component | Technology | Role |
|---|---|---|
| vote | Python | Frontend voting UI |
| redis | Redis | In-memory vote queue |
| worker | .NET | Processes votes from Redis to PostgreSQL |
| db | PostgreSQL | Persistent vote store |
| result | Node.js | Displays vote results |

---

## Table of Contents

1. [Concepts Demonstrated](#1-concepts-demonstrated)
2. [Prerequisites](#2-prerequisites)
3. [Cluster Setup — GKE](#3-cluster-setup--gke)
4. [Namespaces — Logical Isolation](#4-namespaces--logical-isolation)
5. [Deploying the Application](#5-deploying-the-application)
6. [Services — Internal and External Networking](#6-services--internal-and-external-networking)
7. [ConfigMaps and Secrets](#7-configmaps-and-secrets)
8. [Persistent Storage for PostgreSQL](#8-persistent-storage-for-postgresql)
9. [Health Checks — Liveness and Readiness Probes](#9-health-checks--liveness-and-readiness-probes)
10. [Rolling Updates and Rollback](#10-rolling-updates-and-rollback)
11. [Debugging — What Breaks and How to Fix It](#11-debugging--what-breaks-and-how-to-fix-it)
12. [Cleanup](#12-cleanup)
13. [What I Would Do Next](#13-what-i-would-do-next)

---

## 1. Concepts Demonstrated

| Kubernetes Concept | How It Appears in This Project                         |
|---|--------------------------------------------------------|
| Pods | Each app component runs as a Pod                       |
| Deployments | Manages desired replica count and rollout behavior     |
| ReplicaSets | Automatically maintained by Deployments                |
| Services (ClusterIP) | Internal communication between components              |
| Services (LoadBalancer) | External access to vote and result UIs                 |
| Namespaces | All components deployed into `voting` namespace        |
| ConfigMaps | Non-sensitive configuration injected into pods         |
| Secrets | PostgreSQL credentials stored as a Kubernetes Secret   |
| PersistentVolumeClaim | PostgreSQL data survives pod restarts                  |
| Liveness Probe | Kubernetes restarts unhealthy pods automatically       |
| Readiness Probe | Traffic only routed to pods that are ready             |
| Rolling Update | Deploy a new version with zero downtime                |
| Rollback | Revert to a previous working version                   |
| kubectl debugging | Diagnosing CrashLoopBackOff, ImagePullBackOff, Pending |

---
<br>

## 2. Prerequisites

### Kubernetes Manifest Development and POC
> Development of Kubernetes YAML manifests and proof-of-concept on local machine.
* **Hypervisor:** used for creating single node deployment of Kubernetes components and container runtime.
  * *MacOS `vfkit` is the chosen hypervisor because it is the native MacOS virtualization framework and bypasses the complexities of setting up  socket layer configurations and permissions.*
* **Package Manager:** used for installation of the required software packages for this project.
  * *Homebrew is the preferred PM on MacOS.  For Ubuntu/Debian systems, `apt` is commonly used.* 
* **CLI Utility:** installation of `kubectl` utility to interact with Kubernetes cluster.
  * *`kubectl` deployment instructions followed:* [Kubectl Install Instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)
* **MiniKube:** lightweight Kubernetes package that contains all the necessary components to run Kubernetes locally.
  * *Minikube deployment instructions followed:* [MiniKube Install Instructions](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download)
* **GitHub Repository:** Remote repository containing the manifest files for this project.


### Cloud Deployment at Scale
> Deployment of a microservice application at scale using Kubernetes in a cloud environment.
* **GitHub Repository:** for deploying project artifacts into GCP
* **GCP Cloud Account (free tier):** Minimum resources required to implement the following:
  * *A way to  create, manage, and scale virtual machines (VMs).* 
  * *Access to GKE to provision a multi-node cluster (3) to deploy the application at scale.*
  * *Ability to communication with cluster via CLI (`kubectl`).*
  * *Deploy the microservices and manage the application lifecycle using Kubernetes.*
  * *Expose application to external users using Kubernetes Services.*
* **Create a GCP Project:** 
  * *Provide a name for the project.*
---
<br>

## 3. Cluster Setup — GKE

**Concept:** A Kubernetes cluster is the foundation — a set of nodes (VMs) managed by a control plane that schedules and runs your workloads.

GKE is a *managed* Kubernetes service: Google operates the control plane, you manage what runs on it. 

### Create the Cluster

Cluster creation is a one-time process done through the GCP Dashboard.
1. Go to the [GCP Console](https://console.cloud.google.com/).
2. Provide loging and authorization credentials.
3. From the main navigation menu in the upper left hand corner, select **Kubernetes Engine**.
4. In this space you will create a standard GKE cluster. If asked **Enable Kubernetes API**.
5. Select **Create Cluster**. (See alternate instructions below for manual cluster creation)
6. Your default cluster will be the "GKE AutoPilot Cluster", in the upper right hand corner select **Switch to Standard Cluster**.
7. Provide a name for your cluster: [`vote-app-deployment`].
8. Provide a region, in this case keep default "Zonal:" [`us-central1`]. (Free tier status does not allow for regional clusters due to quota limits)
9. Configure the following under Node Pool "default-pool" and "Nodes" settings:
   * **[default-pool]: Node Count:** 3
   * **[default-pool]: Enable Spot VMs** (VMs can be reclaimed at any time)
   * **[Nodes]: Machine Type:** `General Purpose, e2-small(2 vCPUs, 1 core, 2 GB RAM)`
   * **[Nodes]: Disk Size:** `10` GB
10. Click **Create**.
11. Cluster creation can take a few minutes. 
12. After successful creation of the cluster, you will the cluster details in the dashboard with 3 nodes, 6 virtual CPUs, and 12 GB of RAM. 
13. To see node details, click on the cluster name and select the tab **Nodes**.


Alternatively, the GCP Cloud Shell can be used to create the cluster starting from step 5 above. 
1. From the upper right hand corner of the dashboard, select **Cloud Shell**.
2. Once the shell initializes, enter the following command:
```bash
ggcloud container clusters create voting-app-deployment \
    --zone=us-central1-a \
    --num-nodes=3 \
    --machine-type=e2-small \
    --spot \
    --disk-size=30
    
```
3. After successful creation of the cluster, you will see the cluster details in the dashboard with 3 nodes, 6 vCPUs, 6 GB of memory.


### Connect kubectl to the Cluster
1. The cluster can be accessed via the `gcloud` CLI utility.
2. From the UI menu at the top, select **Connect** and choose **Command-line access** and **Run in Cloud Shell**.
3. The GCP Cloud Shell will open up and will be pre-prompted with the command to connect `kubectl` to the cluster.
```bash
gcloud container clusters get-credentials voting-app-deployment --zone us-central1-a --project <accept-default-project-id>
```

### Verify the Cluster is Healthy
```bash
kubectl get nodes
```
```
💡 Pro Tip: It gets very tiresome typing 'kubectl' all the time...

alias k='kubectl'

k get nodes

```
Expected output: three nodes in `Ready` status.

```
NAME                                   STATUS   ROLES    AGE   VERSION
gke-voting-cluster-default-pool-...   Ready    <none>   2m    v1.28.x
gke-voting-cluster-default-pool-...   Ready    <none>   2m    v1.28.x
gke-voting-cluster-default-pool-...   Ready    <none>   2m    v1.28.x
```

### Clone the release 

```bash
git clone https://github.com/damienncooke-dev/Orchestration-Project.git
```
---
<br>

## 4. Namespaces — Logical Isolation

**Concept:** Namespaces divide a cluster into isolated virtual workspaces. In GitLab Dedicated, different tenant environments live in isolated spaces — this is a foundational pattern, even at small scale.

### Create the Namespace
```bash
kubectl create namespace voting
```

### Set it as the Default for This Session
```bash
kubectl config set-context $(kubectl config current-context) --namespace=voting
```

> **Why this matters:** Without namespaces, everything lands in `default`. In a team or multi-tenant environment, we need to isolate workloads from each other.

### Verify
```bash
kubectl get namespaces
```

---
<br>

## 5. Deploying the Application

**Concepts:** Deployments declare the *desired state* of a workload — how many replicas, which image, what resources. Kubernetes continuously works to match reality to that declaration. ReplicaSets are created automatically by the Deployment to maintain the replica count.


### Apply All Deployments
```bash
kubectl apply -f Orchestration-Project/K8s-app-deploy/Deployments/ -n voting
```

### Watch the Pods Come Up
```bash
kubectl get pods -w
```

Wait until all pods show `Running` with `1/1` under `READY`. Ctrl-C to stop watching.

## Uh-oh! 
### Inspect a Deployment

```bash
kubectl get depoyments 
```

```
The db deployment is failing to start! 

NAME     READY   UP-TO-DATE   AVAILABLE   AGE
db       0/1     1            0           3m28s
redis    1/1     1            1           3m28s
result   1/1     1            1           3m28s
vote     1/1     1            1           3m28s
worker   1/1     1            1           3m28s

...checking the pod events

(~)$ kubectl get pods

NAME                      READY   STATUS                       RESTARTS   AGE
db-5867dd7898-8lmp2       0/1     CreateContainerConfigError   0          29s
redis-65cd7744b9-whtld    1/1     Running                      0          29s
result-5dcb6664d4-27j9c   1/1     Running                      0          29s
vote-7cff9b8dfc-zdg8g     1/1     Running                      0          29s
worker-545d464b96-qnxmn   1/1     Running                      0          28s

(~)$ kubectl describe db-5867dd7898-8lmp2

Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  6m33s                 default-scheduler  Successfully assigned voting/db-5867dd7898-8lmp2 to gke-voting-app-deploymen-default-pool-02a1fe9b-lj02
  Normal   Pulled     75s (x26 over 6m32s)  kubelet            spec.containers{postgres}: Container image "postgres:15-alpine" already present on machine and can be accessed by the pod
  Warning  Failed     75s (x26 over 6m32s)  kubelet            spec.containers{postgres}: Error: secret "db-secrets" not found
  
 The secres has not been created yet, and this is expected. It will be addressed in an upcoming section (seting 7).

```

Note the sections: desired replicas, current replicas, image, events. This is the first place to look when something isn't working.

### Key Relationship to Understand With Deployments

Deployments manage ReplicaSets and the desired number of pods running in the cluster.

```
Deployment (desired state: 2 replicas of vote)
    └── ReplicaSet (maintains 2 pods)
            ├── Pod (vote-abc123)
            └── Pod (vote-def456)
```

If you delete a pod manually, the ReplicaSet creates a replacement immediately. Try it:
```bash
# Scale up the voting app to 2 replicas
kubectl scale deployment vote --replicas=2 

# Get a pod name
kubectl get pods   # Two pods are now running

# Delete it  
kubectl delete pod <vote-pod-name>   # Note the last 5 digits of the pod name

# Watch it get recreated 
kubectl get pods -w  # Ctrl-C to stop watching.

```
The pod will be recreated with a new name. This demonstrates that 2 pods will always be running. 



---
<br>

## 6. Services — Internal and External Networking

**Concept:** Pods are temporary and get new IP addresses when restarted. Services provide a stable network endpoint in front of pods, using label selectors to find the right pods automatically.

There are two types in use here:
- **ClusterIP** — internal only, for pod-to-pod communication (Redis, PostgreSQL, worker)
- **LoadBalancer** — provisions a cloud load balancer with an external IP (vote UI, result UI)

### Apply Services
```bash
kubectl apply -f manifests/services/ -n voting
```

### Verify
```bash
kubectl get services -n voting
```

The vote and result services will show an `EXTERNAL-IP`. It may take 1–2 minutes for GCP to provision the load balancer.

### Test Internal DNS Resolution

Kubernetes automatically gives every Service a DNS name in the format `<service-name>.<namespace>.svc.cluster.local`. This is how the worker finds Redis and PostgreSQL without hardcoded IPs.

```bash
# Exec into a running pod
kubectl exec -it <vote-pod-name> -n voting -- /bin/sh

# Inside the pod — test that redis is reachable by DNS name
nslookup redis
# or
cat /etc/resolv.conf
```

> **Why this matters for GitLab Dedicated:** In a multi-tenant environment, services in one namespace are isolated from those in another. Understanding how Service DNS works — and where it doesn't reach — is core to understanding environment isolation.

---
<br>


## 7. ConfigMaps and Secrets

**Concept:** Configuration and credentials should never be baked into container images. ConfigMaps hold non-sensitive config. Secrets hold sensitive values (passwords, tokens). Both inject values into pods at runtime, keeping images portable and environment-agnostic.

### Create a ConfigMap for Redis Hostname
```bash
kubectl create configmap app-config \
  --from-literal=REDIS_HOST=redis \
  -n voting
```

### Create a Secret for PostgreSQL Credentials
```bash
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_PASSWORD=your_password \
  --from-literal=POSTGRES_USER=postgres \
  -n voting
```

### Verify
```bash
kubectl get configmaps -n voting
kubectl get secrets -n voting

# Inspect (values are base64-encoded, not plaintext)
kubectl describe secret db-credentials -n voting
```

### Reference in a Deployment (example)
```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: POSTGRES_PASSWORD
```

> **Note:** Kubernetes Secrets are base64-encoded, not encrypted. In production (including GitLab Dedicated), secrets are backed by cloud KMS (GCP Secret Manager, AWS Secrets Manager, Azure Key Vault). This project demonstrates the Kubernetes-native pattern; the cloud-native enhancement is the next step.

---
<br>


## 8. Persistent Storage for PostgreSQL

**Concept:** Pods are ephemeral — when a pod is deleted, its local storage is gone with it. PostgreSQL needs to retain data across pod restarts. PersistentVolumeClaims (PVCs) request durable storage from the cluster; GKE provisions a GCP Persistent Disk automatically.

### Apply the PVC
```yaml
# manifests/pvc/postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: voting
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f manifests/pvc/postgres-pvc.yaml -n voting
kubectl get pvc -n voting
```

Watch the STATUS column change from `Pending` to `Bound`. GKE is dynamically provisioning the disk.

### Mount It in the PostgreSQL Deployment
```yaml
volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
containers:
  - name: db
    volumeMounts:
      - mountPath: /var/lib/postgresql/data
        name: postgres-storage
```

### Verify Persistence
```bash
# Cast a few votes through the UI, then delete the db pod
kubectl delete pod <db-pod-name> -n voting

# Kubernetes recreates it, reattaching the same PVC
kubectl get pods -n voting -w

# Open the result UI — votes should still be there
```

---
<br>


## 9. Health Checks — Liveness and Readiness Probes

**Concept:** Kubernetes doesn't know if your application is actually healthy just because the process is running. Probes let you define what "healthy" means. Liveness probes restart a broken pod. Readiness probes control whether a pod receives traffic.

### Add Probes to the vote Deployment
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

```bash
kubectl apply -f manifests/deployments/vote-deployment.yaml -n voting
```

### Observe Probe Behavior

**Simulate a readiness failure** (misconfigure the path temporarily):
```yaml
readinessProbe:
  httpGet:
    path: /healthz   # This path doesn't exist
    port: 80
```

```bash
kubectl apply -f manifests/deployments/vote-deployment.yaml -n voting
kubectl get pods -n voting   # Pod stays Running but READY shows 0/1
kubectl describe pod <vote-pod> -n voting   # Check Events section
```

The pod is running but removed from the Service endpoints — no traffic reaches it.

Restore the correct path and reapply to resolve.

> **Key distinction to document:**
> - Liveness failure → pod is killed and restarted
> - Readiness failure → pod stays running but receives zero traffic

---
<br>


## 10. Rolling Updates and Rollback

**Concept:** Kubernetes can update a running Deployment to a new image version with zero downtime by gradually replacing old pods with new ones. If the new version is broken, you can roll back to the previous version instantly.

### Trigger a Rolling Update
```bash
# Simulate updating to a new image version
kubectl set image deployment/vote vote=vote:v2 -n voting

# Watch the rollout
kubectl rollout status deployment/vote -n voting
```

Kubernetes brings up new pods before terminating old ones — traffic continues during the update.

### View Rollout History
```bash
kubectl rollout history deployment/vote -n voting
```

### Simulate a Bad Deploy
```bash
# Set a non-existent image tag
kubectl set image deployment/vote vote=vote:this-tag-does-not-exist -n voting

# Watch it fail
kubectl get pods -n voting -w
# New pods will show ImagePullBackOff
```

### Roll Back
```bash
kubectl rollout undo deployment/vote -n voting

# Confirm recovery
kubectl rollout status deployment/vote -n voting
kubectl get pods -n voting
```

> **Why this matters:** This is the fundamental operational loop in any Kubernetes environment — deploy, verify, rollback if needed. GitLab Dedicated automates this at scale across hundreds of environments; understanding the manual version is the prerequisite.

---
<br>


## 11. Debugging — What Breaks and How to Fix It

**Concept:** Knowing *how* to read failure signals is as important as knowing how to deploy. These are the most common failure modes you will encounter in real Kubernetes work.

### The Four Common Failures

#### CrashLoopBackOff
The container starts, crashes, Kubernetes restarts it, it crashes again.

```bash
kubectl get pods -n voting           # See CrashLoopBackOff in STATUS
kubectl logs <pod-name> -n voting    # Read the crash output
kubectl logs <pod-name> -n voting --previous   # Logs from the last crash
kubectl describe pod <pod-name> -n voting      # Check Events and Last State
```

**Common causes:** bad command/entrypoint, missing environment variable, failed DB connection on startup.

#### ImagePullBackOff
Kubernetes can't pull the container image.

```bash
kubectl describe pod <pod-name> -n voting
# Look at Events: "Failed to pull image ... not found"
```

**Common causes:** typo in image name or tag, private registry credentials missing.

#### OOMKilled
The container exceeded its memory limit and was killed by the kernel.

```bash
kubectl describe pod <pod-name> -n voting
# Look for: Last State: Terminated, Reason: OOMKilled
```

**Fix:** Increase the memory limit in the Deployment spec.

#### Pending (Unschedulable)
The pod cannot be placed on any node.

```bash
kubectl describe pod <pod-name> -n voting
# Look for: Events: 0/3 nodes are available: Insufficient cpu
```

**Common causes:** resource requests too high for available nodes, or no nodes with the required labels.

### Debugging Quick Reference

| Symptom | First Command | What to Look For |
|---|---|---|
| Pod not starting | `kubectl describe pod <name>` | Events section at the bottom |
| Pod crashing | `kubectl logs <name> --previous` | Crash output / error message |
| Can't reach service | `kubectl get endpoints <svc>` | No endpoints = no ready pods |
| General confusion | `kubectl get events --sort-by=.lastTimestamp` | Most recent cluster events |

---
<br>


## 12. Cleanup

When finished, tear down the cluster to avoid ongoing GCP charges.

```bash
# Delete all resources in the namespace first
kubectl delete namespace voting

# Delete the GKE cluster
gcloud container clusters delete voting-cluster --zone us-central1-a
```

---
<br>

## 13. What I Would Do Next

This project covers the foundational layer. The following represents the natural progression into intermediate and advanced SRE Kubernetes work — documented in [`README-advanced.md`](./README-advanced.md):

| Next Step | Concept |
|---|---|
| Add HorizontalPodAutoscaler | Scale vote pods automatically under load |
| Add RBAC | Create scoped ServiceAccounts and Roles |
| Add NetworkPolicy | Restrict pod-to-pod communication by namespace |
| Deploy to AWS EKS and Azure AKS | Multi-cloud portability, StorageClass differences |
| Provision cluster with Terraform | Declarative, repeatable infrastructure-as-code |
| Add Helm | Package and template the full app as a reusable chart |
| Add Prometheus + Grafana | Production-grade observability beyond `kubectl top` |

The advanced outline is intentionally documented to show awareness of where this work leads — not as a claim that it's already complete.

---
<br>

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Voting App](https://github.com/dockersamples/example-voting-app)
- [GKE Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

*This project is part of a portfolio demonstrating SRE competencies. Manifests are in `manifests/`. The advanced multi-cloud version is in `README-advanced.md`.*
