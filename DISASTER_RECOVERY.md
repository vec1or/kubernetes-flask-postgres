# Disaster Recovery Plan

## Kubernetes Flask + PostgreSQL Infrastructure

**Project:** Kubernetes Flask PostgreSQL
**Environment:** Local Ubuntu Server VM (VirtualBox)
**Orchestration:** K3s Kubernetes
**Deployment:** Helm
**Database:** PostgreSQL 17
**Container Registry:** GitHub Container Registry (GHCR)

---

## 1. Purpose

This document describes the disaster recovery procedures for a containerized Flask application backed by PostgreSQL and deployed on a single-node K3s Kubernetes cluster.

The objectives are to restore application availability, recover database information from backups, and verify that the system works correctly after a failure.

This runbook is designed for an educational environment and is not a validated production disaster recovery plan.

## 2. Infrastructure Overview

```text
GitHub Repository
       |
       v
GitHub Actions CI/CD
       |
       v
GitHub Container Registry
       |
       v
Ubuntu Server (VirtualBox VM)
       |
       v
K3s Kubernetes Cluster
       |
       +--- Namespace: devops-helm
       |       |
       |       +--- Flask Deployment (2 replicas)
       |       |       |
       |       |       +--- Flask Service
       |       |
       |       +--- PostgreSQL StatefulSet
       |               |
       |               +--- PostgreSQL Service
       |               |
       |               +--- PersistentVolumeClaim
       |
       +--- Helm Chart
       |
       +--- Local Database Backups
```

### Infrastructure Components

| Component | Purpose |
|---|---|
| Ubuntu Server VM | Hosts the local infrastructure |
| K3s | Runs and manages Kubernetes workloads |
| Helm | Manages application releases |
| Flask | Provides the application API |
| PostgreSQL | Stores application data |
| PVC | Provides persistent database storage |
| ConfigMap | Stores non-sensitive application settings |
| Kubernetes Secret | Stores database credentials |
| GHCR | Stores container images |
| GitHub Actions | Automates testing, scanning, building, and deployment |

## 3. Recovery Objectives

| Metric | Target |
|---|---|
| RPO (Recovery Point Objective) | 24 hours |
| RTO (Recovery Time Objective) | 2 hours |

**RPO:** The maximum targeted data-loss window is 24 hours.

**RTO:** The targeted time to restore service availability is 2 hours.

These are proposed training objectives, not measured guarantees. Meeting them requires automated backups, copies outside the affected VM, and successful timed recovery tests.

## 4. Backup Strategy

PostgreSQL backups are created using `pg_dump`.

### Backup Location

Current local backup directory:

```text
/home/devops/devops-labs/postgres-backups/
```

Example backup:

```text
demoapp-no-owner.sql
```

### Create a Database Backup

```bash
mkdir -p ~/devops-labs/postgres-backups
chmod 700 ~/devops-labs/postgres-backups

sudo k3s kubectl exec -n devops-helm demo-app-postgres-0 -- \
  pg_dump -U devops -d demoapp --no-owner \
  > ~/devops-labs/postgres-backups/demoapp-no-owner.sql

chmod 600 ~/devops-labs/postgres-backups/demoapp-no-owner.sql
```

The `--no-owner` option helps restore the database into an environment where the original PostgreSQL role may not exist.

### Backup Recommendations

- Schedule backups at least daily to support the proposed 24-hour RPO.
- Keep multiple backup versions.
- Store copies outside the Ubuntu Server VM.
- Restrict access to backup files.
- Periodically test restoration.
- Monitor backup failures.

**Important:** The current local backup directory is on the same VM as Kubernetes. It is not sufficient for recovery from complete VM or virtual-disk loss.

## 5. Disaster Scenarios

### Scenario A — Flask Pod Failure

**Symptoms:** Flask Pod is unavailable, restarting, or failing readiness checks.

Check the Deployment:

```bash
sudo k3s kubectl get deployments,pods -n devops-helm
```

Inspect a failing Pod:

```bash
sudo k3s kubectl describe pod <flask-pod-name> -n devops-helm
sudo k3s kubectl logs <flask-pod-name> -n devops-helm
```

Kubernetes normally creates a replacement Pod automatically when a Deployment replica is deleted or fails.

Verify recovery:

```bash
sudo k3s kubectl rollout status deployment/demo-app-flask \
  -n devops-helm
```

Expected result: two available Flask replicas.

### Scenario B — Failed Application Deployment

**Symptoms:** New Pods fail to start, the rollout stalls, or readiness checks fail.

Inspect the Deployment:

```bash
sudo k3s kubectl get pods -n devops-helm
sudo k3s kubectl describe deployment demo-app-flask -n devops-helm
```

Inspect Kubernetes events:

```bash
sudo k3s kubectl get events -n devops-helm \
  --sort-by=.lastTimestamp
```

If the failed change was deployed using Helm, inspect Helm history:

```bash
helm history demo-app -n devops-helm
```

Identify the previous successful release revision and roll back:

```bash
helm rollback demo-app <known-good-revision> \
  -n devops-helm \
  --wait --timeout 5m
```

Verify availability:

```bash
sudo k3s kubectl get deployments,pods -n devops-helm
```

Avoid mixing direct `kubectl set image` changes with Helm-managed image updates, because this can cause field ownership conflicts.

### Scenario C — PostgreSQL Data Loss

**Symptoms:** Application database tables are missing, records have been deleted, or the database requires restoration.

Do not overwrite the active database immediately.

First identify an appropriate backup and restore it into an isolated test database.

Create the test database:

```bash
sudo k3s kubectl exec -n devops-helm demo-app-postgres-0 -- \
  createdb -U devops demoapp_restore_test
```

Restore the backup:

```bash
sudo k3s kubectl exec -i -n devops-helm demo-app-postgres-0 -- \
  psql -X -v ON_ERROR_STOP=1 \
  -U devops -d demoapp_restore_test \
  < ~/devops-labs/postgres-backups/demoapp-no-owner.sql
```

This procedure assumes the test database does not already contain objects from a previous restore.

Verify data:

```bash
sudo k3s kubectl exec -n devops-helm demo-app-postgres-0 -- \
  psql -U devops -d demoapp_restore_test \
  -c "SELECT * FROM messages;"
```

Only after validating backup integrity and coordinating application downtime should a recovery into the active database be considered.

For a complete loss of PostgreSQL storage, a separate database instance and new persistent storage would first need to be provisioned.

### Scenario D — Kubernetes Cluster Failure

**Symptoms:** Kubernetes API is unavailable or workloads cannot be managed.

Check the K3s service:

```bash
sudo systemctl status k3s --no-pager
```

Inspect service logs:

```bash
sudo journalctl -u k3s -n 100 --no-pager
```

Check system resources:

```bash
free -h
df -h /
```

Check Kubernetes availability:

```bash
sudo k3s kubectl get nodes
```

If K3s can safely be restarted after diagnosis:

```bash
sudo systemctl restart k3s
```

Verify recovery:

```bash
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A
```

Do not uninstall K3s or delete Kubernetes storage as an initial troubleshooting step.

### Scenario E — Complete VM Loss

If the Ubuntu VM or virtual disk is lost, recovery requires:

1. Provisioning a replacement Ubuntu Server VM.
2. Installing and configuring K3s.
3. Installing Helm.
4. Restoring the Kubernetes application manifests and Helm Chart from GitHub.
5. Recreating Kubernetes Secrets from a secure external source.
6. Making the Flask container image available from GHCR.
7. Installing the Helm release.
8. Restoring PostgreSQL from an external backup.
9. Reconfiguring GitHub Actions deployment access.
10. Verifying application functionality.

A complete cluster-state recovery additionally requires suitable K3s datastore and configuration backups. The Helm repository and SQL dump alone do not restore the entire previous cluster state.

## 6. Application Recovery with Helm

Repository:

[https://github.com/vec1or/kubernetes-flask-postgres](https://github.com/vec1or/kubernetes-flask-postgres)

Clone the project:

```bash
git clone https://github.com/vec1or/kubernetes-flask-postgres.git
cd kubernetes-flask-postgres
```

Ensure that the K3s cluster is operational and the desired Flask image is accessible from GHCR.

Create the namespace if needed:

```bash
kubectl create namespace devops-helm
```

Provision the required `postgres-secret` securely before installing the chart. The actual database password must be managed separately from this Git repository.

Validate:

```bash
helm lint ./demo-app-chart
```

Install in a clean environment:

```bash
helm install demo-app ./demo-app-chart \
  -n devops-helm \
  --wait --timeout 5m
```

If the release already exists, use `helm upgrade` after reviewing the proposed changes and ensuring that the desired image tag is correct.

**Important:** Installing a new PostgreSQL StatefulSet creates a new database if its storage is empty. Historical data must then be restored from a backup. The initialization SQL only creates the initial schema and demonstration records.

## 7. Post-Recovery Verification

### Kubernetes Health

```bash
sudo k3s kubectl get nodes
sudo k3s kubectl get pods,svc,pvc -n devops-helm
```

Verify the Deployment:

```bash
sudo k3s kubectl rollout status deployment/demo-app-flask \
  -n devops-helm
```

### Application Health

Start port forwarding:

```bash
sudo k3s kubectl port-forward \
  -n devops-helm \
  svc/demo-app-flask-service 5003:5000
```

In another terminal:

```bash
curl -i http://127.0.0.1:5003/
curl -i http://127.0.0.1:5003/health
curl -i http://127.0.0.1:5003/messages
curl -i http://127.0.0.1:5003/metrics
```

Expected outcomes:

- `/` returns HTTP 200.
- `/health` reports application and database availability.
- `/messages` returns expected records.
- `/metrics` exposes Prometheus metrics.

### Database Integrity

Compare record counts between the restored database and the last known backup or documented baseline.

For stronger integrity validation, compare schema, important records, constraints, and application-level functionality—not only row counts.

## 8. Security Considerations

- Never store database credentials in Git.
- Protect Kubernetes kubeconfig files.
- Apply least-privilege RBAC to deployment automation.
- Prefer immutable container image tags or digests.
- Keep backups in access-controlled storage outside the primary VM.
- Regularly rotate credentials and review access.
- Do not grant a self-hosted GitHub Actions Runner unrestricted cluster-admin privileges.

## 9. Limitations and Future Improvements

The current implementation runs on a single-node K3s cluster. Multiple Flask replicas improve application-level redundancy but cannot protect against loss of the entire VM.

Future improvements include:

- Automated PostgreSQL backup schedules.
- Off-host or object-storage backup copies.
- Backup retention and verification.
- K3s datastore snapshots and restore testing.
- Dedicated multi-node Kubernetes infrastructure.
- Timed disaster recovery drills to measure actual RPO/RTO.
- Automated post-recovery smoke tests.
- Aligning Helm and CI/CD as one consistent deployment mechanism.

## 10. Recovery Checklist

- [ ] Incident identified and recorded.
- [ ] Affected components identified.
- [ ] Last known good container image identified.
- [ ] Latest valid backup located.
- [ ] Kubernetes cluster restored or verified.
- [ ] PostgreSQL data restored and validated.
- [ ] Flask Deployment available.
- [ ] Health and API endpoints checked.
- [ ] Monitoring and alerts checked.
- [ ] Incident details and recovery time documented.

---

**Document status:** Educational disaster recovery runbook
**RPO target:** 24 hours
**RTO target:** 2 hours
**Validation:** Partial — application rollback and isolated PostgreSQL restoration have been practiced; complete VM-loss recovery has not been tested.
