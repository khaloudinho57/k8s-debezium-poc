## 📘 `README.md` — Kafka + PostgreSQL + Debezium on Kubernetes

### 🛠️ Stack Overview

This setup uses:

* **PostgreSQL** (with logical replication enabled)
* **Apache Kafka** (Zookeeper + Kafka)
* **Debezium Connect** (for CDC from Postgres to Kafka)
* **Kubernetes** (Kind or any cluster)

All components run as K8s Deployments with `ClusterIP` services. Communication is internal unless explicitly port-forwarded.

---

### 📁 Components

| Component     | Image                            | Port |
| ------------- | -------------------------------- | ---- |
| PostgreSQL    | `quay.io/debezium/postgres:15`   | 5432 |
| Kafka         | `quay.io/debezium/kafka:2.5`     | 9092 |
| Zookeeper     | `quay.io/debezium/zookeeper:2.5` | 2181 |
| Debezium Conn | `quay.io/debezium/connect:2.5`   | 8083 |

---

### 🚀 Deployment

All manifests are in `k8s-debezium/`. Apply them with:

```bash
kubectl apply -f k8s-debezium/
```

Order doesn’t matter — Kubernetes handles dependencies via selectors.

---

### 🔌 Debezium Connector Setup

After everything is running, register the connector:

#### 1. Port-forward Debezium Connect:

```bash
kubectl port-forward svc/debezium-connect 8083:8083
```

#### 2. Create `postgres-connector.json`:

```json
{
  "name": "postgres-users-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "postgres",
    "database.dbname": "testdb",
    "topic.prefix": "dbserver1",
    "table.include.list": "public.users",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_pub"
  }
}
```

#### 3. Register it via `curl`:

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @postgres-connector.json
```

---

### 🧪 Testing Debezium CDC

#### Insert into PostgreSQL:

```sql
INSERT INTO users (username, email, password_hash)
VALUES ('alice', 'alice@example.com', 'hash123');
```

#### Read from Kafka:

Launch Kafka tools pod:

```bash
kubectl run kafka-tools --rm -it \
  --image=bitnami/kafka:latest \
  --restart=Never \
  --overrides='
  {
    "apiVersion": "v1",
    "spec": {
      "securityContext": {
        "runAsUser": 0
      }
    }
  }' -- bash
```

Inside the pod:

```bash
/opt/bitnami/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic dbserver1.public.users \
  --from-beginning
```

Expected output:

```json
{
  "op": "c",
  "after": {
    "id": 1,
    "username": "alice",
    "email": "alice@example.com",
    ...
  }
}
```

---

### 🧹 How to Clean a Kafka Topic

Delete and recreate topic:

```bash
/opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --delete --topic dbserver1.public.users

/opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --create --topic dbserver1.public.users \
  --partitions 1 --replication-factor 1
```

---

### 🛠️ Troubleshooting

* **No messages in Kafka?**

  * Verify connector status: `curl localhost:8083/connectors/postgres-users-connector/status`
  * Make sure Debezium sees DB inserts after connector is running
  * Check logs: `kubectl logs deployment/debezium-connect`
  * Check that Kafka topic exists
