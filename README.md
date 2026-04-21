# PostgreSQL 18 Deployment on Kubernetes (Minikube)

This project demonstrates a simple Proof-of-Concept deployment of PostgreSQL 18 on Kubernetes using Minikube.

The objective of this lab is to help DBAs understand:

- Running PostgreSQL inside containers
- Kubernetes Stateful workloads
- Persistent storage for databases
- Pod restart and data persistence validation

---

# Architecture

```
+-----------------------------------+
|          Minikube Cluster         |
|                                   |
|   +---------------------------+   |
|   |        Service            |   |
|   |        postgres           |   |
|   +-------------+-------------+   |
|                 |                 |
|        +--------v--------+        |
|        |   StatefulSet   |        |
|        |     postgres    |        |
|        +--------+--------+        |
|                 |                 |
|            +----v----+            |
|            |  Pod    |            |
|            |postgres-0|           |
|            +----+----+            |
|                 |                 |
|       +---------v----------+      |
|       | Persistent Volume  |      |
|       | Claim (PVC)        |      |
|       +--------------------+      |
|                                   |
+-----------------------------------+
```

---

# Environment

| Component | Version |
|----------|---------|
| OS | Linux / Ubuntu |
| Kubernetes | Minikube |
| PostgreSQL | 18 |
| Container Runtime | Docker |

---

# 1 Install kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify installation

```bash
kubectl version --client
```

---

# 2 Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

chmod +x minikube-linux-amd64

sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

Verify installation

```bash
minikube version
```

---

# 3 Start Kubernetes Cluster

```bash
minikube start
```

Verify cluster

```bash
kubectl get nodes
```

Expected output

```
NAME       STATUS   ROLES
minikube   Ready    control-plane
```

---

# 4 Create PostgreSQL Deployment File

Create file

```bash
nano postgres-k8s.yaml
```

Paste the following configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  ports:
    - port: 5432
  clusterIP: None
  selector:
    app: postgres

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:18
        ports:
        - containerPort: 5432

        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: secret
        - name: POSTGRES_DB
          value: testdb

        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql

  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

---

# 5 Deploy PostgreSQL

```bash
kubectl apply -f postgres-k8s.yaml
```

Expected output

```
service/postgres created
statefulset.apps/postgres created
```

---

# 6 Verify Deployment

Check pods

```bash
kubectl get pods
```

Expected

```
postgres-0   1/1   Running
```

Check persistent storage

```bash
kubectl get pvc
```

Example

```
postgres-storage-postgres-0   Bound   1Gi
```

---

# 7 Check PostgreSQL Logs

```bash
kubectl logs postgres-0
```

Expected message

```
database system is ready to accept connections
```

---

# 8 Connect to PostgreSQL

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d testdb
```

Verify version

```sql
select version();
```

Expected

```
PostgreSQL 18.x
```

---

# 9 Verify Database Storage Path

Enter container

```bash
kubectl exec -it postgres-0 -- bash
```

Navigate to database directory

```bash
cd /var/lib/postgresql/18/docker
```

List files

```bash
ls
```

Typical directories

```
base
global
pg_wal
pg_stat
pg_xact
PG_VERSION
postgresql.conf
```

Exit container

```bash
exit
```

---

# 10 Test Persistent Storage

Connect again

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d testdb
```

Create table

```sql
CREATE TABLE k8s_test(id INT, name TEXT);

INSERT INTO k8s_test VALUES
(1,'PostgreSQL'),
(2,'Kubernetes'),
(3,'Minikube');

SELECT * FROM k8s_test;
```

---

# Restart Pod

```bash
kubectl delete pod postgres-0
```

Kubernetes automatically recreates the pod.

Verify

```bash
kubectl get pods
```

Reconnect and confirm the data still exists.

This confirms that **Persistent Volume storage works correctly**.

---

# Kubernetes Objects Used

| Object | Purpose |
|------|---------|
| Service | Internal cluster networking |
| StatefulSet | Stateful application management |
| Pod | Runs PostgreSQL container |
| PVC | Persistent storage for database |
| PV | Physical storage resource |

---

# Key Learning Points

- PostgreSQL runs inside a container managed by Kubernetes
- StatefulSet provides stable identity and storage
- Persistent Volumes protect database data
- Kubernetes automatically recreates failed pods

---

# Author

Clement
