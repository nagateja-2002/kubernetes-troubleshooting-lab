# Kubernetes Troubleshooting Lab

A hands-on **local Kind lab** for practising how to investigate, remediate, and validate common Kubernetes workload failures. Each scenario starts with an intentionally incorrect configuration and documents the commands used to narrow down the cause.

> **Scope and safety:** These are learning exercises, not evidence of production incident handling. Verify your active Kubernetes context before applying anything. Do not run these demonstration manifests against a company or production cluster.

## What this lab covers

| Scenario | Common symptom | Investigation focus |
|---|---|---|
| CrashLoopBackOff | Container repeatedly exits | Pod status, current/previous logs, restart count, events |
| Image pull failure | `ErrImagePull` / `ImagePullBackOff` | Image reference, registry access, Pod events |
| Pending pod | Pod cannot be scheduled | CPU requests, node capacity, scheduler events |
| Probe failures | Pod unready or container restarts | Liveness/readiness configuration, logs, events |
| Service selector mismatch | Service has no endpoints | Service selector, pod labels, endpoint list |
| Failed rollout and rollback | New Deployment revision does not become ready | Rollout status/history, ReplicaSets, image reference |

## Technology

- Kubernetes and Kind
- Docker
- kubectl
- YAML
- NGINX and BusyBox demo containers

## Repository contents

| File | Purpose |
|---|---|
| `01-crashloop-demo.yaml` | CrashLoopBackOff scenario |
| `02-imagepull-demo.yaml` | Image pull failure scenario |
| `03-pending-demo.yaml` | Scheduling / insufficient CPU scenario |
| `04-probe-failure.yaml` | Liveness and readiness probe scenario |
| `05-service-networking.yaml` | Service selector and endpoint scenario |
| `06-rollout-demo.yaml` | Known-good revision for rollout/rollback scenario |
| `06-rollout-failure.yaml` | Deliberately bad image update to trigger a failed rollout |
| `RUNBOOKS.md` | Investigation steps, remediation examples, and validation commands |
| `VALIDATION-EVIDENCE.md` | Recorded observations and the limitations of each validation |

## Prerequisites

- Docker running locally
- Kind installed
- kubectl installed

The documented local context is `kind-sre-lab` and the namespace is `troubleshooting-lab`. If you create the cluster specifically for this lab, Kind's cluster name `sre-lab` normally produces the context `kind-sre-lab`.

Before applying a manifest, verify the context and namespace:

```bash
kubectl config current-context
kubectl get nodes
kubectl get namespace troubleshooting-lab
```

If the namespace has not been created in your local lab, create it only after confirming the context:

```bash
kubectl create namespace troubleshooting-lab
```

## Run a scenario

Apply **one scenario at a time** from the repository root. For example:

```bash
kubectl apply -f 01-crashloop-demo.yaml
kubectl get pods -n troubleshooting-lab
kubectl describe pod <pod-name> -n troubleshooting-lab
kubectl logs <pod-name> -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
```

Replace `<pod-name>` with the actual Pod name shown by `kubectl get pods`. Use the matching manifest for another scenario, then follow the remediation and validation steps in [RUNBOOKS.md](RUNBOOKS.md).

General troubleshooting sequence:

1. **Identify:** locate the affected Pod, Deployment, or Service.
2. **Inspect:** check resource state, logs, descriptions, and events.
3. **Diagnose:** use evidence to determine the likely root cause.
4. **Remediate:** make the smallest suitable configuration change.
5. **Validate:** confirm recovery using rollout status, Pod readiness, endpoints, or restart counts as appropriate.
6. **Record:** capture what was observed and any remaining limitation.

Useful commands:

```bash
kubectl get pods,deployments,services -n troubleshooting-lab
kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp
kubectl rollout history deployment/rollout-demo -n troubleshooting-lab
kubectl rollout status deployment/rollout-demo -n troubleshooting-lab
```

## Continuous validation

GitHub Actions runs a pinned Kubeconform schema validation on all six scenario manifests. This validates Kubernetes resource schemas; it does not run a cluster, prove that a failure can be reproduced, or certify recovery.

[View CI runs](https://github.com/nagateja-2002/kubernetes-troubleshooting-lab/actions).

## Validation notes

The repository's evidence file records observations from prior local lab exercises. Important limitations are kept explicit:

- Probe stability was observed for a limited period; that does not prove long-term stability.
- The Service scenario validates selector matching and endpoint population. The BusyBox container is not an HTTP server, so HTTP connectivity is not claimed.
- A command example is not the same as a retained terminal transcript; rerun the checks to validate your current environment.

See [VALIDATION-EVIDENCE.md](VALIDATION-EVIDENCE.md) and [RUNBOOKS.md](RUNBOOKS.md) for details.

## Safety checklist

- Confirm `kubectl config current-context` before every apply or rollback.
- Keep the exercises within a disposable local Kind cluster.
- Never commit credentials, kubeconfig files, access tokens, customer data, or confidential logs.
- Use workload-specific evidence to confirm recovery rather than assuming that a manifest applying successfully means the application is healthy.
