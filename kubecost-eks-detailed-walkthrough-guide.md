# Kubecost for EKS: Complete Detailed Walkthrough Guide

## 🎯 Introduction: Understanding Kubernetes Cost Management

Imagine you're running a successful e-commerce platform on Amazon EKS. Your business is growing rapidly, and so are your Kubernetes clusters. You started with a simple setup - maybe 3 nodes running a few microservices. Fast forward six months, and you now have multiple clusters across different regions, dozens of namespaces, hundreds of pods, and your AWS bill has grown from $500 to $15,000 per month.

The problem is, you have no idea where that money is going. Which team is consuming the most resources? Which applications are the most expensive to run? Are you over-provisioning resources? Could you save money by rightsizing your pods or using spot instances more effectively?

This is exactly the problem Kubecost solves. Kubecost is like having a detailed financial analyst for your Kubernetes infrastructure. It breaks down your costs by namespace, deployment, service, label, and even individual pods. It shows you not just what you're spending, but where you can optimize to save money without impacting performance.

## 🏗️ What is Kubecost and Why Do You Need It?

Kubecost is an open-source tool that provides real-time cost visibility and insights for Kubernetes workloads. Think of it as Google Analytics for your Kubernetes spending. Just as Google Analytics shows you which pages on your website are most popular and where users are coming from, Kubecost shows you which parts of your Kubernetes infrastructure are most expensive and where your money is going.

### The Business Problem Kubecost Solves

Let's say you work for a company called "TechCorp" that runs a microservices-based application on EKS. Your application consists of:

- Frontend service (React application)
- User authentication service
- Product catalog service
- Order processing service
- Payment processing service
- Notification service
- Analytics service
- Background job processors

Without Kubecost, when your CFO asks "Why did our AWS bill increase by $3,000 this month?", you might give answers like:
- "We added more users, so we needed more resources"
- "We deployed some new features"
- "Traffic increased during the holiday season"

These are vague answers that don't help with decision-making. With Kubecost, you can give specific, actionable answers:
- "The analytics service is consuming 40% more CPU than last month due to a new reporting feature"
- "The payment processing service is over-provisioned - we're requesting 2 CPU cores but only using 0.3 CPU cores on average"
- "Our development namespace is costing $800/month because developers are leaving test deployments running"

### Key Features of Kubecost

**Real-time Cost Monitoring**: See your costs as they happen, not at the end of the month when you get your AWS bill.

**Multi-dimensional Cost Breakdown**: View costs by namespace, deployment, service, pod, node, cluster, or any Kubernetes label.

**Resource Efficiency Insights**: Identify over-provisioned and under-provisioned workloads.

**Cost Allocation**: Accurately allocate shared infrastructure costs to different teams or projects.

**Savings Recommendations**: Get specific recommendations for reducing costs without impacting performance.

**Budget Alerts**: Set up alerts when spending exceeds predefined thresholds.

**Multi-cluster Support**: Monitor costs across multiple EKS clusters from a single dashboard.

## 🚀 Complete EKS Setup: From Zero to Kubecost

Let's walk through setting up Kubecost on a real EKS cluster. We'll create a complete environment that mirrors what you'd have in production.

### Step 1: Prerequisites and Environment Setup

Before we install Kubecost, we need to set up our EKS cluster and ensure we have the necessary tools and permissions.

**Required Tools:**
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install AWS CLI (if not already installed)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**AWS Configuration:**
```bash
# Configure AWS credentials
aws configure
# Enter your Access Key ID, Secret Access Key, Region (e.g., us-east-1), and output format (json)

# Verify your AWS identity
aws sts get-caller-identity
```

### Step 2: Creating the EKS Cluster

We'll create a production-like EKS cluster with multiple node groups to simulate a real environment.

**Create the cluster configuration file:**
```yaml
# eks-cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: techcorp-production
  region: us-east-1
  version: "1.28"

# Enable logging for better observability
cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]

# IAM settings
iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: ebs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      ebsCSIController: true
  - metadata:
      name: kubecost-cost-analyzer
      namespace: kubecost
    attachPolicyARNs:
    - "arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess"
    - "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"

# Node groups for different workload types
nodeGroups:
  # General purpose node group
  - name: general-purpose
    instanceType: m5.large
    desiredCapacity: 3
    minSize: 2
    maxSize: 10
    volumeSize: 100
    volumeType: gp3
    labels:
      workload-type: general
      cost-center: engineering
    tags:
      Environment: production
      Team: platform
      Project: techcorp-main
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

  # Compute-intensive node group
  - name: compute-intensive
    instanceType: c5.xlarge
    desiredCapacity: 2
    minSize: 1
    maxSize: 5
    volumeSize: 100
    volumeType: gp3
    labels:
      workload-type: compute
      cost-center: analytics
    tags:
      Environment: production
      Team: data-science
      Project: analytics-pipeline
    taints:
      - key: workload-type
        value: compute
        effect: NoSchedule

  # Spot instances for cost optimization
  - name: spot-instances
    instancesDistribution:
      maxPrice: 0.10
      instanceTypes: ["m5.large", "m5.xlarge", "m4.large"]
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 0
      spotInstancePools: 3
    desiredCapacity: 2
    minSize: 0
    maxSize: 8
    labels:
      workload-type: batch
      cost-center: engineering
      instance-lifecycle: spot
    tags:
      Environment: production
      Team: platform
      Project: batch-processing

# Add-ons
addons:
- name: vpc-cni
  version: latest
- name: coredns
  version: latest
- name: kube-proxy
  version: latest
- name: aws-ebs-csi-driver
  version: latest
  serviceAccountRoleARN: arn:aws:iam::ACCOUNT_ID:role/eksctl-techcorp-production-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa-Role1
```

**Create the cluster:**
```bash
# Create the EKS cluster (this takes 15-20 minutes)
eksctl create cluster -f eks-cluster-config.yaml

# Verify cluster creation
kubectl get nodes
kubectl get namespaces

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name techcorp-production
```

### Step 3: Installing Essential Cluster Components

Before installing Kubecost, we need to set up some essential components that will help us demonstrate cost monitoring effectively.

**Install AWS Load Balancer Controller:**
```bash
# Download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

# Create IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=techcorp-production \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**Install Metrics Server (required for Kubecost):**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify metrics server is running
kubectl get deployment metrics-server -n kube-system
```

**Install Prometheus (Kubecost dependency):**
```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace for monitoring
kubectl create namespace monitoring

# Install Prometheus with custom values
cat > prometheus-values.yaml << EOF
server:
  persistentVolume:
    enabled: true
    size: 50Gi
    storageClass: gp2
  retention: "30d"
  
alertmanager:
  enabled: true
  persistentVolume:
    enabled: true
    size: 10Gi
    storageClass: gp2

nodeExporter:
  enabled: true
  
pushgateway:
  enabled: true

serverFiles:
  prometheus.yml:
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
      
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/\${1}/proxy/metrics
      
      - job_name: 'kubernetes-cadvisor'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/\${1}/proxy/metrics/cadvisor
EOF

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  -n monitoring \
  -f prometheus-values.yaml

# Verify Prometheus installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## 📊 Installing Kubecost: Step-by-Step Process

Now we're ready to install Kubecost. We'll install it with a comprehensive configuration that includes all the features you'd want in a production environment.

### Step 1: Create Kubecost Namespace and Configuration

```bash
# Create dedicated namespace for Kubecost
kubectl create namespace kubecost

# Label the namespace for better organization
kubectl label namespace kubecost cost-monitoring=enabled
kubectl label namespace kubecost team=platform
```

### Step 2: Configure Kubecost Values

Create a comprehensive Kubecost configuration that includes all the features we'll explore:

```yaml
# kubecost-values.yaml
global:
  # Prometheus configuration
  prometheus:
    enabled: false  # We're using our existing Prometheus
    fqdn: http://prometheus-server.monitoring.svc.cluster.local:80
  
  # Grafana configuration
  grafana:
    enabled: true
    domainName: kubecost.techcorp.local
    
# Cost Analyzer configuration
costAnalyzer:
  # Resource requests and limits
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 800m
      memory: 2Gi
  
  # Persistence for cost data
  persistentVolume:
    enabled: true
    size: 32Gi
    storageClass: gp2
  
  # Service configuration
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  
  # Cost model configuration
  costModel:
    # AWS-specific pricing
    awsSpotDataRegion: us-east-1
    awsSpotDataBucket: spot-data-feed-bucket
    
    # Custom pricing (optional)
    customPricing:
      enabled: false
    
    # Resource allocation
    maxQueryConcurrency: 5
    warmCache: true
    warmSavingsCache: true
    
    # ETL configuration for large clusters
    etl:
      enabled: true
      
# Prometheus configuration (connecting to existing instance)
prometheus:
  server:
    # We're using external Prometheus
    enabled: false
  
  # Node exporter (already installed with Prometheus)
  nodeExporter:
    enabled: false
    
  # Service monitors for cost metrics
  serviceAccounts:
    server:
      create: false

# Grafana configuration
grafana:
  # Enable Grafana for advanced dashboards
  enabled: true
  
  # Admin credentials
  adminUser: admin
  adminPassword: "TechCorp2024!"  # Change this in production
  
  # Persistence
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: gp2
  
  # Service configuration
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  
  # Grafana configuration
  grafana.ini:
    server:
      domain: grafana.techcorp.local
      root_url: "http://grafana.techcorp.local"
    
    security:
      admin_user: admin
      admin_password: "TechCorp2024!"
    
    auth.anonymous:
      enabled: false
    
    dashboards:
      default_home_dashboard_path: /var/lib/grafana/dashboards/kubecost/cluster-metrics.json

# Network costs (for multi-AZ and cross-region traffic)
networkCosts:
  enabled: true
  
  # AWS-specific network cost configuration
  config:
    services:
      amazon-web-services:
        enabled: true
        key: "AWS_ACCESS_KEY_ID"  # Will be provided via service account
        secret: "AWS_SECRET_ACCESS_KEY"
        
# Cluster controller for multi-cluster management
clusterController:
  enabled: true
  
# Reporting and alerts
reporting:
  # Enable cost reports
  enabled: true
  
  # Report configurations
  reports:
    - name: "weekly-team-costs"
      schedule: "0 9 * * 1"  # Every Monday at 9 AM
      format: "csv"
      
    - name: "monthly-namespace-costs"
      schedule: "0 9 1 * *"  # First day of month at 9 AM
      format: "pdf"

# Alerts configuration
alerts:
  enabled: true
  
  # Slack integration (optional)
  slack:
    enabled: false
    webhook: ""  # Add your Slack webhook URL
    
  # Email alerts (optional)
  email:
    enabled: false
    
  # Alert rules
  rules:
    - name: "high-namespace-cost"
      threshold: 1000  # $1000
      window: "7d"
      aggregation: "namespace"
      
    - name: "cluster-cost-spike"
      threshold: 5000  # $5000
      window: "1d"
      aggregation: "cluster"

# Service account for AWS permissions
serviceAccount:
  create: true
  name: kubecost-cost-analyzer
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eksctl-techcorp-production-addon-iamserviceaccount-kubecost-kubecost-cost-analyzer-Role1

# Ingress configuration (optional)
ingress:
  enabled: false  # We'll use LoadBalancer for this demo
  className: "alb"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - host: kubecost.techcorp.com
      paths:
        - path: /
          pathType: Prefix
```

### Step 3: Install Kubecost

```bash
# Add Kubecost Helm repository
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update

# Install Kubecost with our custom configuration
helm install kubecost kubecost/cost-analyzer \
  -n kubecost \
  -f kubecost-values.yaml \
  --version 1.108.1

# Verify installation
kubectl get pods -n kubecost
kubectl get svc -n kubecost

# Check if all pods are running (this may take 5-10 minutes)
kubectl wait --for=condition=ready pod --all -n kubecost --timeout=600s
```

### Step 4: Access Kubecost Dashboard

```bash
# Get the LoadBalancer URL
kubectl get svc -n kubecost kubecost-cost-analyzer -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Or use port-forward for immediate access
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090

# Access Kubecost at http://localhost:9090
```

## 🎛️ Kubecost Dashboard Deep Dive

Once you access the Kubecost dashboard, you'll see a comprehensive interface that provides multiple views of your Kubernetes costs. Let's walk through each section in detail.

### Overview Dashboard

The Overview dashboard is your mission control for Kubernetes costs. When you first open Kubecost, this is what you'll see.

**Key Metrics Displayed:**

**Monthly Cluster Cost**: This shows your total cluster cost for the current month. For our example cluster, you might see something like "$2,847.32" which includes:
- Compute costs (EC2 instances): $1,890.45
- Storage costs (EBS volumes): $234.67
- Network costs (data transfer): $89.23
- Load balancer costs: $632.97

**Daily Cost Trend**: A line graph showing how your daily costs have changed over time. You might notice patterns like:
- Higher costs during business hours (auto-scaling)
- Lower costs on weekends (reduced traffic)
- Spikes during deployments or traffic surges

**Cost by Namespace**: A pie chart breaking down costs by namespace. In our TechCorp example, you might see:
- production-frontend: $892.34 (31.3%)
- production-backend: $1,245.67 (43.7%)
- production-analytics: $456.78 (16.0%)
- development: $123.45 (4.3%)
- monitoring: $129.08 (4.5%)

**Efficiency Score**: Kubecost calculates an efficiency score based on resource utilization. A score of 65% means you're using 65% of the resources you're paying for, indicating 35% waste that could be optimized.

### Allocations View: Understanding Where Your Money Goes

The Allocations view is where Kubecost really shines. This is where you can drill down into your costs with incredible granularity.

**Viewing Costs by Different Dimensions:**

**By Namespace**: Click on "Namespace" in the allocation view to see costs broken down by namespace. You'll see a table like this:

```
Namespace               | Cost (7d) | CPU Cost | Memory Cost | Storage Cost | Network Cost
production-frontend     | $156.78   | $89.45   | $34.23      | $23.10       | $10.00
production-backend      | $234.56   | $145.67  | $56.78      | $25.11       | $7.00
production-analytics    | $89.34    | $45.67   | $23.45      | $15.22       | $5.00
development            | $45.67    | $25.34   | $12.33      | $6.00        | $2.00
monitoring             | $34.56    | $20.45   | $8.11       | $4.00        | $2.00
```

**By Deployment**: Switch to "Deployment" view to see which specific deployments are most expensive:

```
Deployment                    | Cost (7d) | Pods | CPU Req | Memory Req | Efficiency
frontend-web                  | $89.45    | 6    | 3.0     | 6Gi        | 72%
backend-api                   | $123.67   | 4    | 4.0     | 8Gi        | 58%
analytics-processor           | $67.89    | 2    | 2.0     | 4Gi        | 85%
user-authentication          | $34.56    | 3    | 1.5     | 3Gi        | 91%
payment-service              | $45.78    | 2    | 2.0     | 4Gi        | 45%
```

Notice how the payment-service has low efficiency (45%), indicating it's over-provisioned and could be optimized.

**By Labels**: This is particularly powerful for cost allocation. If you've labeled your resources with team, project, or cost-center labels, you can see costs by these dimensions:

```
Team Label        | Cost (7d) | Percentage
frontend-team     | $156.78   | 28.5%
backend-team      | $234.56   | 42.7%
data-team         | $89.34    | 16.2%
platform-team     | $69.32    | 12.6%
```

### Assets View: Infrastructure Cost Breakdown

The Assets view shows you the cost of your underlying infrastructure - the nodes, persistent volumes, and load balancers that support your applications.

**Node Costs**: You'll see a breakdown like this:

```
Node Name                           | Instance Type | Cost (7d) | CPU Util | Memory Util | Efficiency
ip-10-0-1-123.ec2.internal         | m5.large      | $67.20    | 45%      | 62%         | 53%
ip-10-0-2-456.ec2.internal         | m5.large      | $67.20    | 78%      | 85%         | 81%
ip-10-0-3-789.ec2.internal         | c5.xlarge     | $134.40   | 92%      | 88%         | 90%
ip-10-0-4-012.ec2.internal (Spot)  | m5.large      | $20.16    | 65%      | 70%         | 67%
```

This view immediately shows you that the first node (ip-10-0-1-123) is underutilized and could potentially be downsized or terminated.

**Storage Costs**: See the cost of your persistent volumes:

```
PV Name                    | Size  | Storage Class | Cost (7d) | Utilization
pvc-database-main         | 100Gi | gp3          | $8.00     | 67%
pvc-prometheus-data       | 50Gi  | gp2          | $5.00     | 45%
pvc-grafana-data          | 10Gi  | gp2          | $1.00     | 23%
```

**Load Balancer Costs**: See the cost of your AWS load balancers:

```
Load Balancer                    | Type | Cost (7d) | Data Processed
kubecost-cost-analyzer          | NLB  | $18.48    | 1.2TB
aws-load-balancer-controller    | ALB  | $22.68    | 2.1TB
```

### Savings Recommendations: Actionable Cost Optimization

One of Kubecost's most valuable features is its savings recommendations. These are specific, actionable suggestions for reducing costs.

**Right-sizing Recommendations**: Kubecost analyzes your actual resource usage and recommends optimal resource requests:

```
Workload: production-backend/payment-service
Current Request: 2 CPU, 4Gi Memory
Recommended: 0.8 CPU, 1.5Gi Memory
Potential Monthly Savings: $156.78
Confidence: High (based on 30 days of data)

Reasoning: This workload consistently uses only 40% of requested CPU and 37% of requested memory. 
Reducing requests will not impact performance but will significantly reduce costs.
```

**Cluster Sizing Recommendations**: Suggestions for optimizing your node groups:

```
Node Group: general-purpose
Current: 3x m5.large instances
Recommended: 2x m5.xlarge instances
Potential Monthly Savings: $234.56
Confidence: Medium

Reasoning: Current setup has low utilization across multiple small instances. 
Consolidating to fewer, larger instances will improve efficiency and reduce costs.
```

**Spot Instance Opportunities**: Recommendations for using spot instances:

```
Workload: production-analytics/batch-processor
Current: On-demand m5.large
Recommended: Spot m5.large with mixed instance types
Potential Monthly Savings: $445.67 (67% cost reduction)
Risk: Low (workload is fault-tolerant)
```

### Reports and Alerts: Staying on Top of Costs

Kubecost provides comprehensive reporting and alerting capabilities to help you stay on top of your costs.

**Cost Reports**: You can generate detailed cost reports in various formats:

**Weekly Team Report (CSV format)**:
```csv
Team,Namespace,Cost_7d,Cost_30d,CPU_Cost,Memory_Cost,Storage_Cost,Network_Cost
frontend-team,production-frontend,$156.78,$672.34,$89.45,$34.23,$23.10,$10.00
backend-team,production-backend,$234.56,$1006.78,$145.67,$56.78,$25.11,$7.00
data-team,production-analytics,$89.34,$383.46,$45.67,$23.45,$15.22,$5.00
```

**Monthly Executive Summary (PDF format)**: A high-level summary suitable for executives, including:
- Total monthly spend and trend
- Top cost drivers
- Efficiency metrics
- Savings opportunities
- Budget vs. actual spending

**Cost Alerts**: Set up alerts to notify you when costs exceed thresholds:

```
Alert: High Namespace Cost
Trigger: When any namespace exceeds $500 in 7 days
Action: Send Slack notification to #platform-team
Status: Active

Alert: Cluster Cost Spike  
Trigger: When daily cluster cost increases by >50% compared to 7-day average
Action: Send email to platform-team@techcorp.com
Status: Active

Alert: Low Efficiency
Trigger: When cluster efficiency drops below 60%
Action: Create Jira ticket for optimization review
Status: Active
```

## 🔧 Advanced Kubecost Configuration

Now that we understand the basics, let's explore advanced Kubecost configurations that you'd use in a production environment.

### Multi-Cluster Cost Management

In real-world scenarios, you often have multiple EKS clusters (development, staging, production, different regions). Kubecost can aggregate costs across all these clusters.

**Setting up Kubecost Federation:**

First, install Kubecost on each cluster with federation enabled:

```yaml
# kubecost-federation-values.yaml
global:
  prometheus:
    enabled: false
    fqdn: http://prometheus-server.monitoring.svc.cluster.local:80

# Enable federation
federatedETL:
  enabled: true
  
# Primary cluster configuration (where you'll view aggregated data)
costAnalyzer:
  federatedStorageConfigSecret: "federated-store-config"
  
# Thanos configuration for long-term storage
thanos:
  enabled: true
  
  # S3 bucket for storing metrics data
  objstore:
    config:
      type: S3
      config:
        bucket: "kubecost-thanos-storage"
        endpoint: "s3.us-east-1.amazonaws.com"
        region: "us-east-1"
        
# Multi-cluster service account
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/KubecostFederationRole
```

**Create S3 bucket for federated storage:**
```bash
# Create S3 bucket for Thanos storage
aws s3 mb s3://kubecost-thanos-storage-techcorp --region us-east-1

# Create IAM policy for S3 access
cat > kubecost-s3-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::kubecost-thanos-storage-techcorp",
                "arn:aws:s3:::kubecost-thanos-storage-techcorp/*"
            ]
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name KubecostS3Access \
    --policy-document file://kubecost-s3-policy.json
```

**Configure federated storage secret:**
```bash
# Create secret for S3 configuration
kubectl create secret generic federated-store-config -n kubecost --from-literal=object-store.yaml="
type: S3
config:
  bucket: kubecost-thanos-storage-techcorp
  endpoint: s3.us-east-1.amazonaws.com
  region: us-east-1
"
```

### Custom Pricing Configuration

For accurate cost allocation, you might want to configure custom pricing that reflects your actual AWS costs, including Reserved Instances, Savings Plans, or enterprise discounts.

```yaml
# custom-pricing-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pricing-configs
  namespace: kubecost
data:
  default.json: |
    {
      "CPU": "0.031611",
      "spotCPU": "0.006655",
      "RAM": "0.004237",
      "spotRAM": "0.000892",
      "GPU": "0.95",
      "storage": "0.00005",
      "zoneNetworkEgress": "0.01",
      "regionNetworkEgress": "0.02",
      "internetNetworkEgress": "0.143",
      "spotLabel": "lifecycle",
      "spotLabelValue": "spot",
      "awsServiceKeyName": "service",
      "awsServiceKeySecret": "service-key",
      "awsSpotDataRegion": "us-east-1",
      "awsSpotDataBucket": "spot-data-feed",
      "projectID": "techcorp-production"
    }
```

Apply the custom pricing:
```bash
kubectl apply -f custom-pricing-config.yaml

# Update Kubecost to use custom pricing
helm upgrade kubecost kubecost/cost-analyzer \
  -n kubecost \
  --set costAnalyzer.costModel.customPricing.enabled=true \
  --set costAnalyzer.costModel.customPricing.configmapName=pricing-configs
```

### Advanced Monitoring and Alerting

Set up comprehensive monitoring and alerting for your Kubecost deployment:

```yaml
# kubecost-monitoring.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubecost-cost-analyzer
  namespace: kubecost
  labels:
    app: cost-analyzer
spec:
  selector:
    matchLabels:
      app: cost-analyzer
  endpoints:
  - port: http
    interval: 30s
    path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubecost-alerts
  namespace: kubecost
spec:
  groups:
  - name: kubecost.rules
    rules:
    - alert: KubecostHighClusterCost
      expr: kubecost_cluster_cost_total > 5000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High cluster cost detected"
        description: "Cluster cost has exceeded $5000 in the last 24 hours"
        
    - alert: KubecostLowEfficiency
      expr: kubecost_cluster_efficiency < 0.6
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Low cluster efficiency"
        description: "Cluster efficiency is below 60%: {{ $value }}"
        
    - alert: KubecostHighNamespaceCost
      expr: kubecost_namespace_cost_total > 1000
      for: 5m
      labels:
        severity: info
      annotations:
        summary: "High namespace cost"
        description: "Namespace {{ $labels.namespace }} cost exceeds $1000"
```

## 📈 Real-World Cost Optimization Scenarios

Let's walk through several real-world scenarios where Kubecost helps identify and resolve cost issues.

### Scenario 1: The Runaway Development Environment

**Problem Discovery**: Your monthly AWS bill increased by $2,000, and you need to find out why.

**Investigation with Kubecost**:
1. Open the Allocations view and sort by cost (7 days)
2. You notice the "development" namespace is costing $1,456 for the week
3. Drill down into the development namespace
4. You see multiple deployments with names like "feature-branch-abc123", "test-deployment-xyz789"

**Root Cause**: Developers are creating test deployments for feature branches but not cleaning them up after testing.

**Solution Implementation**:
```bash
# Create a cleanup policy using a CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-deployments
  namespace: development
spec:
  schedule: "0 2 * * *"  # Run daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Delete deployments older than 7 days with specific labels
              kubectl get deployments -n development \
                -l temporary=true \
                -o json | \
              jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 604800)) | .metadata.name' | \
              xargs -r kubectl delete deployment -n development
              
              # Delete services older than 7 days
              kubectl get services -n development \
                -l temporary=true \
                -o json | \
              jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 604800)) | .metadata.name' | \
              xargs -r kubectl delete service -n development
          restartPolicy: OnFailure
```

**Result**: Monthly development costs reduced from $6,224 to $1,890 (70% reduction).

### Scenario 2: Over-Provisioned Analytics Workload

**Problem Discovery**: The analytics team complains about high infrastructure costs, but they need the processing power for their ML workloads.

**Investigation with Kubecost**:
1. Filter allocations by namespace: "production-analytics"
2. Look at the efficiency column - you see 23% efficiency
3. Click on the analytics-processor deployment
4. View the resource utilization over time

**Analysis**: The deployment requests 8 CPU cores and 16Gi memory, but actual usage shows:
- Average CPU usage: 1.2 cores (15% of requested)
- Peak CPU usage: 3.4 cores (42% of requested)  
- Average Memory usage: 4.2Gi (26% of requested)
- Peak Memory usage: 8.1Gi (51% of requested)

**Solution Implementation**:
```yaml
# Optimized analytics deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics-processor-optimized
  namespace: production-analytics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: analytics-processor
  template:
    metadata:
      labels:
        app: analytics-processor
    spec:
      containers:
      - name: processor
        image: techcorp/analytics-processor:v2.1
        resources:
          requests:
            cpu: 2000m      # Reduced from 8000m
            memory: 6Gi     # Reduced from 16Gi
          limits:
            cpu: 4000m      # Allow bursting for peak loads
            memory: 10Gi    # Allow some headroom for spikes
        env:
        - name: PROCESSING_MODE
          value: "batch"
        - name: MAX_CONCURRENT_JOBS
          value: "4"
---
# Horizontal Pod Autoscaler for handling load spikes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: analytics-processor-hpa
  namespace: production-analytics
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: analytics-processor-optimized
  minReplicas: 1
  maxReplicas: 5
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

**Result**: Analytics workload costs reduced from $2,340/month to $890/month (62% reduction) while maintaining the same processing capability through auto-scaling.

### Scenario 3: Inefficient Storage Usage

**Problem Discovery**: Storage costs are higher than expected for the amount of data you're storing.

**Investigation with Kubecost**:
1. Go to Assets view
2. Click on "Storage" tab
3. Sort by cost to see most expensive volumes
4. Check utilization percentages

**Analysis**: You discover:
- Database PV: 500Gi allocated, 127Gi used (25% utilization)
- Log storage PV: 200Gi allocated, 45Gi used (22% utilization)
- Backup PV: 1000Gi allocated, 234Gi used (23% utilization)

**Solution Implementation**:
```bash
# Resize persistent volumes (requires CSI driver support)
# First, check if your storage class supports volume expansion
kubectl get storageclass gp3 -o yaml | grep allowVolumeExpansion

# Create optimized storage classes with different performance tiers
cat > storage-classes.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-optimized
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-high-performance
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "16000"
  throughput: "1000"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-archive
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f storage-classes.yaml

# Implement automated storage cleanup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: storage-cleanup
  namespace: production
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: techcorp/storage-cleanup:v1.0
            command:
            - /bin/bash
            - -c
            - |
              # Clean up old log files
              find /var/log -name "*.log" -mtime +7 -delete
              
              # Compress old backup files
              find /backups -name "*.sql" -mtime +3 -exec gzip {} \;
              
              # Remove backups older than 30 days
              find /backups -name "*.gz" -mtime +30 -delete
            volumeMounts:
            - name: log-storage
              mountPath: /var/log
            - name: backup-storage
              mountPath: /backups
          volumes:
          - name: log-storage
            persistentVolumeClaim:
              claimName: log-storage-pvc
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-storage-pvc
          restartPolicy: OnFailure
```

**Result**: Storage costs reduced from $1,200/month to $450/month (62% reduction) through rightsizing and automated cleanup.

## 🎯 Best Practices and Production Tips

Based on real-world experience with Kubecost in production environments, here are essential best practices:

### 1. Proper Resource Labeling Strategy

Implement a consistent labeling strategy across all your Kubernetes resources:

```yaml
# Standard labels for cost allocation
metadata:
  labels:
    # Business labels
    team: "backend-team"
    project: "user-authentication"
    cost-center: "engineering"
    environment: "production"
    
    # Technical labels  
    component: "api-server"
    version: "v2.1.0"
    
    # Cost optimization labels
    criticality: "high"        # high, medium, low
    scaling-policy: "auto"     # auto, manual, fixed
    instance-preference: "on-demand"  # on-demand, spot, mixed
```

### 2. Set Up Proper RBAC for Kubecost

Create role-based access control to ensure teams can only see their own costs:

```yaml
# Team-specific RBAC for cost visibility
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production-frontend
  name: kubecost-frontend-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "persistentvolumeclaims"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubecost-frontend-binding
  namespace: production-frontend
subjects:
- kind: User
  name: frontend-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: kubecost-frontend-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. Implement Cost Budgets and Governance

Set up automated cost governance using Kubernetes policies:

```yaml
# ResourceQuota for cost control
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-budget-quota
  namespace: production-frontend
spec:
  hard:
    requests.cpu: "20"      # Max 20 CPU cores
    requests.memory: 40Gi   # Max 40Gi memory
    persistentvolumeclaims: "10"  # Max 10 PVCs
    count/deployments: "20"       # Max 20 deployments
---
# LimitRange for preventing resource waste
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production-frontend
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 2000m
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi
    type: Container
```

### 4. Automated Cost Optimization

Implement automated cost optimization using custom controllers:

```yaml
# VPA for automatic rightsizing
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: analytics-processor-vpa
  namespace: production-analytics
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: analytics-processor
  updatePolicy:
    updateMode: "Auto"  # Automatically apply recommendations
  resourcePolicy:
    containerPolicies:
    - containerName: processor
      maxAllowed:
        cpu: 4000m
        memory: 8Gi
      minAllowed:
        cpu: 100m
        memory: 256Mi
      controlledResources: ["cpu", "memory"]
```

### 5. Cost Monitoring and Alerting Integration

Integrate Kubecost with your existing monitoring stack:

```yaml
# Prometheus AlertManager configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'
    
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'cost-alerts'
      routes:
      - match:
          severity: critical
        receiver: 'cost-critical'
    
    receivers:
    - name: 'cost-alerts'
      slack_configs:
      - channel: '#cost-optimization'
        title: 'Kubecost Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        
    - name: 'cost-critical'
      slack_configs:
      - channel: '#platform-team'
        title: 'CRITICAL: High Cost Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      email_configs:
      - to: 'platform-team@techcorp.com'
        subject: 'CRITICAL: Kubernetes Cost Alert'
        body: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

## 🔍 Troubleshooting Common Kubecost Issues

### Issue 1: Kubecost Shows No Data

**Symptoms**: Kubecost dashboard loads but shows "No data available" or zero costs.

**Diagnosis Steps**:
```bash
# Check if Kubecost pods are running
kubectl get pods -n kubecost

# Check Kubecost logs
kubectl logs -n kubecost deployment/kubecost-cost-analyzer

# Verify Prometheus connectivity
kubectl exec -n kubecost deployment/kubecost-cost-analyzer -- \
  curl -s http://prometheus-server.monitoring.svc.cluster.local:80/api/v1/query?query=up

# Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system
```

**Common Solutions**:
```bash
# Restart Kubecost deployment
kubectl rollout restart deployment/kubecost-cost-analyzer -n kubecost

# Verify Prometheus configuration
kubectl get configmap prometheus-server -n monitoring -o yaml | grep kubecost

# Check service discovery
kubectl get endpoints -n kubecost
```

### Issue 2: Inaccurate Cost Data

**Symptoms**: Costs shown in Kubecost don't match AWS billing.

**Diagnosis and Solutions**:
```bash
# Check custom pricing configuration
kubectl get configmap pricing-configs -n kubecost -o yaml

# Verify AWS pricing data
kubectl logs -n kubecost deployment/kubecost-cost-analyzer | grep -i pricing

# Update pricing data manually
kubectl exec -n kubecost deployment/kubecost-cost-analyzer -- \
  /bin/sh -c "curl -X POST http://localhost:9003/refreshPricing"
```

### Issue 3: High Memory Usage

**Symptoms**: Kubecost pods consuming excessive memory or getting OOMKilled.

**Solutions**:
```yaml
# Increase memory limits and optimize ETL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubecost-cost-analyzer
  namespace: kubecost
spec:
  template:
    spec:
      containers:
      - name: cost-analyzer-frontend
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
        env:
        - name: ETL_ENABLED
          value: "true"
        - name: ETL_MAX_PROMETHEUS_QUERY_DURATION
          value: "5m"
        - name: CACHE_WARMING_ENABLED
          value: "false"  # Disable if memory constrained
```

This comprehensive Kubecost guide provides everything you need to implement effective cost monitoring and optimization for your EKS clusters. The key to success is starting with proper setup, implementing good labeling practices, and regularly reviewing and acting on the insights Kubecost provides.

Remember that cost optimization is an ongoing process, not a one-time activity. Set up regular reviews of your Kubecost data, implement the recommended optimizations, and continuously monitor the results to ensure you're getting the best value from your Kubernetes infrastructure.