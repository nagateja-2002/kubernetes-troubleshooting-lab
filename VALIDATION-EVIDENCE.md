# Kubernetes Troubleshooting Lab — Validation Evidence

## Environment

- Kubernetes distribution: Kind

- Namespace: `troubleshooting-lab`

- Container runtime: Docker

- CLI: kubectl

> Results documented here are observations recorded during earlier local lab exercises. The current repository manifests deliberately reproduce failure states for the first five scenarios; scenario 6 uses a good base revision plus a separate failing update. The GitHub Actions workflow validates Kubernetes schemas only. It does not run a cluster or re-test runtime recovery. Re-run the scenarios from the current commit before claiming fresh runtime validation.

## 1. CrashLoopBackOff

**Initial condition:** The demonstration container exited with a non-zero status, resulting in repeated restarts.

**Remediation:** Updated the container command so the process remained running.

**Observed result:** The pod reached `Running`. Subsequent checks showed recovery.

**Evidence commands:**

```bash

kubectl get pods -n troubleshooting-lab

kubectl describe pod <pod-name> -n troubleshooting-lab

kubectl logs <pod-name> --previous -n troubleshooting-lab

```

## 2. ImagePullBackOff

**Initial condition:** An invalid image reference prevented the container from starting.

**Remediation:** Changed the image to `busybox:1.36`.

**Observed result:** The pod reached `Running` and `Ready`.

**Evidence commands:**

```bash

kubectl describe pod <pod-name> -n troubleshooting-lab

kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp

kubectl get pods -n troubleshooting-lab

```

## 3. Pending Pod — Insufficient CPU

**Initial condition:** A CPU request of `100` cores prevented scheduling.

**Remediation:** Reduced the demonstration request to `100m`.

**Observed result:** The pod was scheduled and became ready.

**Evidence commands:**

```bash

kubectl describe pod <pod-name> -n troubleshooting-lab

kubectl get pods -n troubleshooting-lab

```

## 4. Liveness and Readiness Probe Failures

**Initial condition:** An incorrect probe path caused health-check failures.

**Remediation:** Corrected the probe paths and adjusted the probe settings and container resource limits.

**Observed result:** The latest recorded observation showed the pod at `1/1 Running` with zero restarts after approximately two minutes.

**Limitation:** This observation does not establish long-term stability.

**Evidence commands:**

```bash

kubectl describe pod <pod-name> -n troubleshooting-lab

kubectl get pods -n troubleshooting-lab

kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp

```

## 5. Service Selector and Endpoints

**Initial condition:** A mismatched Service selector resulted in no endpoints.

**Remediation:** For the current manifest, change the selector to `app=endpoint-target`.

**Historical observation:** An endpoint was populated after selector correction in an earlier local exercise. The self-contained target Deployment now in the repository has not been runtime-tested by GitHub Actions; repeat the scenario from the current commit to confirm its endpoint state.

**Limitation:** Endpoint selection does not prove HTTP connectivity, because the BusyBox container is not configured as an HTTP server.

**Evidence commands:**

```bash

kubectl describe service crash-demo-service -n troubleshooting-lab

kubectl get endpoints crash-demo-service -n troubleshooting-lab

kubectl get pods --show-labels -n troubleshooting-lab

```

## 6. Deployment Rollout Failure and Rollback

**Initial condition:** Applied `06-rollout-demo.yaml` with a valid NGINX image and waited for two available replicas, then applied `06-rollout-failure.yaml` with an invalid image tag to create the failed revision.

**Remediation:** Executed a Deployment rollback.

```bash

kubectl rollout undo deployment/rollout-demo -n troubleshooting-lab

```

**Observed result:** The Deployment returned to the previous valid image, and both replicas became available.

**Evidence commands:**

```bash

kubectl rollout history deployment/rollout-demo -n troubleshooting-lab

kubectl rollout status deployment/rollout-demo -n troubleshooting-lab

kubectl get deployment rollout-demo -n troubleshooting-lab

kubectl get pods -n troubleshooting-lab

```

## Validation Scope

- The observations above were recorded during prior local lab exercises; the latest CI workflow validates schemas, not runtime failure/recovery behavior.

- The six scenarios describe local lab exercises, not production incident validation.

- Kubernetes manifest dry-run checks validate manifest acceptance but do not prove that a workload operates correctly.

- Command examples in this document show how to inspect the scenarios; they are not retained terminal logs.

- No HTTP connectivity test is claimed for the Service scenario.

- Probe stability is limited to the documented observation period.
