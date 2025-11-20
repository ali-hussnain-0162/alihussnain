# Optiva Analytics Service

A comprehensive Helm chart for deploying a production-grade analytics platform on Kubernetes. This chart orchestrates multiple data processing, workflow orchestration, and visualization components into a unified analytics stack.

## Overview

**Optiva Analytics Service** is a Helm chart that deploys and manages a complete analytics infrastructure by integrating several industry-standard open-source tools. It provides a scalable, cloud-native platform for data ingestion, processing, workflow orchestration, and visualization.

### Key Components

The chart provisions the following workloads:

- **Percona XtraDB Cluster (PXC)** - High-availability MySQL database cluster
- **Apache Flink** - Stream processing framework for real-time analytics
- **Apache Airflow** - Workflow orchestration and scheduling
- **Apache NiFi** - Data ingestion and flow management
- **Kafka Connect** - Streaming data integration
- **Apache Superset** - Modern data exploration and visualization platform

## Component Details

### pxc-operator (v1.18.0)
The Percona XtraDB Cluster Operator manages the lifecycle of PXC database clusters. It handles automated provisioning, scaling, backup/restore operations, and ensures high availability of the MySQL database layer that stores metadata for Airflow and other services.

### pxc-db (v1.18.0)
Deploys a managed Percona XtraDB Cluster instance through the operator. This provides a highly available, ACID-compliant relational database with automatic failover, making it ideal for storing critical application metadata and configuration.

### flink-kubernetes-operator (v1.12.0)
Manages Apache Flink applications on Kubernetes using custom resources. It automates deployment, scaling, savepointing, and upgrades of Flink jobs, enabling efficient stream processing workloads with native Kubernetes integration.

### airflow (v1.18.0)
Apache Airflow provides workflow orchestration through Directed Acyclic Graphs (DAGs). It schedules and monitors data pipelines, ETL jobs, and complex workflows across the analytics platform with extensive plugin support and a web-based UI.

### nifi (v1.2.1)
Apache NiFi offers a visual interface for designing data flows. It excels at data ingestion from diverse sources, transformation, routing, and system mediation logic. Its provenance tracking and data lineage features are essential for compliance and debugging.

### kafka-connect (v0.4.0)
Kafka Connect provides scalable, reliable streaming data integration between Apache Kafka and external systems. It supports both source connectors (ingesting data into Kafka) and sink connectors (exporting data from Kafka) with fault tolerance.

### superset (v0.15.0)
Apache Superset is a modern data exploration and visualization platform. It provides rich interactive dashboards, a SQL IDE, and supports a wide variety of data sources, making analytics accessible to both analysts and business users.

## Prerequisites

Before deploying Optiva Analytics Service, ensure the following requirements are met:

### Required Infrastructure
- **Kubernetes cluster** (v1.24+)
- **Helm 3** (v3.8+)
- **kubectl** configured with cluster access
- Sufficient cluster resources (CPU, memory, storage)

### Critical Dependencies
- **cert-manager** (v1.11+) - Must be installed before deploying this chart
- Access to the OCI registry: `us-central1-docker.pkg.dev/product-obp/oas/charts`
- Valid credentials for OCI registry authentication

### Custom Resource Definitions (CRDs)
The following operators require their CRDs to be installed:
- Percona XtraDB Cluster Operator CRDs
- Flink Kubernetes Operator CRDs
- cert-manager CRDs (via cert-manager installation)

## Deployment Sequence

⚠️ **Critical**: Components must be installed in this specific order to avoid dependency conflicts and initialization failures.

### Step 1: Install cert-manager
```bash
helm install cert-manager oci://us-central1-docker.pkg.dev/product-obp/oas/charts/cert-manager \
  --version v1.18.2 \
  -f cert-manager-values.yaml \
  -n cert-manager --create-namespace
```

Wait for cert-manager pods to be ready:
```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

### Step 2: Install pxc-operator
The PXC operator must be deployed before the database cluster:
```bash
helm registry login us-central1-docker.pkg.dev
helm install pxc-operator oci://us-central1-docker.pkg.dev/product-obp/oas/charts/pxc-operator \
  --version 1.18.0 \
  -f values-files/oas-pxc-operator.yaml
```

### Step 3: Install flink-kubernetes-operator
```bash
helm install flink-operator oci://us-central1-docker.pkg.dev/product-obp/oas/charts/flink-kubernetes-operator \
  --version 1.12.0 \
  -f values-files/oas-flink-operator.yaml
```

### Step 4: Install Optiva Analytics Service
Once operators are running, deploy the main chart:
```bash
cd optiva-analytics-service/
helm dependency update
helm install optiva . \
  -f values-files/oas-pxcdb-values.yaml \
  -f values-files/oas-airflow-values.yaml \
  -f values-files/oas-nifi-values.yaml \
  -f values-files/oas-kafka-connect-values.yaml \
  -f values-files/oas-superset-values.yaml
```

**Why this sequence matters:**
- cert-manager must exist before any certificate resources are created
- Operators must be running before their custom resources (PXC clusters, Flink jobs) can be deployed
- Database must be available before Airflow and other services attempt to connect

## Deployment Instructions

### 1. Authenticate to OCI Registry

```bash
# Login to the OCI registry
helm registry login us-central1-docker.pkg.dev

# Enter credentials when prompted
# Username: _json_key
# Password: <contents of service account JSON key>
```

### 2. Update Chart Dependencies

Navigate to the chart directory and pull dependencies:

```bash
cd optiva-analytics-service/
helm dependency update
```

This downloads all dependent charts specified in `Chart.yaml` to the `charts/` subdirectory.

### 3. Install the Chart

Basic installation:
```bash
helm install oas optiva-analytics-service/
```

Installation with custom values:
```bash
helm install optiva optiva-analytics-service/ \
  -f values-files/oas-airflow-values.yaml \
  -f values-files/oas-superset-values.yaml \
  -f values-files/oas-nifi-values.yaml \
  --namespace oas-analytics \
  --create-namespace
```

### 4. Enable/Disable Components

Individual components can be toggled in `values.yaml`:

```yaml
pxc-operator:
  enabled: true

pxc-db:
  enabled: true

flink-kubernetes-operator:
  enabled: true

airflow:
  enabled: true

nifi:
  enabled: false  # Disable NiFi if not needed

kafka-connect:
  enabled: true

superset:
  enabled: true
```

### 5. Verify Deployment

```bash
# Check all pods are running
kubectl get pods -n oas-analytics

# Check Helm release status
helm status optiva -n oas-analytics

# View all deployed resources
kubectl get all -n oas-analytics
```

## Using values-files

The `values-files/` directory contains environment-specific and component-specific configuration overrides:

| File | Purpose |
|------|---------|
| `oas-airflow-values.yaml` | Airflow executor config, DAG settings, connections, resource limits |
| `oas-flink-operator.yaml` | Flink operator configuration, job manager settings |
| `oas-pxc-operator.yaml` | PXC operator settings, backup configuration |
| `oas-pxcdb-values.yaml` | Database cluster sizing, storage class, replication settings |
| `oas-nifi-values.yaml` | NiFi cluster size, persistence, security configuration |
| `oas-kafka-connect-values.yaml` | Connector configurations, Kafka bootstrap servers |
| `oas-superset-values.yaml` | Database connections, authentication, secret keys |
| `values-cert-manager.yaml` | Certificate issuer configuration |

**Best Practice**: Create environment-specific value files (e.g., `values-dev.yaml`, `values-prod.yaml`) and store sensitive data in external secret management systems (HashiCorp Vault, AWS Secrets Manager, etc.).

## Access Information

Access methods depend on your Kubernetes service configuration (NodePort, LoadBalancer, or Ingress).

### Apache Superset

**NodePort:**
```bash
kubectl get svc superset -n analytics
# Access at http://<node-ip>:<node-port>
```

**Port Forward (for testing):**
```bash
kubectl port-forward svc/oas-superset 8088:8088 -n oas-analytics
# Access at http://localhost:8088
```

**Default Credentials**: Check `oas-superset-values.yaml` or chart documentation

---

### Apache Airflow

**NodePort:**
```bash
kubectl get svc airflow-webserver -n oas-analytics
# Access at http://<node-ip>:<node-port>
```

**Port Forward:**
```bash
kubectl port-forward svc/airflow-api-server 8080:8080 -n oas-analytics
# Access at http://localhost:8080
```

**Default Credentials**: Typically `admin` / check `oas-airflow-values.yaml`

---

### Apache NiFi

**NodePort:**
```bash
kubectl get svc nifi -n oas-analytics
# Access at https://<node-ip>:<node-port>/nifi
```

**Port Forward:**
```bash
kubectl port-forward svc/osa-nifi 8443:8443 -n oas-analytics
# Access at https://localhost:8443/nifi
```

**Note**: NiFi uses HTTPS by default. Accept the self-signed certificate or configure proper TLS.

---

### Kafka Connect

**REST API Access:**
```bash
kubectl get svc kafka-connect -n oas-analytics
# REST API at http://<service-ip>:8083
```

**Check Connectors:**
```bash
kubectl port-forward svc/kafka-connect 8083:8083 -n oas-analytics
curl http://localhost:8083/connectors
```

---

### Apache Flink Dashboard

**Port Forward:**
```bash
# Find the Flink JobManager service
kubectl get svc -l component=jobmanager -n oas-analytics
kubectl port-forward svc/<flink-jobmanager-svc> 8081:8081 -n oas-analytics
# Access at http://localhost:8081
```

---

## Directory Structure

```
Optiva-Analytics-Service/
├── deployments/          # Kubernetes manifests for CI/CD pipelines
│                         # May include GitOps configs, ArgoCD applications
├── optiva-analytics-service/  # Main Helm chart directory
│   ├── Chart.yaml        # Chart metadata and dependencies
│   ├── values.yaml       # Default configuration values
│   ├── templates/        # Kubernetes resource templates
│   └── charts/           # Downloaded dependency charts (generated)
├── scripts/              # Helper scripts for deployment, testing, backups
│                         # May include init scripts, migration tools
├── values-files/         # Environment and component-specific overrides
│   └── *.yaml            # Separate values files per component/environment
└── README.md             # This file
```

### Key Directories

- **deployments/**: Contains deployment automation scripts, CI/CD pipeline definitions, or GitOps manifests for different environments
- **scripts/**: Utility scripts for common operations (database migrations, backup/restore, health checks, troubleshooting)
- **optiva-analytics-service/**: The primary Helm chart with all templates and default configurations
- **values-files/**: Organized configuration overrides to avoid modifying the base `values.yaml`

### Linting the Chart

Validate chart syntax and best practices:

```bash
# Lint the chart
helm lint optiva-analytics-service/

# Validate with specific values
helm lint optiva-analytics-service/ -f values-files/oas-airflow-values.yaml

# Dry-run to check rendered manifests
helm install --dry-run --debug optiva optiva-analytics-service/
```

### Testing Chart Changes

```bash
# Template the chart to see rendered YAML
helm template optiva optiva-analytics-service/ > rendered.yaml

# Install in a test namespace
helm install test-optiva optiva-analytics-service/ \
  --namespace test-analytics \
  --create-namespace \
  --dry-run

# Run Helm tests (if defined)
helm test optiva -n analytics
```

## Troubleshooting

### cert-manager Issues

**Problem**: Pods fail with certificate errors or ACME challenges timeout

**Solutions**:
```bash
# Verify cert-manager is running
kubectl get pods -n cert-manager

# Check certificate status
kubectl get certificates -n analytics
kubectl describe certificate <cert-name> -n oas-analytics

# Check issuer configuration
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

**Common Causes**:
- cert-manager not installed or not ready
- ClusterIssuer or Issuer misconfigured
- DNS not propagating for HTTP-01 challenges
- Network policies blocking ACME validation

---

### Operator CRDs Not Installed

**Problem**: Error applying custom resources (e.g., `PerconaXtraDBCluster`, `FlinkDeployment`)

**Solutions**:
```bash
# For PXC Operator
kubectl get crd perconaxtradbclusters.pxc.percona.com

# For Flink Operator
kubectl get crd flinkdeployments.flink.apache.org

# Install CRDs manually if missing
helm install pxc-operator oci://us-central1-docker.pkg.dev/product-obp/oas/charts/pxc-operator \
  --version 1.18.0 \
  --set installCRDs=true
```

---

### OCI Registry Authentication Failures

**Problem**: `Error: failed to download chart` or `403 Forbidden`

**Solutions**:
```bash
# Re-login to registry
helm registry logout us-central1-docker.pkg.dev
helm registry login us-central1-docker.pkg.dev

# Verify credentials
# Ensure service account has Artifact Registry Reader role

# Pull charts manually to test
helm pull oci://us-central1-docker.pkg.dev/product-obp/oas/charts/airflow --version 1.18.0
```

**Common Causes**:
- Expired credentials
- Incorrect service account permissions
- Network/firewall blocking registry access

---

### Pods Stuck in Init or Pending

**Problem**: Pods remain in `Init:0/1` or `Pending` state indefinitely

**Solutions**:
```bash
# Check pod events
kubectl describe pod <pod-name> -n oas-analytics

# Check init container logs
kubectl logs <pod-name> -c <init-container-name> -n oas-analytics

# Common issues to check:
# 1. PVC not bound (check storage class availability)
kubectl get pvc -n oas-analytics

# 2. Resource constraints (insufficient CPU/memory)
kubectl describe nodes

# 3. Database not ready (for Airflow, Superset)
kubectl logs <airflow-scheduler-pod> -n oas-analytics

# 4. Image pull errors
kubectl get events -n analytics --sort-by='.lastTimestamp'
```

---

### Dependency Download Failures

**Problem**: `helm dependency update` fails or times out

**Solutions**:
```bash
# Clear Helm cache
rm -rf ~/.cache/helm/repository

# Update dependencies with verbose output
helm dependency update optiva-analytics-service/ --debug

# Manually download problematic charts
helm pull oci://us-central1-docker.pkg.dev/product-obp/oas/charts/pxc-operator \
  --version 1.18.0 \
  --untar \
  --untardir optiva-analytics-service/charts/

# Verify Chart.lock exists
cat optiva-analytics-service/Chart.lock
```

---

### Database Connection Failures

**Problem**: Airflow or Superset cannot connect to PXC database

**Solutions**:
```bash
# Check PXC cluster status
kubectl get pxc -n oas-analytics
kubectl describe pxc <cluster-name> -n oas-analytics

# Verify database credentials secret
kubectl get secret <db-secret-name> -n oas-analytics -o yaml

# Test database connectivity from a debug pod
kubectl run -it --rm debug --image=mysql:8.0 --restart=Never -n oas-analytics -- \
  mysql -h <pxc-service> -u<username> -p<password>

# Check Airflow database initialization
kubectl logs <airflow-scheduler-pod> -n oas-analytics | grep -i database
```

---

### High Resource Usage

**Problem**: Nodes running out of CPU or memory

**Solutions**:
```bash
# Check resource usage
kubectl top nodes
kubectl top pods -n oas-analytics

# Review resource requests/limits
kubectl describe pod <pod-name> -n oas-analytics | grep -A 5 Resources

# Adjust in values files:
# values-files/oas-airflow-values.yaml
airflow:
  workers:
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2000m"
        memory: "4Gi"
```

---

### Checking Logs

```bash
# Airflow scheduler logs
kubectl logs -f deployment/airflow-scheduler -n oas-analytics

# Superset logs
kubectl logs -f deployment/superset -n oas-analytics

# Flink JobManager
kubectl logs -f <flink-jobmanager-pod> -n oas-analytics

# NiFi logs
kubectl logs -f statefulset/nifi -n oas-analytics

# PXC cluster logs
kubectl logs -f <pxc-pod> -n oas-analytics
```

---

## Additional Resources

- [Helm Documentation](https://helm.sh/docs/)
- [Percona XtraDB Cluster Operator](https://docs.percona.com/percona-operator-for-mysql/pxc/index.html)
- [Apache Flink Kubernetes Operator](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/)
- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Apache NiFi Documentation](https://nifi.apache.org/docs.html)
- [Kafka Connect Documentation](https://docs.confluent.io/platform/current/connect/index.html)
- [Apache Superset Documentation](https://superset.apache.org/docs/intro)
- [cert-manager Documentation](https://cert-manager.io/docs/)

---
