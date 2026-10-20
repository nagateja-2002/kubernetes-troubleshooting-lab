# Kubernetes Troubleshooting Runbooks
## Overview
This document describes symptoms, investigation commands, root causes, remediation steps, and validation procedures for six Kubernetes troubleshooting scenarios.
**Environment:** Local Kind Kubernetes cluster

**Namespace:** `troubleshooting-lab`

> These scenarios were tested in a local lab. They do not represent production incident validation. Verify the active Kubernetes context before applying changes.
## General Investigation Workflow
Use this workflow when troubleshooting a Kubernetes workload:
1. Identify the affected namespace, pod, deployment, or service.
2. Check the resource status and restart count.
3. Inspect pod descriptions, container logs, and Kubernetes events.
4. Identify the most likely root cause using the available evidence.
5. Apply the smallest appropriate correction.
6. Verify recovery and monitor for recurring failures.
7. Document the root cause, resolution, and validation evidence.
Useful commands:
```bash
kubectl config current-context
kubectl get pods -n troubleshooting-lab
kubectl get deployments -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
```
## 1. CrashLoopBackOff
### Symptoms
- Pod repeatedly restarts.
- Pod status may show `CrashLoopBackOff`.
- Restart count increases.
### Investigation
```bash
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
kubectl logs <pod-name> -n troubleshooting-lab
kubectl logs <pod-name> --previous -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
```
### Root Cause
The demonstration container was configured to exit after a short period, causing Kubernetes to restart it repeatedly.
### Remediation
Update the container command so the demonstration process remains running:
```yaml
command: ["sh", "-c", "echo Container started; while true; do sleep 3600; done"]
```
Apply the corrected manifest:
```bash
kubectl apply -f 01-crashloop-demo.yaml
```
### Validation
```bash
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
```
Confirm that the pod reaches `Running` and that the restart count is not continuing to increase.
## 2. ImagePullBackOff
### Symptoms
- Pod remains in `ImagePullBackOff` or `ErrImagePull`.
- Container does not start.
- Events report image-pull failures.
### Investigation
```bash
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
kubectl get deployment imagepull-demo -n troubleshooting-lab -o yaml
```
### Root Cause
The demonstration Deployment referenced a deliberately invalid image name and tag.
In real environments, image-pull failures may also result from incorrect registry paths, unavailable tags, registry connectivity, or authentication problems.
### Remediation
Correct the image reference in `02-imagepull-demo.yaml`:
```yaml
image: busybox:1.36
```
Apply the corrected manifest:
```bash
kubectl apply -f 02-imagepull-demo.yaml
```
### Validation
```bash
kubectl rollout status deployment/imagepull-demo -n troubleshooting-lab
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
```
Confirm that the container starts successfully and the pod reaches `Running`.
## 3. Pending Pod: Insufficient CPU
### Symptoms
- Pod remains in `Pending`.
- No node is assigned to the pod.
- Events report `Insufficient cpu`.
### Investigation
```bash
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
kubectl get nodes
kubectl describe node <node-name>
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
```
### Root Cause
The demonstration Deployment requested `100` CPU cores, exceeding the available schedulable CPU in the local cluster.
### Remediation
Change the CPU request to a suitable value for the local lab:
```yaml
resources:
  requests:
    cpu: 100m
```
Apply the corrected manifest:
```bash
kubectl apply -f 03-pending-demo.yaml
```
### Validation
```bash
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
```
Confirm that the pod is scheduled and reaches `Running`.
> CPU requests affect scheduling. Select requests based on actual workload requirements in real environments rather than simply reducing them to force scheduling.
## 4. Liveness and Readiness Probe Failures
### Symptoms
- Liveness probe failures may cause container restarts.
- Readiness probe failures may leave a pod unready.
- Events contain probe-failure messages.
### Investigation
```bash
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
kubectl logs <pod-name> -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
kubectl get deployment probe-demo -n troubleshooting-lab -o yaml
```
### Root Cause
The demonstration initially used a probe that checked a file that did not exist in the container.
A liveness probe failure can cause Kubernetes to restart a container. A readiness probe failure marks the container as not ready to receive Service traffic but does not, by itself, restart the container.
### Remediation
Configure both probes to check the file created by the container's startup command:
```yaml
livenessProbe:
  exec:
    command: ["sh", "-c", "test -f /tmp/health/live"]
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
readinessProbe:
  exec:
    command: ["sh", "-c", "test -f /tmp/health/live"]
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```
Apply the corrected manifest:
```bash
kubectl apply -f 04-probe-failure.yaml
```
### Validation
```bash
kubectl rollout status deployment/probe-demo -n troubleshooting-lab
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
```
Confirm that the pod becomes ready and that the restart count remains stable during the observation period.
## 5. Service Selector and Endpoints
### Symptoms
- A Service may have no endpoints.
- The Service selector may not match any pod labels.
- Requests through a Service may fail when no suitable backend is available.
### Investigation
```bash
kubectl get services -n troubleshooting-lab
kubectl describe service crash-demo-service -n troubleshooting-lab
kubectl get endpoints crash-demo-service -n troubleshooting-lab
kubectl get pods --show-labels -n troubleshooting-lab
kubectl get service crash-demo-service -n troubleshooting-lab -o yaml
```
### Root Cause
The demonstration Service initially used a selector that did not match the target pod's labels.
### Remediation
Correct the Service selector to match the labels on the intended pod:
```yaml
selector:
  app: crash-demo
```
Apply the corrected manifest:
```bash
kubectl apply -f 05-service-networking.yaml
```
### Validation
```bash
kubectl get service crash-demo-service -n troubleshooting-lab
kubectl get endpoints crash-demo-service -n troubleshooting-lab
kubectl get pods --show-labels -n troubleshooting-lab
```
Confirm that the Service has an endpoint corresponding to the selected pod.
> **Validation limitation:** This scenario validates Service selector matching and endpoint selection. The BusyBox demonstration container is not an HTTP server, so endpoint presence does not prove that HTTP requests will succeed.
## 6. Failed Deployment Rollout and Rollback
### Symptoms
- A Deployment rollout does not complete successfully.
- New pods fail to start.
- Events report image-pull errors.
- The previous replica set may continue serving available replicas.
### Investigation
```bash
kubectl get deployments -n troubleshooting-lab
kubectl rollout status deployment/rollout-demo -n troubleshooting-lab
kubectl rollout history deployment/rollout-demo -n troubleshooting-lab
kubectl get pods -n troubleshooting-lab
kubectl describe deployment rollout-demo -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
```
### Root Cause
The demonstration Deployment was updated to use an invalid NGINX image tag. Kubernetes could not pull the image for the new pods.
### Remediation
Roll back the Deployment to its previous revision:
```bash
kubectl rollout undo deployment/rollout-demo -n troubleshooting-lab
```
### Validation
```bash
kubectl rollout status deployment/rollout-demo -n troubleshooting-lab
kubectl get deployment rollout-demo -n troubleshooting-lab
kubectl get pods -n troubleshooting-lab
kubectl get deployment rollout-demo -n troubleshooting-lab -o jsonpath="{.spec.template.spec.containers[0].image}"
```
Confirm that the rollout completes, the intended replicas become available, and the image reference returns to the previous valid version.
> Rollback restores the previous Deployment revision. It does not automatically reverse database changes, external side effects, or other resources changed outside that revision.
## Safety and Scope
- These manifests are intended for a local Kind cluster.
- Verify `kubectl config current-context` before applying manifests.
- Do not apply demonstration manifests to company or production clusters.
- Do not include production credentials, access tokens, customer information, or confidential logs in this repository.
- Treat command output as evidence; do not claim a recovery was validated unless it was actually observed.
