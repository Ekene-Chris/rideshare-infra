# Rideshare Platform Infrastructure

Kubernetes manifests for the rideshare platform's cluster-wide resources. Individual microservice manifests live in their respective repositories.

## Repository Structure

```
├── namespaces.yaml                 # Namespace definitions
├── aws/                            # AWS/EKS infrastructure
│   ├── storage-classes.yaml        # EBS gp3 storage classes
│   ├── iam-roles.yaml              # IRSA service accounts
│   └── node-groups.yaml            # Node group reference
├── platform/
│   ├── ingress/
│   │   └── ingress-rules.yaml      # API routing rules
│   ├── secrets/
│   │   ├── secret-store.yaml       # ClusterSecretStore for AWS SM
│   │   └── external-secrets.yaml   # ExternalSecret CRs
│   ├── scaling/
│   │   ├── cluster-autoscaler.yaml # Cluster Autoscaler deployment
│   │   └── pod-disruption-budgets.yaml
│   └── stateful/
│       ├── redis-cluster.yaml      # 6-node Redis cluster
│       └── postgres-primary-replica.yaml  # Primary + 2 replicas
```

## Prerequisites

- EKS cluster with EBS CSI driver installed
- NGINX Ingress Controller (installed via Helm)
- External Secrets Operator (installed via Helm)
- `kubectl` configured with cluster access
- AWS Secrets Manager secrets created:
  - `rideshare/postgres` - PostgreSQL credentials
  - `rideshare/redis` - Redis password
  - `rideshare/app` - Application secrets
  - `rideshare/payment` - Payment provider keys
  - `rideshare/notifications` - Notification service keys

## Namespaces

| Namespace | Purpose |
|-----------|---------|
| `rideshare-platform` | Shared platform services, ingress rules, application secrets |
| `rideshare-data` | Stateful workloads (Redis, PostgreSQL) |

## Deployment Order

### 1. Initial Setup (one-time)

```bash
# Create namespaces
kubectl apply -f namespaces.yaml

# Apply storage classes
kubectl apply -f aws/storage-classes.yaml

# Apply IRSA service accounts
kubectl apply -f aws/iam-roles.yaml
```

### 2. Secrets Management

```bash
# Create ClusterSecretStore
kubectl apply -f platform/secrets/secret-store.yaml

# Create ExternalSecrets
kubectl apply -f platform/secrets/external-secrets.yaml

# Verify secrets are synced
kubectl get externalsecrets -n rideshare-data
kubectl get externalsecrets -n rideshare-platform
```

### 3. Stateful Workloads

```bash
# Deploy PostgreSQL
kubectl apply -f platform/stateful/postgres-primary-replica.yaml

# Deploy Redis cluster
kubectl apply -f platform/stateful/redis-cluster.yaml
```

### 4. Platform Components

```bash
# Deploy Cluster Autoscaler and PDBs
kubectl apply -f platform/scaling/cluster-autoscaler.yaml
kubectl apply -f platform/scaling/pod-disruption-budgets.yaml

# Apply Ingress rules (after microservices are deployed)
kubectl apply -f platform/ingress/ingress-rules.yaml
```

## Service Endpoints

### Internal (cluster DNS)

| Service | Endpoint |
|---------|----------|
| PostgreSQL Primary (writes) | `postgres-primary.rideshare-data.svc.cluster.local:5432` |
| PostgreSQL Replicas (reads) | `postgres-replicas.rideshare-data.svc.cluster.local:5432` |
| Redis Cluster | `redis-cluster.rideshare-data.svc.cluster.local:6379` |

### External (via Ingress)

| Path | Service |
|------|---------|
| `/api/v1/users`, `/api/v1/auth` | user-service |
| `/api/v1/rides` | ride-service |
| `/api/v1/drivers` | driver-service |
| `/api/v1/locations` | location-service |
| `/api/v1/payments` | payment-service |
| `/api/v1/notifications` | notification-service |

## Configuration

### Updating Cluster Name

Replace `rideshare-cluster` with your actual cluster name in:
- `aws/iam-roles.yaml` - IRSA role ARN
- `platform/scaling/cluster-autoscaler.yaml` - autodiscovery tags

### Updating AWS Region

Default region is `us-east-2`. Update in:
- `platform/secrets/secret-store.yaml`
- `.github/workflows/deploy.yaml`

### Updating Domain

Replace `rideshare.example.com` with your domain in:
- `platform/ingress/ingress-rules.yaml`

## GitHub Actions

The repository includes a GitHub Actions workflow (`.github/workflows/deploy.yaml`) that:
- Automatically deploys changed components on push to `main`
- Supports manual deployment of specific components via `workflow_dispatch`
- Detects file changes and only deploys affected resources
- Configures AWS and kubeconfig once per run

### Required Secrets

Configure these secrets in your GitHub repository:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

### Manual Deployment

Trigger a manual deployment from the Actions tab, selecting which component to deploy:
- `all` - Deploy everything
- `namespaces` - Namespace definitions only
- `aws` - Storage classes and IAM roles
- `secrets` - SecretStore and ExternalSecrets
- `scaling` - Cluster Autoscaler and PDBs
- `ingress` - Ingress rules
- `stateful` - Redis and PostgreSQL

## Troubleshooting

### Check External Secrets sync status
```bash
kubectl describe externalsecret postgres-credentials -n rideshare-data
```

### Check Redis cluster status
```bash
kubectl exec -it redis-cluster-0 -n rideshare-data -- redis-cli cluster info
kubectl exec -it redis-cluster-0 -n rideshare-data -- redis-cli cluster nodes
```

### Check PostgreSQL replication status
```bash
kubectl exec -it postgres-primary-0 -n rideshare-data -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

### View Cluster Autoscaler logs
```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

## Migration to Managed Services

This setup uses self-managed Redis and PostgreSQL for initial deployment. For production, consider migrating to:
- **Amazon ElastiCache** for Redis
- **Amazon RDS** or **Amazon Aurora** for PostgreSQL

The ExternalSecrets are already configured to pull credentials from AWS Secrets Manager, making the migration straightforward — update connection strings in your microservice configs.
