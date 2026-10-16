# Kubernetes Flask + PostgreSQL

A cloud-native deployment of a Flask application and PostgreSQL database using Kubernetes (K3s) and Helm.

## Architecture

```text
                   Kubernetes (K3s)
                         |
                  Flask Service
                         |
                 Flask Deployment
                  /           \
           REST API        /metrics
               |
         PostgreSQL Service
               |
        PostgreSQL StatefulSet
               |
               PVC
               |
        PersistentVolume
```

## Tech Stack

- Kubernetes (K3s)
- Helm
- Docker / containerd
- Python Flask
- PostgreSQL 17
- ConfigMaps and Secrets
- Persistent Volumes (PVC)
- Readiness and Liveness Probes

## Features

- Flask application deployed using Kubernetes Deployment
- PostgreSQL deployed using StatefulSet
- Persistent database storage using PVC
- Automatic database initialization with `init.sql`
- Configurable application settings through Helm Values
- PostgreSQL credentials loaded from Kubernetes Secret
- Internal communication through Kubernetes Services
- HTTP health checks and Prometheus metrics
- Helm installation, upgrades and rollbacks

## Project Structure

```text
kubernetes-demo-app/
├── demo-app-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── init.sql
│   └── templates/
│       ├── configmap.yaml
│       ├── flask-deployment.yaml
│       ├── flask-service.yaml
│       ├── postgres-init-configmap.yaml
│       ├── postgres-service.yaml
│       └── postgres-statefulset.yaml
├── flask.yaml
├── postgres.yaml
└── README.md
```

## Prerequisites

- A running Kubernetes cluster (tested using K3s)
- Helm CLI
- Docker Engine to build the Flask image
- Access to the companion [Flask application repository](https://github.com/vec1or/demo-app)

## Build and Import Flask Image

K3s uses its own containerd image store.

```bash
git clone https://github.com/vec1or/demo-app.git
cd demo-app

docker build -t demo-flask:k3s-v2 .
docker save demo-flask:k3s-v2 -o /tmp/demo-flask.tar

sudo k3s ctr images import /tmp/demo-flask.tar
rm /tmp/demo-flask.tar
```

The included Helm values use `imagePullPolicy: Never`, so the image must exist on the Kubernetes node. This example is intended for a single-node local cluster.

## Helm Installation

Clone this repository and navigate into the project.

Create a namespace:

```bash
kubectl create namespace devops-helm
```

Create a PostgreSQL Secret locally:

```bash
kubectl -n devops-helm create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD=replace-with-local-password
```

Install the chart:

```bash
helm install demo-app ./demo-app-chart \
  -n devops-helm \
  --wait --timeout 5m
```

The chart references an existing Secret named `postgres-secret`. The Secret is not created by Helm.

## Verify Deployment

```bash
kubectl get pods,svc,pvc -n devops-helm
helm list -n devops-helm
```

Check PostgreSQL:

```bash
kubectl exec -n devops-helm demo-app-postgres-0 -- \
  psql -U devops -d demoapp -c "SELECT * FROM messages;"
```

## Test Flask API

Forward the Flask Service:

```bash
kubectl port-forward -n devops-helm \
  svc/demo-app-flask-service 5003:5000
```

In another terminal:

```bash
curl http://127.0.0.1:5003/
curl http://127.0.0.1:5003/health
curl http://127.0.0.1:5003/messages
curl http://127.0.0.1:5003/metrics
```

## Helm Upgrade and Rollback

Scale Flask to two replicas:

```bash
helm upgrade demo-app ./demo-app-chart \
  -n devops-helm \
  --set flask.replicas=2 \
  --wait
```

Check history:

```bash
helm history demo-app -n devops-helm
```

Roll back to the previous release revision (replace `<revision>` with the correct number):

```bash
helm rollback demo-app <revision> -n devops-helm
```

## Data Persistence

PostgreSQL uses a StatefulSet with a dynamically provisioned PVC through the `local-path` StorageClass.

The `init.sql` script runs only when PostgreSQL initializes an empty database directory.

**Warning:** Removing PVCs may permanently delete database data, depending on the storage reclaim policy. Back up important data before deleting storage resources.

## Security Notes

- PostgreSQL credentials are stored in a Kubernetes Secret rather than embedded in deployment manifests.
- Kubernetes Secrets are not automatically encrypted at rest.
- The default example is intended for local development, not production.
- Kubeconfig files, local secrets and database backups should not be committed to Git.
- For production, use a secure container registry, stronger credential management, RBAC and a production WSGI server.

## Related Projects

- [Flask Demo App](https://github.com/vec1or/demo-app)
- [Terraform Docker Infrastructure](https://github.com/vec1or/terraform-docker-infrastructure)
- [Ansible Ubuntu Automation](https://github.com/vec1or/ansible-ubuntu-automation)
- [DevOps Monitoring Stack](https://github.com/vec1or/devops-monitoring-stack)
