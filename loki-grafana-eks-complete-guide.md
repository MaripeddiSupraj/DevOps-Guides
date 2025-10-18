# Loki-Grafana on EKS: Complete Production Guide

## 🎯 Overview

This guide provides a complete implementation of Loki-Grafana logging stack on EKS with a real e-commerce application POC. You'll learn centralized logging, log aggregation, and observability for microservices.

## 📋 What You'll Build

- **EKS Cluster** with Loki-Grafana stack
- **E-commerce Microservices** (Frontend, API, Database)
- **Centralized Logging** with structured logs
- **Real-time Dashboards** with alerts
- **Production-grade Configuration** with persistence and security

## 🏗️ Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   API Service   │    │   Database      │
│   (React)       │────│   (Node.js)     │────│   (MongoDB)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   Promtail      │
                    │   (Log Agent)   │
                    └─────────────────┘
                                 │
                    ┌─────────────────┐
                    │      Loki       │
                    │  (Log Storage)  │
                    └─────────────────┘
                                 │
                    ┌─────────────────┐
                    │    Grafana      │
                    │ (Visualization) │
                    └─────────────────┘
```

## 🚀 Prerequisites

```bash
# Required tools
aws --version
kubectl version --client
eksctl version
helm version
```

## 📦 Step 1: EKS Cluster Setup

### 1.1 Create EKS Cluster

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: loki-demo-cluster
  region: us-west-2
  version: "1.28"

nodeGroups:
  - name: worker-nodes
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 2
    maxSize: 5
    volumeSize: 20
    ssh:
      allow: true

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
```

```bash
# Create cluster
eksctl create cluster -f cluster-config.yaml

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

### 1.2 Install EBS CSI Driver

```bash
# Create IAM role for EBS CSI
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster loki-demo-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/Amazon_EBS_CSI_DriverPolicy \
  --approve \
  --override-existing-serviceaccounts

# Install EBS CSI driver
eksctl create addon --name aws-ebs-csi-driver --cluster loki-demo-cluster --service-account-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/eksctl-loki-demo-cluster-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa-Role1-*
```

## 📊 Step 2: Install Loki Stack

### 2.1 Add Grafana Helm Repository

```bash
# Add Grafana helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring
```

### 2.2 Loki Configuration

```yaml
# loki-values.yaml
loki:
  auth_enabled: false
  
  server:
    http_listen_port: 3100
    
  ingester:
    lifecycler:
      address: 127.0.0.1
      ring:
        kvstore:
          store: inmemory
        replication_factor: 1
    chunk_idle_period: 5m
    chunk_retain_period: 30s
    
  schema_config:
    configs:
    - from: 2020-05-15
      store: boltdb-shipper
      object_store: s3
      schema: v11
      index:
        prefix: index_
        period: 24h
        
  storage_config:
    boltdb_shipper:
      active_index_directory: /loki/index
      cache_location: /loki/index_cache
      shared_store: s3
    aws:
      s3: s3://loki-storage-bucket-$(aws sts get-caller-identity --query Account --output text)
      region: us-west-2

  limits_config:
    enforce_metric_name: false
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    
persistence:
  enabled: true
  size: 10Gi
  storageClassName: gp2

serviceMonitor:
  enabled: true
```

### 2.3 Create S3 Bucket for Loki

```bash
# Create S3 bucket
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3 mb s3://loki-storage-bucket-${ACCOUNT_ID} --region us-west-2

# Create IAM policy for Loki S3 access
cat > loki-s3-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::loki-storage-bucket-${ACCOUNT_ID}",
                "arn:aws:s3:::loki-storage-bucket-${ACCOUNT_ID}/*"
            ]
        }
    ]
}
EOF

aws iam create-policy --policy-name LokiS3Policy --policy-document file://loki-s3-policy.json

# Create service account with IAM role
eksctl create iamserviceaccount \
  --name loki \
  --namespace monitoring \
  --cluster loki-demo-cluster \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/LokiS3Policy \
  --approve
```

### 2.4 Install Loki

```bash
# Install Loki
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values loki-values.yaml \
  --set loki.serviceAccount.create=false \
  --set loki.serviceAccount.name=loki
```

## 📈 Step 3: Install Grafana

### 3.1 Grafana Configuration

```yaml
# grafana-values.yaml
persistence:
  enabled: true
  size: 10Gi
  storageClassName: gp2

adminPassword: "admin123"

service:
  type: LoadBalancer

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      isDefault: true

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards/default

dashboards:
  default:
    loki-dashboard:
      gnetId: 13639
      revision: 2
      datasource: Loki

plugins:
  - grafana-piechart-panel
  - grafana-worldmap-panel

env:
  GF_EXPLORE_ENABLED: true
  GF_PANELS_DISABLE_SANITIZE_HTML: true
  GF_LOG_LEVEL: info
```

### 3.2 Install Grafana

```bash
# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-values.yaml

# Get Grafana URL
kubectl get svc grafana -n monitoring
```

## 🛍️ Step 4: Deploy E-commerce POC Application

### 4.1 MongoDB Database

```yaml
# mongodb-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password123"
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
      volumes:
      - name: mongodb-storage
        persistentVolumeClaim:
          claimName: mongodb-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: ecommerce
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: gp2
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: ecommerce
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

### 4.2 API Service (Node.js)

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-api
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ecommerce-api
  template:
    metadata:
      labels:
        app: ecommerce-api
    spec:
      containers:
      - name: api
        image: node:16-alpine
        command: ["/bin/sh"]
        args: ["-c", "npm install express mongoose winston && node server.js"]
        ports:
        - containerPort: 3000
        env:
        - name: MONGODB_URI
          value: "mongodb://admin:password123@mongodb-service:27017/ecommerce?authSource=admin"
        - name: NODE_ENV
          value: "production"
        volumeMounts:
        - name: app-code
          mountPath: /app
        workingDir: /app
      volumes:
      - name: app-code
        configMap:
          name: api-code
---
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-api-service
  namespace: ecommerce
spec:
  selector:
    app: ecommerce-api
  ports:
  - port: 3000
    targetPort: 3000
  type: ClusterIP
```

### 4.3 API Code ConfigMap

```yaml
# api-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-code
  namespace: ecommerce
data:
  server.js: |
    const express = require('express');
    const mongoose = require('mongoose');
    const winston = require('winston');
    
    // Configure Winston logger
    const logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.Console()
      ]
    });
    
    const app = express();
    app.use(express.json());
    
    // Connect to MongoDB
    mongoose.connect(process.env.MONGODB_URI)
      .then(() => logger.info('Connected to MongoDB'))
      .catch(err => logger.error('MongoDB connection error:', err));
    
    // Product Schema
    const productSchema = new mongoose.Schema({
      name: String,
      price: Number,
      category: String,
      stock: Number
    });
    
    const Product = mongoose.model('Product', productSchema);
    
    // Order Schema
    const orderSchema = new mongoose.Schema({
      userId: String,
      products: [{
        productId: String,
        quantity: Number,
        price: Number
      }],
      total: Number,
      status: String,
      createdAt: { type: Date, default: Date.now }
    });
    
    const Order = mongoose.model('Order', orderSchema);
    
    // Middleware for request logging
    app.use((req, res, next) => {
      logger.info({
        method: req.method,
        url: req.url,
        ip: req.ip,
        userAgent: req.get('User-Agent'),
        timestamp: new Date().toISOString()
      });
      next();
    });
    
    // Routes
    app.get('/health', (req, res) => {
      logger.info('Health check requested');
      res.json({ status: 'healthy', timestamp: new Date().toISOString() });
    });
    
    app.get('/products', async (req, res) => {
      try {
        const products = await Product.find();
        logger.info(`Retrieved ${products.length} products`);
        res.json(products);
      } catch (error) {
        logger.error('Error fetching products:', error);
        res.status(500).json({ error: 'Internal server error' });
      }
    });
    
    app.post('/products', async (req, res) => {
      try {
        const product = new Product(req.body);
        await product.save();
        logger.info('Product created:', { productId: product._id, name: product.name });
        res.status(201).json(product);
      } catch (error) {
        logger.error('Error creating product:', error);
        res.status(400).json({ error: 'Bad request' });
      }
    });
    
    app.post('/orders', async (req, res) => {
      try {
        const order = new Order(req.body);
        await order.save();
        logger.info('Order created:', { 
          orderId: order._id, 
          userId: order.userId, 
          total: order.total,
          productCount: order.products.length
        });
        res.status(201).json(order);
      } catch (error) {
        logger.error('Error creating order:', error);
        res.status(400).json({ error: 'Bad request' });
      }
    });
    
    app.get('/orders/:userId', async (req, res) => {
      try {
        const orders = await Order.find({ userId: req.params.userId });
        logger.info(`Retrieved ${orders.length} orders for user ${req.params.userId}`);
        res.json(orders);
      } catch (error) {
        logger.error('Error fetching orders:', error);
        res.status(500).json({ error: 'Internal server error' });
      }
    });
    
    // Error handling middleware
    app.use((error, req, res, next) => {
      logger.error('Unhandled error:', error);
      res.status(500).json({ error: 'Internal server error' });
    });
    
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
      logger.info(`Server running on port ${PORT}`);
    });
  package.json: |
    {
      "name": "ecommerce-api",
      "version": "1.0.0",
      "main": "server.js",
      "dependencies": {
        "express": "^4.18.0",
        "mongoose": "^6.0.0",
        "winston": "^3.8.0"
      }
    }
```

### 4.4 Frontend Service

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-frontend
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ecommerce-frontend
  template:
    metadata:
      labels:
        app: ecommerce-frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: frontend-config
          mountPath: /etc/nginx/conf.d
        - name: frontend-html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: frontend-config
        configMap:
          name: frontend-nginx-config
      - name: frontend-html
        configMap:
          name: frontend-html
---
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-frontend-service
  namespace: ecommerce
spec:
  selector:
    app: ecommerce-frontend
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

### 4.5 Frontend ConfigMaps

```yaml
# frontend-configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-nginx-config
  namespace: ecommerce
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
        
        location /api/ {
            proxy_pass http://ecommerce-api-service:3000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: ecommerce
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>E-commerce Demo</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            .container { max-width: 800px; margin: 0 auto; }
            .product { border: 1px solid #ddd; padding: 20px; margin: 10px 0; }
            button { background: #007cba; color: white; padding: 10px 20px; border: none; cursor: pointer; }
            button:hover { background: #005a87; }
            .form-group { margin: 10px 0; }
            input, select { padding: 8px; margin: 5px; width: 200px; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>E-commerce Demo Store</h1>
            
            <div id="add-product">
                <h2>Add Product</h2>
                <div class="form-group">
                    <input type="text" id="productName" placeholder="Product Name">
                    <input type="number" id="productPrice" placeholder="Price">
                    <input type="text" id="productCategory" placeholder="Category">
                    <input type="number" id="productStock" placeholder="Stock">
                    <button onclick="addProduct()">Add Product</button>
                </div>
            </div>
            
            <div id="products">
                <h2>Products</h2>
                <button onclick="loadProducts()">Load Products</button>
                <div id="productList"></div>
            </div>
            
            <div id="orders">
                <h2>Create Order</h2>
                <div class="form-group">
                    <input type="text" id="userId" placeholder="User ID">
                    <input type="text" id="productId" placeholder="Product ID">
                    <input type="number" id="quantity" placeholder="Quantity">
                    <button onclick="createOrder()">Create Order</button>
                </div>
            </div>
        </div>
        
        <script>
            const API_BASE = '/api';
            
            async function addProduct() {
                const product = {
                    name: document.getElementById('productName').value,
                    price: parseFloat(document.getElementById('productPrice').value),
                    category: document.getElementById('productCategory').value,
                    stock: parseInt(document.getElementById('productStock').value)
                };
                
                try {
                    const response = await fetch(`${API_BASE}/products`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(product)
                    });
                    
                    if (response.ok) {
                        alert('Product added successfully!');
                        loadProducts();
                    } else {
                        alert('Error adding product');
                    }
                } catch (error) {
                    console.error('Error:', error);
                    alert('Error adding product');
                }
            }
            
            async function loadProducts() {
                try {
                    const response = await fetch(`${API_BASE}/products`);
                    const products = await response.json();
                    
                    const productList = document.getElementById('productList');
                    productList.innerHTML = products.map(product => `
                        <div class="product">
                            <h3>${product.name}</h3>
                            <p>Price: $${product.price}</p>
                            <p>Category: ${product.category}</p>
                            <p>Stock: ${product.stock}</p>
                            <p>ID: ${product._id}</p>
                        </div>
                    `).join('');
                } catch (error) {
                    console.error('Error:', error);
                    alert('Error loading products');
                }
            }
            
            async function createOrder() {
                const order = {
                    userId: document.getElementById('userId').value,
                    products: [{
                        productId: document.getElementById('productId').value,
                        quantity: parseInt(document.getElementById('quantity').value),
                        price: 10.00 // Simplified for demo
                    }],
                    total: 10.00 * parseInt(document.getElementById('quantity').value),
                    status: 'pending'
                };
                
                try {
                    const response = await fetch(`${API_BASE}/orders`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(order)
                    });
                    
                    if (response.ok) {
                        alert('Order created successfully!');
                    } else {
                        alert('Error creating order');
                    }
                } catch (error) {
                    console.error('Error:', error);
                    alert('Error creating order');
                }
            }
            
            // Load products on page load
            window.onload = loadProducts;
        </script>
    </body>
    </html>
```

### 4.6 Deploy E-commerce Application

```bash
# Create namespace
kubectl create namespace ecommerce

# Deploy all components
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f api-configmap.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f frontend-configmaps.yaml
kubectl apply -f frontend-deployment.yaml

# Wait for deployments
kubectl wait --for=condition=available --timeout=300s deployment/mongodb -n ecommerce
kubectl wait --for=condition=available --timeout=300s deployment/ecommerce-api -n ecommerce
kubectl wait --for=condition=available --timeout=300s deployment/ecommerce-frontend -n ecommerce

# Get frontend URL
kubectl get svc ecommerce-frontend-service -n ecommerce
```

## 📊 Step 5: Configure Promtail for Log Collection

### 5.1 Promtail Configuration

```yaml
# promtail-values.yaml
config:
  logLevel: info
  serverPort: 3101
  clients:
    - url: http://loki:3100/loki/api/v1/push

scrapeConfigs: |
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels:
          - __meta_kubernetes_pod_controller_name
        regex: ([0-9a-z-.]+?)(-[0-9a-f]{8,10})?
        action: replace
        target_label: __tmp_controller_name
      - source_labels:
          - __meta_kubernetes_pod_label_app_kubernetes_io_name
          - __meta_kubernetes_pod_label_app
          - __tmp_controller_name
          - __meta_kubernetes_pod_name
        regex: ^;*([^;]+)(;.*)?$
        action: replace
        target_label: app
      - source_labels:
          - __meta_kubernetes_pod_label_app_kubernetes_io_instance
          - __meta_kubernetes_pod_label_instance
        regex: ^;*([^;]+)(;.*)?$
        action: replace
        target_label: instance
      - source_labels:
          - __meta_kubernetes_pod_label_app_kubernetes_io_component
          - __meta_kubernetes_pod_label_component
        regex: ^;*([^;]+)(;.*)?$
        action: replace
        target_label: component
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node_name
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        replacement: $1
        separator: /
        source_labels:
        - namespace
        - app
        target_label: job
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_container_name
        target_label: container
      - action: replace
        replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_uid
        - __meta_kubernetes_pod_container_name
        target_label: __path__
      - action: replace
        regex: true/(.*)
        replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_annotationpresent_kubernetes_io_config_hash
        - __meta_kubernetes_pod_annotation_kubernetes_io_config_hash
        - __meta_kubernetes_pod_container_name
        target_label: __path__

  - job_name: kubernetes-pods-app
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - action: drop
        regex: .+
        source_labels:
          - __meta_kubernetes_pod_label_name
      - source_labels:
          - __meta_kubernetes_pod_label_app
        target_label: __service__
      - source_labels:
          - __meta_kubernetes_pod_node_name
        target_label: __host__
      - action: drop
        regex: ''
        source_labels:
          - __service__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        replacement: $1
        separator: /
        source_labels:
          - __meta_kubernetes_namespace
          - __service__
        target_label: job
      - action: replace
        source_labels:
          - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
          - __meta_kubernetes_pod_name
        target_label: pod
      - action: replace
        source_labels:
          - __meta_kubernetes_pod_container_name
        target_label: container
      - replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
          - __meta_kubernetes_pod_uid
          - __meta_kubernetes_pod_container_name
        target_label: __path__

serviceMonitor:
  enabled: true

resources:
  limits:
    cpu: 200m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### 5.2 Install Promtail

```bash
# Install Promtail
helm install promtail grafana/promtail \
  --namespace monitoring \
  --values promtail-values.yaml

# Verify installation
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail
```

## 📈 Step 6: Create Grafana Dashboards

### 6.1 E-commerce Application Dashboard

```json
# ecommerce-dashboard.json
{
  "dashboard": {
    "id": null,
    "title": "E-commerce Application Logs",
    "tags": ["ecommerce", "loki"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "API Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate({namespace=\"ecommerce\", app=\"ecommerce-api\"} |= \"method\" [5m])",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "palette-classic"
            },
            "custom": {
              "displayMode": "list",
              "orientation": "horizontal"
            },
            "mappings": [],
            "thresholds": {
              "steps": [
                {
                  "color": "green",
                  "value": null
                },
                {
                  "color": "red",
                  "value": 80
                }
              ]
            }
          }
        },
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate({namespace=\"ecommerce\", app=\"ecommerce-api\"} |= \"error\" [5m])",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {
                  "color": "green",
                  "value": null
                },
                {
                  "color": "yellow",
                  "value": 0.1
                },
                {
                  "color": "red",
                  "value": 1
                }
              ]
            }
          }
        },
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      },
      {
        "id": 3,
        "title": "Recent API Logs",
        "type": "logs",
        "targets": [
          {
            "expr": "{namespace=\"ecommerce\", app=\"ecommerce-api\"}",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 12,
          "w": 24,
          "x": 0,
          "y": 8
        }
      },
      {
        "id": 4,
        "title": "Order Creation Logs",
        "type": "logs",
        "targets": [
          {
            "expr": "{namespace=\"ecommerce\", app=\"ecommerce-api\"} |= \"Order created\"",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 20
        }
      },
      {
        "id": 5,
        "title": "Product Operations",
        "type": "logs",
        "targets": [
          {
            "expr": "{namespace=\"ecommerce\", app=\"ecommerce-api\"} |= \"Product\"",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 20
        }
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
```

### 6.2 Import Dashboard to Grafana

```bash
# Get Grafana admin password
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Port forward to access Grafana
kubectl port-forward --namespace monitoring svc/grafana 3000:80

# Access Grafana at http://localhost:3000
# Username: admin
# Password: (from above command)
```

## 🚨 Step 7: Set Up Alerts

### 7.1 Alert Rules Configuration

```yaml
# alert-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-alert-rules
  namespace: monitoring
data:
  alert-rules.yaml: |
    groups:
      - name: ecommerce-alerts
        rules:
          - alert: HighErrorRate
            expr: rate({namespace="ecommerce", app="ecommerce-api"} |= "error" [5m]) > 0.1
            for: 2m
            labels:
              severity: warning
              service: ecommerce-api
            annotations:
              summary: "High error rate detected in e-commerce API"
              description: "Error rate is {{ $value }} errors per second"
              
          - alert: APIDown
            expr: absent(rate({namespace="ecommerce", app="ecommerce-api"} [5m]))
            for: 1m
            labels:
              severity: critical
              service: ecommerce-api
            annotations:
              summary: "E-commerce API is down"
              description: "No logs received from e-commerce API for 1 minute"
              
          - alert: DatabaseConnectionError
            expr: rate({namespace="ecommerce", app="ecommerce-api"} |= "MongoDB connection error" [5m]) > 0
            for: 1m
            labels:
              severity: critical
              service: mongodb
            annotations:
              summary: "Database connection issues detected"
              description: "MongoDB connection errors detected in API logs"
```

### 7.2 Apply Alert Rules

```bash
# Apply alert rules
kubectl apply -f alert-rules.yaml

# Restart Grafana to pick up new rules
kubectl rollout restart deployment/grafana -n monitoring
```

## 🧪 Step 8: Generate Test Data and Logs

### 8.1 Load Testing Script

```bash
# Create load test script
cat > load-test.sh << 'EOF'
#!/bin/bash

# Get frontend service URL
FRONTEND_URL=$(kubectl get svc ecommerce-frontend-service -n ecommerce -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

if [ -z "$FRONTEND_URL" ]; then
    echo "Frontend service not ready. Using port-forward..."
    kubectl port-forward svc/ecommerce-frontend-service -n ecommerce 8080:80 &
    FRONTEND_URL="localhost:8080"
    sleep 5
fi

API_URL="http://${FRONTEND_URL}/api"

echo "Testing E-commerce API at: $API_URL"

# Create sample products
echo "Creating sample products..."
for i in {1..10}; do
    curl -X POST "$API_URL/products" \
        -H "Content-Type: application/json" \
        -d "{
            \"name\": \"Product $i\",
            \"price\": $((RANDOM % 100 + 10)),
            \"category\": \"Category $((i % 3 + 1))\",
            \"stock\": $((RANDOM % 50 + 10))
        }" &
done

wait

# Generate orders
echo "Creating sample orders..."
for i in {1..20}; do
    curl -X POST "$API_URL/orders" \
        -H "Content-Type: application/json" \
        -d "{
            \"userId\": \"user$((i % 5 + 1))\",
            \"products\": [{
                \"productId\": \"prod$i\",
                \"quantity\": $((RANDOM % 5 + 1)),
                \"price\": $((RANDOM % 50 + 10))
            }],
            \"total\": $((RANDOM % 200 + 50)),
            \"status\": \"pending\"
        }" &
    
    # Add some delay
    sleep 0.1
done

wait

# Generate some errors (invalid requests)
echo "Generating error scenarios..."
for i in {1..5}; do
    curl -X POST "$API_URL/products" \
        -H "Content-Type: application/json" \
        -d "{\"invalid\": \"data\"}" &
    
    curl -X GET "$API_URL/nonexistent" &
done

wait

echo "Load test completed!"
EOF

chmod +x load-test.sh
```

### 8.2 Run Load Test

```bash
# Run the load test
./load-test.sh

# Monitor logs in real-time
kubectl logs -f deployment/ecommerce-api -n ecommerce
```

## 📊 Step 9: Explore Logs in Grafana

### 9.1 Access Grafana

```bash
# Port forward to Grafana
kubectl port-forward --namespace monitoring svc/grafana 3000:80

# Open browser to http://localhost:3000
# Login with admin/admin123
```

### 9.2 Useful LogQL Queries

```logql
# All e-commerce API logs
{namespace="ecommerce", app="ecommerce-api"}

# Only error logs
{namespace="ecommerce", app="ecommerce-api"} |= "error"

# Order creation logs with JSON parsing
{namespace="ecommerce", app="ecommerce-api"} |= "Order created" | json

# HTTP requests by method
{namespace="ecommerce", app="ecommerce-api"} |= "method" | json | line_format "{{.method}} {{.url}}"

# Error rate over time
rate({namespace="ecommerce", app="ecommerce-api"} |= "error" [5m])

# Top error messages
topk(10, count by (level) (rate({namespace="ecommerce", app="ecommerce-api"} |= "error" [5m])))

# Logs from specific user actions
{namespace="ecommerce", app="ecommerce-api"} |= "userId" | json | userId="user1"

# Database connection logs
{namespace="ecommerce", app="ecommerce-api"} |= "MongoDB"

# Performance monitoring - slow requests
{namespace="ecommerce", app="ecommerce-api"} |= "method" | json | duration > 1000
```

## 🔧 Step 10: Production Optimizations

### 10.1 Loki Storage Optimization

```yaml
# loki-production-values.yaml
loki:
  config:
    limits_config:
      retention_period: 744h  # 31 days
      max_query_series: 500
      max_query_parallelism: 32
      
    chunk_store_config:
      max_look_back_period: 0s
      
    table_manager:
      retention_deletes_enabled: true
      retention_period: 744h
      
  persistence:
    enabled: true
    size: 50Gi
    storageClassName: gp3
    
  resources:
    limits:
      cpu: 1000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 1Gi
```

### 10.2 Grafana Production Configuration

```yaml
# grafana-production-values.yaml
replicas: 2

persistence:
  enabled: true
  size: 20Gi
  storageClassName: gp3

resources:
  limits:
    cpu: 500m
    memory: 1Gi
  requests:
    cpu: 250m
    memory: 512Mi

env:
  GF_SERVER_ROOT_URL: "https://grafana.yourdomain.com"
  GF_SECURITY_ADMIN_PASSWORD: "your-secure-password"
  GF_INSTALL_PLUGINS: "grafana-piechart-panel,grafana-worldmap-panel"

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: "alb"
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - grafana.yourdomain.com
```

### 10.3 Update Production Configurations

```bash
# Upgrade Loki with production settings
helm upgrade loki grafana/loki-stack \
  --namespace monitoring \
  --values loki-production-values.yaml

# Upgrade Grafana with production settings
helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-production-values.yaml
```

## 🧹 Step 11: Cleanup

```bash
# Delete e-commerce application
kubectl delete namespace ecommerce

# Delete monitoring stack
helm uninstall loki -n monitoring
helm uninstall grafana -n monitoring
helm uninstall promtail -n monitoring

# Delete monitoring namespace
kubectl delete namespace monitoring

# Delete EKS cluster
eksctl delete cluster --name loki-demo-cluster

# Delete S3 bucket
aws s3 rb s3://loki-storage-bucket-$(aws sts get-caller-identity --query Account --output text) --force

# Delete IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/LokiS3Policy
```

## 📚 Key Learnings

### Loki Concepts
- **Log Aggregation**: Centralized collection from multiple sources
- **Label-based Indexing**: Efficient storage and querying
- **LogQL**: Powerful query language for log analysis
- **Retention Policies**: Automated log lifecycle management

### Production Best Practices
- **Resource Limits**: Prevent resource exhaustion
- **Persistent Storage**: Ensure data durability
- **Monitoring**: Track Loki and Grafana health
- **Security**: Implement proper RBAC and network policies
- **Backup Strategy**: Regular backups of configurations and data

### Cost Optimization
- **Log Retention**: Balance storage costs with compliance needs
- **Query Optimization**: Efficient LogQL queries reduce compute costs
- **Storage Tiering**: Use appropriate storage classes
- **Resource Right-sizing**: Monitor and adjust resource allocations

## 🎯 Next Steps

1. **Advanced Alerting**: Implement PagerDuty/Slack integrations
2. **Log Parsing**: Add structured logging with JSON parsing
3. **Multi-tenancy**: Implement tenant isolation for multiple teams
4. **Backup & Recovery**: Set up automated backup procedures
5. **Security Hardening**: Implement authentication and authorization
6. **Performance Tuning**: Optimize for high-volume log ingestion

This guide provides a complete production-ready Loki-Grafana implementation on EKS with real-world examples. The e-commerce POC demonstrates practical log aggregation, monitoring, and alerting scenarios you'll encounter in production environments.