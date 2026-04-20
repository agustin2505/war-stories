---
title: Silent rollout in Airflow — image was pushed but the pod never restarted
project: banking-ai-platform
date: 2026-04-16
stack: [airflow, gke, helm, docker]
topics: [rollout-strategy, image-tags, debugging]
severity: medium
status: draft
---

# Raw notes

## The symptom

- Data team runs an Airflow task that needs new code.
- Fails with `ModuleNotFoundError: src.infrastructure.rag.chunking.contextualizer`.
- The module exists in `main`. The CD built and pushed the image successfully.
- But on `exec` into the scheduler pod: the file is not there.

## The diagnosis

```
$ kubectl get pods -n airflow airflow-scheduler-5c48dbd746-s8tqh
NAME                                 READY   STATUS    RESTARTS   AGE
airflow-scheduler-5c48dbd746-s8tqh   4/4     Running   0          16d
```

16 days with no restart. The container image was frozen since 2026-03-31.

## Why

- CD pushes the new image to AR with tag `latest` and a semver tag (`0.1.0-bN-sha-XYZ`).
- But the Airflow deployment in the chart references a fixed tag (or `latest` with `imagePullPolicy: IfNotPresent`).
- Kubernetes detects no change in `image:` → it does not trigger a rollout.
- The image could have been pushed 50 times to the registry; without a change in the Deployment manifest, the pods keep running the image they pulled on day one.

## Immediate fix

```bash
kubectl rollout restart deployment/airflow-scheduler -n airflow
kubectl rollout restart deployment/airflow-dag-processor -n airflow
kubectl rollout restart statefulset/airflow-worker -n airflow
```

## Permanent fix (still thinking)

Options:
1. Change the CD so it updates the image tag in the values with the commit SHA; each deploy changes the manifest and K8s triggers a rollout.
2. Use `imagePullPolicy: Always` + `checksum/config` annotation on the pod template based on something that changes (git SHA, build date). Less explicit but robust.
3. A post-push hook in the CD that explicitly runs `kubectl rollout restart`. This couples the CD to the cluster state — I do not like it.

## Trap: worker with terminationGracePeriodSeconds: 600

- `rollout restart` of the worker did not kill it in a few seconds.
- The worker has `terminationGracePeriodSeconds: 600` (10 min) — it waits for Celery tasks to finish before dying.
- If there are long-running tasks, the restart can hang up to 10 min.
- When you do not have time: `kubectl delete pod airflow-worker-0 --force --grace-period=0`. But you lose in-flight tasks.

## Takeaways to develop

- Pushing image to the registry is not a rollout.
- A pod with 0 restarts in 16 days on a platform with active CI/CD is a red flag, not a sign of stability.
- Restarting stateful workers (Celery/Airflow) has real costs: in-flight tasks, grace period, reschedule.

## To include in the polished story

- Visual flow (diagram): registry push → does NOT trigger rollout → pod still runs old image.
- Compare strategies: pinned tag + rollout annotation vs latest + imagePullPolicy:Always.
