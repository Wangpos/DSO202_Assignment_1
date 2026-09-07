# DSO202 Assignment 1 Report

## Three-Tier Task Tracker Deployment on Kubernetes

**Module:** DSO202 - Scaling, Orchestration, Monitoring & Observability  
**Scope:** Unit I  
**Cluster:** kind cluster `dso202`  
**Namespace:** `dso202-assignment-01`

## 1. Executive summary

This assignment deployed a provided three-tier Task Tracker application to a
local Kubernetes kind cluster. The application consists of a frontend, a
backend REST API, and a PostgreSQL database. No application code was written
for the assignment; the work focused on Kubernetes configuration, service
discovery, configuration management, secret handling, persistent storage,
resource governance, troubleshooting, and verification.

The final deployment runs one Pod for each tier. The frontend is exposed to
the host through a NodePort, while the backend and database remain internal to
the namespace. The database uses a PersistentVolumeClaim, so its data remains
available when an application Pod is replaced. The complete CRUD workflow,
internal DNS resolution, backend self-healing, and data persistence were
verified successfully.

## 2. Requirements and architecture

The application uses the following resources:

| Tier     | Image                               | Internal port | Kubernetes Service             |
| -------- | ----------------------------------- | ------------: | ------------------------------ |
| Frontend | `sarojisanyasi/dso202-frontend:1.0` |          8080 | `frontend-svc`, NodePort 30080 |
| Backend  | `sarojisanyasi/dso202-backend:1.0`  |          8080 | `backend-svc`, ClusterIP       |
| Database | `sarojisanyasi/dso202-db:1.0`       |          5432 | `db-svc`, headless Service     |

The kind configuration maps host port `8080` to NodePort `30080`, allowing the
frontend to be opened at `http://localhost:8080`.

The control-plane node provides the Kubernetes API server, scheduler,
controller manager, and kubelet. The scheduler places the three Pods on the
node. Deployments create and manage ReplicaSets and Pods. Services provide
stable names and networking. The ConfigMap provides non-sensitive settings,
the Secret provides database credentials, the PVC provides persistent storage,
and the ResourceQuota and LimitRange govern resource usage in the namespace.

The resulting traffic flow is:

Apply Patch

```mermaid
flowchart TD
    browser[Browser] --> frontendSvc["frontend-svc:30080"]
    frontendSvc --> frontendPod["Frontend Pod:8080"]
    frontendPod -->|"/api/ through nginx"| backendSvc["backend-svc:8080"]
    backendSvc --> dbSvc["db-svc:5432"]
    dbSvc --> databasePod["Database Pod"]
    databasePod --> dbPvc["db-pvc: /var/lib/postgresql/data"]
`````````


The backend and database are not exposed using NodePort or LoadBalancer.

## 3. Kubernetes resources implemented

### Namespace

All assignment resources use the dedicated namespace:

```text
dso202-assignment-01
```

This provides namespace isolation from unrelated workloads.

### ConfigMap and Secret

The ConfigMap contains the non-sensitive values:

```text
DB_HOST=db-svc
DB_PORT=5432
DB_NAME=tasksdb
POSTGRES_DB=tasksdb
APP_PORT=8080
CORS_ORIGIN=*
BACKEND_URL=http://backend-svc:8080
```

The Secret contains matching application and PostgreSQL credential keys:

| Backend key   | PostgreSQL image key |
| ------------- | -------------------- |
| `DB_USER`     | `POSTGRES_USER`      |
| `DB_PASSWORD` | `POSTGRES_PASSWORD`  |

The PostgreSQL image uses `POSTGRES_*`, while the backend uses `DB_*`. Supplying
both names is necessary because they are consumed by different containers.
Kubernetes Secret values are base64-encoded and are not encrypted at rest by
default. Production deployments would require encryption at rest and stronger
secret-management controls.

### Database tier

The database Deployment uses one replica, mounts `db-pvc` at
`/var/lib/postgresql/data`, and uses the provided PostgreSQL image. `db-svc` is
headless (`clusterIP: None`) and is therefore reachable only through internal
cluster DNS.

### Backend tier

The backend Deployment uses one replica and consumes the ConfigMap and Secret.
It listens on port 8080. `backend-svc` is a ClusterIP Service, so the REST API
is available to other Pods but is not directly exposed outside the cluster.

The verified API endpoints are:

```text
GET    /api/status
GET    /api/tasks
GET    /api/tasks/{id}
POST   /api/tasks
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
```

### Frontend tier

The frontend Deployment uses the provided frontend image and listens on port 8080. Its nginx configuration proxies `/api/` to `backend-svc:8080`. The
frontend runtime configuration is generated from `config.js.template` at
container startup, so the backend address is supplied at runtime rather than
hardcoded into the image build.

### Persistent storage

The database PVC requests 1Gi using kind's `standard` StorageClass. The PVC
was verified as `Bound`, and the database Pod mounts it at the required
PostgreSQL data path.

### Resource governance

The ResourceQuota limits the namespace to:

```text
Pods:             6
CPU requests:     1
Memory requests:  1Gi
CPU limits:       2
Memory limits:    2Gi
```

The three single-replica Deployments fit within these limits while retaining
room for a replacement Pod during an update. The LimitRange supplies default
requests of 100m CPU and 128Mi memory, default limits of 250m CPU and 256Mi
memory, maximums of 500m CPU and 512Mi memory, and minimums of 50m CPU and
64Mi memory.

## 4. Deployment procedure

The assignment cluster was selected before running kubectl commands:

```bash
kubectl config use-context kind-dso202
```

The resources were applied in dependency order:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml -f secret.yaml -f quota.yaml
kubectl apply -f frontend/configmap.yaml
kubectl apply -f database/ -f backend/ -f frontend/
```

The final cluster state was checked with:

```bash
kubectl get pods,svc,pvc,resourcequota,limitrange \
	-n dso202-assignment-01
```

The final state showed all three Pods running, the PVC bound, the frontend
Service as NodePort, the backend Service as ClusterIP, the database Service as
headless, and both governance resources present.

## 5. Verification and evidence

### 5.1 Resource and workload verification

![Namespace resources and workload status](screenshot/Screenshot%201.png)

This screenshot shows the running Pods, Deployments, Services, ReplicaSets,
PVC, ConfigMap, and Secret in the assignment namespace. It demonstrates that
the three tiers were created and connected inside Kubernetes.

### 5.2 Backend health and database connection

![Backend status from inside the frontend Pod](screenshot/connection%20with%20backend.png)

The backend status endpoint returned:

```json
{ "status": "ok", "db": "connected" }
```

This proves that the backend is serving requests and can connect to
PostgreSQL.

The same check can be reproduced with:

```bash
kubectl exec -n dso202-assignment-01 deploy/frontend-deployment -- \
	curl http://backend-svc:8080/api/status
```

### 5.3 Listing and retrieving tasks

![Initial task list through the backend](screenshot/apitask%20list%20using%20localhost%208081.png)

The backend returned task objects containing IDs, titles, descriptions,
statuses, and creation timestamps. This verifies the read/list operation.

### 5.4 Full CRUD cycle

![CRUD terminal commands and responses](screenshot/crud%20final.png)

The CRUD test created a task, updated its status, deleted it, and listed the
remaining tasks. The API responses demonstrated:

```text
POST /api/tasks  -> task created
PUT  /api/tasks/11 -> status changed to done
DELETE /api/tasks/11 -> HTTP 204 No Content
GET  /api/tasks -> deleted task absent from final list
```

Additional CRUD evidence is provided by:

![CRUD operation evidence](screenshot/curd%20operatiion%20.png)

![Task creation evidence](screenshot/Screenshot%202.2.png)

![Task listing evidence](screenshot/Screenshot%203.png)

![Task update evidence](screenshot/Screenshot%204.png)

![Task deletion evidence](screenshot/Screenshot%205.png)

### 5.5 Service DNS resolution

![Backend Service DNS resolution from the frontend Pod](screenshot/Screenshot%206.png)

The frontend Pod successfully reached:

```text
http://backend-svc:8080/api/status
```

This proves that Kubernetes Service DNS resolution works within the namespace.

### 5.6 Self-healing

![Backend Pod deletion command](screenshot/delete%20pod%20-n%20dso202-assignment-01%20-ltier=backend.png)

The backend Pod was deliberately deleted:

```bash
kubectl delete pod -n dso202-assignment-01 -l tier=backend
```

![Backend Pod replacement observed with kubectl watch](screenshot/Screenshot%207.png)

![Replacement backend Pod running](screenshot/Screenshot%207.1.png)

The Deployment's ReplicaSet created a replacement Pod. This demonstrates
Kubernetes self-healing and the desired-state behavior of a Deployment.

### 5.7 Data persistence

![Task data after backend Pod replacement](screenshot/Screenshot%209.png)

The task data remained available after the backend Pod was deleted and
recreated. This demonstrates that application Pod lifecycle and database
storage lifecycle are independent because database data is stored on the PVC.

### 5.8 Frontend browser verification

![Frontend connected to backend and database](screenshot/sucess%20connection%20with%20backend%20.png)

The frontend displayed `BACKEND + DB ONLINE` and loaded tasks through the
browser. The additional browser evidence is shown below:

![Frontend Task Tracker](screenshot/crud%20final.png)

The frontend can be opened at:

```text
http://localhost:8080
```

## 6. Declarative and imperative management

Declarative management was demonstrated with:

```bash
kubectl apply -f namespace.yaml
```

The namespace already existed and kubectl reported it as unchanged, showing
that the command can safely reconcile the desired state.

Imperative management was demonstrated separately with:

```bash
kubectl create configmap imperative-evidence \
	--from-literal=method=imperative \
	-n dso202-assignment-01
kubectl get configmap imperative-evidence -n dso202-assignment-01
kubectl delete configmap imperative-evidence -n dso202-assignment-01
```

Declarative YAML is version-controlled, repeatable, reviewable, and suitable
for maintaining the assignment. Imperative commands are useful for quick,
one-off actions, but they do not store the complete desired state in the
repository.

## 7. Errors encountered and solutions

### 7.1 Wrong Kubernetes context

**Error:**

```text
namespaces "dso202-assignment-01" not found
```

**Cause:** kubectl was using the `docker-desktop` context instead of the
assignment's `kind-dso202` context.

**Solution:**

```bash
kubectl config use-context kind-dso202
```

After switching contexts, the namespace, quota, and workloads were found.

### 7.2 Browser could not resolve the backend Service

**Error:** The browser displayed `BACKEND UNREACHABLE` and `Failed to fetch`,
while a curl command executed inside the cluster succeeded.

**Cause:** `backend-svc` is a Kubernetes-internal DNS name. A browser running
on the host cannot resolve or directly access that name.

**Solution:** nginx was configured to proxy the browser's same-origin
`/api/` requests internally to `http://backend-svc:8080/api/`. The browser
therefore calls `http://localhost:8080/api/...`, and nginx performs the
cluster-internal Service lookup. The backend remains protected as a ClusterIP.

![Earlier browser connection error](screenshot/backend%20unreachable%20ui%20.png)

![Successful frontend/backend connection](screenshot/connection%20with%20backend.png)

### 7.3 Incorrect API path

**Error:**

```text
Cannot GET /api/task
```

**Cause:** The required endpoint is `/api/tasks` plural, not `/api/task`.

**Solution:** The correct command is:

```bash
curl http://localhost:8081/api/tasks
```

### 7.4 Frontend nginx configuration was initially ignored

**Error:** Requests to `/api/status` returned the frontend HTML page instead of
backend JSON.

**Cause:** The provided image's nginx master configuration did not include the
mounted `/etc/nginx/conf.d/default.conf` file.

**Solution:** A complete nginx configuration was supplied through a ConfigMap,
copied to a writable temporary path, and started with:

```text
nginx -c /tmp/nginx.conf -g 'daemon off;'
```

The corrected response became:

```json
{ "status": "ok", "db": "connected" }
```

### 7.5 ConfigMap key and indentation issue

**Error:** The frontend container could not find `nginx.conf` after the
configuration mount was changed.

**Cause:** The ConfigMap content was initially stored under the old
`default.conf` key and the replacement block had incorrect YAML indentation.

**Solution:** The ConfigMap was corrected to contain a properly indented
`nginx.conf` block. The Deployment was restarted and the live ConfigMap was
verified before testing the endpoint again.

### 7.6 Runtime frontend configuration

**Error:** The frontend image expected `config.js`, but overriding its default
entrypoint could skip creation of that file.

**Cause:** The image contains `config.js.template`, which must be rendered at
container startup using `BACKEND_URL`.

**Solution:** The startup command renders the template before starting nginx:

```text
config.js.template -> config.js
```

The generated configuration was verified inside the running Pod.

### 7.7 Secret and configuration contract risk

The assignment specification distinguishes `DB_*` backend variables from
`POSTGRES_*` database-image variables. Both sets were supplied with matching
values. The report also documents that Kubernetes Secrets are base64-encoded,
not encrypted at rest by default. In a production system, the example
development password would be replaced and stronger secret management would
be used.

## 8. Learning outcomes

### LO1 - Core Kubernetes concepts and architecture

**Achieved.** The report explains the control-plane components, scheduling,
Deployments, ReplicaSets, Pods, Services, ConfigMap, Secret, PVC, ResourceQuota,
and LimitRange.

### LO2 - Deploy and manage applications using Kubernetes resources

**Achieved.** The application uses a Namespace, ConfigMap, Secret, three
Deployments, three Services, a PVC, ResourceQuota, and LimitRange. The live
cluster verifies that these resources work together.

### LO3 - Operate kubectl for management and troubleshooting

**Achieved.** kubectl was used to switch contexts, apply resources, inspect
workloads, port-forward the backend, execute curl from a Pod, delete a Pod,
watch its replacement, inspect quota usage, and troubleshoot failed
configuration.

### LO4 - Implement persistent storage using volumes

**Achieved.** The database uses a 1Gi PVC mounted at the PostgreSQL data path.
Task data remained available after backend Pod replacement, and the PVC was
verified as `Bound`.

### LO5 - Apply namespace-based multi-tenancy

**Achieved for Unit I scope.** All assignment resources are isolated in
`dso202-assignment-01`, with namespace-level ResourceQuota and LimitRange.
RBAC is optional bonus material and was not required for the Unit I core task.

## 9. Final conclusion

The required Kubernetes deployment outcomes have been achieved. The final
system runs the three provided images in separate Pods, exposes only the
frontend externally, connects the tiers through Kubernetes Services, persists
database data through a PVC, applies namespace resource governance, and
demonstrates CRUD, DNS resolution, self-healing, and persistence.

The screenshots and terminal evidence document both successful operation and
the troubleshooting process used to reach the final state. The most important
technical issue was the difference between cluster-internal Service DNS and
host-browser networking; the nginx same-origin proxy solved that issue without
exposing the backend Service.

## Appendix A - Complete screenshot index

All screenshots supplied in the submission folder are included below. Some
show the same verification from different points in the workflow; they are
retained so the complete evidence set is documented.

![Screenshot 1 - Kubernetes resources](screenshot/Screenshot%201.png)

![Screenshot 2.1 - application/API verification](screenshot/Screenshot%202.1.png)

![Screenshot 2.2 - task creation](screenshot/Screenshot%202.2.png)

![Screenshot 3 - task listing](screenshot/Screenshot%203.png)

![Screenshot 4 - task update](screenshot/Screenshot%204.png)

![Screenshot 5 - task deletion](screenshot/Screenshot%205.png)

![Screenshot 6 - Service DNS resolution](screenshot/Screenshot%206.png)

![Screenshot 7 - Pod replacement watch](screenshot/Screenshot%207.png)

![Screenshot 7.1 - replacement Pod running](screenshot/Screenshot%207.1.png)

![Screenshot 9 - persistence after replacement](screenshot/Screenshot%209.png)

![Task list through port-forward](screenshot/apitask%20list%20using%20localhost%208081.png)

![Initial browser connection error](screenshot/backend%20unreachable%20ui%20.png)

![Backend connection check](screenshot/connection%20with%20backend.png)

![Final CRUD terminal evidence](screenshot/crud%20final.png)

![CRUD operation terminal evidence](screenshot/curd%20operatiion%20.png)

![Backend Pod deletion](screenshot/delete%20pod%20-n%20dso202-assignment-01%20-ltier=backend.png)

![Browser and terminal evidence captured at 4:19 pm](screenshot/Screenshot%202026-09-08%20at%204.19.34%E2%80%AFpm.png)

![Browser and terminal evidence captured at 4:26 pm](screenshot/Screenshot%202026-09-08%20at%204.26.11%E2%80%AFpm.png)

![Successful browser connection](screenshot/sucess%20connection%20with%20backend%20.png)
