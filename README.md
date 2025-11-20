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

## Secret Management

### Creating the Deployment Secret

Before deploying any components, you must create a Kubernetes secret containing all required credentials and configuration values. A template is provided in `dummy-secret/oas-dummy-sec.yaml`.

**Steps:**

1. Copy the template:
```bash
cp dummy-secret/oas-dummy-sec.yaml oas-secret.yaml
```

2. Edit `oas-secret.yaml` and replace all values with actual credentials:
```bash
# Use a secure editor
vim oas-secret.yaml
# or
nano oas-secret.yaml
```

3. Apply the secret to your cluster:
```bash
kubectl create namespace oas-analytics
kubectl apply -f oas-secret.yaml -n oas-analytics
```

4. **For CI/CD pipelines**: Base64-encode the entire secret file and store it as a GitHub secret:
```bash
# Encode the secret
base64 -w 0 oas-secret.yaml > oas-secret.yaml.b64

# Copy the contents and add to GitHub Secrets as OAS_SECRET
cat oas-secret.yaml.b64
```

### Security Best Practices

⚠️ **IMPORTANT**:
- Never commit `oas-secret.yaml` to version control
- Store production secrets in a secure vault (HashiCorp Vault, AWS Secrets Manager, etc.)
- Rotate credentials regularly
- Use RBAC to restrict secret access
- The `oas-secret-template.yaml` is safe to commit (contains no real credentials)

### Secret Fields Reference

The deployment secret contains credentials for:
- **Database**: Root password, application user passwords
- **Airflow**: Fernet key (for encrypting connections), webserver secret key, admin password
- **Superset**: Secret key, admin password, database connection
- **Kafka**: Bootstrap servers, SASL credentials
- **NiFi**: Keystore/truststore passwords, admin credentials
- **Registry**: OCI registry credentials

See `dummy-secret/oas-dummy-sec.yaml` for the complete list of required fields.

## Chart Versions

All charts are deployed from the OCI registry: `oci://us-central1-docker.pkg.dev/product-obp/oas/charts`

| Component | Chart Version | Description |
|-----------|---------------|-------------|
| pxc-operator | 1.18.0 | Percona XtraDB Cluster Operator |
| pxc-db | 1.18.0 | Percona XtraDB Cluster Database |
| flink-kubernetes-operator | 1.12.0 | Apache Flink Kubernetes Operator |
| airflow | 1.18.0 | Apache Airflow |
| nifi | 1.2.1 | Apache NiFi |
| kafka-connect | 0.4.0 | Kafka Connect |
| superset | 0.15.0 | Apache Superset |

## Deployment Sequence

⚠️ **CRITICAL**: Components must be installed in this exact order to satisfy dependencies and avoid initialization failures.

### Prerequisites Check

Before starting deployment:
```bash
# Verify Helm version
helm version

# Verify kubectl access
kubectl cluster-info

# Create namespace
kubectl create namespace oas-analytics
```

---

### Step 1: Install cert-manager

cert-manager is required for TLS certificate management across all components.
```bash
# Install cert-manager
helm install cert-manager oci://us-central1-docker.pkg.dev/product-obp/oas/charts/cert-manager \
  --version v1.18.2 \
  -f values-cert-manager.yaml \
  -n cert-manager --create-namespace

# Wait for cert-manager to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager \
  --timeout=300s

# Verify installation
kubectl get pods -n cert-manager
```

---

### Step 2: Apply Secrets
```bash
# Apply the deployment secret (created in Secret Management section)
kubectl apply -f oas-secret.yaml -n oas-analytics

# Verify secret is created
kubectl get secret oas-secret -n oas-analytics
```

---

### Step 3: Authenticate to OCI Registry
```bash
# Login to the OCI registry
helm registry login us-central1-docker.pkg.dev

# When prompted, enter:
# Username: _json_key
# Password: 
```

**Alternative**: Use a credentials file:
```bash
cat ~/my-gcp-key.json | helm registry login \
  us-central1-docker.pkg.dev \
  --username _json_key \
  --password-stdin
```

---

### Step 4: Install PXC Operator

The PXC Operator must be running before any database clusters can be deployed.
```bash
helm install pxc-operator \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/pxc-operator \
  --version 1.18.0 \
  -f values-files/oas-pxc-operator.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 10m

# Verify operator is running
kubectl get pods -l app.kubernetes.io/name=pxc-operator -n oas-analytics
kubectl get crd perconaxtradbclusters.pxc.percona.com
```

---

### Step 5: Install PXC Database

Deploy the actual database cluster.
```bash
helm install pxc-db \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/pxc-db \
  --version 1.18.0 \
  -f values-files/oas-pxcdb-values.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 15m

# Wait for cluster to be ready (this may take 5-10 minutes)
kubectl get pxc -n oas-analytics
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=pxc-db \
  -n oas-analytics \
  --timeout=900s

# Verify database is accessible
kubectl get svc -l app.kubernetes.io/instance=pxc-db -n oas-analytics
```

---

### Step 6: Install Flink Kubernetes Operator

Required for running Flink jobs.
```bash
helm install flink-operator \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/flink-kubernetes-operator \
  --version 1.12.0 \
  -f values-files/oas-flink-operator.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 10m

# Verify operator is running
kubectl get pods -l app.kubernetes.io/name=flink-kubernetes-operator -n oas-analytics
kubectl get crd flinkdeployments.flink.apache.org
```

---

### Step 7: Install Apache Airflow

Airflow will automatically create its database and users during the migration job.
```bash
helm install airflow \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/airflow \
  --version 1.18.0 \
  -f values-files/oas-airflow-values.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 15m

# Monitor migration job
kubectl get jobs -l app.kubernetes.io/component=migration -n oas-analytics
kubectl logs -f job/airflow-migration -n oas-analytics

# Verify Airflow pods are running
kubectl get pods -l app.kubernetes.io/name=airflow -n oas-analytics
```

**Note**: The Airflow migration job creates:
- Airflow metadata database
- ETL user in PXC (using credentials from `oas-secret`)
- Required database schema and initial admin user

---

### Step 8: Install Apache NiFi
```bash
helm install nifi \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/nifi \
  --version 1.2.1 \
  -f values-files/oas-nifi-values.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 15m

# Verify NiFi StatefulSet
kubectl get statefulset nifi -n oas-analytics
kubectl get pods -l app.kubernetes.io/name=nifi -n oas-analytics
```

---

### Step 9: Install Kafka Connect
```bash
helm install kafka-connect \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/kafka-connect \
  --version 0.4.0 \
  -f values-files/oas-kafka-connect-values.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 10m

# Verify Kafka Connect
kubectl get pods -l app.kubernetes.io/name=kafka-connect -n oas-analytics

# Check REST API is accessible
kubectl port-forward svc/kafka-connect 8083:8083 -n oas-analytics &
curl http://localhost:8083/
```

---

### Step 10: Install Apache Superset

Superset will automatically create its database and users during the migration job.
```bash
helm install superset \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/superset \
  --version 0.15.0 \
  -f values-files/oas-superset-values.yaml \
  --namespace oas-analytics \
  --wait \
  --timeout 15m

# Monitor migration job
kubectl get jobs -l app.kubernetes.io/component=migration -n oas-analytics
kubectl logs -f job/superset-migration -n oas-analytics

# Verify Superset pods
kubectl get pods -l app.kubernetes.io/name=superset -n oas-analytics
```

---

### Deployment Verification

After all components are deployed:
```bash
# Check all pods are running
kubectl get pods -n oas-analytics

# Check all services
kubectl get svc -n oas-analytics

# Check persistent volumes
kubectl get pvc -n oas-analytics

# View all deployed Helm releases
helm list -n oas-analytics
```

Expected output should show all components with STATUS: `deployed`.

---

## Values Files Configuration

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

## Accessing Components

Access methods vary based on your Kubernetes service configuration (NodePort or LoadBalancer).

### Apache Superset

**Port Forward (Development/Testing):**
```bash
kubectl port-forward svc/superset 8088:8088 -n oas-analytics
# Access at http://localhost:8088
```

**NodePort:**
```bash
kubectl get svc superset -n oas-analytics -o jsonpath='{.spec.ports[0].nodePort}'
# Access at http://:
```

**Default Credentials**: Set in `oas-superset-values.yaml` under `superset.init.adminUser`

---

### Apache Airflow

**Port Forward:**
```bash
kubectl port-forward svc/airflow-webserver 8080:8080 -n oas-analytics
# Access at http://localhost:8080
```

**NodePort:**
```bash
kubectl get svc airflow-webserver -n oas-analytics -o jsonpath='{.spec.ports[0].nodePort}'
# Access at http://:
```

**Default Credentials**: Check `oas-airflow-values.yaml` under `airflow.webserver.defaultUser`

**Accessing DAGs:**
- DAGs location: Configured in `oas-airflow-values.yaml`
- Sync method: Git-sync, volume mount, or S3
- DAG examples: Check Airflow UI → DAGs page

---

### Apache NiFi

**Port Forward:**
```bash
kubectl port-forward svc/nifi 8443:8443 -n oas-analytics
# Access at https://localhost:8443/nifi
```

**NodePort:**
```bash
kubectl get svc nifi -n oas-analytics -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'
# Access at https://:/nifi
```

**Note**: 
- NiFi uses HTTPS by default with self-signed certificates
- Accept certificate warning in browser or configure proper TLS
- Credentials: Set in `oas-secret.yaml` (nifi-admin-password)

---

### Kafka Connect

**REST API Port Forward:**
```bash
kubectl port-forward svc/kafka-connect 8083:8083 -n oas-analytics
```

**Check Status:**
```bash
curl http://localhost:8083/
```
---

### Apache Flink Dashboard

**Find Flink JobManager Service:**
```bash
kubectl get svc -l component=jobmanager -n oas-analytics
```

**Port Forward:**
```bash
kubectl port-forward svc/ 8081:8081 -n oas-analytics
# Access at http://localhost:8081
```

**Submit Flink Jobs:**
- Via FlinkDeployment CRDs
- Via REST API
- Via Flink CLI (from within cluster)

---

### PXC Database

**Direct Connection (from within cluster):**
```bash
# Create a MySQL client pod
kubectl run mysql-client --image=mysql:8.0 -it --rm --restart=Never -n oas-analytics -- \
  mysql -h pxc-db-haproxy -u root -p

# Enter root password from oas-secret
```

**Port Forward (for external tools):**
```bash
kubectl port-forward svc/pxc-db-haproxy 3306:3306 -n oas-analytics
# Connect using MySQL client at localhost:3306
```

---

## Uninstalling Components

To remove components, uninstall in reverse order:
```bash
# Remove applications first
helm uninstall superset -n oas-analytics
helm uninstall kafka-connect -n oas-analytics
helm uninstall nifi -n oas-analytics
helm uninstall airflow -n oas-analytics

# Remove operators
helm uninstall flink-operator -n oas-analytics

# Remove database (WARNING: This deletes all data!)
helm uninstall pxc-db -n oas-analytics
helm uninstall pxc-operator -n oas-analytics

# Remove secrets
kubectl delete secret oas-secret -n oas-analytics

# Optionally remove namespace
kubectl delete namespace oas-analytics
```

**⚠️ WARNING**: Uninstalling the database will delete all persistent data unless you have backups configured.

---

## Upgrading Components

### Upgrade Individual Components
```bash
# Check for available versions
helm search repo charts/airflow --versions

# Upgrade to new version
helm upgrade airflow \
  oci://us-central1-docker.pkg.dev/product-obp/oas/charts/airflow \
  --version 1.19.0 \
  -f values-files/oas-airflow-values.yaml \
  --namespace oas-analytics

# Rollback if needed
helm rollback airflow -n oas-analytics
```

### Update Chart Versions in Documentation

When upgrading, update the version table in this README:

1. Edit the **Chart Versions** section above
2. Update version numbers
3. Test deployment with new versions
4. Commit changes to repository

---

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
