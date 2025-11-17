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

### 🎛 Custom values.yaml Files

Each chart has a corresponding custom values file:

oas-airflow-values.yaml
oas-kafka-connect-values.yaml
oas-nifi-values.yaml
oas-superset-values.yaml
oas-pxcdb-values.yaml
oas-operators-values.yaml

markdown
Copy code

These files provide opinionated defaults for the OAS environment. They help ensure:

- Consistency across environments  
- Clean separation of config vs. chart templates  
- Minimal edits inside chart sources  

### 🔐 Secrets Management Pattern

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
├── superset-secret-example.yaml

yaml
Copy code

This ensures:

- Secrets do **not** get accidentally packaged with charts  
- Only approved personnel edit real secrets  
- New users can copy templates easily  
- Git remains clean of sensitive data  

---

## 📁 Folder-by-Folder Explanation

### **airflow/**  
Contains the Helm chart for deploying Apache Airflow. Includes templates for webserver, scheduler, workers, ingress, configmaps, etc.

### **kafka-connect/**  
Helm chart for Kafka Connect, including connector configurations and deployment/statefulset definitions.

### **nifi/**  
Apache NiFi chart with cluster configuration, node identity, Zookeeper settings, and persistent storage templates.

### **superset/**  
Helm chart for Apache Superset, including webserver, async workers, database migration jobs, and ingress.

### **flink-chart/**  
Custom chart wrapping Apache Flink deployment logic (job cluster, session cluster, configmaps, etc.).

### **pxc-db/**  
Helm chart for deploying Percona XtraDB Cluster (MySQL-compatible database).

### **oas-operators/**  
Chart that deploys platform operators or CRDs required by the OAS ecosystem.

---

## 🔐 How Secrets Work

### Why are secrets separated?

- Avoid committing real credentials inside charts  
- Prevent accidental packaging of secrets when users run `helm package`  
- Allow different teams to manage secrets independently of chart logic  
- Maintain a “plug-and-play” system for onboarding new environments  

### How to use secret templates

Users should:

1. Navigate to `secrets/secret-examples/`
2. Copy the template they need:

   ```bash
   cp secrets/secret-examples/airflow-secret-example.yaml secrets/airflow-secret.yaml
Edit the copied file with real sensitive values.

(Optional) Rename based on environment, e.g.:

bash
Copy code
secrets/prod-airflow-secret.yaml
Applying secrets before Helm installs
Secrets must be created before installing any dependent chart:

bash
Copy code
kubectl apply -f secrets/oas-secret.yaml
kubectl apply -f secrets/airflow-secret.yaml
Or apply entire folder:

bash
Copy code
kubectl apply -f secrets/
🚀 Installation Instructions
You must apply secrets first, then install each chart using its values file.

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
Variables users should update
Database usernames/passwords inside secrets

OAuth / Keycloak client secrets

Internal service tokens

Storage class names in values files

External DNS / ingress FQDNs

Compatibility & Dependencies
Component	Depends On
Airflow	PXC-DB, Keycloak (optional)
NiFi	Kafka, Zookeeper
Kafka Connect	Kafka
Superset	Database (PXC-DB), authentication provider
Operators	None, but may manage CRDs for others

Ensure dependent services exist before deploying higher-level applications.
