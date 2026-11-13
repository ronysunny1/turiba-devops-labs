# Lab 8 · Run the whole app

GitHub user: ronysunny1 · Fallback image used: no · Fallback Mission 7 files used: no

## 1 · Database with storage

- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I predicted yes because PostgreSQL stores its data in `/var/lib/postgresql/data` backed by the `db-data` PersistentVolumeClaim, which persists independently of Pod lifecycles. What happened: the replacement `db` Pod mounted the same PersistentVolume and `GET /api/todos` still returned the `"survive a restart"` todo.

```text
$ kubectl -n course get pvc,pv
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-ca484c5a-94ad-4316-b820-1418d91b639f   1Gi        RWO            standard       <unset>                 77s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-ca484c5a-94ad-4316-b820-1418d91b639f   1Gi        RWO            Delete           Bound    course/db-data   standard       <unset>                          57s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"survive a restart"}' localhost:8080/api/todos
{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-13T17:44:26.835Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
NAME                 READY   STATUS    RESTARTS   AGE
db-547768cbd-jvwkr   1/1     Running   0          74s

$ kubectl -n course delete pod db-547768cbd-jvwkr
pod "db-547768cbd-jvwkr" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
NAME                 READY   STATUS    RESTARTS   AGE
db-547768cbd-m2kfh   1/1     Running   0          1s

$ curl -s localhost:8080/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-13T17:44:26.835Z"}]
```

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
db:5432 - no response
waiting for the database
db:5432 - no response
waiting for the database
db:5432 - no response
waiting for the database
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
course-api 0.1.0 starting (Node.js v24.22.2)
API listening on port 5000
Database ready
Database connection lost: terminating connection due to administrator command
```

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I predicted it would show the `password` key encoded in base64 rather than plaintext because Kubernetes Secrets store `data` fields base64-encoded. What I saw, in words (don't paste the value): the Secret manifest displayed the `password` key under `data` as an opaque base64-encoded string.

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I predicted the Pods would still receive traffic because their readiness probe still checked `/healthz`, while kubelet would repeatedly restart the containers after failing liveness checks on `/nope`. What happened: the Pods stayed `1/1 Running` (Ready) and served traffic, while `RESTARTS` incremented to `1` as kubelet killed and restarted containers after HTTP 404 liveness probe failures.

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
NAME                    READY   STATUS    RESTARTS      AGE
api-68d485d7cd-dm7dg    1/1     Running   1 (6s ago)    46s
api-68d485d7cd-dp584    1/1     Running   1 (16s ago)   56s
api-68d485d7cd-zjdgm    1/1     Running   1 (13s ago)   53s

$ kubectl -n course describe pod api-68d485d7cd-dm7dg         (only the Events)
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  46s                default-scheduler  Successfully assigned course/api-68d485d7cd-dm7dg to kind-control-plane
  Normal   Pulled     46s                kubelet            spec.initContainers{wait-for-db}: Container image "postgres:17-alpine" already present on machine
  Normal   Created    46s                kubelet            spec.initContainers{wait-for-db}: Created container: wait-for-db
  Normal   Started    46s                kubelet            spec.initContainers{wait-for-db}: Started container wait-for-db
  Normal   Pulled     6s (x2 over 45s)   kubelet            spec.containers{api}: Container image "ghcr.io/ronysunny1/turiba-devops-labs/api:0.2.0" already present on machine
  Normal   Created    6s (x2 over 45s)   kubelet            spec.containers{api}: Created container: api
  Normal   Started    6s (x2 over 45s)   kubelet            spec.containers{api}: Started container api
  Warning  Unhealthy  6s (x3 over 26s)   kubelet            spec.containers{api}: Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    6s                 kubelet            spec.containers{api}: Container api failed liveness probe, will be restarted
```

- How I made Git and the cluster agree again afterwards: ran `git restore k8s/ && kubectl apply -f k8s/`, verified `git diff` was empty, and confirmed `deployment/api` rolled out cleanly.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: database connection or `wait-for-db` init container startup if the ConfigMap host or Secret key was changed.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | `wait-for-db` init container stuck in `Init:0/1` | `kubectl -n course logs -l app=api -c wait-for-db` → `database:5432 - no response` | `fix(k8s): restore DB_HOST to db in app-config ConfigMap` |
| 2 | API Pods blocked in `CreateContainerConfigError` | `kubectl -n course describe pod -l app=api` → `Error: couldn't find key pasword in Secret course/db-secret` | `fix(k8s): fix typo in db-secret key reference to password` |
| 3 | API Pods `Running` but `0/1` Not Ready | `kubectl -n course describe pod -l app=api` → `Readiness probe failed: HTTP probe failed / connection refused` on port `5001` | `fix(k8s): restore readinessProbe port to 5000` |
