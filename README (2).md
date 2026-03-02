# Items Microservice

A production-ready Python CRUD REST microservice built with **FastAPI**, **PostgreSQL**, **Docker**, and **Kubernetes**.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                      │
│   ┌──────────────┐        ┌──────────────────────┐  │
│   │   Ingress    │──────▶ │   items-api (x2+)    │  │
│   │  (nginx)     │        │   FastAPI / uvicorn  │  │
│   └──────────────┘        └─────────┬────────────┘  │
│                                     │                │
│                            ┌────────▼─────────┐      │
│                            │  PostgreSQL 16   │      │
│                            │  (StatefulSet)   │      │
│                            └──────────────────┘      │
└─────────────────────────────────────────────────────┘
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| API Framework | FastAPI 0.115 |
| ORM | SQLAlchemy 2.0 |
| Database | PostgreSQL 16 |
| Migrations | Alembic |
| Server | Uvicorn |
| Container | Docker (multi-stage) |
| Orchestration | Kubernetes |
| Scaling | HPA (CPU + Memory) |

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness check |
| `GET` | `/ready` | Readiness check (DB ping) |
| `POST` | `/items` | Create item |
| `GET` | `/items` | List items (paginated) |
| `GET` | `/items/{id}` | Get item by ID |
| `PUT` | `/items/{id}` | Update item |
| `DELETE` | `/items/{id}` | Delete item |
| `GET` | `/docs` | Interactive Swagger UI |

## Project Structure

```
microservice/
├── app/
│   ├── __init__.py
│   ├── main.py        # FastAPI app, routes
│   ├── database.py    # SQLAlchemy engine & session
│   ├── models.py      # ORM models
│   ├── schemas.py     # Pydantic request/response schemas
│   └── crud.py        # Database operations
├── migrations/
│   └── versions/
│       └── 001_initial.py
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-secrets.yaml
│   ├── 02-configmap.yaml
│   ├── 03-postgres.yaml      # StatefulSet + headless Service
│   ├── 04-deployment.yaml    # Deployment + ClusterIP Service
│   ├── 05-hpa-ingress.yaml   # HPA + Ingress
│   └── 06-networkpolicy.yaml # Network isolation
├── Dockerfile             # Multi-stage build
├── docker-compose.yml     # Local development
├── requirements.txt
└── alembic.ini
```

---

## Quick Start (Local with Docker Compose)

```bash
# Start API + DB
docker compose up --build

# API is available at http://localhost:8000
# Swagger docs at http://localhost:8000/docs

# Stop
docker compose down -v
```

## Example API Usage

```bash
# Create an item
curl -X POST http://localhost:8000/items \
  -H "Content-Type: application/json" \
  -d '{"name": "Widget", "description": "A useful widget", "price": 9.99, "in_stock": true}'

# List items
curl http://localhost:8000/items

# Get item by ID
curl http://localhost:8000/items/1

# Update item
curl -X PUT http://localhost:8000/items/1 \
  -H "Content-Type: application/json" \
  -d '{"price": 12.99, "in_stock": false}'

# Delete item
curl -X DELETE http://localhost:8000/items/1
```

---

## Kubernetes Deployment

### Prerequisites
- `kubectl` configured for your cluster
- Container image pushed to a registry

### 1. Build & Push Image

```bash
docker build -t your-registry/items-microservice:1.0.0 .
docker push your-registry/items-microservice:1.0.0
```

### 2. Update Image Reference

Edit `k8s/04-deployment.yaml`:
```yaml
image: your-registry/items-microservice:1.0.0
```

### 3. Update Secrets

Generate base64 values and update `k8s/01-secrets.yaml`:
```bash
echo -n "your-password" | base64
```

### 4. Apply Manifests

```bash
# Apply in order
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-secrets.yaml
kubectl apply -f k8s/02-configmap.yaml
kubectl apply -f k8s/03-postgres.yaml
kubectl apply -f k8s/04-deployment.yaml
kubectl apply -f k8s/05-hpa-ingress.yaml
kubectl apply -f k8s/06-networkpolicy.yaml

# Or apply all at once
kubectl apply -f k8s/
```

### 5. Verify Deployment

```bash
kubectl get all -n items-microservice
kubectl logs -l app=items-api -n items-microservice
```

### 6. Run Migrations

```bash
kubectl exec -it deployment/items-api -n items-microservice -- \
  alembic upgrade head
```

---

## Database Migrations (Alembic)

```bash
# Create a new migration
alembic revision --autogenerate -m "add category column"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

---

## Production Considerations

- **Secrets**: Replace static secrets with a secrets manager (AWS Secrets Manager, HashiCorp Vault, or Sealed Secrets)
- **TLS**: Uncomment the TLS section in the Ingress manifest and provision a certificate (cert-manager recommended)
- **Monitoring**: Add Prometheus annotations and a `/metrics` endpoint (e.g. `prometheus-fastapi-instrumentator`)
- **Logging**: Ship structured JSON logs to a log aggregator (Loki, Elasticsearch)
- **Database**: For high availability, use a managed PostgreSQL service (RDS, Cloud SQL) or a Postgres operator (CloudNativePG)
