# Lab 7 write-up · Run it on a cluster

## The first apply

### My prediction
Before running kubectl apply -f k8s/, I predicted that Kubernetes would create three Pods with names starting with api- followed by the ReplicaSet template hash and a random unique identifier (for example, api-689677cfd-...).

### Actual ouput
```text
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   3/3     3            3           12s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/api-689677cfd   3         3         3       12s

NAME                         READY   STATUS    RESTARTS   AGE
pod/api-689677cfd-fl86v      1/1     Running   0          12s
pod/api-689677cfd-s5jdn      1/1     Running   0          12s
pod/api-689677cfd-tm7ff      1/1     Running   0          12s
```

`curl -s http://localhost:8080/` response:
``json
{"name":"course-api","version":"0.1.0"}
```

---

## Heal and scale

### Deleting a Pod
When deleting one of the running Pods, the ReplicaSet controller immediately detected that the current replica count was 2 instead of 3 and created a brand-new replacement Pod within seconds.

Output of `kubectl -n course get pods`:
```text
NAME                  READY   STATUS    RESTARTS        AGE
api-689677cfd-btzdd   1/1     Running   0               33s
api-689677cfd-s5jdn   1/1     Running   1 (5m40s ago)   8m57s
api-689677cfd-tm7ff   1/1     Running   1 (5m40s ago)   8m57s
```

### Scaling up and down
Changing `replicas: 5` scaled the Deployment to 5 active Pods:
``text
NAME                  READY   STATUS    RESTARTS   AGE
api-689677cfd-btzdd   1/1     Running   0          22m14s
api-689677cfd-s5jdn   1/1     Running   1          10m
api-689677cfd-tm7ff   1/1     Running   1          10m
api-689677cfd-k48x9   1/1     Running   0          8s
api-689677cfd-q71lm   1/1     Running   0          8s
```
Scaling back to `replicas: 3` terminated the two newest Pods, returning the Deployment to 3 healthy Pods.

---

## Roll and roll back

### Rolling update to a new image
Updated `k8s/api-deployment.yaml` from image tag `0.2.0` to `d85c1c0bc2a0809c2aee1c1d9f70511cd14aa481`.

Output of `kubectl -n course rollout status deployment/api`:
```text
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
deployment "api" successfully rolled out
b``

### Broken rollout
When applying the non-existent image tag `9.9.9`, I predicted that all three existing Pods would continue serving traffic without downtime because rolling updates require new Pods to pass readiness before terminating old ones.

Output of `kubectl -n course get pods` during the stall:
``text
NAME                   READY   STATUS             RESTARTS   AGE
api-689677cfd-btzdd    1/1     Running            0          6m12s
api-689677cfd-s5jdn    1/1     Running            1          14m
api-689677cfd-tm7ff    1/1     Running            1          14m
api-7d498c8bf5-2f9pl   0/1     ImagePullBackOff   0          42s
```

### The rollout undo warning
When running `kubectl -n course rollout undo deployment/api`, the terminal prints a warning regarding `last-applied-configuration`. This warning means `rollout undo` imperatively mutates the live resource in the cluster (etcd), which causes the cluster state to drift from the declarative configuration files saved locally in Git.

---

## What broke and how I found it

### Fault 1: Image tag typo
* **Symptom:** Pods were stuck with status `ImagePullBackOff` / `ErrImagePull`.
* **How I found it:** Running `kubectl -n course get pods` and inspecting `kubectl -n course describe pod` showed image pull failures because the tag ended with capital letter `O` (`0.2.O`) instead of digit `0` (`0.2.0`).
* **Fix commit:** `fix(k8s): correct image tag typo from 0.2.O to 0.2.0`

### Fault 2: Service selector mismatch
 **Symptom:** Traffic was not being routed to any Pods.
* **How I found it:** Running `kubectl -n course get endpointslices -l kubernetes.io/service-name=api` showed `<unset>` endpoints. In `k8s/api-service.yaml`, the selector was `app.kubernetes.io/name: api`, whereas the Pod labels were `app: api`.
* **Fix commit:** `fix(k8s): align service selector with pod label app: api`

### Fault 3: Service targetPort mismatch
* **Symptom:** `curl -s http://localhost:8080/` through port-forward failed with connection refused / empty reply.
* **How I found it:** In `k8s/api-service.yaml`, `targetPort` was set to `8080`, but the Node.js API container listens on port `5000`.
* **Fix commit:** `fix(k8s): route service targetPort to container port 5000`
