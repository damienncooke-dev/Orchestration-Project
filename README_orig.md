# Kubernetes Fundamentals: Deploying the Docker Voting App on GKE
## Overview

This project demonstrates foundational Kubernetes operational concepts such as deployment, service discovery, 
persistent storage, health monitoring, and troubleshooting using Docker's Example Voting App Microservice. 
The purpose of this project was to build and strengthen practical Site Reliability Engineering skills and show production-readiness thinking.
---
## Objectives
- Deploy containerized workloads into Kubernetes
- Understand pod lifecycle management
- Implement service communication
- Configure externalized application settings
- Persist application data
- Add health checks for reliability
- Practice troubleshooting failed workloads
---
## Architecture
Application components:
- Voting App
- Redis
- Worker
- PostgreSQL
- Result App
Focus:
This repository focuses on Kubernetes orchestration, not application development.
---
## Kubernetes Concepts Demonstrated
### Deployments
Concepts:
- Pod creation
- ReplicaSets
- Scaling
- Rolling updates
- Rollbacks
Commands:
kubectl get deployments
kubectl rollout history deployment voting-app
kubectl rollout undo deployment voting-app
---
### Services
Concepts:
- ClusterIP
- NodePort
Purpose:
- internal communication
- external application access
Commands:
kubectl get svc
---
### ConfigMaps
Concepts:
- external configuration
- environment variable injection
Purpose:
Keep configuration separate from application images.
Commands:
kubectl get configmaps
---
### Persistent Volumes
Concepts:
- PersistentVolumeClaims
Purpose:
Persist PostgreSQL data across pod restarts.
Commands:
kubectl get pvc
---
### Health Checks
Concepts:
- livenessProbe
- readinessProbe
Purpose:
Ensure unhealthy containers are restarted automatically.
Commands:
kubectl describe pod <pod>
---
### Troubleshooting
Scenarios tested:
1. Broken container image
2. Failed readiness probe
Commands used:
kubectl logs <pod>
kubectl describe pod <pod>
kubectl get events
---
## Deployment Steps
### Create Cluster (GKE)
### Apply Manifests
kubectl apply -f .
### Verify Resources
kubectl get all
### Test Application
Access NodePort service.
---
## Lessons Learned
- Kubernetes deployment lifecycle
- Service communication patterns
- Persistent storage basics
- Health monitoring concepts
- Debugging failed pods
---
## Future Improvements
- Helm packaging
- Terraform cluster provisioning
- Multi-cloud deployment (AWS, Azure)
- Monitoring with Prometheus/Grafana
- Horizontal Pod Autoscaling
