# Kubernetes Troubleshooting Lab



A hands-on Kubernetes troubleshooting project using a local Kind cluster. This project demonstrates six common Kubernetes failure scenarios, their investigation, remediation, and validation.



## Technology Stack



- Kubernetes

- Kind

- Docker

- kubectl

- YAML

- NGINX

- BusyBox



## Scenarios



### 1. CrashLoopBackOff

**Manifest:** `01-crashloop-demo.yaml`



- **Issue:** The original container command exited with a non-zero status.

- **Investigation:** Inspected pod status, restart count, logs, and events.

- **Resolution:** Corrected the container command so the process remains running.

- **Validation:** Confirmed the pod reached `Running` and `Ready` status.



### 2. Image Pull Failure

**Manifest:** `02-imagepull-demo.yaml`



- **Issue:** An invalid container image prevented the pod from starting.

- **Investigation:** Inspected pod events and the configured image.

- **Resolution:** Changed the image to `busybox:1.36`.

- **Validation:** Confirmed the pod reached `Running` and `Ready` status.



### 3. Pending Pod - Insufficient CPU

**Manifest:** `03-pending-demo.yaml`



- **Issue:** The pod could not be scheduled because its CPU request exceeded node capacity.

- **Investigation:** Reviewed pod events and resource requests.

- **Resolution:** Reduced the CPU request from `100` cores to `100m`.

- **Validation:** Confirmed the pod was scheduled and became ready.



### 4. Liveness and Readiness Probe Failures

**Manifest:** `04-probe-failure.yaml`



- **Issue:** Incorrect probe paths caused health-check failures.

- **Investigation:** Inspected probe configuration, pod status, logs, and events.

- **Resolution:** Corrected the probes and increased their timeout settings and container resource limits.

- **Validation:** The latest observed pod was `1/1 Running` with zero restarts after approximately two minutes. Longer observation is needed before claiming sustained stability.



### 5. Service Selector Mismatch

**Manifest:** `05-service-networking.yaml`



- **Issue:** A mismatched Service selector initially resulted in no endpoints.

- **Investigation:** Compared Service selectors with pod labels and inspected endpoints.

- **Resolution:** Corrected the selector to `app=crash-demo`.

- **Validation:** Confirmed an endpoint was populated for the matching pod.

- **Limitation:** This validates endpoint selection only. The BusyBox container is not an HTTP server, so HTTP connectivity was not tested.



### 6. Failed Deployment Rollout and Rollback

**Manifest:** `06-rollout-demo.yaml`



- **Issue:** An invalid NGINX image tag caused the updated Deployment to fail.

- **Investigation:** Inspected pod events, Deployment status, and ReplicaSets.

- **Resolution:** Rolled back to the previous working revision:



```bash

kubectl rollout undo deployment/rollout-demo -n troubleshooting-lab

```



- **Validation:** Confirmed the Deployment returned to its intended image and both replicas became available.



## General Troubleshooting Workflow



1\. Check resource status with `kubectl get`.

2\. Inspect resource details with `kubectl describe`.

3\. Review application logs and Kubernetes events.

4\. Identify the root cause before changing configuration.

5\. Apply the smallest appropriate correction.

6\. Validate recovery and monitor for recurring failures.

7\. Document the issue, resolution, and evidence.



## Run the Lab



Verify the active Kubernetes context before applying any manifest:



```bash

kubectl config current-context

```



Apply an individual scenario:



```bash

kubectl apply -f 01-crashloop-demo.yaml

```



Inspect the pods and events:



```bash

kubectl get pods -n troubleshooting-lab

kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp

```



See `RUNBOOKS.md` for detailed investigation commands and remediation guidance.



## Safety



These manifests are intended for a local Kind lab. Verify that the active context is `kind-sre-lab` before applying them. Do not apply demonstration manifests to company or production clusters.


