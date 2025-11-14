# K8s 微服務完整部署實戰案例

> 基於 K8s 基礎設施的 Python 微服務完整部署示例
> 
> 包含：應用開發 → 容器化 → CI/CD → K8s 部署 → 監控日誌

---

## 📋 目錄

- [案例概述](#案例概述)
- [應用架構](#應用架構)
- [1. 應用開發](#1-應用開發)
- [2. 容器化](#2-容器化)
- [3. K8s 資源配置](#3-k8s-資源配置)
- [4. CI/CD 流程](#4-cicd-流程)
- [5. 部署到 K8s](#5-部署到-k8s)
- [6. 驗證與測試](#6-驗證與測試)
- [7. 監控與日誌](#7-監控與日誌)
- [完整工作流程](#完整工作流程)
- [故障排查](#故障排查)

---

## 案例概述

### 應用說明

我們將構建一個 **圖片上傳與處理服務**：

- 📸 用戶上傳圖片到 MinIO
- 🔄 發送處理任務到 Kafka
- 💾 處理結果存入 Redis 緩存
- 📊 暴露 Prometheus metrics
- 📝 結構化日誌輸出到 ELK

### 使用的基礎設施組件

| 組件 | 用途 |
|------|------|
| **MinIO** | 存儲上傳的圖片 |
| **Redis** | 緩存處理結果 |
| **Kafka** | 異步任務隊列 |
| **Longhorn** | 持久化存儲 |
| **Harbor** | 存儲 Docker 鏡像 |
| **ArgoCD** | GitOps 部署 |
| **Prometheus** | 監控指標 |
| **ELK** | 日誌收集 |
| **Traefik** | 入口流量 |
| **cert-manager** | HTTPS 證書 |

---

## 應用架構

```mermaid
graph TB
    User[用戶]
    Traefik[Traefik Ingress]
    API[API Service<br/>Flask App]
    Worker[Worker Service<br/>後台處理]
    MinIO[MinIO<br/>圖片存儲]
    Redis[Redis<br/>結果緩存]
    Kafka[Kafka<br/>任務隊列]
    
    subgraph Monitoring[監控層]
        Prometheus[Prometheus<br/>指標收集]
        ELK[ELK<br/>日誌收集]
    end
    
    User -->|HTTPS| Traefik
    Traefik --> API
    API -->|上傳圖片| MinIO
    API -->|發送任務| Kafka
    API -->|讀取緩存| Redis
    Kafka -->|消費任務| Worker
    Worker -->|處理圖片| MinIO
    Worker -->|存儲結果| Redis
    API -.metrics.-> Prometheus
    Worker -.metrics.-> Prometheus
    API -.logs.-> ELK
    Worker -.logs.-> ELK
```

---

## 1. 應用開發

### 項目結構

```
image-service/
├── api/                      # API 服務
│   ├── app.py               # Flask 應用
│   ├── requirements.txt
│   └── Dockerfile
├── worker/                   # Worker 服務
│   ├── worker.py
│   ├── requirements.txt
│   └── Dockerfile
├── k8s/                      # K8s 配置
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   ├── configmap.yaml
│   │   └── secret.yaml
│   └── overlays/
│       ├── dev/
│       └── prod/
├── .gitlab-ci.yml           # GitLab CI/CD
└── README.md
```

### API 服務代碼 (api/app.py)

> k8s service domain name 規則
> ` <service-name>.<namespace>.svc.cluster.local`


```python
from flask import Flask, request, jsonify
from minio import Minio
from kafka import KafkaProducer
import redis
import json
import logging
import structlog
from prometheus_client import Counter, Histogram, generate_latest
import time
import uuid

# 結構化日誌配置
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)
logger = structlog.get_logger()

app = Flask(__name__)

# Prometheus 指標
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'HTTP request duration')
UPLOAD_COUNT = Counter('image_uploads_total', 'Total image uploads')

# MinIO 客戶端
minio_client = Minio(
    "minio.minio.svc.cluster.local:9000",
    access_key="admin",
    secret_key="YourStrongPassword123!",
    secure=False
)

# Redis 客戶端
redis_client = redis.Redis(
    host='redis-master.redis.svc.cluster.local',
    port=6379,
    password='YourRedisPassword123!',
    decode_responses=True
)

# Kafka 生產者
kafka_producer = KafkaProducer(
    bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# 確保 bucket 存在
BUCKET_NAME = "images"
if not minio_client.bucket_exists(BUCKET_NAME):
    minio_client.make_bucket(BUCKET_NAME)
    logger.info("bucket_created", bucket=BUCKET_NAME)

@app.route('/health')
def health():
    """健康檢查"""
    return jsonify({"status": "healthy"}), 200

@app.route('/ready')
def ready():
    """就緒檢查"""
    try:
        # 檢查 Redis 連接
        redis_client.ping()
        # 檢查 MinIO 連接
        minio_client.bucket_exists(BUCKET_NAME)
        return jsonify({"status": "ready"}), 200
    except Exception as e:
        logger.error("readiness_check_failed", error=str(e))
        return jsonify({"status": "not ready", "error": str(e)}), 503

@app.route('/metrics')
def metrics():
    """Prometheus metrics 端點"""
    return generate_latest()

@app.route('/upload', methods=['POST'])
@REQUEST_DURATION.time()
def upload_image():
    """上傳圖片"""
    start_time = time.time()
    
    try:
        # 檢查文件
        if 'file' not in request.files:
            REQUEST_COUNT.labels(method='POST', endpoint='/upload', status='400').inc()
            return jsonify({"error": "No file provided"}), 400
        
        file = request.files['file']
        if file.filename == '':
            REQUEST_COUNT.labels(method='POST', endpoint='/upload', status='400').inc()
            return jsonify({"error": "No file selected"}), 400
        
        # 生成唯一 ID
        image_id = str(uuid.uuid4())
        object_name = f"{image_id}/{file.filename}"
        
        # 上傳到 MinIO
        minio_client.put_object(
            BUCKET_NAME,
            object_name,
            file.stream,
            length=-1,
            part_size=10*1024*1024,
            content_type=file.content_type
        )
        
        UPLOAD_COUNT.inc()
        logger.info("image_uploaded", 
                   image_id=image_id, 
                   filename=file.filename,
                   size=file.content_length)
        
        # 發送處理任務到 Kafka
        task = {
            "image_id": image_id,
            "object_name": object_name,
            "filename": file.filename,
            "timestamp": time.time()
        }
        kafka_producer.send('image-processing', task)
        kafka_producer.flush()
        
        logger.info("task_sent_to_kafka", task=task)
        
        REQUEST_COUNT.labels(method='POST', endpoint='/upload', status='200').inc()
        
        duration = time.time() - start_time
        return jsonify({
            "message": "Image uploaded successfully",
            "image_id": image_id,
            "duration": f"{duration:.2f}s"
        }), 200
        
    except Exception as e:
        logger.error("upload_failed", error=str(e))
        REQUEST_COUNT.labels(method='POST', endpoint='/upload', status='500').inc()
        return jsonify({"error": str(e)}), 500

@app.route('/status/<image_id>', methods=['GET'])
def get_status(image_id):
    """查詢處理狀態"""
    try:
        # 從 Redis 緩存讀取
        cache_key = f"image:status:{image_id}"
        status = redis_client.get(cache_key)
        
        if status:
            logger.info("status_cache_hit", image_id=image_id)
            REQUEST_COUNT.labels(method='GET', endpoint='/status', status='200').inc()
            return jsonify(json.loads(status)), 200
        else:
            logger.info("status_not_found", image_id=image_id)
            REQUEST_COUNT.labels(method='GET', endpoint='/status', status='404').inc()
            return jsonify({"error": "Status not found"}), 404
            
    except Exception as e:
        logger.error("status_query_failed", error=str(e))
        REQUEST_COUNT.labels(method='GET', endpoint='/status', status='500').inc()
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    logger.info("api_service_starting", port=8080)
    app.run(host='0.0.0.0', port=8080)
```

### Worker 服務代碼 (worker/worker.py)

```python
from kafka import KafkaConsumer
from minio import Minio
import redis
import json
import structlog
from prometheus_client import Counter, Gauge, start_http_server
import time
from PIL import Image
import io

# 結構化日誌
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)
logger = structlog.get_logger()

# Prometheus 指標
TASKS_PROCESSED = Counter('tasks_processed_total', 'Total tasks processed', ['status'])
PROCESSING_DURATION = Gauge('task_processing_duration_seconds', 'Task processing duration')
ACTIVE_TASKS = Gauge('active_tasks', 'Currently active tasks')

# MinIO 客戶端
minio_client = Minio(
    "minio.minio.svc.cluster.local:9000",
    access_key="admin",
    secret_key="YourStrongPassword123!",
    secure=False
)

# Redis 客戶端
redis_client = redis.Redis(
    host='redis-master.redis.svc.cluster.local',
    port=6379,
    password='YourRedisPassword123!',
    decode_responses=True
)

# Kafka 消費者
consumer = KafkaConsumer(
    'image-processing',
    bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092',
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    group_id='image-processor-group',
    auto_offset_reset='earliest'
)

def process_image(object_name):
    """處理圖片（生成縮略圖）"""
    try:
        # 從 MinIO 下載圖片
        response = minio_client.get_object("images", object_name)
        image_data = response.read()
        
        # 使用 PIL 處理圖片
        image = Image.open(io.BytesIO(image_data))
        
        # 生成縮略圖
        thumbnail = image.copy()
        thumbnail.thumbnail((200, 200))
        
        # 保存縮略圖到 MinIO
        thumbnail_buffer = io.BytesIO()
        thumbnail.save(thumbnail_buffer, format=image.format)
        thumbnail_buffer.seek(0)
        
        thumbnail_name = object_name.replace('/', '/thumbnail_')
        minio_client.put_object(
            "images",
            thumbnail_name,
            thumbnail_buffer,
            length=thumbnail_buffer.getbuffer().nbytes,
            content_type='image/jpeg'
        )
        
        return {
            "success": True,
            "thumbnail": thumbnail_name,
            "original_size": image.size,
            "thumbnail_size": thumbnail.size
        }
        
    except Exception as e:
        logger.error("image_processing_failed", error=str(e))
        return {"success": False, "error": str(e)}

def main():
    """主處理循環"""
    logger.info("worker_starting")
    
    # 啟動 Prometheus metrics 服務器
    start_http_server(8081)
    logger.info("metrics_server_started", port=8081)
    
    logger.info("waiting_for_tasks")
    
    for message in consumer:
        ACTIVE_TASKS.inc()
        start_time = time.time()
        
        task = message.value
        image_id = task['image_id']
        object_name = task['object_name']
        
        logger.info("task_received", task=task)
        
        try:
            # 處理圖片
            result = process_image(object_name)
            
            # 保存結果到 Redis
            cache_key = f"image:status:{image_id}"
            status_data = {
                "image_id": image_id,
                "status": "completed" if result["success"] else "failed",
                "result": result,
                "processed_at": time.time()
            }
            redis_client.setex(cache_key, 3600, json.dumps(status_data))  # 1小時過期
            
            duration = time.time() - start_time
            PROCESSING_DURATION.set(duration)
            
            if result["success"]:
                TASKS_PROCESSED.labels(status='success').inc()
                logger.info("task_completed", 
                           image_id=image_id, 
                           duration=duration,
                           result=result)
            else:
                TASKS_PROCESSED.labels(status='failed').inc()
                logger.error("task_failed", image_id=image_id, error=result.get("error"))
                
        except Exception as e:
            logger.error("task_processing_error", image_id=image_id, error=str(e))
            TASKS_PROCESSED.labels(status='error').inc()
        
        finally:
            ACTIVE_TASKS.dec()

if __name__ == '__main__':
    main()
```

### 依賴文件

**api/requirements.txt**
```txt
Flask==3.0.0
minio==7.2.0
kafka-python==2.0.2
redis==5.0.1
prometheus-client==0.19.0
structlog==23.2.0
```

**worker/requirements.txt**
```txt
kafka-python==2.0.2
minio==7.2.0
redis==5.0.1
prometheus-client==0.19.0
structlog==23.2.0
Pillow==10.1.0
```

---

## 2. 容器化

### API Dockerfile (api/Dockerfile)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安裝依賴
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 複製應用代碼
COPY app.py .

# 非 root 用戶運行
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8080/health')"

EXPOSE 8080

CMD ["python", "app.py"]
```

### Worker Dockerfile (worker/Dockerfile)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安裝系統依賴（PIL 需要）
RUN apt-get update && apt-get install -y \
    libjpeg-dev \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

# 安裝 Python 依賴
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 複製應用代碼
COPY worker.py .

# 非 root 用戶
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8081

CMD ["python", "worker.py"]
```

---

## 3. K8s 資源配置

### ConfigMap (k8s/base/configmap.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: image-service-config
data:
  MINIO_ENDPOINT: "minio.minio.svc.cluster.local:9000"
  MINIO_BUCKET: "images"
  REDIS_HOST: "redis-master.redis.svc.cluster.local"
  REDIS_PORT: "6379"
  KAFKA_BOOTSTRAP_SERVERS: "my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092"
  KAFKA_TOPIC: "image-processing"
  LOG_LEVEL: "INFO"
```

### Secret (k8s/base/secret.yaml)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: image-service-secret
type: Opaque
stringData:
  MINIO_ACCESS_KEY: "admin"
  MINIO_SECRET_KEY: "YourStrongPassword123!"
  REDIS_PASSWORD: "YourRedisPassword123!"
```

### API Deployment (k8s/base/api-deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-api
  labels:
    app: image-api
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: image-api
  template:
    metadata:
      labels:
        app: image-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      imagePullSecrets:
      - name: harbor-secret
      
      containers:
      - name: api
        image: harbor.yourdomain.com/image-service/api:latest
        ports:
        - containerPort: 8080
          name: http
        
        envFrom:
        - configMapRef:
            name: image-service-config
        - secretRef:
            name: image-service-secret
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

### Worker Deployment (k8s/base/worker-deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-worker
  labels:
    app: image-worker
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: image-worker
  template:
    metadata:
      labels:
        app: image-worker
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/metrics"
    spec:
      imagePullSecrets:
      - name: harbor-secret
      
      containers:
      - name: worker
        image: harbor.yourdomain.com/image-service/worker:latest
        ports:
        - containerPort: 8081
          name: metrics
        
        envFrom:
        - configMapRef:
            name: image-service-config
        - secretRef:
            name: image-service-secret
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

### Service (k8s/base/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: image-api
  labels:
    app: image-api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: image-api
---
apiVersion: v1
kind: Service
metadata:
  name: image-worker
  labels:
    app: image-worker
spec:
  type: ClusterIP
  ports:
  - port: 8081
    targetPort: 8081
    protocol: TCP
    name: metrics
  selector:
    app: image-worker
```

### Ingress (k8s/base/ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: image-api-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  tls:
  - hosts:
    - api.yourdomain.com
    secretName: image-api-tls
  rules:
  - host: api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: image-api
            port:
              number: 80
```

### HPA (k8s/base/hpa.yaml)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: image-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: image-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 4. CI/CD 流程

### GitLab CI/CD (.gitlab-ci.yml)

```yaml
stages:
  - test
  - build
  - deploy

variables:
  HARBOR_REGISTRY: harbor.yourdomain.com
  HARBOR_PROJECT: image-service
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

# 測試階段
test:
  stage: test
  image: python:3.11-slim
  script:
    - cd api
    - pip install -r requirements.txt
    - python -m pytest tests/
  only:
    - merge_requests
    - main

# 構建 API 鏡像
build-api:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  before_script:
    - docker login $HARBOR_REGISTRY -u $HARBOR_USER -p $HARBOR_PASSWORD
  script:
    - cd api
    - docker build -t $HARBOR_REGISTRY/$HARBOR_PROJECT/api:$IMAGE_TAG .
    - docker tag $HARBOR_REGISTRY/$HARBOR_PROJECT/api:$IMAGE_TAG $HARBOR_REGISTRY/$HARBOR_PROJECT/api:latest
    - docker push $HARBOR_REGISTRY/$HARBOR_PROJECT/api:$IMAGE_TAG
    - docker push $HARBOR_REGISTRY/$HARBOR_PROJECT/api:latest
  only:
    - main

# 構建 Worker 鏡像
build-worker:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  before_script:
    - docker login $HARBOR_REGISTRY -u $HARBOR_USER -p $HARBOR_PASSWORD
  script:
    - cd worker
    - docker build -t $HARBOR_REGISTRY/$HARBOR_PROJECT/worker:$IMAGE_TAG .
    - docker tag $HARBOR_REGISTRY/$HARBOR_PROJECT/worker:$IMAGE_TAG $HARBOR_REGISTRY/$HARBOR_PROJECT/worker:latest
    - docker push $HARBOR_REGISTRY/$HARBOR_PROJECT/worker:$IMAGE_TAG
    - docker push $HARBOR_REGISTRY/$HARBOR_PROJECT/worker:latest
  only:
    - main

# 更新 K8s manifest
update-manifests:
  stage: deploy
  image: alpine/git
  script:
    - git config --global user.email "ci@example.com"
    - git config --global user.name "GitLab CI"
    - git clone https://oauth2:${GITLAB_TOKEN}@gitlab.com/yourorg/k8s-manifests.git
    - cd k8s-manifests/image-service
    - sed -i "s|image:.*api:.*|image: $HARBOR_REGISTRY/$HARBOR_PROJECT/api:$IMAGE_TAG|g" deployment.yaml
    - sed -i "s|image:.*worker:.*|image: $HARBOR_REGISTRY/$HARBOR_PROJECT/worker:$IMAGE_TAG|g" deployment.yaml
    - git add deployment.yaml
    - git commit -m "Update image to $IMAGE_TAG"
    - git push origin main
  only:
    - main
```

---

## 5. 部署到 K8s

### 使用 kubectl 直接部署

```bash
# 1. 創建命名空間
kubectl create namespace image-service

# 2. 創建 Harbor Secret
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.yourdomain.com \
  --docker-username=admin \
  --docker-password=YourPassword \
  -n image-service

# 3. 應用配置
kubectl apply -f k8s/base/ -n image-service

# 4. 查看部署狀態
kubectl get pods -n image-service -w

# 5. 查看服務
kubectl get svc -n image-service

# 6. 查看 Ingress
kubectl get ingress -n image-service
```

### 使用 ArgoCD 部署

```bash
# 1. 創建 ArgoCD Application
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: image-service
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/yourorg/k8s-manifests.git
    targetRevision: main
    path: image-service
  
  destination:
    server: https://kubernetes.default.svc
    namespace: image-service
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

# 2. 查看同步狀態
kubectl get application -n argocd image-service

# 3. 查看詳細信息
kubectl describe application -n argocd image-service
```

---

## 6. 驗證與測試

### 測試 API

```bash
# 1. Port Forward（如果沒有 Ingress）
kubectl port-forward -n image-service svc/image-api 8080:80

# 2. 健康檢查
curl http://localhost:8080/health

# 3. 上傳圖片
curl -X POST http://localhost:8080/upload \
  -F "file=@test-image.jpg"

# 輸出示例:
# {
#   "message": "Image uploaded successfully",
#   "image_id": "550e8400-e29b-41d4-a716-446655440000",
#   "duration": "0.52s"
# }

# 4. 查詢處理狀態
curl http://localhost:8080/status/550e8400-e29b-41d4-a716-446655440000

# 輸出示例:
# {
#   "image_id": "550e8400-e29b-41d4-a716-446655440000",
#   "status": "completed",
#   "result": {
#     "success": true,
#     "thumbnail": "550e8400.../thumbnail_test-image.jpg",
#     "original_size": [1920, 1080],
#     "thumbnail_size": [200, 112]
#   },
#   "processed_at": 1699999999.123
# }
```

---

## 7. 監控與日誌

### Prometheus 監控

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: image-service
  namespace: image-service
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: image-api
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

**常用 PromQL 查詢**

```promql
# API 請求速率
rate(http_requests_total[5m])

# 上傳成功率
rate(image_uploads_total[5m])

# Worker 處理速率
rate(tasks_processed_total[5m])

# Pod 內存使用
container_memory_usage_bytes{pod=~"image-.*"}
```

### ELK 日誌查詢

```kql
# 查看 API 日誌
kubernetes.namespace: "image-service" AND kubernetes.labels.app: "image-api"

# 查看錯誤日誌
kubernetes.namespace: "image-service" AND level: "error"
```

---

## 完整工作流程

```mermaid
flowchart TD
    A[開發者提交代碼] --> B[GitLab CI 觸發]
    B --> C[運行測試]
    C -->|測試通過| D[構建 Docker 鏡像]
    C -->|測試失敗| Z[通知開發者]
    D --> E[推送到 Harbor]
    E --> F[掃描鏡像漏洞]
    F -->|安全| G[更新 K8s manifest]
    F -->|有漏洞| Z
    G --> H[Git push manifest 倉庫]
    H --> I[ArgoCD 檢測變更]
    I --> J[ArgoCD 自動同步]
    J --> K[部署到 K8s]
    K --> L[健康檢查]
    L -->|通過| M[流量切換]
    L -->|失敗| N[自動回滾]
    M --> O[Prometheus 收集指標]
    M --> P[ELK 收集日誌]
    N --> Z
```

---

## 故障排查

### 常見問題

#### 1. Pod 無法啟動

```bash
# 查看 Pod 狀態
kubectl get pods -n image-service

# 查看詳細信息
kubectl describe pod <pod-name> -n image-service

# 查看日誌
kubectl logs <pod-name> -n image-service
```

#### 2. 無法連接中間件

```bash
# 測試網絡連通性
kubectl run tmp-shell --rm -i --tty \
  --image nicolaka/netshoot \
  -n image-service -- /bin/bash

# 測試 DNS
nslookup minio.minio.svc.cluster.local
nslookup redis-master.redis.svc.cluster.local
```

---

## 總結

### 完成的工作

✅ **應用開發**
- Flask API 服務（上傳、查詢）
- Kafka Worker 服務（異步處理）
- 結構化日誌輸出
- Prometheus metrics 暴露

✅ **容器化**
- Dockerfile 優化
- 健康檢查
- 非 root 用戶

✅ **K8s 配置**
- Deployment、Service、Ingress
- ConfigMap、Secret
- HPA 自動擴縮容

✅ **CI/CD**
- GitLab CI 自動化
- Harbor 鏡像管理
- ArgoCD GitOps 部署

✅ **可觀測性**
- Prometheus 監控
- Grafana 可視化
- ELK 日誌收集

### 架構優勢

1. **高可用**: 多副本 + HPA + 健康檢查
2. **可擴展**: 水平擴容 + Kafka 解耦
3. **可觀測**: 完整的監控和日誌
4. **自動化**: GitOps + CI/CD
5. **安全性**: HTTPS + RBAC + 鏡像掃描

---

**最後更新：2024年**
