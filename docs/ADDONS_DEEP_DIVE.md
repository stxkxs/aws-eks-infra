# Addon Deep-Dive Documentation

## Overview

This document provides comprehensive details about each addon and Helm chart included in the EKS infrastructure, including their purpose, configuration options, troubleshooting, and integration patterns.

## AWS Managed Addons

### 1. VPC CNI (Amazon VPC Container Network Interface)

#### Purpose and Benefits
The VPC CNI provides native AWS networking for Kubernetes pods by assigning AWS VPC IP addresses directly to pods, enabling:
- **Native AWS Networking**: Pods get VPC IP addresses for seamless AWS service integration
- **High Performance**: Direct VPC networking without overlay networks
- **Security**: Native VPC security groups and NACLs apply to pods
- **Scalability**: Supports large-scale deployments with efficient IP allocation

#### Configuration Details
```yaml
awsVpcCni:
  name: vpc-cni
  version: v1.19.6-eksbuild.1
  preserveOnDelete: false
  resolveConflicts: preserve
  serviceAccount:
    metadata:
      name: aws-node
      namespace: kube-system
    role:
      name: {{hosted:id}}-vpc-cni
      managedPolicyNames:
        - AmazonEKS_CNI_Policy
```

#### Key Features
- **IP Address Management**: Efficient ENI and IP allocation
- **Network Performance**: High-performance networking with SR-IOV support
- **Security Groups for Pods**: Pod-level security group assignment
- **IPv4/IPv6 Support**: Dual-stack networking capabilities

#### Troubleshooting Common Issues

**Issue: Pod IP Allocation Failures**
```bash
# Check CNI plugin logs
kubectl logs -n kube-system -l k8s-app=aws-node

# Verify ENI capacity
aws ec2 describe-instances --instance-ids $(kubectl get nodes -o jsonpath='{.items[*].spec.providerID}' | tr '/' ' ' | awk '{print $NF}')

# Check IP allocation
kubectl describe node <node-name> | grep "pods:"
```

**Issue: Network Connectivity Problems**
```bash
# Test pod-to-pod communication
kubectl run test1 --image=busybox --rm -it -- ping <pod-ip>

# Check security groups
aws ec2 describe-security-groups --group-ids <sg-id>

# Verify route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"
```

#### Performance Tuning
```yaml
# CNI configuration optimizations
spec:
  containers:
  - name: aws-node
    env:
    - name: ENABLE_PREFIX_DELEGATION
      value: "true"
    - name: WARM_PREFIX_TARGET
      value: "2"
    - name: MINIMUM_IP_TARGET
      value: "10"
```

### 2. EBS CSI Driver (Elastic Block Store Container Storage Interface)

#### Purpose and Benefits
The EBS CSI driver enables dynamic provisioning and management of Amazon EBS volumes for persistent storage:
- **Dynamic Provisioning**: Automatic volume creation and attachment
- **Volume Expansion**: Online volume resize capabilities
- **Snapshot Support**: Volume backup and restore functionality
- **Encryption**: KMS encryption for data at rest

#### Configuration Details
```yaml
awsEbsCsi:
  name: aws-ebs-csi-driver
  version: v1.44.0-eksbuild.1
  serviceAccount:
    role:
      managedPolicyNames:
        - service-role/AmazonEBSCSIDriverPolicy
      customPolicies:
        - name: {{hosted:id}}-eks-ebs-encryption
          policy: policy/kms-eks-ebs-encryption.mustache
  defaultStorageClass: eks/storage-class.yaml
  kms:
    alias: {{hosted:id}}-eks-ebs-encryption
    enabled: true
```

#### Storage Classes
```yaml
# Default GP3 encrypted storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: alias/{{hosted:id}}-eks-ebs-encryption
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

#### Volume Types and Use Cases
- **gp3**: General purpose, balanced price/performance
- **io1/io2**: High IOPS for databases and intensive workloads
- **st1**: Throughput optimized for big data and data warehouses
- **sc1**: Cold storage for infrequently accessed data

#### Troubleshooting Storage Issues

**Issue: PVC Stuck in Pending**
```bash
# Check PVC status
kubectl describe pvc <pvc-name>

# Check storage class
kubectl get storageclass

# Verify CSI driver
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl logs -n kube-system -l app=ebs-csi-controller
```

**Issue: Volume Mount Failures**
```bash
# Check node CSI driver
kubectl get pods -n kube-system -l app=ebs-csi-node

# Verify volume attachment
aws ec2 describe-volumes --volume-ids <volume-id>

# Check node capacity
kubectl describe node <node-name> | grep "Attached Volumes"
```

#### Performance Optimization
```yaml
# High-performance storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "4000"
  throughput: "250"
  encrypted: "true"
```

### 3. CoreDNS (DNS Server for Service Discovery)

#### Purpose and Benefits
CoreDNS provides cluster DNS services for service discovery and name resolution:
- **Service Discovery**: Automatic DNS records for services and pods
- **Custom DNS**: Support for custom DNS configurations
- **Caching**: DNS response caching for improved performance
- **Extensibility**: Plugin-based architecture for customization

#### Configuration Details
```yaml
coreDns:
  name: coredns
  version: v1.12.1-eksbuild.2
  preserveOnDelete: false
  resolveConflicts: overwrite
```

#### DNS Resolution Flow
```
Pod DNS Query → CoreDNS → Upstream DNS (if external) → Response
                     ↓
              Kubernetes API (for cluster services)
```

#### Custom DNS Configuration
```yaml
# CoreDNS ConfigMap customization
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    company.local:53 {
        errors
        cache 30
        forward . 10.0.0.10
    }
```

#### Troubleshooting DNS Issues

**Issue: DNS Resolution Failures**
```bash
# Test DNS resolution
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Verify DNS configuration
kubectl get configmap coredns -n kube-system -o yaml
```

**Issue: Slow DNS Resolution**
```bash
# Check DNS performance
kubectl run dns-perf --image=busybox --rm -it -- time nslookup kubernetes.default

# Monitor DNS metrics
kubectl top pods -n kube-system -l k8s-app=kube-dns
```

### 4. Kube Proxy (Network Proxy for Services)

#### Purpose and Benefits
Kube Proxy maintains network rules for service communication:
- **Service Load Balancing**: Distributes traffic across service endpoints
- **Network Rules**: Manages iptables/IPVS rules for service routing
- **Session Affinity**: Supports sticky sessions when needed
- **External Traffic**: Handles ingress traffic routing

#### Configuration Details
```yaml
kubeProxy:
  name: kube-proxy
  version: v1.33.0-eksbuild.2
  preserveOnDelete: false
  resolveConflicts: overwrite
```

#### Proxy Modes
- **iptables**: Default mode using iptables rules
- **IPVS**: High-performance mode for large clusters
- **kernelspace**: Legacy mode (not recommended)

#### Troubleshooting Network Issues

**Issue: Service Connectivity Problems**
```bash
# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Verify service endpoints
kubectl get endpoints <service-name>

# Check iptables rules
kubectl exec -n kube-system <kube-proxy-pod> -- iptables -t nat -L
```

### 5. Pod Identity Agent (IAM Roles for Service Accounts)

#### Purpose and Benefits
Pod Identity Agent enables secure AWS API access for pods:
- **Secure Credentials**: Temporary AWS credentials via OIDC
- **Fine-grained Permissions**: Service account level IAM policies
- **No Long-term Keys**: Eliminates need for static AWS keys
- **Audit Trail**: Complete audit trail of AWS API calls

#### Configuration Details
```yaml
podIdentityAgent:
  name: eks-pod-identity-agent
  version: v1.3.7-eksbuild.2
  preserveOnDelete: false
  resolveConflicts: overwrite
```

#### IRSA Setup Pattern
```yaml
# Service account with IAM role
serviceAccount:
  metadata:
    name: app-service-account
    namespace: default
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/app-role
  role:
    name: app-role
    customPolicies:
      - name: app-policy
        policy: |
          {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Allow",
                "Action": ["s3:GetObject"],
                "Resource": "arn:aws:s3:::app-bucket/*"
              }
            ]
          }
```

#### Troubleshooting IRSA Issues

**Issue: AWS API Access Denied**
```bash
# Check service account annotations
kubectl describe serviceaccount <sa-name> -n <namespace>

# Verify IAM role trust policy
aws iam get-role --role-name <role-name>

# Test AWS credentials in pod
kubectl exec -it <pod-name> -- aws sts get-caller-identity
```

### 6. Container Insights (CloudWatch Monitoring)

#### Purpose and Benefits
Container Insights provides comprehensive observability:
- **Metrics Collection**: Container, pod, and node metrics
- **Log Aggregation**: Centralized logging for troubleshooting
- **X-Ray Integration**: Distributed tracing capabilities
- **Performance Insights**: Resource utilization analysis

#### Configuration Details
```yaml
containerInsights:
  name: amazon-cloudwatch-observability
  version: v4.1.0-eksbuild.1
  serviceAccount:
    role:
      managedPolicyNames:
        - CloudWatchAgentServerPolicy
        - AWSXrayWriteOnlyAccess
```

#### Metrics and Dashboards
- **Cluster Overview**: Cluster-level resource utilization
- **Node Performance**: Node CPU, memory, disk usage
- **Pod Insights**: Pod-level metrics and logs
- **Service Map**: X-Ray service dependency mapping

#### Custom Metrics
```yaml
# Custom CloudWatch metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: cwagentconfig
  namespace: amazon-cloudwatch
data:
  cwagentconfig.json: |
    {
      "metrics": {
        "namespace": "CustomApp/Metrics",
        "metrics_collected": {
          "cpu": {
            "measurement": ["cpu_usage_idle", "cpu_usage_iowait"]
          }
        }
      }
    }
```

## Custom Helm Chart Addons

### 1. Cert-Manager (Certificate Management)

#### Purpose and Benefits
Cert-Manager automates TLS certificate lifecycle management:
- **Automated Issuance**: Automatic certificate provisioning
- **Renewal Management**: Automatic certificate renewal
- **Multiple Issuers**: Support for Let's Encrypt, private CAs
- **Kubernetes Integration**: Native Kubernetes certificate resources

#### Configuration Details
```yaml
certManager:
  chart:
    name: cert-manager
    repository: https://charts.jetstack.io
    release: cert-manager
    version: v1.17.1
    namespace: cert-manager
    values: helm/cert-manager.mustache
```

#### Issuer Configuration
```yaml
# Let's Encrypt staging issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@company.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: alb
```

#### Certificate Request
```yaml
# Certificate resource
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: default
spec:
  secretName: app-tls-secret
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
  - app.company.com
```

#### Troubleshooting Certificate Issues

**Issue: Certificate Not Issued**
```bash
# Check certificate status
kubectl describe certificate <cert-name>

# Check certificate request
kubectl describe certificaterequest <cr-name>

# Check issuer status
kubectl describe clusterissuer <issuer-name>

# Check cert-manager logs
kubectl logs -n cert-manager -l app.kubernetes.io/name=cert-manager
```

### 2. AWS Load Balancer Controller

#### Purpose and Benefits
AWS Load Balancer Controller manages AWS load balancers for Kubernetes:
- **ALB Integration**: Application Load Balancer for HTTP/HTTPS
- **NLB Integration**: Network Load Balancer for TCP/UDP
- **Ingress Support**: Kubernetes Ingress resource support
- **Target Groups**: Direct pod targeting for improved performance

#### Configuration Details
```yaml
awsLoadBalancer:
  chart:
    name: aws-load-balancer-controller
    repository: https://aws.github.io/eks-charts
    release: aws-load-balancer-controller
    version: 1.11.0
    namespace: aws-load-balancer
    values: helm/aws-load-balancer.mustache
  serviceAccount:
    role:
      customPolicies:
        - name: aws-load-balancer-controller
          policy: policy/aws-load-balancer-controller.mustache
```

#### Ingress Examples

**Application Load Balancer Ingress**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/cert-id
spec:
  rules:
  - host: app.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

**Network Load Balancer Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: backend-app
```

#### Troubleshooting Load Balancer Issues

**Issue: Load Balancer Not Created**
```bash
# Check controller logs
kubectl logs -n aws-load-balancer -l app.kubernetes.io/name=aws-load-balancer-controller

# Check ingress events
kubectl describe ingress <ingress-name>

# Verify IAM permissions
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names elasticloadbalancing:CreateLoadBalancer --resource-arns "*"
```

### 3. Karpenter (Node Autoscaling)

#### Purpose and Benefits
Karpenter provides advanced node autoscaling capabilities:
- **Just-in-Time Provisioning**: Rapid node provisioning for pending pods
- **Cost Optimization**: Right-sized instances based on workload requirements
- **Spot Instance Support**: Automatic spot instance utilization
- **Multi-AZ Scheduling**: Intelligent placement across availability zones

#### Configuration Details
```yaml
karpenter:
  chart:
    name: karpenter
    repository: oci://public.ecr.aws/karpenter/karpenter
    release: karpenter
    version: 1.5.0
    namespace: kube-system
    values: helm/karpenter.mustache
  podIdentity:
    role:
      customPolicies:
        - name: karpenter
          policy: policy/karpenter.mustache
        - name: karpenter-interrupt
          policy: policy/karpenter-interrupt.mustache
```

#### Node Pool Configuration
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        node-type: karpenter
    spec:
      nodeClassRef:
        name: default
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m5.large", "m5.xlarge", "m5.2xlarge"]
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
```

#### EC2NodeClass Configuration
```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: "{{hosted:id}}-vpc"
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: "{{hosted:id}}-vpc"
  instanceStorePolicy: RAID0
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh {{hosted:id}}-eks
```

#### Troubleshooting Karpenter Issues

**Issue: Nodes Not Provisioning**
```bash
# Check Karpenter logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter

# Check pending pods
kubectl get pods -A --field-selector=status.phase=Pending

# Verify node pool configuration
kubectl describe nodepool default

# Check IAM permissions
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter | grep -i permission
```

**Issue: Spot Instance Interruptions**
```bash
# Check interruption queue
aws sqs get-queue-attributes --queue-url <queue-url>

# Monitor spot interruption logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter | grep -i interrupt

# Check node events
kubectl get events --field-selector involvedObject.kind=Node
```

### 4. CSI Secrets Store (Secret Management)

#### Purpose and Benefits
CSI Secrets Store enables mounting secrets from external systems:
- **Volume Integration**: Mount secrets as files in pods
- **Multiple Providers**: Support for AWS, Azure, GCP, Vault
- **Automatic Rotation**: Secret rotation without pod restart
- **Secure Access**: No secrets in environment variables

#### Configuration Details
```yaml
csiSecretsStore:
  chart:
    name: secrets-store-csi-driver
    repository: https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
    release: csi-secrets
    version: 1.4.8
    namespace: aws-secrets-store

awsSecretsStore:
  chart:
    name: secrets-store-csi-driver-provider-aws
    repository: https://aws.github.io/secrets-store-csi-driver-provider-aws
    release: aws-secrets
    version: 1.0.1
    namespace: aws-secrets-store
```

#### SecretProviderClass Example
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "database-password"
        objectType: "secretsmanager"
        objectAlias: "db-password"
      - objectName: "api-key"
        objectType: "ssmparameter"
        objectAlias: "api-key"
  secretObjects:
  - secretName: app-secret
    type: Opaque
    data:
    - objectName: db-password
      key: password
```

#### Pod Configuration
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: app-service-account
  containers:
  - name: app
    image: app:latest
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: app-secrets
```

#### Troubleshooting Secrets Issues

**Issue: Secret Mount Failures**
```bash
# Check CSI driver logs
kubectl logs -n aws-secrets-store -l app=secrets-store-csi-driver

# Check AWS provider logs
kubectl logs -n aws-secrets-store -l app=secrets-store-provider-aws

# Verify SecretProviderClass
kubectl describe secretproviderclass <spc-name>

# Check IAM permissions
aws secretsmanager get-secret-value --secret-id <secret-name>
```

### 5. Grafana K8s Monitoring

#### Purpose and Benefits
Grafana K8s Monitoring provides comprehensive observability:
- **Unified Observability**: Metrics, logs, and traces in one platform
- **Cloud Integration**: Direct integration with Grafana Cloud
- **Pre-built Dashboards**: Ready-to-use Kubernetes dashboards
- **Alerting**: Advanced alerting and notification capabilities

#### Configuration Details
```yaml
grafana:
  chart:
    name: k8s-monitoring
    repository: https://grafana.github.io/helm-charts
    release: k8s-monitoring
    version: 2.0.18
    namespace: monitoring
    values: helm/grafana.mustache
```

#### Monitoring Stack Components
- **Grafana Agent**: Metrics and logs collection
- **Prometheus**: Metrics storage and querying
- **Loki**: Log aggregation and querying
- **Tempo**: Distributed tracing
- **Alertmanager**: Alert routing and management

#### Custom Dashboards
```yaml
# Custom dashboard configuration
grafana:
  dashboards:
    custom:
      application-metrics:
        gnetId: 12345
        revision: 1
        datasource: Prometheus
      error-tracking:
        file: dashboards/errors.json
```

#### Alerting Rules
```yaml
# Prometheus alerting rules
groups:
- name: kubernetes-alerts
  rules:
  - alert: PodCrashLooping
    expr: increase(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} is crash looping"
      
  - alert: NodeHighCPU
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Node {{ $labels.instance }} has high CPU usage"
```

#### Troubleshooting Monitoring Issues

**Issue: Metrics Not Appearing**
```bash
# Check Grafana Agent logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana-agent

# Verify Prometheus targets
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090/targets

# Check service discovery
kubectl get servicemonitors -A
kubectl get podmonitors -A
```

## Integration Patterns

### Cross-Addon Dependencies

#### Certificate Management Flow
```
Cert-Manager → AWS Load Balancer Controller → Application Pods
     ↓
Let's Encrypt/ACM → TLS Certificates → Secure Ingress
```

#### Storage and Security Flow
```
EBS CSI Driver → KMS Encryption → Encrypted Volumes
     ↓
Pod Identity → IAM Roles → Secure AWS API Access
     ↓
CSI Secrets Store → AWS Secrets Manager → Application Secrets
```

#### Observability Chain
```
Applications → Grafana Agent → Grafana Cloud
     ↓
Container Insights → CloudWatch → AWS X-Ray
     ↓
Karpenter Metrics → Cost Optimization
```

### Best Practices for Addon Management

#### 1. Version Management
- Pin specific versions for stability
- Test updates in non-production environments
- Maintain compatibility matrices
- Plan upgrade sequences

#### 2. Resource Management
- Set appropriate resource limits
- Monitor addon resource consumption
- Use node selectors for critical addons
- Implement proper anti-affinity rules

#### 3. Security Considerations
- Follow least privilege principle for IAM roles
- Regularly update addon versions for security patches
- Implement network policies where applicable
- Monitor addon access patterns

#### 4. Operational Excellence
- Implement comprehensive monitoring
- Maintain disaster recovery procedures
- Document troubleshooting procedures
- Automate routine maintenance tasks

## Future Enhancements

### Planned Addon Additions
- **Service Mesh**: Istio or Linkerd integration
- **Policy Engine**: Open Policy Agent (OPA) Gatekeeper
- **Backup Solution**: Velero for cluster backups
- **Security Scanning**: Falco for runtime security

### Optimization Opportunities
- **Multi-cluster Management**: ArgoCD or Flux for GitOps
- **Cost Optimization**: Additional cost monitoring tools
- **Performance**: Advanced performance monitoring
- **Compliance**: Additional compliance and audit tools