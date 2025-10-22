# AI Agent Application Categorized Deployment Guide

## 📋 Table of Contents

1. [Deployment Architecture Overview](#deployment-architecture-overview)
2. [Categorized Deployment Strategy](#categorized-deployment-strategy)
3. [Core Components Deployment](#core-components-deployment)
4. [Application Layer Deployment](#application-layer-deployment)
5. [Monitoring System Deployment](#monitoring-system-deployment)
6. [Operations & Maintenance Guide](#operations--maintenance-guide)
7. [Troubleshooting Handbook](#troubleshooting-handbook)

---

## Deployment Architecture Overview

### Application Layered Architecture

```
┌──────────────────────────────────────────────────────┐
│            External Access Layer (NodePort/Ingress)   │
│  Web UI (30080) | API (30093) | Monitoring (30090)    │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│              Application Layer                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │AIAgent   │  │AIAgent   │  │Presidio  │            │
│  │Web (Java)│  │API (Py)  │  │Analyzer  │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│            Data Layer                                 │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────┐    │
│  │ Redis  │ │MongoDB │ │ MySQL  │ │Elasticsearch │    │
│  │ (cache)│ │(document)││(relational)││ (search)   │    │
│  └────────┘ └────────┘ └────────┘ └──────────────┘    │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│          Monitoring Layer                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │Prometheus│  │ Grafana  │  │Exporters │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└──────────────────────────────────────────────────────┘
```

### Component Classification Description

| Category | Component | Deployment Priority | Dependencies |
|---------:|----------|---------------------|-------------|
| **Core infrastructure** | Redis, MongoDB, MySQL, Elasticsearch | P0 (highest) | None |
| **Application core** | AIAgent Web, AIAgent API | P1 (high) | Depends on core infrastructure |
| **Privacy protection** | Presidio Analyzer | P2 (medium) | Runs independently |
| **Monitoring system** | Prometheus, Grafana, Exporters | P3 (medium) | Runs independently |
| **Host monitoring** | Node Exporter, cAdvisor | P4 (low) | Runs independently |

---

## Categorized Deployment Strategy

### Deployment Mode Selection

#### Mode 1: One-click Full Deployment (development/test)
```bash
# Applicable scenario: quick verification, development & testing
# Characteristics: deploy all components at once
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh
# Select: 3. Application Deployment -> 1. One-click Deploy
```

#### Mode 2: Incremental Categorized Deployment (recommended for production)
```bash
# Applicable scenario: production, fault isolation
# Characteristics: deploy step-by-step by category for better control and validation
# Order: Core infrastructure -> Application core -> Monitoring system
```

#### Mode 3: On-demand Selective Deployment (special scenarios)
```bash
# Applicable scenario: partial component migration, disaster recovery
# Characteristics: deploy only required components
```

---

## Core Components Deployment

### Step 1: Redis Cache Service

#### Using the deployment script

```bash
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh

# Menu path: 3. Application Deployment -> 4. System Components Deployment
# Choose: Deploy only Redis
```

#### Manual deployment with Helm

```bash
# 1. Prepare the values file
cat > redis-values.yaml <<EOF
redis:
  enabled: true
  image: ubuntu/redis:6.0-22.04_edge
  password: "YOUR_REDIS_PASSWORD"
  maxmemory: "2gb"
  maxmemoryPolicy: "allkeys-lru"
  resources:
    requests:
      memory: 512Mi
      cpu: 200m
    limits:
      memory: 2Gi
      cpu: 1000m

# Disable other components
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy Redis
helm upgrade --install redis-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  --create-namespace \
  -f redis-values.yaml

# 3. Verify deployment
kubectl get pods -n aiagent-system -l app=redis
kubectl get svc -n aiagent-system redis

# 4. Test connectivity
kubectl run -it --rm redis-test \
  --image=redis:6 \
  --restart=Never \
  -n aiagent-system \
  -- redis-cli -h redis -a YOUR_REDIS_PASSWORD ping
# Expected output: PONG
```

#### Verification checklist
- [ ] Pod status is Running
- [ ] Service ClusterIP is reachable
- [ ] PVC is Bound
- [ ] Data directory exists (`/opt/aiagent-k3s/data/redis`)
- [ ] Can connect with password

---

### Step 2: MongoDB Document Database

```bash
# 1. Values file
cat > mongodb-values.yaml <<EOF
mongodb:
  enabled: true
  image: mongodb/mongodb-community-server:8.0.4-ubuntu2204
  database: aiagent
  username: aiagent
  password: "YOUR_MONGODB_PASSWORD"
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 4Gi
      cpu: 2000m

redis:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install mongodb-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f mongodb-values.yaml

# 3. Verify
kubectl get pods -n aiagent-system -l app=mongodb
kubectl logs -f -l app=mongodb -n aiagent-system

# 4. Test connectivity
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p YOUR_MONGODB_PASSWORD --authenticationDatabase aiagent
# Run: db.runCommand({ ping: 1 })
```

#### Initialize the database

```bash
# Create initialization script
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p YOUR_MONGODB_PASSWORD --authenticationDatabase aiagent <<EOF
use aiagent;
db.createCollection("conversations");
db.createCollection("users");
db.conversations.createIndex({ "userId": 1, "createdAt": -1 });
show collections;
EOF
```

---

### Step 3: MySQL Relational Database

```bash
# 1. Values file
cat > mysql-values.yaml <<EOF
mysql:
  enabled: true
  image: ubuntu/mysql:8.0-22.04_edge
  database: aiagent
  username: aiagent
  password: "YOUR_MYSQL_PASSWORD"
  rootPassword: "YOUR_MYSQL_ROOT_PASSWORD"
  maxConnections: 2000
  init:
    enabled: true
    readonlyUser:
      username: "aiagent_user_ro"
      password: "YOUR_READONLY_PASSWORD"
  resources:
    requests:
      memory: 2Gi
      cpu: 1000m
    limits:
      memory: 8Gi
      cpu: 4000m

redis:
  enabled: false
mongodb:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install mysql-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f mysql-values.yaml

# 3. Wait for readiness (may take 2-3 minutes)
kubectl wait --for=condition=ready pod \
  -l app=mysql \
  -n aiagent-system \
  --timeout=300s

# 4. Test connectivity
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -pYOUR_MYSQL_PASSWORD -e "SHOW DATABASES;"
```

#### Initialize table schema

```bash
# Execute initialization SQL
kubectl exec -i -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -pYOUR_MYSQL_PASSWORD aiagent <<EOF
CREATE TABLE IF NOT EXISTS users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS sessions (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  session_token VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

SHOW TABLES;
EOF

# Verify read-only user
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent_user_ro -pYOUR_READONLY_PASSWORD -e "SHOW DATABASES;"
```

---

### Step 4: Elasticsearch Search Engine

```bash
# 1. Values file
cat > elasticsearch-values.yaml <<EOF
elasticsearch:
  enabled: true
  image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
  clusterName: aiagent-cluster-prod
  password: "YOUR_ES_PASSWORD"
  javaOpts: "-Xms4g -Xmx4g"
  discoveryType: "single-node"
  xpackSecurityEnabled: true

  # Plugin configuration
  plugins:
    enabled: true
    hostPath: "/opt/aiagent-k3s/data/elasticsearch/plugin"
    ik:
      enabled: true
      version: "8.17.0"

  # Initialization scripts
  init:
    enabled: true
    scripts:
      dataset: "create_dataset.sh"
      longTermMemory: "create_long_term_memory.sh"

  resources:
    requests:
      memory: 4Gi
      cpu: 1000m
    limits:
      memory: 8Gi
      cpu: 4000m

redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
EOF

# 2. Prepare the IK tokenizer plugin
sudo mkdir -p /opt/aiagent-k3s/data/elasticsearch/plugin
sudo cp -r aiagent_component/elasticsearch/plugin/ik \
  /opt/aiagent-k3s/data/elasticsearch/plugin/
sudo chown -R 1000:1000 /opt/aiagent-k3s/data/elasticsearch/plugin

# 3. Deploy
helm upgrade --install elasticsearch-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f elasticsearch-values.yaml

# 4. Wait for readiness (may take 3-5 minutes)
kubectl wait --for=condition=ready pod \
  -l app=elasticsearch \
  -n aiagent-system \
  --timeout=600s

# 5. Test connectivity
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') \
  -- curl -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200
```

#### Initialize indices

```bash
# 1. Get pod
ES_POD=$(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}')

# 2. Create dataset indices (64 indices)
kubectl exec -n aiagent-system $ES_POD -- bash /scripts/create_dataset.sh

# 3. Create long-term memory indices (32 indices)
kubectl exec -n aiagent-system $ES_POD -- bash /scripts/create_long_term_memory.sh

# 4. Verify indices were created
kubectl exec -n aiagent-system $ES_POD -- \
  curl -s -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200/_cat/indices?v

# 5. Verify IK plugin
kubectl exec -n aiagent-system $ES_POD -- \
  curl -s -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200/_cat/plugins?v

# 6. Test IK analyzer
kubectl exec -n aiagent-system $ES_POD -- \
  curl -s -u elastic:YOUR_ES_PASSWORD -k -X POST \
  "https://localhost:9200/_analyze" \
  -H 'Content-Type: application/json' \
  -d '{"analyzer": "ik_max_word", "text": "中华人民共和国"}'
```

---

### Core Components Verification

```bash
# Create the verification script
cat > verify-components.sh <<'EOF'
#!/bin/bash

echo "========================================="
echo "Core Components Deployment Verification"
echo "========================================="

# Check Redis
echo -e "\n1. Redis check:"
kubectl get pods -n aiagent-system -l app=redis
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=redis -o jsonpath='{.items[0].metadata.name}') \
  -- redis-cli -a YOUR_REDIS_PASSWORD ping

# Check MongoDB
echo -e "\n2. MongoDB check:"
kubectl get pods -n aiagent-system -l app=mongodb
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p YOUR_MONGODB_PASSWORD --authenticationDatabase aiagent --eval "db.runCommand({ ping: 1 })"

# Check MySQL
echo -e "\n3. MySQL check:"
kubectl get pods -n aiagent-system -l app=mysql
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -pYOUR_MYSQL_PASSWORD -e "SELECT VERSION();"

# Check Elasticsearch
echo -e "\n4. Elasticsearch check:"
kubectl get pods -n aiagent-system -l app=elasticsearch
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200/_cluster/health?pretty

echo -e "\n========================================="
echo "Verification complete"
echo "========================================="
EOF

chmod +x verify-components.sh
./verify-components.sh
```

---

## Application Layer Deployment

### Step 5: AIAgent Web Application (Java + Spring Boot)

```bash
# 1. Prepare license file (optional but recommended)
sudo mkdir -p /opt/install_init_file
# Place the [projectID].license file into this directory
# Example: 68a6ed09ec70d46f26ed6e4e.license

# 2. Values file
cat > aiagent-web-values.yaml <<EOF
aiagent:
  web:
    enabled: true
    image: aiagent-web:latest
    replicas: 3  # 3 replicas recommended for production

    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

    # Horizontal Pod Autoscaler
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70

    # Health checks
    healthcheck:
      enabled: true
      liveness:
        httpGet:
          path: /actuator/health
          port: 8080
        initialDelaySeconds: 60
        periodSeconds: 30
      readiness:
        httpGet:
          path: /actuator/health
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10

  api:
    enabled: false

# Core infrastructure already deployed, set to false
redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 3. Deploy
helm upgrade --install aiagent-web \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f aiagent-web-values.yaml

# 4. Wait for rollout to complete
kubectl rollout status deployment/aiagent-web -n aiagent-system

# 5. Verify
kubectl get pods -n aiagent-system -l app=aiagent-web -o wide
kubectl logs -f -l app=aiagent-web -n aiagent-system --tail=100

# 6. Verify license processing
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=aiagent-web -o jsonpath='{.items[0].metadata.name}') \
  -- ls -la /opt/developer/config/

# 7. Test access
curl http://NODE_IP:30080
```

#### Viewing application logs

```bash
# Live logs
kubectl logs -f -l app=aiagent-web -n aiagent-system

# View the last 100 lines
kubectl logs --tail=100 -l app=aiagent-web -n aiagent-system

# Logs for a specific pod
kubectl logs -f aiagent-web-xxxxx-xxxxx -n aiagent-system

# Previous container logs (for crash investigation)
kubectl logs --previous aiagent-web-xxxxx-xxxxx -n aiagent-system
```

---

### Step 6: AIAgent API Application (Python + FastAPI)

```bash
# 1. Values file
cat > aiagent-api-values.yaml <<EOF
aiagent:
  web:
    enabled: false

  api:
    enabled: true
    image: aiagent-api:latest
    replicas: 3  # 3 replicas recommended for production

    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

    # Horizontal Pod Autoscaler
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70

    # Health checks
    healthcheck:
      enabled: true
      liveness:
        httpGet:
          path: /health
          port: 9300
        initialDelaySeconds: 30
        periodSeconds: 30
      readiness:
        httpGet:
          path: /health
          port: 9300
        initialDelaySeconds: 15
        periodSeconds: 10

redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install aiagent-api \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f aiagent-api-values.yaml

# 3. Verify
kubectl rollout status deployment/aiagent-api -n aiagent-system
kubectl get pods -n aiagent-system -l app=aiagent-api -o wide

# 4. Test API health endpoint
curl http://NODE_IP:30093/health
```

---

### Step 7: Presidio Privacy Protection Service

```bash
# 1. Values file
cat > presidio-values.yaml <<EOF
presidio:
  enabled: true
  image: presidio_analyzer-22358:20250612

  resources:
    requests:
      memory: 512Mi
      cpu: 200m
    limits:
      memory: 2Gi
      cpu: 1000m

aiagent:
  web:
    enabled: false
  api:
    enabled: false

redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install presidio \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f presidio-values.yaml

# 3. Verify
kubectl get pods -n aiagent-system -l app=presidio

# 4. Test privacy detection
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=presidio -o jsonpath='{.items[0].metadata.name}') \
  -- curl -X POST http://localhost:3000/analyze \
  -H 'Content-Type: application/json' \
  -d '{"text":"My email is test@example.com","language":"en"}'
```

---

## Monitoring System Deployment

### Step 8: Prometheus Monitoring

```bash
# 1. Values file
cat > prometheus-values.yaml <<EOF
monitoring:
  enabled: true

  prometheus:
    enabled: true
    image: prom/prometheus:latest
    retention: 30d  # retain 30 days of data

    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2000m

    # Alerting rules
    alerting:
      enabled: true

  grafana:
    enabled: false

  exporters:
    enabled: false

aiagent:
  web:
    enabled: false
  api:
    enabled: false

redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install prometheus \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f prometheus-values.yaml

# 3. Verify
kubectl get pods -n aiagent-system -l app=prometheus
kubectl get svc -n aiagent-system prometheus

# 4. Access Prometheus UI
kubectl port-forward -n aiagent-system svc/prometheus 9090:9090
# Browser access: http://localhost:9090
```

---

### Step 9: Grafana Visualization

```bash
# 1. Values file
cat > grafana-values.yaml <<EOF
monitoring:
  enabled: true

  prometheus:
    enabled: false

  grafana:
    enabled: true
    image: grafana/grafana:latest
    adminPassword: "YOUR_GRAFANA_PASSWORD"  # ⚠️ change this password

    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 1Gi
        cpu: 500m

    # Pre-installed dashboards
    dashboards:
      enabled: true

aiagent:
  web:
    enabled: false
  api:
    enabled: false

redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install grafana \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f grafana-values.yaml

# 3. Retrieve admin password
kubectl get secret -n aiagent-system grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d

# 4. Access Grafana UI
kubectl port-forward -n aiagent-system svc/grafana 3000:80
# Browser access: http://localhost:3000
# Username: admin
# Password: YOUR_GRAFANA_PASSWORD
```

#### Configure Grafana data source

```bash
# Automatically configure Prometheus as a data source
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=grafana -o jsonpath='{.items[0].metadata.name}') \
  -- grafana-cli admin reset-admin-password YOUR_GRAFANA_PASSWORD
```

---

### Step 10: Database Exporters

```bash
# 1. Values file
cat > exporters-values.yaml <<EOF
monitoring:
  enabled: true

  prometheus:
    enabled: false

  grafana:
    enabled: false

  exporters:
    mongodb:
      enabled: true
      image: percona/mongodb_exporter:0.44

    mysql:
      enabled: true
      image: prom/mysqld-exporter:v0.17.2

    elasticsearch:
      enabled: true
      image: prometheuscommunity/elasticsearch-exporter:v1.9.0

    redis:
      enabled: true
      image: oliver006/redis_exporter:latest

aiagent:
  web:
    enabled: false
  api:
    enabled: false

redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. Deploy
helm upgrade --install exporters \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f exporters-values.yaml

# 3. Verify
kubectl get pods -n aiagent-system | grep exporter
```

---

## Operations & Maintenance Guide

### 1. Daily Operations

#### View application status

```bash
# Quick status check
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh status

# Or use kubectl
kubectl get all -n aiagent-system
kubectl get pods -n aiagent-system -o wide
kubectl top pods -n aiagent-system
kubectl top nodes
```

#### View logs

```bash
# Use the deployment script logs menu
./aiagent-deploy.sh logs

# Manual log viewing
# Web application logs
kubectl logs -f -l app=aiagent-web -n aiagent-system --tail=100

# API application logs
kubectl logs -f -l app=aiagent-api -n aiagent-system --tail=100

# Redis logs
kubectl logs -f -l app=redis -n aiagent-system --tail=100

# All pod logs for the web app
kubectl logs -f --all-containers -l app=aiagent-web -n aiagent-system
```

#### Enter containers for debugging

```bash
# Enter Web container
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=aiagent-web -o jsonpath='{.items[0].metadata.name}') \
  -- /bin/bash

# Enter MySQL container
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root -p

# Temporary debug pod
kubectl run debug --rm -it --image=busybox -n aiagent-system -- sh
```

---

### 2. Application Update Procedures

#### Rolling update the application

```bash
# Method 1: Helm upgrade
helm upgrade aiagent-web \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f aiagent-web-values.yaml \
  --set aiagent.web.image=aiagent-web:v2.0

# Method 2: Update image directly
kubectl set image deployment/aiagent-web \
  aiagent-web=aiagent-web:v2.0 \
  -n aiagent-system

# Check rolling update status
kubectl rollout status deployment/aiagent-web -n aiagent-system

# View update history
kubectl rollout history deployment/aiagent-web -n aiagent-system
```

#### Rollback to previous version

```bash
# Rollback to previous revision
kubectl rollout undo deployment/aiagent-web -n aiagent-system

# Rollback to a specific revision
kubectl rollout undo deployment/aiagent-web \
  -n aiagent-system \
  --to-revision=2

# Verify rollback
kubectl rollout status deployment/aiagent-web -n aiagent-system
```

---

### 3. Scaling Operations

#### Manual scaling

```bash
# Scale up to 5 replicas
kubectl scale deployment/aiagent-web \
  --replicas=5 \
  -n aiagent-system

# Scale down to 2 replicas
kubectl scale deployment/aiagent-web \
  --replicas=2 \
  -n aiagent-system

# Verify
kubectl get pods -n aiagent-system -l app=aiagent-web
```

#### Configure HPA (Horizontal Pod Autoscaler)

```bash
# Create HPA
kubectl autoscale deployment aiagent-web \
  --cpu-percent=70 \
  --min=3 \
  --max=10 \
  -n aiagent-system

# View HPA status
kubectl get hpa -n aiagent-system
kubectl describe hpa aiagent-web -n aiagent-system

# Delete HPA
kubectl delete hpa aiagent-web -n aiagent-system
```

---

### 4. Backup and Restore

#### Backup databases

```bash
# MySQL backup
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysqldump -u root -pYOUR_ROOT_PASSWORD aiagent > aiagent_backup_$(date +%Y%m%d).sql

# MongoDB backup
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongodump --uri="mongodb://aiagent:YOUR_PASSWORD@localhost:27017/aiagent" \
  --out=/tmp/backup

# Copy backup files to local
kubectl cp aiagent-system/$(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}'):/tmp/backup \
  ./mongodb_backup_$(date +%Y%m%d)
```

#### Restore databases

```bash
# MySQL restore
kubectl exec -i -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root -pYOUR_ROOT_PASSWORD aiagent < aiagent_backup_20250122.sql

# MongoDB restore
kubectl cp ./mongodb_backup_20250122 \
  aiagent-system/$(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}'):/tmp/restore

kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongorestore --uri="mongodb://aiagent:YOUR_PASSWORD@localhost:27017/aiagent" \
  /tmp/restore/aiagent
```

---

### 5. Configuration Management

#### Update ConfigMap

```bash
# View existing ConfigMaps
kubectl get configmap -n aiagent-system

# Edit a ConfigMap
kubectl edit configmap aiagent-web-config -n aiagent-system

# Restart application to apply configuration
kubectl rollout restart deployment/aiagent-web -n aiagent-system
```

#### Update Secret

```bash
# Create/update Secret
kubectl create secret generic aiagent-web-secret \
  --from-literal=db-password=NEW_PASSWORD \
  -n aiagent-system \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart application
kubectl rollout restart deployment/aiagent-web -n aiagent-system
```

---

### 6. Resource Cleanup

#### Remove a single component

```bash
# Delete Deployment
kubectl delete deployment aiagent-web -n aiagent-system

# Delete Service
kubectl delete svc aiagent-web -n aiagent-system

# Delete PVC (⚠️ will remove data)
kubectl delete pvc redis-data -n aiagent-system
```

#### Full uninstall

```bash
# Use the deployment script to undeploy
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh undeploy

# Manually uninstall all Helm releases
helm list -n aiagent-system
helm uninstall redis-only -n aiagent-system
helm uninstall mongodb-only -n aiagent-system
helm uninstall mysql-only -n aiagent-system
helm uninstall elasticsearch-only -n aiagent-system
helm uninstall aiagent-web -n aiagent-system
helm uninstall aiagent-api -n aiagent-system
helm uninstall presidio -n aiagent-system
helm uninstall prometheus -n aiagent-system
helm uninstall grafana -n aiagent-system

# Delete namespace (⚠️ will remove all resources)
kubectl delete namespace aiagent-system
```

---

## Troubleshooting Handbook

### Common issue diagnosis

#### Pod fails to start

```bash
# 1. Check pod status
kubectl get pods -n aiagent-system -o wide

# 2. Describe pod for details
kubectl describe pod POD_NAME -n aiagent-system

# 3. Check events
kubectl get events -n aiagent-system --sort-by='.lastTimestamp' | tail -20

# 4. Common causes and checks:

# Image pull failure
kubectl describe pod POD_NAME -n aiagent-system | grep -A 10 "Events:"
# Fix: verify image name, imagePullPolicy

# Resource shortage
kubectl describe nodes | grep -A 5 "Allocated resources"
# Fix: add nodes or adjust resource limits

# Configuration errors
kubectl logs POD_NAME -n aiagent-system
# Fix: verify ConfigMap and Secret configurations
```

#### Application inaccessible

```bash
# 1. Check Service
kubectl get svc -n aiagent-system
kubectl describe svc aiagent-web -n aiagent-system

# 2. Check Endpoints
kubectl get endpoints aiagent-web -n aiagent-system

# 3. Test ClusterIP access
kubectl run curl-test --rm -it --image=curlimages/curl -n aiagent-system \
  -- curl http://aiagent-web:8080/actuator/health

# 4. Test NodePort access
curl http://NODE_IP:30080
```

#### Database connection failures

```bash
# 1. Check database pods
kubectl get pods -n aiagent-system -l app=mysql

# 2. Test DB connection
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -p

# 3. Check application logs for connection errors
kubectl logs -f -l app=aiagent-web -n aiagent-system | grep -i "connection\|mysql"

# 4. Verify Service DNS
kubectl run dns-test --rm -it --image=busybox -n aiagent-system \
  -- nslookup mysql.aiagent-system.svc.cluster.local
```

#### Storage troubleshooting

```bash
# 1. Check PVC status
kubectl get pvc -n aiagent-system

# 2. Check PV status
kubectl get pv

# 3. PVC Pending investigation
kubectl describe pvc PVC_NAME -n aiagent-system

# 4. Check StorageClass
kubectl get sc
kubectl describe sc longhorn

# 5. Check disk space
df -h /opt/aiagent-k3s/data/
```

---

### Performance Troubleshooting

#### High CPU/memory usage

```bash
# 1. Check resource usage
kubectl top pods -n aiagent-system
kubectl top nodes

# 2. Check resource limits
kubectl describe pod POD_NAME -n aiagent-system | grep -A 5 "Limits\|Requests"

# 3. Check HPA status
kubectl get hpa -n aiagent-system

# 4. Temporary scale-out
kubectl scale deployment/aiagent-web --replicas=5 -n aiagent-system
```

#### Database performance issues

```bash
# MySQL slow queries
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root -p -e "SHOW FULL PROCESSLIST;"

# MongoDB performance stats
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p --authenticationDatabase aiagent \
  --eval "db.serverStatus()"

# Elasticsearch performance
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s -u elastic:PASSWORD -k https://localhost:9200/_nodes/stats/jvm,thread_pool
```

---

### Monitoring Alerts

#### Configure alerting rules

```bash
# Create alert rules ConfigMap
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: aiagent-system
data:
  alerts.yml: |
    groups:
    - name: aiagent_alerts
      interval: 30s
      rules:
      # Pod restart alert
      - alert: PodRestarting
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ \$labels.pod }} is restarting frequently"

      # High memory usage
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ \$labels.pod }} memory usage exceeds 90%"

      # High MySQL connections
      - alert: MySQLHighConnections
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL connections exceed 80%"
EOF
```

---

### Quick Operations Command Reference

```bash
# ============ View status ============
kubectl get all -n aiagent-system
kubectl get pods -n aiagent-system -o wide
kubectl top pods -n aiagent-system

# ============ View logs ============
kubectl logs -f -l app=aiagent-web -n aiagent-system --tail=100
kubectl logs --previous POD_NAME -n aiagent-system

# ============ Troubleshooting ============
kubectl describe pod POD_NAME -n aiagent-system
kubectl get events -n aiagent-system --sort-by='.lastTimestamp'

# ============ Enter container ============
kubectl exec -it POD_NAME -n aiagent-system -- /bin/bash

# ============ Update application ============
kubectl set image deployment/aiagent-web aiagent-web=new-image:tag -n aiagent-system
kubectl rollout status deployment/aiagent-web -n aiagent-system

# ============ Rollback application ============
kubectl rollout undo deployment/aiagent-web -n aiagent-system

# ============ Scale ============
kubectl scale deployment/aiagent-web --replicas=5 -n aiagent-system

# ============ Restart application ============
kubectl rollout restart deployment/aiagent-web -n aiagent-system

# ============ Delete Pod (auto-recreate) ============
kubectl delete pod POD_NAME -n aiagent-system
```

---

## Appendix

### A. Full Deployment Checklist

#### Core infrastructure checks
- [ ] Redis Pod Running and reachable
- [ ] MongoDB Pod Running, reachable, and database initialized
- [ ] MySQL Pod Running, reachable, table schema created, read-only user OK
- [ ] Elasticsearch Pod Running, indices created (96 total), IK tokenizer functioning

#### Application layer checks
- [ ] AIAgent Web Pod Running (3 replicas), health checks passing
- [ ] AIAgent API Pod Running (3 replicas), health checks passing
- [ ] Presidio Pod Running, privacy detection working
- [ ] License file processed (if applicable)

#### Monitoring system checks
- [ ] Prometheus Pod Running, metrics collection normal
- [ ] Grafana Pod Running, login successful, data source configured
- [ ] Database Exporters Pod Running, metrics available

#### Access verification
- [ ] Web UI accessible: http://NODE_IP:30080
- [ ] API reachable: http://NODE_IP:30093/health
- [ ] Prometheus reachable: http://NODE_IP:30090
- [ ] Grafana reachable: http://NODE_IP:30300

#### Storage verification
- [ ] All PVCs are Bound
- [ ] Data directory permissions are correct
- [ ] Data persistence verified (no data loss after pod restart)

---

### B. Categorized Deployment Script Template

Create a convenient script for categorized deployment:

```bash
cat > deploy-by-category.sh <<'EOF'
#!/bin/bash

# AIAgent categorized deployment script

set -e

NAMESPACE="aiagent-system"
CHART_DIR="../helm-charts/aiagent"

log_info() {
    echo -e "\033[0;36m[INFO]\033[0m $1"
}

log_success() {
    echo -e "\033[0;32m[SUCCESS]\033[0m $1"
}

# Deploy Redis
deploy_redis() {
    log_info "Deploying Redis..."
    cat > /tmp/redis-values.yaml <<YAML
redis:
  enabled: true
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
aiagent:
  web:
    enabled: false
  api:
    enabled: false
YAML

    helm upgrade --install redis-only $CHART_DIR \
        -n $NAMESPACE --create-namespace \
        -f /tmp/redis-values.yaml

    kubectl wait --for=condition=ready pod -l app=redis -n $NAMESPACE --timeout=300s
    log_success "Redis deployment complete"
}

# Deploy MongoDB
deploy_mongodb() {
    log_info "Deploying MongoDB..."
    # ... (similar to Redis configuration)
}

# Deploy MySQL
deploy_mysql() {
    log_info "Deploying MySQL..."
    # ... (similar to Redis configuration)
}

# Deploy Elasticsearch
deploy_elasticsearch() {
    log_info "Deploying Elasticsearch..."
    # ... (similar to Redis configuration)
}

# Main menu
show_menu() {
    echo "========================================"
    echo "AIAgent Categorized Deployment Menu"
    echo "========================================"
    echo "1. Deploy core component (Redis)"
    echo "2. Deploy core component (MongoDB)"
    echo "3. Deploy core component (MySQL)"
    echo "4. Deploy core component (Elasticsearch)"
    echo "5. Deploy all core components"
    echo "6. Deploy AIAgent Web"
    echo "7. Deploy AIAgent API"
    echo "8. Deploy Presidio"
    echo "9. Deploy monitoring system"
    echo "0. Exit"
    echo "========================================"
}

# Main loop
while true; do
    show_menu
    read -p "Select an option [0-9]: " choice

    case $choice in
        1) deploy_redis ;;
        2) deploy_mongodb ;;
        3) deploy_mysql ;;
        4) deploy_elasticsearch ;;
        5) deploy_redis; deploy_mongodb; deploy_mysql; deploy_elasticsearch ;;
        0) exit 0 ;;
        *) echo "Invalid selection" ;;
    esac

    echo ""
    read -p "Press Enter to continue..."
done
EOF

chmod +x deploy-by-category.sh
```

---

Document version: v1.0  
Last updated: 2025-01-22  
Maintainer: DevOps Team
