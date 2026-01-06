# External Database Stack for Kubernetes

A production-ready setup for running PostgreSQL and S3-compatible storage on dedicated hardware, designed to be consumed by Kubernetes clusters via external services. This mirrors cloud-native patterns where databases are managed separately from compute clusters (like connecting GKE to Cloud SQL or EKS to RDS).

## Why This Approach?

In production cloud environments, you typically don't run stateful services like databases inside your Kubernetes cluster. Instead, you connect to managed services (RDS, Cloud SQL, etc.) that run outside the cluster. This setup replicates that pattern for homelab and on-premise environments:

- **Separation of concerns**: Database management is isolated from cluster operations
- **Resource efficiency**: No need to reserve cluster resources for storage overhead
- **Simplified upgrades**: Update cluster or databases independently
- **Cloud-like architecture**: Patterns that translate to production environments
- **Cost effective**: Repurpose older hardware as dedicated storage instead of scaling cluster

## What You'll Deploy

This stack provides two core services plus monitoring:

### Core Services (Tested & Production-Ready)
- **PostgreSQL 18** - Primary relational database on port 5432
- **SeaweedFS** - Distributed S3-compatible object storage on port 8333
- **PostgreSQL Exporter** - Prometheus metrics for database monitoring

### Optional Services (Commented Out)
- **Qdrant** - Vector database (untested, can be enabled)
- **Ollama** - AI model serving (untested, can be enabled)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster (192.168.1.x)                            │
│                                                             │
│  ┌──────────────┐      ┌──────────────┐                     │
│  │ Application  │      │ Application  │                     │
│  │   Pod A      │      │   Pod B      │                     │
│  └───────┬──────┘      └──────┬───────┘                     │
│          │                     │                            │
│          └─────────┬───────────┘                            │
│                    │                                        │
│          ┌─────────▼───────────────┐                        │
│          │  External Services      │                        │
│          │  - postgres-external    │                        │
│          │  - seaweedfs-s3-external│                        │
│          └─────────┬───────────────┘                        │
│                    │                                        │
└────────────────────┼────────────────────────────────────────┘
                     │ Network
                     │
┌────────────────────▼─────────────────────────────────────────┐
│ Docker Host (192.168.1.100)                                  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐      │
│  │  PostgreSQL  │  │  SeaweedFS   │  │   Postgres     │      │
│  │    :5432     │  │  S3 :8333    │  │ Exporter :9187 │      │
│  └──────────────┘  └──────────────┘  └────────────────┘      │
│                                                              │
│  Data: /mnt/data/{postgres,seaweedfs}                        │
└──────────────────────────────────────────────────────────────┘
```

**Two-Machine Setup:**
- **Docker Host** (Mini PC): Runs PostgreSQL and S3 services via Docker Compose
- **Kubernetes Workstation**: Where you run `kubectl` commands to configure cluster access

These can be the same physical machine if you run Kubernetes on the Docker host, but typically they are separate.

## Prerequisites

### Hardware Requirements
- **Minimum**: 4 CPU cores, 8GB RAM, 250GB storage
- **Recommended**: 8+ CPU cores, 16GB+ RAM, 1TB+ SSD/NVMe
- **OS**: Ubuntu 22.04/24.04, Debian 12, or similar Linux distribution

### Software Requirements
- Docker Engine 24.0+
- Docker Compose v2.20+
- Access to `/mnt/data` or similar mount point with sufficient space
- Network connectivity between cluster and Docker host

### Network Requirements
- Static IP address for Docker host (example: `192.168.1.100`)
- Network connectivity between cluster nodes and Docker host (same network recommended)
- Ports: 5432 (PostgreSQL), 8333 (S3), 9187 (metrics)
- Optional: Tailscale for secure mesh networking across different networks

---

## Part 1: Deploy Docker Services

> **📍 Location: Docker Host (Mini PC)**
> All commands in Part 1 are executed on the machine running Docker (e.g., your mini PC at 192.168.1.100)

### Step 1: Install Docker

```bash
# Install Docker (if not already installed)
curl -fsSL https://get.docker.com | sh

# Add your user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

### Step 2: Clone or Download This Repository

```bash
# Clone the repository
git clone <your-repo-url> /opt/local-db
cd /opt/local-db

# Or if you're using this as a template, download the files to your preferred location
```

### Step 3: Create Data Directories

```bash
# Create directory structure
sudo mkdir -p /mnt/data/{seaweedfs/{master,volume,filer},postgres}

# Set correct ownership
# PostgreSQL runs as user 999
sudo chown -R 999:999 /mnt/data/postgres

# SeaweedFS runs as your user
sudo chown -R $USER:$USER /mnt/data/seaweedfs

# Verify permissions
ls -la /mnt/data/
```

### Step 4: Configure Environment Variables

```bash
# Copy environment template
cp .env.example .env

# Generate secure credentials
echo "PostgreSQL Password: $(openssl rand -base64 32)"
echo "S3 Access Key: $(openssl rand -base64 24)"
echo "S3 Secret Key: $(openssl rand -base64 24)"

# Edit .env file with your generated credentials
vim .env
```

#### Why Credentials Appear in Two Places

You'll need to configure S3 credentials in **both** `.env` and `s3-config.json`:

- **`.env`**: Used by Docker Compose to pass environment variables to containers (specifically the S3 gateway container)
- **`s3-config.json`**: Used by SeaweedFS S3 gateway to validate incoming S3 API requests and manage access control

Both files must have **identical** S3 credentials. The S3 gateway reads `s3-config.json` to know which access keys are authorized, while the environment variables are used for initial configuration.

```bash
# Edit s3-config.json with the SAME credentials from .env
vim s3-config.json
```

Update the accessKey and secretKey to match your `.env` file:
```json
{
  "identities": [
    {
      "name": "default",
      "credentials": [
        {
          "accessKey": "your_s3_access_key_here",
          "secretKey": "your_s3_secret_key_here"
        }
      ],
      "actions": ["Admin", "Read", "Write"]
    }
  ]
}
```

**Tip**: Save your generated credentials temporarily so you can easily copy them to both files and later use them in Kubernetes secrets.

### Step 5: Deploy the Stack

```bash
# Start all services
docker compose up -d

# Watch logs (Ctrl+C to exit)
docker compose logs -f

# Check service health
docker compose ps
```

All services should show status as `healthy` after 30-60 seconds.

### Step 6: Verify Services

```bash
# Test PostgreSQL
docker exec -it postgres psql -U postgres -d homelab -c "SELECT version();"

# Test S3 Gateway
curl http://localhost:8333/

# Test metrics endpoint
curl http://localhost:9187/metrics

# Check all services are running
docker ps
```

Expected output: All containers should be `Up` and `healthy`.

### Step 7: Create S3 Buckets

Create buckets on your **SeaweedFS S3 instance** (not Amazon AWS). We use the AWS CLI tool because SeaweedFS is S3-compatible:

> **Note**: These commands create buckets on your local SeaweedFS server (localhost:8333), not on Amazon AWS.

```bash
# Set your credentials from .env file
export AWS_ACCESS_KEY_ID="your_s3_access_key"
export AWS_SECRET_ACCESS_KEY="your_s3_secret_key"

# Create buckets
docker run --rm \
  --network host \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  amazon/aws-cli \
  s3 mb s3://test-bucket --endpoint-url=http://localhost:8333

docker run --rm \
  --network host \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  amazon/aws-cli \
  s3 mb s3://authentik-media --endpoint-url=http://localhost:8333

docker run --rm \
  --network host \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  amazon/aws-cli \
  s3 mb s3://app-backups --endpoint-url=http://localhost:8333

# List buckets to verify
docker run --rm \
  --network host \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  amazon/aws-cli \
  s3 ls --endpoint-url=http://localhost:8333

# Test upload
echo "test" > /tmp/test.txt
docker run --rm \
  --network host \
  -v /tmp:/tmp \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  amazon/aws-cli \
  s3 cp /tmp/test.txt s3://test-bucket/ --endpoint-url=http://localhost:8333
```

You can view the bucket and its contents on port `8888`

```bash
http://192.168.1.100:8888
```

**Common bucket names:**
- `loki-logs` - For Loki log storage
- `authentik-media` - For Authentik user-uploaded media
- `longhorn-backups` - For Kubernetes volume backups
- `app-backups` - For application backups

---

## Part 2: Connect Kubernetes Cluster

> **📍 Location: Kubernetes Workstation**
> All commands in Part 2 are executed on your workstation with `kubectl` configured (NOT on the Docker host)

Now that your Docker services are running, let's connect your Kubernetes cluster to use them.

### Step 1: Networking

**If your cluster and Docker host are on the same trusted network**, no firewall configuration is needed. The services will be accessible directly.

**For additional security**, consider using Tailscale to create a secure mesh network between your cluster nodes and Docker host without exposing services to the broader network.


### Step 2: Update k8s-external-services.yaml

Update the IP address to match your Docker host:

```bash
# Replace 192.168.1.100 with your Docker host IP
sed -i 's/192.168.1.100/YOUR_DOCKER_HOST_IP/g' k8s-external-services.yaml
```

### Step 3: Deploy External Services to Kubernetes

```bash
# Apply the configuration
kubectl apply -f k8s-external-services.yaml

# Verify services were created
kubectl get svc -n external-services
kubectl get endpointslice -n external-services

# Check endpoints are correct
kubectl describe endpointslice -n external-services postgres-external
```

Expected output:
```
NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
postgres-external   ClusterIP   None         <none>        5432/TCP   10s
seaweedfs-s3-external ClusterIP None         <none>        8333/TCP   10s
```

### Step 4: Test Connectivity from Kubernetes

```bash
# Test PostgreSQL connection
kubectl run -it --rm pg-test \
  --image=postgres:18-alpine \
  --restart=Never \
  --namespace=external-services \
  -- psql -h postgres-external -U postgres -d homelab -c "SELECT version();"

# You'll be prompted for the password from your .env file
```

If this works, your cluster can successfully connect to the external database!

```bash
# Test S3 endpoint
kubectl run -it --rm curl-test \
  --image=curlimages/curl \
  --restart=Never \
  --namespace=external-services \
  -- curl -v http://seaweedfs-s3-external:8333/
```

### Step 5: Create Kubernetes Secrets

Now create secrets in Kubernetes to securely store your database and S3 credentials. Use the same credentials you configured in your `.env` file.

#### PostgreSQL Credentials Secret

```bash
# Create a secret for PostgreSQL connection details
# Replace the values with your actual credentials from .env
kubectl create secret generic external-db-credentials \
  --namespace=external-services \
  --from-literal=postgres-host=postgres-external.external-services.svc.cluster.local \
  --from-literal=postgres-port=5432 \
  --from-literal=postgres-database=homelab \
  --from-literal=postgres-user=postgres \
  --from-literal=postgres-password='YOUR_POSTGRES_PASSWORD_FROM_ENV'

# Verify the secret was created
kubectl get secret -n external-services external-db-credentials
```

#### S3 Credentials Secret

```bash
# Create a secret for S3 connection details
# Replace with your S3 credentials from .env / s3-config.json
kubectl create secret generic external-s3-credentials \
  --namespace=external-services \
  --from-literal=s3-endpoint=http://seaweedfs-s3-external.external-services.svc.cluster.local:8333 \
  --from-literal=s3-region=us-east-1 \
  --from-literal=s3-access-key='YOUR_S3_ACCESS_KEY_FROM_ENV' \
  --from-literal=s3-secret-key='YOUR_S3_SECRET_KEY_FROM_ENV'

# Verify the secret was created
kubectl get secret -n external-services external-s3-credentials
```

#### Creating Secrets in Application Namespaces

Your applications will likely run in their own namespaces. Create the secrets in each application namespace:

```bash
# For apps in 'default' namespace
kubectl create secret generic external-db-credentials \
  --namespace=default \
  --from-literal=postgres-host=postgres-external.external-services.svc.cluster.local \
  --from-literal=postgres-port=5432 \
  --from-literal=postgres-database=homelab \
  --from-literal=postgres-user=postgres \
  --from-literal=postgres-password='YOUR_POSTGRES_PASSWORD_FROM_ENV'

kubectl create secret generic external-s3-credentials \
  --namespace=default \
  --from-literal=s3-endpoint=http://seaweedfs-s3-external.external-services.svc.cluster.local:8333 \
  --from-literal=s3-region=us-east-1 \
  --from-literal=s3-access-key='YOUR_S3_ACCESS_KEY_FROM_ENV' \
  --from-literal=s3-secret-key='YOUR_S3_SECRET_KEY_FROM_ENV'

# Repeat for other namespaces where you have applications
# kubectl create secret ... --namespace=my-app-namespace ...
```

**Alternative: Use a Secret Management Tool**

For production, consider using:
- **External Secrets Operator**: Sync from external secret stores
- **Sealed Secrets**: Encrypt secrets in Git
- **Vault**: HashiCorp Vault integration

### Step 6: Test with Disposable Pods

Test the connections with temporary pods before updating your applications.

#### Test PostgreSQL Connection

```bash
# Test PostgreSQL with psql
kubectl run -it --rm postgres-test \
  --image=postgres:18-alpine \
  --restart=Never \
  --namespace=default \
  --env="PGPASSWORD=$(kubectl get secret external-db-credentials -n default -o jsonpath='{.data.postgres-password}' | base64 -d)" \
  -- psql -h postgres-external.external-services.svc.cluster.local -U postgres -d homelab -c "SELECT version();"

# Create a test table and insert data
kubectl run -it --rm postgres-test \
  --image=postgres:18-alpine \
  --restart=Never \
  --namespace=default \
  --env="PGPASSWORD=$(kubectl get secret external-db-credentials -n default -o jsonpath='{.data.postgres-password}' | base64 -d)" \
  -- psql -h postgres-external.external-services.svc.cluster.local -U postgres -d homelab -c "
    CREATE TABLE IF NOT EXISTS test_connection (
      id SERIAL PRIMARY KEY,
      message TEXT,
      created_at TIMESTAMP DEFAULT NOW()
    );
    INSERT INTO test_connection (message) VALUES ('Connection from Kubernetes successful!');
    SELECT * FROM test_connection;
  "
```

#### Test S3 Connection

```bash
# Test S3 with AWS CLI
kubectl run -it --rm s3-test \
  --image=amazon/aws-cli \
  --restart=Never \
  --namespace=default \
  --env="AWS_ACCESS_KEY_ID=$(kubectl get secret external-s3-credentials -n default -o jsonpath='{.data.s3-access-key}' | base64 -d)" \
  --env="AWS_SECRET_ACCESS_KEY=$(kubectl get secret external-s3-credentials -n default -o jsonpath='{.data.s3-secret-key}' | base64 -d)" \
  -- s3 --endpoint-url=http://seaweedfs-s3-external.external-services.svc.cluster.local:8333 ls

# Upload a file to existing test-bucket (created in Part 1 Step 7)
kubectl run -it --rm s3-test \
  --image=amazon/aws-cli \
  --restart=Never \
  --namespace=default \
  --env="AWS_ACCESS_KEY_ID=$(kubectl get secret external-s3-credentials -n default -o jsonpath='{.data.s3-access-key}' | base64 -d)" \
  --env="AWS_SECRET_ACCESS_KEY=$(kubectl get secret external-s3-credentials -n default -o jsonpath='{.data.s3-secret-key}' | base64 -d)" \
  -- sh -c "
    echo 'Test from Kubernetes' > /tmp/k8s-test.txt &&
    aws s3 --endpoint-url=http://seaweedfs-s3-external.external-services.svc.cluster.local:8333 cp /tmp/k8s-test.txt s3://test-bucket/ &&
    aws s3 --endpoint-url=http://seaweedfs-s3-external.external-services.svc.cluster.local:8333 ls s3://test-bucket/
  "
```

#### Update Your Applications

Once tests pass, update your application deployments to use the external services. Reference the secrets created in Step 5:

**For PostgreSQL applications**, use environment variables from `external-db-credentials`:
- `postgres-host`
- `postgres-port`
- `postgres-database`
- `postgres-user`
- `postgres-password`

**For S3 applications**, use environment variables from `external-s3-credentials`:
- `s3-endpoint`
- `s3-region`
- `s3-access-key`
- `s3-secret-key`

**Real-World Examples:**
This setup is currently running:
- **Authentik** (identity provider) → PostgreSQL for user data and sessions
- **Loki** (log aggregation) → SeaweedFS S3 for log storage

Your application configuration will vary based on the specific app, but the pattern is the same: reference the secrets and point to the external services.

### Step 7: Configure Prometheus to Scrape Metrics

If you have Prometheus in your cluster, add the postgres-exporter as a scrape target:

```yaml
# Add to your Prometheus configuration or ServiceMonitor
apiVersion: v1
kind: Service
metadata:
  name: postgres-exporter-external
  namespace: external-services
  labels:
    app: postgres-exporter
spec:
  ports:
  - port: 9187
    name: metrics
  clusterIP: None
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: postgres-exporter-external
  namespace: external-services
  labels:
    kubernetes.io/service-name: postgres-exporter-external
addressType: IPv4
endpoints:
- addresses:
  - "192.168.1.100"
ports:
- port: 9187
  name: metrics
```

This is already included in `k8s-external-services.yaml` if you applied it earlier.

---

## Migration Story: From CNPG to External PostgreSQL

This setup was born from migrating my Kubernetes cluster running CloudNativePG (CNPG) to external PostgreSQL. Here's how I did it:

### The Challenge

Previously, I ran PostgreSQL inside my 6-node Talos Kubernetes cluster using CloudNativePG. While this worked, it had drawbacks:
- High resource overhead for HA PostgreSQL
- Complicated backup strategies
- Cluster upgrades affected database availability
- Not how production cloud environments work

### The Solution

I migrated to a dedicated mini PC running Docker with PostgreSQL and SeaweedFS, then connected the Kubernetes cluster to it via external services.

### Migration Process

1. **Deployed external stack**: Set up Docker compose on dedicated hardware
2. **Exported data**: Used `pg_dumpall` to export from CNPG cluster
3. **Imported to external**: Restored dump to external PostgreSQL
4. **Tested connectivity**: Verified K8s pods could reach external DB
5. **Updated applications**: Changed connection strings one app at a time
6. **Monitored**: Watched logs for connection issues
7. **Decommissioned CNPG**: Removed cluster database after successful migration

### Key Takeaways

- **Zero downtime**: I updated apps one at a time, old DB stayed up during migration
- **Easy rollback**: I could revert connection strings if issues arose
- **Improved performance**: Dedicated resources, no cluster overhead
- **Simplified operations**: Single docker-compose.yaml instead of complex K8s operators

### Export Command Used

```bash
# From cluster
kubectl exec -n cnpg-system postgres-1 -- \
  pg_dumpall -U postgres > cluster-backup.sql

# Import to external
docker exec -i postgres psql -U postgres < cluster-backup.sql
```

For detailed migration steps, see the archived documentation in `archive/MIGRATION_GUIDE.md`.

---

## Troubleshooting

### Services Won't Start

```bash
# Check logs
docker compose logs postgres
docker compose logs seaweedfs-s3

# Check permissions
ls -la /mnt/data/postgres
ls -la /mnt/data/seaweedfs

# Restart specific service
docker compose restart postgres
```

### Kubernetes Can't Connect

```bash
# Test from cluster node (not pod)
ssh <cluster-node-ip>
curl http://192.168.1.100:5432

# Check service endpoints
kubectl get endpointslice -n external-services postgres-external -o yaml

# If using firewall, check rules (optional)
sudo ufw status
```

### PostgreSQL Authentication Failed

```bash
# Check password in .env matches what you're using
cat .env | grep POSTGRES_PASSWORD

# Test locally on Docker host
docker exec -it postgres psql -U postgres -d homelab
```

### S3 Credentials Invalid

```bash
# Verify .env and s3-config.json match
cat .env | grep S3_
cat s3-config.json

# They must be identical
# Restart S3 gateway after changing config
docker compose restart seaweedfs-s3
```

### Connection Refused from Kubernetes

1. Verify Docker host IP is correct in `k8s-external-services.yaml`
2. Ensure services are listening on `0.0.0.0` not `127.0.0.1`
3. Check network connectivity between cluster and Docker host
4. If using firewall, verify it allows cluster subnet (optional)
5. Test with debug pod:

```bash
kubectl run -it --rm debug \
  --image=nicolaka/netshoot \
  --namespace=external-services \
  --restart=Never \
  -- bash

# Inside pod:
telnet postgres-external 5432
curl http://seaweedfs-s3-external:8333/
```

---

## Maintenance & Operations

> **📍 Location: Docker Host (Mini PC)**
> Maintenance commands are executed on the Docker host

### Backups

PostgreSQL backup:
```bash
# Daily backup script
docker exec postgres pg_dumpall -U postgres | \
  gzip > /backups/postgres-$(date +%Y%m%d).sql.gz

# Or use pg_dump for specific database
docker exec postgres pg_dump -U postgres -Fc homelab > \
  /backups/homelab-$(date +%Y%m%d).dump
```

Data directory backup:
```bash
# Backup everything
tar -czf /backups/data-$(date +%Y%m%d).tar.gz /mnt/data/
```

### Updates

```bash
# Update container images
docker compose pull

# Recreate containers with new images
docker compose up -d

# Check status
docker compose ps
```

### Monitoring

```bash
# View logs
docker compose logs -f postgres
docker compose logs -f seaweedfs-s3

# Check resource usage
docker stats

# Health checks
docker compose ps
curl http://localhost:9187/metrics | grep postgres_up
```

### Resource Tuning

PostgreSQL is configured for 16GB RAM systems. Adjust in `docker-compose.yaml`:

```yaml
# For 8GB systems:
- "shared_buffers=512MB"
- "effective_cache_size=1536MB"

# For 32GB systems:
- "shared_buffers=2GB"
- "effective_cache_size=6GB"
```

---

## Optional Services

The docker-compose.yaml includes commented-out services that haven't been tested yet:

### Qdrant (Vector Database)

Uncomment the qdrant service section in docker-compose.yaml to enable. Don't forget to:
1. Create data directory: `sudo mkdir -p /mnt/data/vectordb`
2. Deploy K8s service: Uncomment qdrant section in `k8s-external-services.yaml`

### Ollama (AI Models)

Uncomment the ollama service section to enable. Don't forget to:
1. Create data directory: `sudo mkdir -p /mnt/data/ai-models/ollama`
2. Deploy K8s service: Uncomment ollama section in `k8s-external-services.yaml`
3. If you have GPU, uncomment the nvidia runtime lines

---

## Service Reference

### Connection Details

**PostgreSQL:**
- Host: `postgres-external.external-services.svc.cluster.local` (from K8s)
- Host: `192.168.1.100` (direct)
- Port: `5432`
- User: `postgres` (or as configured in .env)
- Database: `homelab` (or as configured in .env)

**SeaweedFS S3:**
- Endpoint: `http://seaweedfs-s3-external.external-services.svc.cluster.local:8333` (from K8s)
- Endpoint: `http://192.168.1.100:8333` (direct)
- Region: `us-east-1` (default)
- Access Key: From .env / s3-config.json
- Secret Key: From .env / s3-config.json

**Metrics:**
- PostgreSQL metrics: `http://192.168.1.100:9187/metrics`
- SeaweedFS metrics: Various ports (9324, 9327, 9328, 9329)

### Port Reference

| Service | Port | Purpose |
|---------|------|---------|
| postgres | 5432 | PostgreSQL database |
| seaweedfs-master | 9333 | Master server |
| seaweedfs-volume | 8080 | Volume server |
| seaweedfs-filer | 8888 | Filer server |
| seaweedfs-s3 | 8333 | S3 API gateway |
| postgres-exporter | 9187 | Prometheus metrics |
| seaweedfs-master-metrics | 9324 | Metrics |
| seaweedfs-volume-metrics | 9327 | Metrics |
| seaweedfs-filer-metrics | 9328 | Metrics |
| seaweedfs-s3-metrics | 9329 | Metrics |

---

## Files in This Repository

```
.
├── docker-compose.yaml           # Main service definitions
├── k8s-external-services.yaml    # Kubernetes services and endpoints
├── .env.example                  # Environment template
├── s3-config.json               # SeaweedFS S3 credentials
├── .gitignore                   # Git ignore rules
├── README.md                    # This file
└── archive/                     # Historical documentation
    ├── MIGRATION_GUIDE.md       # Original migration guide
    ├── CLUSTER_INTEGRATION.md   # Cluster integration details
    └── NAMESPACE_AND_GATEWAY_GUIDE.md
```

---

## What's Next?

After getting this running, consider:

1. **Automated backups**: Set up cron jobs for PostgreSQL backups
2. **Monitoring dashboards**: Import Grafana dashboards for PostgreSQL metrics
3. **High availability**: Add PostgreSQL replicas if needed
4. **Testing optional services**: Try Qdrant or Ollama if you need them
5. **S3 bucket lifecycle**: Configure retention policies in SeaweedFS
6. **SSL/TLS**: Add reverse proxy with certificates for production
7. **Migrate more apps**: Move other workloads to use external databases

---

## Support & Contributing

**Having issues?**
1. Check the Troubleshooting section above
2. Review logs: `docker compose logs -f`
3. Check archived docs in `archive/` for detailed migration examples

**Service Documentation:**
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [SeaweedFS Docs](https://github.com/seaweedfs/seaweedfs/wiki)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

---

## License

This is free and unencumbered software released into the public domain. See LICENSE or use freely.

---

**Built with the philosophy that homelab infrastructure should mirror production patterns while remaining simple to operate.**
