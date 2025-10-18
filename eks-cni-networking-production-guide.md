# AWS EKS CNI Networking: Complete Production Design Guide

## 🌐 Understanding the Foundation: What is CNI and Why It Matters

Container Network Interface (CNI) is the backbone of Kubernetes networking. Think of CNI as the traffic management system for a massive city - it determines how containers (like cars) communicate with each other, how they reach external services (like highways leading out of the city), and how external traffic reaches them (like entrance ramps into the city).

In Amazon EKS, the AWS VPC CNI plugin is the default networking solution, and understanding it deeply is crucial for designing production-grade clusters. Unlike other CNI plugins that create overlay networks, the AWS VPC CNI provides native VPC networking to Kubernetes pods, meaning each pod gets a real IP address from your VPC subnets.

### The Business Impact of CNI Design Decisions

Imagine you're the platform engineer for a fintech company processing millions of transactions daily. Your CNI design decisions directly impact:

**Performance**: Poor CNI configuration can add 10-50ms latency to every service call. In a microservices architecture with 20+ service hops per transaction, this compounds to 200-1000ms additional latency.

**Security**: Incorrect network policies can expose sensitive financial data. A misconfigured CNI might allow a compromised frontend pod to access the database directly.

**Scalability**: Running out of IP addresses during Black Friday traffic can bring down your entire platform. Poor subnet design might limit you to 250 pods when you need 2,000.

**Cost**: Inefficient networking can increase data transfer costs by 300-500%. Cross-AZ traffic that could be avoided with proper pod placement costs $0.01/GB.

**Compliance**: Financial regulations require network segmentation. Your CNI design must support PCI DSS, SOX, and other compliance requirements.

## 🏗️ AWS VPC CNI Deep Dive: How It Really Works

### The Unique Architecture of AWS VPC CNI

The AWS VPC CNI is fundamentally different from other CNI plugins. Instead of creating an overlay network (like Flannel or Calico), it leverages native AWS networking primitives. Here's how it works at the deepest level:

**Elastic Network Interfaces (ENIs) as the Foundation**

Every EC2 instance in your EKS cluster can have multiple ENIs attached. Each ENI can have multiple secondary IP addresses. The AWS VPC CNI uses these secondary IPs to assign real VPC IP addresses to pods.

Let's break this down with a concrete example:

```
EC2 Instance: m5.large (can support 3 ENIs, 10 IPs per ENI = 30 total IPs)

ENI 0 (Primary): 10.0.1.100 (instance IP)
├── Secondary IP 1: 10.0.1.101 → Pod: frontend-web-abc123
├── Secondary IP 2: 10.0.1.102 → Pod: frontend-web-def456
├── Secondary IP 3: 10.0.1.103 → Pod: backend-api-ghi789
└── ... (up to 9 more secondary IPs)

ENI 1 (Secondary): No primary IP
├── Secondary IP 1: 10.0.1.104 → Pod: analytics-worker-jkl012
├── Secondary IP 2: 10.0.1.105 → Pod: payment-processor-mno345
└── ... (up to 8 more secondary IPs)

ENI 2 (Secondary): No primary IP
├── Secondary IP 1: 10.0.1.106 → Pod: notification-service-pqr678
└── ... (up to 9 more secondary IPs)
```

**The IP Address Lifecycle**

Understanding how IP addresses are managed is crucial for production planning:

1. **Warm Pool**: The CNI maintains a "warm pool" of available IP addresses on each node
2. **Pod Creation**: When a pod is scheduled, it gets an IP from the warm pool instantly
3. **IP Recycling**: When a pod is deleted, its IP returns to the warm pool for reuse
4. **Dynamic Scaling**: If the warm pool runs low, the CNI requests more ENIs/IPs from AWS

**The IPAMD (IP Address Management Daemon)**

The IPAMD runs on every node and is responsible for:
- Managing the warm pool of IP addresses
- Requesting new ENIs when needed
- Releasing unused ENIs to save costs
- Handling IP address assignment to pods

### CNI Configuration Parameters Deep Dive

The AWS VPC CNI behavior is controlled by several environment variables. Understanding these is critical for production tuning:

**WARM_ENI_TARGET**: Number of ENIs to keep in warm pool
```bash
# Default: 1
# Production recommendation: 1-2 for most workloads
# High-churn workloads: 2-3
export WARM_ENI_TARGET=2
```

**WARM_IP_TARGET**: Number of IP addresses to keep available
```bash
# Default: None (uses WARM_ENI_TARGET)
# Production recommendation: 5-10 for predictable workloads
# Burst workloads: 20-50
export WARM_IP_TARGET=10
```

**MAX_ENI**: Maximum ENIs per node
```bash
# Default: Instance type maximum
# Production: Set based on your pod density requirements
export MAX_ENI=3
```

**AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG**: Enable custom networking
```bash
# Default: false
# Use when you need pods in different subnets than nodes
export AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

## 🎯 Production Network Architecture Design

### Scenario: E-commerce Platform Network Design

Let's design a production network for "TechMart," an e-commerce platform handling 100,000 orders per day with the following requirements:

**Business Requirements:**
- 99.99% uptime (4.38 minutes downtime per month)
- PCI DSS compliance for payment processing
- GDPR compliance for EU customers
- Support for 10,000 concurrent users during peak hours
- Multi-region deployment (US East, EU West)
- Disaster recovery with 15-minute RTO

**Technical Requirements:**
- 500+ pods during normal operations
- 2,000+ pods during Black Friday
- Microservices architecture (20+ services)
- Real-time analytics and ML workloads
- Separate environments (dev, staging, prod)

### VPC Design Strategy

**Multi-Account Strategy:**
```
AWS Organization
├── Production Account (111111111111)
│   ├── VPC: prod-us-east-1 (10.0.0.0/16)
│   └── VPC: prod-eu-west-1 (10.1.0.0/16)
├── Staging Account (222222222222)
│   ├── VPC: staging-us-east-1 (10.10.0.0/16)
│   └── VPC: staging-eu-west-1 (10.11.0.0/16)
└── Development Account (333333333333)
    └── VPC: dev-us-east-1 (10.20.0.0/16)
```

**Production VPC Subnet Strategy (10.0.0.0/16):**

```
Availability Zone A (us-east-1a):
├── Public Subnet: 10.0.1.0/24 (251 IPs) - Load Balancers, NAT Gateways
├── Private Subnet (Nodes): 10.0.10.0/22 (1019 IPs) - EKS Worker Nodes
├── Private Subnet (Pods): 10.0.20.0/20 (4091 IPs) - Pod IPs (Custom Networking)
├── Database Subnet: 10.0.100.0/24 (251 IPs) - RDS, ElastiCache
└── Management Subnet: 10.0.110.0/24 (251 IPs) - Bastion, Monitoring

Availability Zone B (us-east-1b):
├── Public Subnet: 10.0.2.0/24 (251 IPs)
├── Private Subnet (Nodes): 10.0.11.0/22 (1019 IPs)
├── Private Subnet (Pods): 10.0.36.0/20 (4091 IPs)
├── Database Subnet: 10.0.101.0/24 (251 IPs)
└── Management Subnet: 10.0.111.0/24 (251 IPs)

Availability Zone C (us-east-1c):
├── Public Subnet: 10.0.3.0/24 (251 IPs)
├── Private Subnet (Nodes): 10.0.12.0/22 (1019 IPs)
├── Private Subnet (Pods): 10.0.52.0/20 (4091 IPs)
├── Database Subnet: 10.0.102.0/24 (251 IPs)
└── Management Subnet: 10.0.112.0/24 (251 IPs)

Reserved for Future Expansion:
└── 10.0.128.0/17 (32,765 IPs) - Future use
```

### Custom Networking Implementation

Custom networking allows pods to use different subnets than nodes, providing better IP utilization and security isolation:

```yaml
# ENIConfig for Availability Zone A
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a-pod-netconfig
spec:
  securityGroups:
    - sg-0123456789abcdef0  # Pod security group
  subnet: subnet-0abcdef1234567890  # Pod subnet (10.0.20.0/20)
---
# ENIConfig for Availability Zone B
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1b-pod-netconfig
spec:
  securityGroups:
    - sg-0123456789abcdef0
  subnet: subnet-0bcdef1234567890a  # Pod subnet (10.0.36.0/20)
---
# ENIConfig for Availability Zone C
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1c-pod-netconfig
spec:
  securityGroups:
    - sg-0123456789abcdef0
  subnet: subnet-0cdef1234567890ab  # Pod subnet (10.0.52.0/20)
```

**Node Group Configuration with Custom Networking:**
```yaml
# eksctl configuration for custom networking
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: techmart-production
  region: us-east-1
  version: "1.28"

vpc:
  id: vpc-0123456789abcdef0
  subnets:
    private:
      us-east-1a-nodes:
        id: subnet-0123456789abcdef1  # Node subnet
        az: us-east-1a
      us-east-1b-nodes:
        id: subnet-0123456789abcdef2
        az: us-east-1b
      us-east-1c-nodes:
        id: subnet-0123456789abcdef3
        az: us-east-1c
    public:
      us-east-1a-public:
        id: subnet-0123456789abcdef4
        az: us-east-1a
      us-east-1b-public:
        id: subnet-0123456789abcdef5
        az: us-east-1b
      us-east-1c-public:
        id: subnet-0123456789abcdef6
        az: us-east-1c

nodeGroups:
  - name: frontend-nodes
    instanceType: m5.xlarge
    desiredCapacity: 6
    minSize: 3
    maxSize: 20
    availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]
    privateNetworking: true
    labels:
      node-type: frontend
      k8s.amazonaws.com/eniConfig: us-east-1a-pod-netconfig  # AZ-specific
    taints:
      - key: frontend-only
        value: "true"
        effect: NoSchedule
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

  - name: backend-nodes
    instanceType: c5.2xlarge
    desiredCapacity: 9
    minSize: 6
    maxSize: 30
    availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]
    privateNetworking: true
    labels:
      node-type: backend
    taints:
      - key: backend-only
        value: "true"
        effect: NoSchedule

  - name: data-nodes
    instanceType: r5.4xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 12
    availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]
    privateNetworking: true
    labels:
      node-type: data-processing
    taints:
      - key: data-processing-only
        value: "true"
        effect: NoSchedule
```

## 🔒 Security Groups and Network Policies

### Layered Security Approach

Security in EKS networking requires multiple layers of protection:

**Layer 1: VPC Security Groups (AWS Native)**
**Layer 2: Kubernetes Network Policies (CNI Plugin)**
**Layer 3: Service Mesh (Istio/Linkerd)**
**Layer 4: Application-Level Security**

### Production Security Group Design

```bash
# Create security groups for different tiers
aws ec2 create-security-group \
    --group-name techmart-alb-sg \
    --description "Application Load Balancer Security Group" \
    --vpc-id vpc-0123456789abcdef0

aws ec2 create-security-group \
    --group-name techmart-node-sg \
    --description "EKS Node Security Group" \
    --vpc-id vpc-0123456789abcdef0

aws ec2 create-security-group \
    --group-name techmart-pod-sg \
    --description "EKS Pod Security Group" \
    --vpc-id vpc-0123456789abcdef0

aws ec2 create-security-group \
    --group-name techmart-db-sg \
    --description "Database Security Group" \
    --vpc-id vpc-0123456789abcdef0

# ALB Security Group Rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-alb123456 \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id sg-alb123456 \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

# Node Security Group Rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-node123456 \
    --protocol tcp \
    --port 22 \
    --source-group sg-bastion123456

aws ec2 authorize-security-group-ingress \
    --group-id sg-node123456 \
    --protocol tcp \
    --port 10250 \
    --source-group sg-node123456  # Kubelet API

aws ec2 authorize-security-group-ingress \
    --group-id sg-node123456 \
    --protocol tcp \
    --port 53 \
    --source-group sg-pod123456  # DNS

# Pod Security Group Rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-pod123456 \
    --protocol tcp \
    --port 443 \
    --source-group sg-alb123456  # HTTPS from ALB

aws ec2 authorize-security-group-ingress \
    --group-id sg-pod123456 \
    --protocol tcp \
    --port 8080 \
    --source-group sg-pod123456  # Inter-pod communication

# Database Security Group Rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-db123456 \
    --protocol tcp \
    --port 5432 \
    --source-group sg-pod123456  # PostgreSQL from pods only
```

### Kubernetes Network Policies

Network policies provide fine-grained control over pod-to-pod communication:

```yaml
# Default deny all ingress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Frontend pods can receive traffic from ALB and communicate with backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to: []  # Allow DNS
    ports:
    - protocol: UDP
      port: 53
---
# Backend pods can only receive traffic from frontend and communicate with database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []  # Allow DNS and external APIs
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 443
---
# Database pods can only receive traffic from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

## ⚡ Performance Optimization and Tuning

### CNI Performance Tuning

**IPAMD Configuration for High-Performance Workloads:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-node
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: aws-node
        env:
        # Aggressive IP pre-allocation for high-churn workloads
        - name: WARM_IP_TARGET
          value: "50"
        - name: MINIMUM_IP_TARGET
          value: "20"
        
        # Faster ENI allocation
        - name: MAX_ENI
          value: "8"  # Use maximum ENIs for instance type
        
        # Reduce API calls to AWS
        - name: AWS_VPC_K8S_CNI_CONFIGURE_RPFILTER
          value: "false"
        
        # Enable prefix delegation for higher pod density
        - name: ENABLE_PREFIX_DELEGATION
          value: "true"
        
        # Optimize for performance over IP conservation
        - name: WARM_PREFIX_TARGET
          value: "1"
        
        resources:
          requests:
            cpu: 25m
            memory: 128Mi
          limits:
            cpu: 100m
            memory: 256Mi
```

### Instance Type Selection for Networking

Different instance types have different networking capabilities:

```
Instance Type | Max ENIs | IPs per ENI | Max Pods | Network Performance
m5.large      | 3        | 10          | 29       | Up to 10 Gbps
m5.xlarge     | 4        | 15          | 58       | Up to 10 Gbps
m5.2xlarge    | 4        | 15          | 58       | Up to 10 Gbps
m5.4xlarge    | 8        | 30          | 234      | Up to 10 Gbps
c5n.large     | 3        | 10          | 29       | Up to 25 Gbps
c5n.xlarge    | 4        | 15          | 58       | Up to 25 Gbps
c5n.2xlarge   | 4        | 15          | 58       | Up to 25 Gbps
c5n.4xlarge   | 8        | 30          | 234      | Up to 25 Gbps
```

**Production Instance Selection Strategy:**

```yaml
# High-performance frontend nodes
nodeGroups:
  - name: frontend-high-perf
    instanceType: c5n.2xlarge  # 25 Gbps networking
    desiredCapacity: 3
    labels:
      workload-type: frontend-high-perf
      network-performance: enhanced
    
# Standard backend nodes
  - name: backend-standard
    instanceType: m5.xlarge    # 10 Gbps networking
    desiredCapacity: 6
    labels:
      workload-type: backend-standard
      
# Data processing nodes with maximum pod density
  - name: data-processing
    instanceType: m5.4xlarge   # 234 max pods
    desiredCapacity: 3
    labels:
      workload-type: data-processing
      pod-density: high
```

### Network Performance Monitoring

```yaml
# Network performance monitoring DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: network-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      name: network-monitor
  template:
    metadata:
      labels:
        name: network-monitor
    spec:
      hostNetwork: true
      containers:
      - name: network-exporter
        image: prom/node-exporter:latest
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --collector.netdev
        - --collector.netstat
        - --collector.sockstat
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

## 🌍 Multi-Region and Cross-AZ Networking

### Cross-Region Cluster Communication

For disaster recovery and global applications, you need secure communication between regions:

```yaml
# VPC Peering between regions
# Primary region: us-east-1 (10.0.0.0/16)
# DR region: us-west-2 (10.2.0.0/16)

# Create VPC peering connection
aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-12345678 \
    --peer-vpc-id vpc-87654321 \
    --peer-region us-west-2

# Accept peering connection in peer region
aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id pcx-1234567890abcdef0 \
    --region us-west-2

# Update route tables
aws ec2 create-route \
    --route-table-id rtb-12345678 \
    --destination-cidr-block 10.2.0.0/16 \
    --vpc-peering-connection-id pcx-1234567890abcdef0
```

### Cross-Region Service Discovery

```yaml
# External DNS for cross-region service discovery
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.13.1
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=techmart.internal
        - --provider=aws
        - --policy=upsert-only
        - --aws-zone-type=private
        - --registry=txt
        - --txt-owner-id=techmart-prod-us-east-1
        env:
        - name: AWS_DEFAULT_REGION
          value: us-east-1
---
# Service with cross-region DNS
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
  annotations:
    external-dns.alpha.kubernetes.io/hostname: user-service.us-east-1.techmart.internal
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

## 🔧 Advanced CNI Features and Configurations

### Prefix Delegation for Higher Pod Density

Prefix delegation allows assigning /28 prefixes to ENIs instead of individual IPs:

```yaml
# Enable prefix delegation
apiVersion: v1
kind: ConfigMap
metadata:
  name: amazon-vpc-cni
  namespace: kube-system
data:
  enable-prefix-delegation: "true"
  warm-prefix-target: "1"
  warm-ip-target: "5"
---
# DaemonSet configuration for prefix delegation
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-node
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: aws-node
        env:
        - name: ENABLE_PREFIX_DELEGATION
          value: "true"
        - name: WARM_PREFIX_TARGET
          value: "1"
        - name: WARM_IP_TARGET
          value: "5"
        - name: MINIMUM_IP_TARGET
          value: "3"
```

**Benefits of Prefix Delegation:**
- Increases pod density from 58 to 110+ pods per m5.xlarge
- Reduces API calls to AWS EC2
- Better IP utilization efficiency
- Faster pod startup times

### Security Groups for Pods

Assign specific security groups to individual pods:

```yaml
# SecurityGroupPolicy for fine-grained control
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: payment-service-sgp
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  securityGroups:
    groupIds:
    - sg-payment-service-123456  # Dedicated SG for payment pods
---
# Payment service deployment with security group
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
      annotations:
        kubernetes.io/ingress-bandwidth: 100M
        kubernetes.io/egress-bandwidth: 100M
    spec:
      containers:
      - name: payment-service
        image: techmart/payment-service:v2.1.0
        ports:
        - containerPort: 8080
        env:
        - name: PCI_COMPLIANCE_MODE
          value: "true"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
```

### IPv6 Support

Enable IPv6 for future-proofing and larger address space:

```yaml
# IPv6 cluster configuration
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: techmart-ipv6
  region: us-east-1
  version: "1.28"

kubernetesNetworkConfig:
  ipFamily: IPv6

vpc:
  enableDnsHostnames: true
  enableDnsSupport: true
  cidr: "10.0.0.0/16"
  
addons:
- name: vpc-cni
  version: latest
  configurationValues: |-
    env:
      ENABLE_IPv6: "true"
      AWS_VPC_K8S_CNI_CONFIGURE_RPFILTER: "false"
```

## 📊 Monitoring and Troubleshooting

### Comprehensive CNI Monitoring

```yaml
# CNI metrics collection
apiVersion: v1
kind: ConfigMap
metadata:
  name: cni-metrics-helper
  namespace: kube-system
data:
  config.yaml: |
    # Metrics configuration
    metrics:
      - name: awscni_assigned_ip_addresses
        help: Number of IP addresses assigned to pods
      - name: awscni_total_ip_addresses  
        help: Total number of IP addresses available
      - name: awscni_eni_allocated
        help: Number of ENIs allocated to the instance
      - name: awscni_eni_max
        help: Maximum number of ENIs that can be allocated
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cni-metrics-helper
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: cni-metrics-helper
  template:
    metadata:
      labels:
        name: cni-metrics-helper
    spec:
      hostNetwork: true
      containers:
      - name: cni-metrics-helper
        image: 602401143452.dkr.ecr.us-east-1.amazonaws.com/cni-metrics-helper:v1.12.1
        ports:
        - containerPort: 61678
          hostPort: 61678
        env:
        - name: USE_CLOUDWATCH
          value: "false"
        - name: AWS_CLUSTER_ID
          value: "techmart-production"
        - name: AWS_REGION
          value: "us-east-1"
```

### Network Troubleshooting Tools

```yaml
# Network debugging pod
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
  namespace: default
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot:latest
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "NET_RAW"]
  hostNetwork: false
  restartPolicy: Never
---
# Network policy testing pod
apiVersion: v1
kind: Pod
metadata:
  name: netpol-test
  namespace: production
  labels:
    app: netpol-test
spec:
  containers:
  - name: test
    image: busybox:1.35
    command: ["/bin/sh"]
    args: ["-c", "sleep 3600"]
```

**Common Troubleshooting Commands:**

```bash
# Check CNI plugin status
kubectl get pods -n kube-system -l k8s-app=aws-node

# View CNI logs
kubectl logs -n kube-system -l k8s-app=aws-node -c aws-node

# Check IP allocation
kubectl exec -n kube-system aws-node-xxxxx -c aws-node -- \
  /opt/cni/bin/aws-k8s-agent introspect

# Test pod connectivity
kubectl exec -it network-debug -- ping 10.0.1.100

# Check network policies
kubectl get networkpolicy -A

# Verify security groups
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# Check ENI allocation
aws ec2 describe-network-interfaces \
  --filters "Name=group-id,Values=sg-xxxxxxxxx"
```

### Performance Benchmarking

```yaml
# Network performance testing
apiVersion: batch/v1
kind: Job
metadata:
  name: network-perf-test
  namespace: default
spec:
  template:
    spec:
      containers:
      - name: iperf-server
        image: networkstatic/iperf3:latest
        command: ["iperf3", "-s"]
        ports:
        - containerPort: 5201
      - name: iperf-client
        image: networkstatic/iperf3:latest
        command: ["iperf3", "-c", "localhost", "-t", "60", "-P", "4"]
      restartPolicy: Never
```

## 🚀 Production Deployment Checklist

### Pre-Deployment Validation

```bash
#!/bin/bash
# CNI Production Readiness Checklist

echo "=== EKS CNI Production Readiness Check ==="

# 1. Verify VPC configuration
echo "Checking VPC configuration..."
aws ec2 describe-vpcs --vpc-ids vpc-xxxxxxxxx

# 2. Validate subnet IP availability
echo "Checking subnet IP availability..."
aws ec2 describe-subnets --subnet-ids subnet-xxxxxxxxx \
  --query 'Subnets[*].[SubnetId,AvailableIpAddressCount,CidrBlock]'

# 3. Verify security groups
echo "Validating security groups..."
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# 4. Check CNI version compatibility
echo "Checking CNI version..."
kubectl get daemonset aws-node -n kube-system -o yaml | grep image:

# 5. Validate node capacity
echo "Checking node networking capacity..."
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODS:.status.capacity.pods

# 6. Test network policies
echo "Testing network policies..."
kubectl auth can-i create networkpolicies

# 7. Verify monitoring setup
echo "Checking monitoring configuration..."
kubectl get pods -n monitoring | grep -E "(prometheus|grafana)"

# 8. Test cross-AZ connectivity
echo "Testing cross-AZ connectivity..."
kubectl run test-pod --image=busybox --rm -it -- ping 10.0.2.100

echo "=== Checklist Complete ==="
```

### Post-Deployment Monitoring

```yaml
# Comprehensive monitoring dashboard
apiVersion: v1
kind: ConfigMap
metadata:
  name: cni-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "EKS CNI Monitoring",
        "panels": [
          {
            "title": "IP Address Utilization",
            "targets": [
              {
                "expr": "awscni_assigned_ip_addresses / awscni_total_ip_addresses * 100"
              }
            ]
          },
          {
            "title": "ENI Utilization",
            "targets": [
              {
                "expr": "awscni_eni_allocated / awscni_eni_max * 100"
              }
            ]
          },
          {
            "title": "Network Policy Violations",
            "targets": [
              {
                "expr": "increase(networkpolicy_drop_count_total[5m])"
              }
            ]
          }
        ]
      }
    }
```

This comprehensive guide provides everything needed to design, implement, and maintain production-grade EKS networking with the AWS VPC CNI. The key to success is understanding the underlying AWS networking primitives, proper capacity planning, and implementing comprehensive monitoring and security controls.

Remember that networking decisions made early in your EKS journey are difficult to change later, so invest time in proper design upfront. Test thoroughly in non-production environments, and always have monitoring and troubleshooting tools ready before going live.