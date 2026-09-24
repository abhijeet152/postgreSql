Here’s a simple example of how you can configure your application Deployment to connect to PostgreSQL running in a **different namespace**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: app-namespace   # 👈 Your application namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          env:
            - name: DATABASE_HOST
              value: "postgres-service.db-namespace.svc.cluster.local"  # 👈 FQDN of Postgres Service
            - name: DATABASE_PORT
              value: "5432"
            - name: DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: DATABASE_NAME
              value: "mydatabase"
```

### 🔎 Breakdown
- `namespace: app-namespace` → Your app runs in its own namespace.
- `DATABASE_HOST` → Uses the **fully qualified domain name (FQDN)** of the Postgres Service:
  ```
  <service-name>.<namespace>.svc.cluster.local
  ```
  Example: `postgres-service.db-namespace.svc.cluster.local`
- Secrets (`postgres-secret`) → Store sensitive credentials securely.
- Configurable environment variables → Your app reads them to build the connection string.

---

💡 **Tip:** If you want to simplify things, you can deploy both the app and Postgres in the same namespace. Then you only need `postgres-service:5432` instead of the full FQDN.

Here’s a clean example of a **Postgres Service manifest** that works with the Deployment snippet I showed earlier:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: db-namespace   # 👈 PostgreSQL namespace
spec:
  selector:
    app: postgres           # 👈 Must match labels in your Postgres Deployment/Pod
  ports:
    - protocol: TCP
      port: 5432            # 👈 Service port
      targetPort: 5432      # 👈 Container port
  type: ClusterIP           # 👈 Default, internal-only access
```

### 🔎 Breakdown
- `namespace: db-namespace` → The Service lives in the same namespace as your Postgres Pods.
- `selector.app: postgres` → Must match the labels defined in your Postgres Deployment/StatefulSet so the Service routes traffic correctly.
- `port: 5432` → Exposes Postgres on the standard port.
- `type: ClusterIP` → Keeps the Service internal to the cluster. Your app connects using DNS, not an external IP.

### 🧩 How it ties together
- Your **application Deployment** (in `app-namespace`) uses:
  ```
  postgres-service.db-namespace.svc.cluster.local:5432
  ```
  as the host.
- The **Postgres Service** (in `db-namespace`) routes traffic to the Postgres Pods.

---

💡 If you ever need external access (e.g., connecting from outside the cluster), you can change `type: ClusterIP` to `NodePort` or `LoadBalancer`. But for app-to-Postgres communication inside Kubernetes, `ClusterIP` is the right choice.
