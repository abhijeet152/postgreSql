# PostgreSql on kubernetes
## Create normal deployment file and service file to deploy 

Below is a simple Kubernetes setup for PostgreSQL using **two YAML files**:

1. `postgres-deployment.yaml` — creates the PostgreSQL Pod
2. `postgres-service.yaml` — exposes PostgreSQL inside the Kubernetes cluster

This is suitable for learning/testing. For production, you would normally add a **PersistentVolume**, **Secret**, and preferably a **StatefulSet**.

### 1. PostgreSQL Deployment

Save it as:

```text
vi postgres-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  labels:
    app: postgres

spec:
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
          image: postgres:16
          ports:
            - containerPort: 5432

          env:
            - name: POSTGRES_DB
              value: mydatabase

            - name: POSTGRES_USER
              value: postgres

            - name: POSTGRES_PASSWORD
              value: postgres123
```



### 2. PostgreSQL Service

Save it as:

```text
vi postgres-service.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service

spec:
  selector:
    app: postgres

  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432

  type: ClusterIP
```


### 3. Deploy PostgreSQL

```bash
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
```

Check the deployment:

```bash
kubectl get deployment
```

Check the Pod:

```bash
kubectl get pods
```

Check the Service:

```bash
kubectl get svc
```

You should see something similar to:

```text
NAME               TYPE        CLUSTER-IP      PORT(S)
postgres-service   ClusterIP   10.96.123.45    5432/TCP
```

### 4. How another application connects

Suppose you have a **FastAPI application** deployed in the same Kubernetes cluster.

It can connect to PostgreSQL using:

```text
Host: postgres-service
Port: 5432
Database: mydatabase
Username: postgres
Password: postgres123
```

For example, a PostgreSQL connection string would be:

```text
postgresql://postgres:postgres123@postgres-service:5432/mydatabase
```

The important part is:

```text
postgres-service
```

Kubernetes DNS resolves that service name to the PostgreSQL Pod.

### Architecture

```text
                Kubernetes Cluster
┌─────────────────────────────────────────────┐
│                                             │
│   FastAPI Pod                               │
│       │                                     │
│       │ postgresql://                       │
│       ▼                                     │
│   postgres-service                          │
│   ClusterIP :5432                           │
│       │                                     │
│       ▼                                     │
│   PostgreSQL Pod                            │
│   postgres:16                               │
│       │                                     │
│       ▼                                     │
│   PostgreSQL Database                       │
│                                             │
└─────────────────────────────────────────────┘
```

**One important point:** this basic Deployment does **not** persist database data if the PostgreSQL Pod is deleted/recreated. For a real application, the next step should be adding a **PersistentVolume/PersistentVolumeClaim** and a **Kubernetes Secret** for the PostgreSQL password.