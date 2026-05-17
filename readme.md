**Kubernetes MySQL Persistent Volume Demo** — proving that database state survives pod crashes through Kubernetes persistent volume storage.

---

## Project at a Glance

| | |
|---|---|
| **Tools Used** | Kubernetes (kubectl), MySQL 5.7, YAML manifests |
| **Platform** | Kubernetes cluster (local or cloud-hosted) |
| **Languages** | YAML, SQL |
| **What It Does** | Deploys a MySQL container in Kubernetes with a persistent volume mount, inserts customer data, kills the pod to simulate a crash, and verifies the data persists across the restart |

---

## The Problem This Project Solves

Containers are ephemeral by design. When a container process terminates — whether due to a crash, an OOM kill, or an intentional restart — the container's writable layer is discarded. For stateless applications this is a feature, not a bug. For databases, it is catastrophic: any data written to the container's filesystem is lost the moment the process exits.

In Kubernetes, this problem is compounded by the pod lifecycle. Pods are not static processes; they are declarative objects that the control plane creates, destroys, and reschedules. A database pod that writes data to its local filesystem and then crashes loses everything when the pod restarts. This makes naive database deployments in Kubernetes inherently unreliable.

Persistent Volumes decouple storage from the pod lifecycle. By mounting a volume at the database's data directory, writes go to a storage backend that outlives the pod. When Kubernetes restarts the pod — whether after a crash or a node reboot — the new pod mounts the same volume and picks up exactly where the previous instance left off. This project demonstrates that guarantee end-to-end: data is inserted, the process is killed, the pod restarts, and the data is still there.

---

## Architecture

```
kubectl apply -f mysql-pod.yml
         |
         v
+----------------------------+
|  Kubernetes Pod: mysql     |
|  Container: mysql:5.7      |
|  Port: 3306                |
|  restartPolicy: Always     |
|                            |
|  VolumeMount               |
|  /var/lib/mysql  <------+  |
+--------------------------|--+
                           |
              +------------+------------+
              |  Volume: mysql-storage  |
              |  (Persistent Storage)   |
              +-------------------------+

Pod crashes (kill 1)
         |
         v
Kubernetes restarts pod automatically
         |
         v
New container mounts same volume
         |
         v
mysql /var/lib/mysql data intact
```

The pod's `restartPolicy: Always` ensures Kubernetes recreates the container on exit. The volume persists independently of the container lifecycle, so the MySQL data directory is preserved across restarts.

---

## Repository Structure

```
kubernetes-mysql-persistent-volume-demo/
├── README.md
├── mysql-pod.yml             # Pod spec: MySQL container with persistent volume mount
└── createCustomerData.sql    # SQL script: creates database, table, and inserts sample data
```

---

## Walkthrough

### Step 1: Deploy the MySQL pod

```bash
kubectl apply -f mysql-pod.yml
```

The manifest creates a pod named `mysql` running `mysql:5.7`, with a volume mounted at `/var/lib/mysql` — the directory MySQL uses to store all database files.

### Step 2: Watch the pod in a dedicated terminal

Open a terminal and run:

```bash
kubectl get pod mysql --watch
```

Leave this running. It will show the pod status in real time, including the restart that happens after the process is killed.

### Step 3: Access the running container

In a second terminal:

```bash
kubectl exec -it mysql -- /bin/bash
```

### Step 4: Connect to MySQL

Inside the container shell:

```bash
mysql -u root -p
# Password: YUCloudOps20XY!
```

### Step 5: Create the database, table, and data

Run the SQL commands from `createCustomerData.sql`:

```sql
CREATE DATABASE YORKU;
USE YORKU;

CREATE TABLE CUSTOMERS (
   NAME VARCHAR(20) NOT NULL,
   ID INT AUTO_INCREMENT,
   AGE INT NOT NULL,
   PRIMARY KEY (ID)
);

INSERT INTO CUSTOMERS (NAME, AGE) VALUES ('Khilan', 25);
INSERT INTO CUSTOMERS (NAME, AGE) VALUES ('Killian', 29);

SELECT * FROM CUSTOMERS;
```

Exit the MySQL prompt and the bash shell.

### Step 6: Kill the MySQL process to simulate a crash

While still inside the container:

```bash
kill 1
```

This sends SIGTERM to PID 1, which is the MySQL server process. The container exits. Observe the first terminal — the pod status transitions from `Running` to `Error` and back to `Running` as Kubernetes restarts it.

### Step 7: Reconnect and verify data persistence

```bash
kubectl exec -it mysql -- /bin/bash
mysql -u root -p
```

```sql
USE YORKU;
SELECT * FROM CUSTOMERS;
```

The data is intact. The volume mount preserved the MySQL data directory across the pod restart.

---

## How to Reproduce

Prerequisites: A running Kubernetes cluster with `kubectl` configured.

```bash
# Clone the repository
git clone <repo-url>
cd kubernetes-mysql-persistent-volume-demo

# Deploy the MySQL pod
kubectl apply -f mysql-pod.yml

# Wait for the pod to be Running
kubectl get pod mysql --watch

# Access the container and set up the database
kubectl exec -it mysql -- /bin/bash
# Inside the container:
mysql -u root -p
# Paste the contents of createCustomerData.sql
exit
exit

# Kill the pod process to simulate a crash
kubectl exec mysql -- kill 1

# Wait for Kubernetes to restart the pod
kubectl get pod mysql --watch

# Reconnect and verify persistence
kubectl exec -it mysql -- /bin/bash
mysql -u root -p
# Run: USE YORKU; SELECT * FROM CUSTOMERS;

# Clean up
kubectl delete pod mysql
```

---

## Key Observations

- The `restartPolicy: Always` on the pod spec is what drives the automatic restart. Without it, the pod would terminate permanently after the kill.
- The volume definition in `mysql-pod.yml` uses an unbound emptyDir-equivalent for local demonstration. In a production cluster this would reference a PersistentVolumeClaim backed by a StorageClass (EBS, GCE PD, Azure Disk, etc.).
- MySQL stores all its data files under `/var/lib/mysql`. Mounting a persistent volume at this path is sufficient to survive any number of container restarts.
- This pattern is a prerequisite for any stateful Kubernetes workload: databases, message queues, key-value stores, and any application that writes files that must outlive the container.

---

*Oluwaseyi Michael Falode · Cybersecurity & Cloud Security Engineer · Toronto, ON*
