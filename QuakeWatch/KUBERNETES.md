# QuakeWatch - Kubernetes Deployment Guide

This guide provides instructions for deploying the QuakeWatch application to a Kubernetes cluster using Docker Desktop.

## Prerequisites

- Docker Desktop installed with Kubernetes enabled
- kubectl command-line tool installed
- Docker image pushed to Docker Hub: `liron29/quakewatch:latest`

## Enable Kubernetes in Docker Desktop

1. Open Docker Desktop
2. Go to Settings/Preferences
3. Navigate to Kubernetes tab
4. Check "Enable Kubernetes"
5. Click "Apply & Restart"
6. Wait for Kubernetes to start (green indicator in bottom-left)

Verify Kubernetes is running:
```bash
kubectl cluster-info
kubectl get nodes
```

## Kubernetes Architecture Overview

The QuakeWatch application uses the following Kubernetes resources:

- **Deployment**: Manages 3 replicas of the QuakeWatch application with rolling updates
- **Service**: LoadBalancer type service exposing the app on port 80
- **HorizontalPodAutoscaler**: Auto-scales pods based on CPU and memory usage
- **ConfigMap**: Stores application configuration
- **Secret**: Stores sensitive data like API keys
- **CronJob**: Runs health checks every 5 minutes from inside the cluster

## Deployment Instructions

### Step 1: Navigate to Project Directory

```bash
cd /path/to/QuakeWatch
```

### Step 2: Apply Kubernetes Manifests

Apply all manifests in the correct order:

```bash
# Create ConfigMap and Secret first
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml

# Deploy the application
kubectl apply -f k8s/deployment.yaml

# Create the service
kubectl apply -f k8s/service.yaml

# Set up autoscaling
kubectl apply -f k8s/hpa.yaml

# Create the health check CronJob
kubectl apply -f k8s/cronjob.yaml
```

Or apply all at once:
```bash
kubectl apply -f k8s/
```

### Step 3: Verify Deployment

Check deployment status:
```bash
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get hpa
kubectl get cronjobs
```

Watch pods come online:
```bash
kubectl get pods -w
```

Check pod logs:
```bash
kubectl logs -l app=quakewatch
```

### Step 4: Access the Application

Get the service external IP:
```bash
kubectl get service quakewatch-service
```

For Docker Desktop, the service will be available at:
```
http://localhost
```

## Kubernetes Resources Explained

### Deployment (deployment.yaml)

The deployment manages 3 replicas of the QuakeWatch application with:
- **Liveness Probe**: Checks if the app is running (restarts container if failing)
  - Endpoint: `GET /`
  - Initial delay: 30 seconds
  - Check interval: 10 seconds
- **Readiness Probe**: Checks if the app is ready to serve traffic
  - Endpoint: `GET /`
  - Initial delay: 10 seconds
  - Check interval: 5 seconds
- **Resource Limits**:
  - Requests: 256Mi memory, 250m CPU
  - Limits: 512Mi memory, 500m CPU

### Service (service.yaml)

LoadBalancer service exposing the application:
- External port: 80
- Internal port: 5000
- Automatically routes traffic to healthy pods

### HorizontalPodAutoscaler (hpa.yaml)

Auto-scales pods between 3-10 replicas based on:
- CPU utilization target: 70%
- Memory utilization target: 80%

View HPA status:
```bash
kubectl get hpa quakewatch-hpa
```

### CronJob (cronjob.yaml)

Runs health checks every 5 minutes:
- Uses curl to test service availability from inside the cluster
- Checks endpoint: `http://quakewatch-service/`
- Keeps history of last 3 successful and 3 failed jobs

View CronJob status:
```bash
kubectl get cronjobs
kubectl get jobs
```

View health check logs:
```bash
kubectl logs -l job-name=<job-name>
```

### ConfigMap (configmap.yaml)

Stores application configuration:
- FLASK_ENV: production
- LOG_LEVEL: INFO

### Secret (secret.yaml)

Stores sensitive data (base64 encoded):
- flask-secret-key: Example secret key

## Useful Commands

### Scaling

Manually scale the deployment:
```bash
kubectl scale deployment quakewatch --replicas=5
```

### Updates

Update the application image:
```bash
kubectl set image deployment/quakewatch quakewatch=liron29/quakewatch:v2
```

Rolling update status:
```bash
kubectl rollout status deployment/quakewatch
```

Rollback to previous version:
```bash
kubectl rollout undo deployment/quakewatch
```

### Monitoring

Watch pod status:
```bash
kubectl get pods -w
```

Describe a pod:
```bash
kubectl describe pod <pod-name>
```

Get pod logs:
```bash
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Follow logs
```

Execute commands in a pod:
```bash
kubectl exec -it <pod-name> -- /bin/sh
```

### Testing Health Checks

Manually trigger a CronJob:
```bash
kubectl create job --from=cronjob/quakewatch-health-check manual-health-check
kubectl logs job/manual-health-check
```

### Resource Usage

View resource usage:
```bash
kubectl top nodes
kubectl top pods
```

## Troubleshooting

### Pods not starting

Check pod events:
```bash
kubectl describe pod <pod-name>
```

Check if image is accessible:
```bash
kubectl get events
```

### Service not accessible

Verify service endpoints:
```bash
kubectl get endpoints quakewatch-service
```

Check if pods are ready:
```bash
kubectl get pods -l app=quakewatch
```

### CronJob not running

Check CronJob status:
```bash
kubectl describe cronjob quakewatch-health-check
```

View job history:
```bash
kubectl get jobs
```

## Cleanup

Remove all resources:
```bash
kubectl delete -f k8s/
```

Or delete individually:
```bash
kubectl delete deployment quakewatch
kubectl delete service quakewatch-service
kubectl delete hpa quakewatch-hpa
kubectl delete configmap quakewatch-config
kubectl delete secret quakewatch-secret
kubectl delete cronjob quakewatch-health-check
```

## Architecture Diagram

```
                                    ┌─────────────────┐
                                    │  LoadBalancer   │
                                    │    Service      │
                                    │    (Port 80)    │
                                    └────────┬────────┘
                                             │
                        ┌────────────────────┼────────────────────┐
                        │                    │                    │
                  ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
                  │   Pod 1   │       │   Pod 2   │       │   Pod 3   │
                  │ QuakeWatch│       │ QuakeWatch│       │ QuakeWatch│
                  │  :5000    │       │  :5000    │       │  :5000    │
                  └───────────┘       └───────────┘       └───────────┘
                        ▲                    ▲                    ▲
                        │                    │                    │
                        │         Liveness & Readiness Probes     │
                        │                                         │
                        └─────────────────────┬───────────────────┘
                                              │
                                    ┌─────────▼─────────┐
                                    │       HPA         │
                                    │   (3-10 replicas) │
                                    └───────────────────┘

                        ┌───────────────────────────────────┐
                        │         CronJob                   │
                        │  (Health Check every 5 minutes)   │
                        │  curl http://quakewatch-service/  │
                        └───────────────────────────────────┘
```

## Features

- **High Availability**: 3 replicas ensure the app stays available
- **Auto-Scaling**: HPA scales pods based on load
- **Health Monitoring**: Liveness/Readiness probes ensure pod health
- **Automated Testing**: CronJob validates service availability
- **Rolling Updates**: Zero-downtime deployments
- **Resource Management**: CPU/Memory limits prevent resource exhaustion
