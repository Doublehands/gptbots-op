# AIAgent 应用分类部署文档

## 📋 目录

1. [部署架构概述](#部署架构概述)
2. [分类部署方案](#分类部署方案)
3. [基础组件部署](#基础组件部署)
4. [应用层部署](#应用层部署)
5. [监控系统部署](#监控系统部署)
6. [运维操作指南](#运维操作指南)
7. [故障排查手册](#故障排查手册)

---

## 部署架构概述

### 应用分层架构

```
┌──────────────────────────────────────────────────────┐
│            外部访问层 (NodePort/Ingress)              │
│  Web UI (30080) | API (30093) | Monitoring (30090)  │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│              应用层 (Application Layer)               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │AIAgent   │  │AIAgent   │  │Presidio  │           │
│  │Web (Java)│  │API (Py)  │  │Analyzer  │           │
│  └──────────┘  └──────────┘  └──────────┘           │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│            数据层 (Data Layer)                        │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────┐  │
│  │ Redis  │ │MongoDB │ │ MySQL  │ │Elasticsearch │  │
│  │ (缓存) │ │(文档)  │ │(关系)  │ │   (搜索)     │  │
│  └────────┘ └────────┘ └────────┘ └──────────────┘  │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│          监控层 (Monitoring Layer)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │Prometheus│  │ Grafana  │  │Exporters │           │
│  └──────────┘  └──────────┘  └──────────┘           │
└──────────────────────────────────────────────────────┘
```

### 组件分类说明

| 分类 | 组件 | 部署优先级 | 依赖关系 |
|------|------|-----------|---------|
| **基础组件** | Redis, MongoDB, MySQL, Elasticsearch | P0 (最高) | 无依赖 |
| **应用核心** | AIAgent Web, AIAgent API | P1 (高) | 依赖基础组件 |
| **隐私保护** | Presidio Analyzer | P2 (中) | 独立运行 |
| **监控系统** | Prometheus, Grafana, Exporters | P3 (中) | 独立运行 |
| **主机监控** | Node Exporter, cAdvisor | P4 (低) | 独立运行 |

---

## 分类部署方案

### 部署模式选择

#### 模式1: 一键完整部署 (开发/测试环境)
```bash
# 适用场景: 快速验证、开发测试
# 特点: 一次性部署所有组件
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh
# 选择: 3. 应用部署 -> 1. 一键部署
```

#### 模式2: 分类渐进部署 (生产环境推荐)
```bash
# 适用场景: 生产环境、故障隔离
# 特点: 按分类逐步部署,便于控制和验证
# 顺序: 基础组件 -> 应用核心 -> 监控系统
```

#### 模式3: 按需选择部署 (特殊场景)
```bash
# 适用场景: 部分组件迁移、灾难恢复
# 特点: 只部署需要的组件
```

---

## 基础组件部署

### 第一步: Redis 缓存服务

#### 使用部署脚本

```bash
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh

# 菜单路径: 3. 应用部署 -> 4. 系统组件部署
# 选择: 仅部署 Redis
```

#### 使用Helm手动部署

```bash
# 1. 准备配置文件
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

# 禁用其他组件
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 2. 部署Redis
helm upgrade --install redis-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  --create-namespace \
  -f redis-values.yaml

# 3. 验证部署
kubectl get pods -n aiagent-system -l app=redis
kubectl get svc -n aiagent-system redis

# 4. 测试连接
kubectl run -it --rm redis-test \
  --image=redis:6 \
  --restart=Never \
  -n aiagent-system \
  -- redis-cli -h redis -a YOUR_REDIS_PASSWORD ping
# 应输出: PONG
```

#### 验证清单
- [ ] Pod 状态为 Running
- [ ] Service ClusterIP 可访问
- [ ] PVC 已绑定 (Bound)
- [ ] 数据目录已创建 (`/opt/aiagent-k3s/data/redis`)
- [ ] 可通过密码连接

---

### 第二步: MongoDB 文档数据库

```bash
# 1. 配置文件
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

# 2. 部署
helm upgrade --install mongodb-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f mongodb-values.yaml

# 3. 验证
kubectl get pods -n aiagent-system -l app=mongodb
kubectl logs -f -l app=mongodb -n aiagent-system

# 4. 测试连接
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p YOUR_MONGODB_PASSWORD --authenticationDatabase aiagent
# 执行: db.runCommand({ ping: 1 })
```

#### 初始化数据库

```bash
# 创建初始化脚本
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

### 第三步: MySQL 关系数据库

```bash
# 1. 配置文件
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

# 2. 部署
helm upgrade --install mysql-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f mysql-values.yaml

# 3. 等待就绪 (可能需要2-3分钟)
kubectl wait --for=condition=ready pod \
  -l app=mysql \
  -n aiagent-system \
  --timeout=300s

# 4. 测试连接
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -pYOUR_MYSQL_PASSWORD -e "SHOW DATABASES;"
```

#### 初始化表结构

```bash
# 执行初始化SQL
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

# 验证只读用户
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent_user_ro -pYOUR_READONLY_PASSWORD -e "SHOW DATABASES;"
```

---

### 第四步: Elasticsearch 搜索引擎

```bash
# 1. 配置文件
cat > elasticsearch-values.yaml <<EOF
elasticsearch:
  enabled: true
  image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
  clusterName: aiagent-cluster-prod
  password: "YOUR_ES_PASSWORD"
  javaOpts: "-Xms4g -Xmx4g"
  discoveryType: "single-node"
  xpackSecurityEnabled: true

  # 插件配置
  plugins:
    enabled: true
    hostPath: "/opt/aiagent-k3s/data/elasticsearch/plugin"
    ik:
      enabled: true
      version: "8.17.0"

  # 初始化配置
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

# 2. 准备ik分词器插件
sudo mkdir -p /opt/aiagent-k3s/data/elasticsearch/plugin
sudo cp -r aiagent_component/elasticsearch/plugin/ik \
  /opt/aiagent-k3s/data/elasticsearch/plugin/
sudo chown -R 1000:1000 /opt/aiagent-k3s/data/elasticsearch/plugin

# 3. 部署
helm upgrade --install elasticsearch-only \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f elasticsearch-values.yaml

# 4. 等待就绪 (可能需要3-5分钟)
kubectl wait --for=condition=ready pod \
  -l app=elasticsearch \
  -n aiagent-system \
  --timeout=600s

# 5. 测试连接
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') \
  -- curl -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200
```

#### 初始化索引

```bash
# 1. 进入Pod
ES_POD=$(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}')

# 2. 创建数据集索引 (64个索引)
kubectl exec -n aiagent-system $ES_POD -- bash /scripts/create_dataset.sh

# 3. 创建长期记忆索引 (32个索引)
kubectl exec -n aiagent-system $ES_POD -- bash /scripts/create_long_term_memory.sh

# 4. 验证索引创建
kubectl exec -n aiagent-system $ES_POD -- \
  curl -s -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200/_cat/indices?v

# 5. 验证ik分词器
kubectl exec -n aiagent-system $ES_POD -- \
  curl -s -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200/_cat/plugins?v

# 6. 测试ik分词
kubectl exec -n aiagent-system $ES_POD -- \
  curl -s -u elastic:YOUR_ES_PASSWORD -k -X POST \
  "https://localhost:9200/_analyze" \
  -H 'Content-Type: application/json' \
  -d '{"analyzer": "ik_max_word", "text": "中华人民共和国"}'
```

---

### 基础组件部署验证

```bash
# 创建验证脚本
cat > verify-components.sh <<'EOF'
#!/bin/bash

echo "========================================="
echo "基础组件部署验证"
echo "========================================="

# 检查Redis
echo -e "\n1. Redis 检查:"
kubectl get pods -n aiagent-system -l app=redis
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=redis -o jsonpath='{.items[0].metadata.name}') \
  -- redis-cli -a YOUR_REDIS_PASSWORD ping

# 检查MongoDB
echo -e "\n2. MongoDB 检查:"
kubectl get pods -n aiagent-system -l app=mongodb
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p YOUR_MONGODB_PASSWORD --authenticationDatabase aiagent --eval "db.runCommand({ ping: 1 })"

# 检查MySQL
echo -e "\n3. MySQL 检查:"
kubectl get pods -n aiagent-system -l app=mysql
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -pYOUR_MYSQL_PASSWORD -e "SELECT VERSION();"

# 检查Elasticsearch
echo -e "\n4. Elasticsearch 检查:"
kubectl get pods -n aiagent-system -l app=elasticsearch
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s -u elastic:YOUR_ES_PASSWORD -k https://localhost:9200/_cluster/health?pretty

echo -e "\n========================================="
echo "验证完成"
echo "========================================="
EOF

chmod +x verify-components.sh
./verify-components.sh
```

---

## 应用层部署

### 第五步: AIAgent Web 应用 (Java + Spring Boot)

```bash
# 1. 准备License许可证文件 (可选但推荐)
sudo mkdir -p /opt/install_init_file
# 将 [projectID].license 文件放到该目录
# 示例: 68a6ed09ec70d46f26ed6e4e.license

# 2. 配置文件
cat > aiagent-web-values.yaml <<EOF
aiagent:
  web:
    enabled: true
    image: aiagent-web:latest
    replicas: 3  # 生产环境3副本

    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

    # 自动伸缩
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70

    # 健康检查
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

# 基础组件已部署,设置为false
redis:
  enabled: false
mongodb:
  enabled: false
mysql:
  enabled: false
elasticsearch:
  enabled: false
EOF

# 3. 部署
helm upgrade --install aiagent-web \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f aiagent-web-values.yaml

# 4. 等待就绪
kubectl rollout status deployment/aiagent-web -n aiagent-system

# 5. 验证
kubectl get pods -n aiagent-system -l app=aiagent-web -o wide
kubectl logs -f -l app=aiagent-web -n aiagent-system --tail=100

# 6. 验证License处理
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=aiagent-web -o jsonpath='{.items[0].metadata.name}') \
  -- ls -la /opt/developer/config/

# 7. 测试访问
curl http://NODE_IP:30080
```

#### 查看应用日志

```bash
# 实时日志
kubectl logs -f -l app=aiagent-web -n aiagent-system

# 查看最近100行
kubectl logs --tail=100 -l app=aiagent-web -n aiagent-system

# 查看特定Pod
kubectl logs -f aiagent-web-xxxxx-xxxxx -n aiagent-system

# 查看上一次启动的日志 (排查崩溃问题)
kubectl logs --previous aiagent-web-xxxxx-xxxxx -n aiagent-system
```

---

### 第六步: AIAgent API 应用 (Python + FastAPI)

```bash
# 1. 配置文件
cat > aiagent-api-values.yaml <<EOF
aiagent:
  web:
    enabled: false

  api:
    enabled: true
    image: aiagent-api:latest
    replicas: 3  # 生产环境3副本

    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

    # 自动伸缩
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70

    # 健康检查
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

# 2. 部署
helm upgrade --install aiagent-api \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f aiagent-api-values.yaml

# 3. 验证
kubectl rollout status deployment/aiagent-api -n aiagent-system
kubectl get pods -n aiagent-system -l app=aiagent-api -o wide

# 4. 测试API健康检查
curl http://NODE_IP:30093/health
```

---

### 第七步: Presidio 隐私保护服务

```bash
# 1. 配置文件
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

# 2. 部署
helm upgrade --install presidio \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f presidio-values.yaml

# 3. 验证
kubectl get pods -n aiagent-system -l app=presidio

# 4. 测试隐私检测
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=presidio -o jsonpath='{.items[0].metadata.name}') \
  -- curl -X POST http://localhost:3000/analyze \
  -H 'Content-Type: application/json' \
  -d '{"text":"My email is test@example.com","language":"en"}'
```

---

## 监控系统部署

### 第八步: Prometheus 监控

```bash
# 1. 配置文件
cat > prometheus-values.yaml <<EOF
monitoring:
  enabled: true

  prometheus:
    enabled: true
    image: prom/prometheus:latest
    retention: 30d  # 保留30天数据

    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2000m

    # 告警规则
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

# 2. 部署
helm upgrade --install prometheus \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f prometheus-values.yaml

# 3. 验证
kubectl get pods -n aiagent-system -l app=prometheus
kubectl get svc -n aiagent-system prometheus

# 4. 访问Prometheus UI
kubectl port-forward -n aiagent-system svc/prometheus 9090:9090
# 浏览器访问: http://localhost:9090
```

---

### 第九步: Grafana 可视化

```bash
# 1. 配置文件
cat > grafana-values.yaml <<EOF
monitoring:
  enabled: true

  prometheus:
    enabled: false

  grafana:
    enabled: true
    image: grafana/grafana:latest
    adminPassword: "YOUR_GRAFANA_PASSWORD"  # ⚠️ 修改密码

    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 1Gi
        cpu: 500m

    # 预装面板
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

# 2. 部署
helm upgrade --install grafana \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f grafana-values.yaml

# 3. 获取密码
kubectl get secret -n aiagent-system grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d

# 4. 访问Grafana UI
kubectl port-forward -n aiagent-system svc/grafana 3000:80
# 浏览器访问: http://localhost:3000
# 用户名: admin
# 密码: YOUR_GRAFANA_PASSWORD
```

#### 配置Grafana数据源

```bash
# 自动配置Prometheus数据源
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=grafana -o jsonpath='{.items[0].metadata.name}') \
  -- grafana-cli admin reset-admin-password YOUR_GRAFANA_PASSWORD
```

---

### 第十步: 数据库监控导出器

```bash
# 1. 配置文件
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

# 2. 部署
helm upgrade --install exporters \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f exporters-values.yaml

# 3. 验证
kubectl get pods -n aiagent-system | grep exporter
```

---

## 运维操作指南

### 1. 日常运维操作

#### 查看应用状态

```bash
# 快速状态检查
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh status

# 或使用kubectl
kubectl get all -n aiagent-system
kubectl get pods -n aiagent-system -o wide
kubectl top pods -n aiagent-system
kubectl top nodes
```

#### 查看日志

```bash
# 使用部署脚本的日志菜单
./aiagent-deploy.sh logs

# 手动查看日志
# Web应用日志
kubectl logs -f -l app=aiagent-web -n aiagent-system --tail=100

# API应用日志
kubectl logs -f -l app=aiagent-api -n aiagent-system --tail=100

# Redis日志
kubectl logs -f -l app=redis -n aiagent-system --tail=100

# 所有Pod日志
kubectl logs -f --all-containers -l app=aiagent-web -n aiagent-system
```

#### 进入容器调试

```bash
# 进入Web容器
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=aiagent-web -o jsonpath='{.items[0].metadata.name}') \
  -- /bin/bash

# 进入MySQL容器
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root -p

# 临时调试Pod
kubectl run debug --rm -it --image=busybox -n aiagent-system -- sh
```

---

### 2. 应用更新操作

#### 滚动更新应用

```bash
# 方式1: 使用Helm升级
helm upgrade aiagent-web \
  ../helm-charts/aiagent \
  -n aiagent-system \
  -f aiagent-web-values.yaml \
  --set aiagent.web.image=aiagent-web:v2.0

# 方式2: 直接更新镜像
kubectl set image deployment/aiagent-web \
  aiagent-web=aiagent-web:v2.0 \
  -n aiagent-system

# 查看滚动更新状态
kubectl rollout status deployment/aiagent-web -n aiagent-system

# 查看更新历史
kubectl rollout history deployment/aiagent-web -n aiagent-system
```

#### 回滚到上一版本

```bash
# 回滚到上一版本
kubectl rollout undo deployment/aiagent-web -n aiagent-system

# 回滚到指定版本
kubectl rollout undo deployment/aiagent-web \
  -n aiagent-system \
  --to-revision=2

# 验证回滚
kubectl rollout status deployment/aiagent-web -n aiagent-system
```

---

### 3. 扩缩容操作

#### 手动扩缩容

```bash
# 扩容到5个副本
kubectl scale deployment/aiagent-web \
  --replicas=5 \
  -n aiagent-system

# 缩容到2个副本
kubectl scale deployment/aiagent-web \
  --replicas=2 \
  -n aiagent-system

# 验证
kubectl get pods -n aiagent-system -l app=aiagent-web
```

#### 配置HPA自动伸缩

```bash
# 创建HPA
kubectl autoscale deployment aiagent-web \
  --cpu-percent=70 \
  --min=3 \
  --max=10 \
  -n aiagent-system

# 查看HPA状态
kubectl get hpa -n aiagent-system
kubectl describe hpa aiagent-web -n aiagent-system

# 删除HPA
kubectl delete hpa aiagent-web -n aiagent-system
```

---

### 4. 备份和恢复

#### 备份数据库

```bash
# MySQL备份
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysqldump -u root -pYOUR_ROOT_PASSWORD aiagent > aiagent_backup_$(date +%Y%m%d).sql

# MongoDB备份
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongodump --uri="mongodb://aiagent:YOUR_PASSWORD@localhost:27017/aiagent" \
  --out=/tmp/backup

# 复制备份文件到本地
kubectl cp aiagent-system/$(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}'):/tmp/backup \
  ./mongodb_backup_$(date +%Y%m%d)
```

#### 恢复数据库

```bash
# MySQL恢复
kubectl exec -i -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root -pYOUR_ROOT_PASSWORD aiagent < aiagent_backup_20250122.sql

# MongoDB恢复
kubectl cp ./mongodb_backup_20250122 \
  aiagent-system/$(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}'):/tmp/restore

kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongorestore --uri="mongodb://aiagent:YOUR_PASSWORD@localhost:27017/aiagent" \
  /tmp/restore/aiagent
```

---

### 5. 配置管理

#### 更新ConfigMap

```bash
# 查看现有ConfigMap
kubectl get configmap -n aiagent-system

# 编辑ConfigMap
kubectl edit configmap aiagent-web-config -n aiagent-system

# 重启应用使配置生效
kubectl rollout restart deployment/aiagent-web -n aiagent-system
```

#### 更新Secret

```bash
# 创建新的Secret
kubectl create secret generic aiagent-web-secret \
  --from-literal=db-password=NEW_PASSWORD \
  -n aiagent-system \
  --dry-run=client -o yaml | kubectl apply -f -

# 重启应用
kubectl rollout restart deployment/aiagent-web -n aiagent-system
```

---

### 6. 资源清理

#### 删除单个组件

```bash
# 删除Deployment
kubectl delete deployment aiagent-web -n aiagent-system

# 删除Service
kubectl delete svc aiagent-web -n aiagent-system

# 删除PVC (⚠️ 会删除数据)
kubectl delete pvc redis-data -n aiagent-system
```

#### 完全卸载

```bash
# 使用部署脚本卸载
cd aiagent-k3s-dev/scripts
./aiagent-deploy.sh undeploy

# 手动卸载所有Helm Release
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

# 删除命名空间 (⚠️ 会删除所有资源)
kubectl delete namespace aiagent-system
```

---

## 故障排查手册

### 常见问题诊断

#### Pod无法启动

```bash
# 1. 查看Pod状态
kubectl get pods -n aiagent-system -o wide

# 2. 查看Pod详细信息
kubectl describe pod POD_NAME -n aiagent-system

# 3. 查看事件
kubectl get events -n aiagent-system --sort-by='.lastTimestamp' | tail -20

# 4. 常见原因排查:

# 镜像拉取失败
kubectl describe pod POD_NAME -n aiagent-system | grep -A 10 "Events:"
# 解决: 检查镜像名称、imagePullPolicy设置

# 资源不足
kubectl describe nodes | grep -A 5 "Allocated resources"
# 解决: 增加节点或调整资源limits

# 配置错误
kubectl logs POD_NAME -n aiagent-system
# 解决: 检查ConfigMap、Secret配置
```

#### 应用无法访问

```bash
# 1. 检查Service
kubectl get svc -n aiagent-system
kubectl describe svc aiagent-web -n aiagent-system

# 2. 检查Endpoints
kubectl get endpoints aiagent-web -n aiagent-system

# 3. 测试ClusterIP访问
kubectl run curl-test --rm -it --image=curlimages/curl -n aiagent-system \
  -- curl http://aiagent-web:8080/actuator/health

# 4. 测试NodePort访问
curl http://NODE_IP:30080
```

#### 数据库连接失败

```bash
# 1. 检查数据库Pod
kubectl get pods -n aiagent-system -l app=mysql

# 2. 测试数据库连接
kubectl exec -it -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u aiagent -p

# 3. 检查应用日志中的连接错误
kubectl logs -f -l app=aiagent-web -n aiagent-system | grep -i "connection\|mysql"

# 4. 验证Service DNS
kubectl run dns-test --rm -it --image=busybox -n aiagent-system \
  -- nslookup mysql.aiagent-system.svc.cluster.local
```

#### 存储问题排查

```bash
# 1. 查看PVC状态
kubectl get pvc -n aiagent-system

# 2. 查看PV状态
kubectl get pv

# 3. PVC Pending排查
kubectl describe pvc PVC_NAME -n aiagent-system

# 4. 检查StorageClass
kubectl get sc
kubectl describe sc longhorn

# 5. 检查磁盘空间
df -h /opt/aiagent-k3s/data/
```

---

### 性能问题排查

#### CPU/内存使用率高

```bash
# 1. 查看资源使用
kubectl top pods -n aiagent-system
kubectl top nodes

# 2. 查看资源限制
kubectl describe pod POD_NAME -n aiagent-system | grep -A 5 "Limits\|Requests"

# 3. 查看HPA状态
kubectl get hpa -n aiagent-system

# 4. 临时扩容
kubectl scale deployment/aiagent-web --replicas=5 -n aiagent-system
```

#### 数据库性能问题

```bash
# MySQL慢查询
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mysql -u root -p -e "SHOW FULL PROCESSLIST;"

# MongoDB性能统计
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=mongodb -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u aiagent -p --authenticationDatabase aiagent \
  --eval "db.serverStatus()"

# Elasticsearch性能
kubectl exec -n aiagent-system \
  $(kubectl get pod -n aiagent-system -l app=elasticsearch -o jsonpath='{.items[0].metadata.name}') \
  -- curl -s -u elastic:PASSWORD -k https://localhost:9200/_nodes/stats/jvm,thread_pool
```

---

### 监控告警

#### 配置告警规则

```bash
# 创建告警规则ConfigMap
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
      # Pod重启告警
      - alert: PodRestarting
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ \$labels.pod }} 频繁重启"

      # 内存使用告警
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ \$labels.pod }} 内存使用超过90%"

      # 数据库连接数告警
      - alert: MySQLHighConnections
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL连接数超过80%"
EOF
```

---

### 快速运维命令参考

```bash
# ============ 查看状态 ============
kubectl get all -n aiagent-system
kubectl get pods -n aiagent-system -o wide
kubectl top pods -n aiagent-system

# ============ 查看日志 ============
kubectl logs -f -l app=aiagent-web -n aiagent-system --tail=100
kubectl logs --previous POD_NAME -n aiagent-system

# ============ 故障排查 ============
kubectl describe pod POD_NAME -n aiagent-system
kubectl get events -n aiagent-system --sort-by='.lastTimestamp'

# ============ 进入容器 ============
kubectl exec -it POD_NAME -n aiagent-system -- /bin/bash

# ============ 更新应用 ============
kubectl set image deployment/aiagent-web aiagent-web=new-image:tag -n aiagent-system
kubectl rollout status deployment/aiagent-web -n aiagent-system

# ============ 回滚应用 ============
kubectl rollout undo deployment/aiagent-web -n aiagent-system

# ============ 扩缩容 ============
kubectl scale deployment/aiagent-web --replicas=5 -n aiagent-system

# ============ 重启应用 ============
kubectl rollout restart deployment/aiagent-web -n aiagent-system

# ============ 删除Pod (自动重建) ============
kubectl delete pod POD_NAME -n aiagent-system
```

---

## 附录

### A. 完整部署检查清单

#### 基础组件检查
- [ ] Redis Pod Running,可连接
- [ ] MongoDB Pod Running,可连接,已初始化数据库
- [ ] MySQL Pod Running,可连接,已创建表结构,只读用户正常
- [ ] Elasticsearch Pod Running,已创建索引 (96个),ik分词器正常

#### 应用层检查
- [ ] AIAgent Web Pod Running (3副本),健康检查通过
- [ ] AIAgent API Pod Running (3副本),健康检查通过
- [ ] Presidio Pod Running,隐私检测功能正常
- [ ] License文件已处理 (如有)

#### 监控系统检查
- [ ] Prometheus Pod Running,数据采集正常
- [ ] Grafana Pod Running,可登录,已配置数据源
- [ ] Database Exporters Pod Running,指标正常

#### 访问验证
- [ ] Web UI可访问: http://NODE_IP:30080
- [ ] API可访问: http://NODE_IP:30093/health
- [ ] Prometheus可访问: http://NODE_IP:30090
- [ ] Grafana可访问: http://NODE_IP:30300

#### 存储验证
- [ ] 所有PVC状态为Bound
- [ ] 数据目录权限正确
- [ ] 数据持久化正常 (重启Pod后数据不丢失)

---

### B. 分类部署脚本模板

创建一个分类部署的便捷脚本:

```bash
cat > deploy-by-category.sh <<'EOF'
#!/bin/bash

# AIAgent 分类部署脚本

set -e

NAMESPACE="aiagent-system"
CHART_DIR="../helm-charts/aiagent"

log_info() {
    echo -e "\033[0;36m[INFO]\033[0m $1"
}

log_success() {
    echo -e "\033[0;32m[SUCCESS]\033[0m $1"
}

# 部署Redis
deploy_redis() {
    log_info "部署 Redis..."
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
    log_success "Redis 部署完成"
}

# 部署MongoDB
deploy_mongodb() {
    log_info "部署 MongoDB..."
    # ... (类似Redis配置)
}

# 部署MySQL
deploy_mysql() {
    log_info "部署 MySQL..."
    # ... (类似Redis配置)
}

# 部署Elasticsearch
deploy_elasticsearch() {
    log_info "部署 Elasticsearch..."
    # ... (类似Redis配置)
}

# 主菜单
show_menu() {
    echo "========================================"
    echo "AIAgent 分类部署菜单"
    echo "========================================"
    echo "1. 部署基础组件 (Redis)"
    echo "2. 部署基础组件 (MongoDB)"
    echo "3. 部署基础组件 (MySQL)"
    echo "4. 部署基础组件 (Elasticsearch)"
    echo "5. 部署所有基础组件"
    echo "6. 部署AIAgent Web"
    echo "7. 部署AIAgent API"
    echo "8. 部署Presidio"
    echo "9. 部署监控系统"
    echo "0. 退出"
    echo "========================================"
}

# 主循环
while true; do
    show_menu
    read -p "请选择操作 [0-9]: " choice

    case $choice in
        1) deploy_redis ;;
        2) deploy_mongodb ;;
        3) deploy_mysql ;;
        4) deploy_elasticsearch ;;
        5) deploy_redis; deploy_mongodb; deploy_mysql; deploy_elasticsearch ;;
        0) exit 0 ;;
        *) echo "无效选择" ;;
    esac

    echo ""
    read -p "按Enter继续..."
done
EOF

chmod +x deploy-by-category.sh
```

---

**文档版本**: v1.0
**最后更新**: 2025-01-22
**维护者**: DevOps Team
