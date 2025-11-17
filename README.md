# Helm Charts Repository

This repository contains a collection of Helm charts and custom value configurations used to deploy core components of the OAS platform. It provides a consistent, modular structure for deploying applications such as Airflow, NiFi, Kafka Connect, Superset, and supporting services like PXC-DB and platform operators.

All sensitive information is stored in a dedicated `secrets/` directory to ensure secure and maintainable deployments.

---

## 📌 Project Overview

This repository aims to:

- Provide **version-controlled Helm charts** for all OAS components  
- Maintain **centralized values.yaml overrides** for each chart  
- Store and manage sensitive data in a **dedicated secrets folder**  
- Offer **deployment-ready templates** and a standardized workflow  
- Simplify onboarding for new developers and operators  

### 📦 Charts Included

| Component        | Purpose |
|------------------|---------|
| **airflow/** | Deployment of Apache Airflow, scheduler, workers, webserver, etc. |
| **nifi/** | Apache NiFi cluster configuration, processors, and connections. |
| **kafka-connect/** | Deployment of Kafka Connect cluster with custom connectors. |
| **superset/** | Apache Superset analytics and UI. |
| **flink-chart/** | Custom Helm chart for Apache Flink. |
| **pxc-db/** | Percona XtraDB Cluster (database for the platform). |
| **oas-operators/** | Operator-level controllers for the OAS platform. |

## 🎛 Custom Values Files

Each chart has a corresponding custom values file:

- `oas-airflow-values.yaml`
- `oas-kafka-connect-values.yaml`
- `oas-nifi-values.yaml`
- `oas-superset-values.yaml`
- `oas-pxcdb-values.yaml`
- `oas-operators-values.yaml`

These files provide opinionated defaults for the OAS environment. They help ensure:

- Consistency across environments  
- Clean separation of config vs. chart templates  
- Minimal edits inside chart sources  

---

## 🔐 Secrets Management Pattern

All secrets are stored outside chart directories inside:

secrets/
├── oas-secret.yaml
└── secret-examples/
├── airflow-secret-example.yaml
├── airflow-git-secret.yaml
├── airflow-keycloak-cm.yaml
├── flink-secret-example.yaml
├── kc-secret-example.yaml
├── nifi-secret-example.yaml
└── superset-secret-example.yaml

yaml
Copy code

This ensures:

- Secrets do **not** get accidentally packaged with charts  
- Only approved personnel edit real secrets  
- New users can copy templates easily  
- Git remains clean of sensitive data  

---

## 📁 Folder-by-Folder Explanation

### `airflow/`
Contains the Helm chart for deploying Apache Airflow (webserver, scheduler, workers, ingress, configmaps, etc).

### `kafka-connect/`
Helm chart for Kafka Connect, including connector configs, deployment/statefulset templates.

### `nifi/`
Apache NiFi cluster chart including node identity, Zookeeper settings, persistent storage templates.

### `superset/`
Helm chart for Apache Superset (webserver, async workers, DB migration jobs, ingress).

### `flink-chart/`
Custom Helm chart wrapping Apache Flink job/session cluster logic.

### `pxc-db/`
Helm chart for Percona XtraDB Cluster (MySQL-compatible DB).

### `oas-operators/`
Chart that deploys platform operators or CRDs.

---

## 🔐 How Secrets Work

### Why separate secrets?

- Prevent committing real credentials  
- Avoid packaging secrets when running `helm package`  
- Allow separate management by different teams  
- Enable easy environment onboarding  

## 🔐 How to Use Secret Templates

Users should follow these steps to create and apply secrets.

### 1. Navigate to the example secrets directory:

cd secrets/secret-examples/

bash
Copy code

### 2. Copy a template:

```bash
cp secrets/secret-examples/airflow-secret-example.yaml secrets/airflow-secret.yaml
3. Edit the copied file and insert real sensitive values.
4. (Optional) Rename for environment:
bash
Copy code
mv secrets/airflow-secret.yaml secrets/prod-airflow-secret.yaml
🔐 Applying Secrets Before Installing Helm Charts
Secrets must be created before installing any chart.

Apply individual secrets:

bash
Copy code
kubectl apply -f secrets/oas-secret.yaml
kubectl apply -f secrets/airflow-secret.yaml
Or apply the entire folder:

bash
Copy code
kubectl apply -f secrets/
🚀 Installation Instructions
You must apply secrets first, then install charts using the provided values files.

1. Create or select a namespace
bash
Copy code
kubectl create namespace oas || true
kubectl config set-context --current --namespace=oas
2. Apply Secrets
bash
Copy code
kubectl apply -f secrets/
3. Install Charts
Airflow
bash
Copy code
helm upgrade --install airflow airflow/ \
  -f oas-airflow-values.yaml
NiFi
bash
Copy code
helm upgrade --install nifi nifi/ \
  -f oas-nifi-values.yaml
Kafka Connect
bash
Copy code
helm upgrade --install kafka-connect kafka-connect/ \
  -f oas-kafka-connect-values.yaml
Superset
bash
Copy code
helm upgrade --install superset superset/ \
  -f oas-superset-values.yaml
PXC-DB
bash
Copy code
helm upgrade --install pxc-db pxc-db/ \
  -f oas-pxcdb-values.yaml
Operators
bash
Copy code
helm upgrade --install oas-operators oas-operators/ \
  -f oas-operators-values.yaml
🛠️ Customization Notes
Variables users must update:
Database usernames/passwords inside secrets

OAuth / Keycloak secrets

Internal API/service tokens

Storage class names

Ingress domain names

🔗 Compatibility & Dependencies
Component	Depends On
Airflow	PXC-DB, Keycloak (optional)
NiFi	Kafka, Zookeeper
Kafka Connect	Kafka
Superset	Database, optional authentication
Operators	None (may provide CRDs)
