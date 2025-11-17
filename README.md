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
├── airflow-git-secret.yaml
├── airflow-keycloak-cm.yaml
├── airflow-secret-example.yaml
├── flink-secret-example.yaml
├── kc-secret-example.yaml
├── nifi-secret-example.yaml
└── superset-secret-example.yaml


Real secrets are **not** stored in the repository.  
Developers instead copy templates from `secret-examples/`.

---

## 📁 Folder-by-Folder Explanation

### **airflow/**
Helm chart for deploying Apache Airflow components such as workers, scheduler, webserver, log configurations, ingress, and configs.

### **kafka-connect/**
Helm chart for deploying Kafka Connect, including connector configs, plugins, and worker configs.

### **nifi/**
Chart for running Apache NiFi clusters with node configs, zookeeper setup, flow configuration, and persistent volumes.

### **superset/**
Apache Superset deployment with webserver, metadata migrations, optional Celery workers, ingress, and database connectivity.

### **flink-chart/**
Custom chart packaging Apache Flink (session clusters, job clusters, configmaps).

### **pxc-db/**
Chart for Percona XtraDB Cluster, acting as the database backend for the OAS platform.

### **oas-operators/**
Contains operators or custom controllers required by the OAS platform (CRDs, RBAC, deployments).

### **secrets/**

#### `secrets/oas-secret.yaml`
Main secret bundle for platform-wide credentials.

#### `secrets/secret-examples/`
Contains **example templates** (safe defaults) for:

- Airflow Git secret  
- Airflow Keycloak configmap  
- NiFi credentials  
- Kafka Connect secrets  
- Flink example credentials  
- Superset admin credentials  

Users copy these templates to create real secrets.

---

## 🔐 How Secrets Work

### Why separate secrets?

- Prevents accidental commits of sensitive data  
- Ensures secrets are **not** packaged inside Helm charts  
- Allows different teams to maintain secrets independently  
- Keeps charts generic, portable, and reusable  

### How to use secret templates

1. Copy an example template:

   ```bash
   cp secrets/secret-examples/airflow-secret-example.yaml secrets/airflow-secret.yaml

2. Edit the copied file and insert real values.
3. Apply secrets before charts:
   ```bash
   kubectl apply -f secrets/
Apply all secrets at once
  ```bash
  kubectl apply -f secrets/


