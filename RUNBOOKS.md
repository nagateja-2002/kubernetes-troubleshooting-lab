\# Kubernetes Troubleshooting Runbooks



\## Overview



This document describes the symptoms, investigation commands, root causes, and corrective actions for six Kubernetes troubleshooting scenarios reproduced in a local Kind cluster.



Namespace: `troubleshooting-lab`



> These scenarios were tested in a local lab. They do not represent production incident validation.



\---



\## 1. CrashLoopBackOff



\### Symptoms

\- Pod repeatedly restarts.

\- Pod status may show `CrashLoopBackOff`.

\- Restart count increases.



\### Investigation



```bash

kubectl get pods -n troubleshooting-lab

kubectl describe pod <pod-name> -n troubleshooting-lab

kubectl logs <pod-name> -n troubleshooting-lab

kubectl logs <pod-name> --previous -n troubleshooting-lab

kubectl get events -n troubleshooting-lab --sort-by=.metadata.creationTimestamp

