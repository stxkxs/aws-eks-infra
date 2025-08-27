# Template Reference Guide

## Overview

This guide provides detailed explanations of all Mustache templates used in the EKS infrastructure project. Templates are located in `src/main/resources/prototype/v1/` and are organized by functional area.

## Template Structure

### Configuration Template (`conf.mustache`)

The main configuration template that defines the entire infrastructure stack.

#### Host Configuration
```yaml
host:
  common:
    id: {{host:id}}                    # Infrastructure identifier
    organization: {{host:organization}} # Organization name
    account: {{host:account}}          # AWS account ID
    region: {{host:region}}            # AWS region
    name: {{host:name}}                # Service name
    alias: {{host:alias}}              # Environment alias
    environment: {{host:environment}}   # Environment type
    version: {{host:version}}          # Version identifier
    domain: {{host:domain}}            # Domain name for tagging
```

#### Hosted Configuration
Contains the actual deployment configuration with VPC and EKS settings.

**VPC Configuration:**
- **CIDR**: 10.0.0.0/16 network range
- **Subnets**: Automatic /24 subnet creation across 3 AZs
- **NAT Gateways**: 2 for cost optimization and redundancy
- **DNS**: Both DNS support and hostnames enabled

**EKS Configuration:**
- **Version**: Kubernetes 1.33
- **Endpoint Access**: Public and private hybrid mode
- **Logging**: All control plane log types enabled
- **Subnets**: Both public and private subnets for flexibility

## EKS Templates

### 1. Addons Template (`eks/addons.mustache`)

Defines all AWS managed addons and custom Helm chart installations.

#### AWS Managed Addons

**VPC CNI (`awsVpcCni`)**
```yaml
name: vpc-cni
version: v1.19.6-eksbuild.1
serviceAccount:
  metadata:
    name: aws-node
    namespace: kube-system
  role:
    managedPolicyNames:
      - AmazonEKS_CNI_Policy
```
- **Purpose**: Pod networking using AWS VPC IPs
- **Permissions**: CNI policy for network interface management
- **Configuration**: IRSA-enabled service account

**EBS CSI Driver (`awsEbsCsi`)**
```yaml
name: aws-ebs-csi-driver
version: v1.44.0-eksbuild.1
serviceAccount:
  role:
    managedPolicyNames:
      - service-role/AmazonEBSCSIDriverPolicy
    customPolicies:
      - name: eks-ebs-encryption
        policy: policy/kms-eks-ebs-encryption.mustache
defaultStorageClass: eks/storage-class.yaml
kms:
  alias: eks-ebs-encryption
  enabled: true
```
- **Purpose**: Persistent volume management with EBS
- **Encryption**: KMS encryption for all EBS volumes
- **Storage Class**: GP3 volumes as default

**Container Insights (`containerInsights`)**
```yaml
name: amazon-cloudwatch-observability
version: v4.1.0-eksbuild.1
serviceAccount:
  role:
    managedPolicyNames:
      - CloudWatchAgentServerPolicy
      - AWSXrayWriteOnlyAccess
```
- **Purpose**: CloudWatch monitoring and X-Ray tracing
- **Permissions**: CloudWatch and X-Ray write access

#### Custom Helm Chart Addons

**Cert-Manager**
```yaml
chart:
  name: cert-manager
  repository: https://charts.jetstack.io
  version: v1.17.1
  namespace: cert-manager
```
- **Purpose**: Automated TLS certificate management
- **Capabilities**: Let's Encrypt integration, certificate renewal

**Karpenter**
```yaml
chart:
  name: karpenter
  repository: oci://public.ecr.aws/karpenter/karpenter
  version: 1.5.0
  namespace: kube-system
podIdentity:
  role:
    customPolicies:
      - name: karpenter
        policy: policy/karpenter.mustache
      - name: karpenter-interrupt
        policy: policy/karpenter-interrupt.mustache
```
- **Purpose**: Advanced node autoscaling and provisioning
- **Features**: Spot instance support, just-in-time provisioning
- **Integration**: SQS for interruption handling

**AWS Load Balancer Controller**
```yaml
chart:
  name: aws-load-balancer-controller
  repository: https://aws.github.io/eks-charts
  version: 1.11.0
  namespace: aws-load-balancer
serviceAccount:
  role:
    customPolicies:
      - name: aws-load-balancer-controller
        policy: policy/aws-load-balancer-controller.mustache
```
- **Purpose**: AWS ALB/NLB integration with Kubernetes
- **Features**: Ingress controller, service integration

**CSI Secrets Store**
```yaml
csiSecretsStore:
  chart:
    name: secrets-store-csi-driver
    version: 1.4.8
    namespace: aws-secrets-store
awsSecretsStore:
  chart:
    name: secrets-store-csi-driver-provider-aws
    version: 1.0.1
    namespace: aws-secrets-store
```
- **Purpose**: Mount secrets as volumes from AWS Secrets Manager
- **Integration**: Parameter Store and Secrets Manager support

**Grafana Monitoring**
```yaml
chart:
  name: k8s-monitoring
  repository: https://grafana.github.io/helm-charts
  version: 2.0.18
  namespace: monitoring
```
- **Purpose**: Comprehensive observability with Grafana Cloud
- **Features**: Metrics, logs, and traces collection

### 2. RBAC Template (`eks/rbac.mustache`)

Defines Kubernetes Role-Based Access Control.

```yaml
userClusterRoleBinding:
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: {{hosted:id}}-eks-read-only-binding
  subjects:
    - kind: Group
      name: eks:read-only
      apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: {{hosted:id}}-eks-read-only-role

userClusterRole:
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: {{hosted:id}}-eks-read-only-role
  rules:
    - apiGroups: [""]
      resources: ["pods", "services", "deployments"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["apps"]
      resources: ["deployments", "replicasets", "statefulsets"]
      verbs: ["get", "list", "watch"]
    - apiGroups: ["batch"]
      resources: ["jobs", "cronjobs"]
      verbs: ["get", "list", "watch"]
```

**Features:**
- **Read-only Access**: Users get read-only access to core resources
- **Group Binding**: AWS IAM roles mapped to Kubernetes groups
- **Resource Scope**: Covers core Kubernetes resources

### 3. Node Groups Template (`eks/node-groups.mustache`)

Defines managed node group configuration for the EKS cluster.

```yaml
- name: {{hosted:id}}-core-node
  amiType: bottlerocket_x86_64
  forceUpdate: true
  instanceClass: m5a
  instanceSize: large
  capacityType: on_demand
  desiredSize: 2
  minSize: 2
  maxSize: 6
  role:
    name: {{hosted:id}}-core-node
    managedPolicyNames:
      - AmazonEKSWorkerNodePolicy
      - AmazonEC2ContainerRegistryReadOnly
      - AmazonSSMManagedInstanceCore
```

**Features:**
- **Bottlerocket OS**: Security-focused, immutable container OS
- **Auto Scaling**: 2-6 node capacity with desired state of 2
- **SSM Integration**: Systems Manager for secure node management
- **Container Registry**: Read-only access to ECR
- **Karpenter Discovery**: Tagged for Karpenter node discovery

### 4. Tenancy Template (`eks/tenancy.mustache`)

Defines cluster access and user management configuration.

```yaml
administrators:
{{#hosted:eks:administrators}}
  {{#role}}
      - role: {{role}}
        username: {{username}}
        email: {{email}}
  {{/role}}
{{/hosted:eks:administrators}}

users:
{{#hosted:eks:users}}
  {{#role}}
      - role: {{role}}
        username: {{username}}
        email: {{email}}
  {{/role}}
{{/hosted:eks:users}}
```

**Features:**
- **Administrator Access**: Maps AWS IAM roles to cluster administrators
- **User Access**: Maps AWS IAM roles to regular users
- **Conditional Rendering**: Empty arrays if no users/administrators defined
- **Email Association**: Links users to email addresses for notifications

### 5. Observability Template (`eks/observability.mustache`)

Comprehensive monitoring, alerting, and dashboard configuration.

**Key Components:**
- **SNS Topics**: Critical and warning alert routing
- **CloudWatch Metrics**: Custom log-based metrics for EKS events
- **Alarms**: Automated alerting for cluster health issues
- **Dashboards**: Comprehensive CloudWatch dashboard with multiple sections

**Monitoring Metrics:**
- API Server errors and latency
- Pod crash loops and failures  
- Node readiness and resource utilization
- Authentication failures
- Scheduler performance

**Dashboard Sections:**
- Cluster Overview (nodes, pods, services)
- Resource Utilization (CPU, memory, storage)
- Pod Metrics (restarts, errors, performance)
- Network Metrics (traffic, drops, EFA, RDMA)
- GPU/Neuron Metrics (utilization, memory, errors)
- API Server Performance (latency, throughput, errors)
- Scheduler Metrics (pending pods, attempts)
- Application Signals (latency, errors, JVM metrics)
- Error Log Queries

### 6. SQS Template (`eks/sqs.mustache`)

SQS queue configuration for Karpenter interruption handling.

```yaml
name: {{hosted:id}}-karpenter
retention: 300
rules:
  - name: {{hosted:id}}-eks-health
    eventPattern:
      source: ["aws.health"]
      detailType: ["AWS Health Event"]
  - name: {{hosted:id}}-eks-spot
    eventPattern:
      source: ["aws.ec2"] 
      detailType: ["EC2 Spot Instance Interruption Warning"]
  - name: {{hosted:id}}-eks-rebalance
    eventPattern:
      source: ["aws.ec2"]
      detailType: ["EC2 Instance Rebalance Recommendation"]
  - name: {{hosted:id}}-eks-state
    eventPattern:
      source: ["aws.ec2"]
      detailType: ["EC2 Instance State-change Notification"]
```

**Features:**
- **Interruption Handling**: Captures spot instance interruption warnings
- **Health Events**: AWS Health events for proactive notifications
- **Instance Lifecycle**: EC2 state change notifications
- **Rebalance Recommendations**: Proactive node replacement signals
- **Message Retention**: 5-minute retention for timely processing

## Helm Chart Templates

### 1. Karpenter Configuration (`helm/karpenter.mustache`)

```yaml
nodeSelector:
  "eks.amazonaws.com/nodegroup": {{hosted:id}}-core-node
settings:
  clusterName: {{hosted:id}}-eks
  interruptionQueue: {{hosted:id}}-karpenter
serviceAccount:
  create: false
  name: {{hosted:id}}-karpenter-sa
serviceMonitor:
  enabled: false
logLevel: debug
```

**Configuration Options:**
- **Node Selector**: Targets specific node groups for Karpenter pods
- **Cluster Integration**: Links to EKS cluster for node management
- **Interruption Queue**: SQS queue for spot instance interruptions
- **Service Account**: Uses pre-created service account with IRSA
- **Logging**: Debug level for troubleshooting

### 2. Cert-Manager Configuration (`helm/cert-manager.mustache`)

*Note: Template content needs to be examined for complete documentation*

### 3. AWS Load Balancer Configuration (`helm/aws-load-balancer.mustache`)

*Note: Template content needs to be examined for complete documentation*

### 4. Grafana Configuration (`helm/grafana.mustache`)

*Note: Template content needs to be examined for complete documentation*

### 5. CSI Secrets Store Configuration (`helm/csi-secrets-store.mustache`)

*Note: Template content needs to be examined for complete documentation*

### 6. AWS Secrets Store Configuration (`helm/aws-secrets-store.mustache`)

*Note: Template content needs to be examined for complete documentation*

## Policy Templates

### 1. Karpenter Policy (`policy/karpenter.mustache`)

Defines IAM permissions for Karpenter node management.

**Key Permissions:**
- EC2 instance management (create, terminate, describe)
- IAM PassRole for node instance profiles
- EKS cluster access for node registration
- Pricing API access for cost optimization

### 2. Karpenter Interrupt Policy (`policy/karpenter-interrupt.mustache`)

SQS permissions for spot instance interruption handling.

### 3. AWS Load Balancer Controller Policy (`policy/aws-load-balancer-controller.mustache`)

IAM permissions for ALB/NLB management.

**Key Permissions:**
- EC2 load balancer operations
- Target group management
- Security group modifications
- WAF integration (if enabled)

### 4. KMS EBS Encryption Policy (`policy/kms-eks-ebs-encryption.mustache`)

KMS permissions for EBS volume encryption.

### 5. Secret Access Policy (`policy/secret-access.mustache`)

Permissions for accessing AWS Secrets Manager and Parameter Store.

### 6. Volume Management Policy (`policy/manage-volume.mustache`)

EBS volume lifecycle management permissions.

### 7. Bucket Access Policy (`policy/bucket-access.mustache`)

S3 bucket access permissions for various services.

### 8. Karpenter Notify Policy (`policy/karpenter-notify.mustache`)

Additional notification permissions for Karpenter.

## Template Customization

### Parameter Mapping

Templates use Mustache syntax with the following parameter sources:

1. **CDK Context**: Values from `cdk.context.json`
2. **Configuration**: Values from the main configuration template
3. **Runtime**: Values computed during CDK synthesis

### Common Parameters

- `{{hosted:id}}`: Infrastructure identifier
- `{{hosted:account}}`: AWS account ID
- `{{hosted:region}}`: AWS region
- `{{hosted:domain}}`: Domain for resource tagging
- `{{hosted:organization}}`: Organization name

### Adding New Templates

1. **Location**: Place in appropriate subdirectory under `prototype/v1/`
2. **Naming**: Use descriptive names with `.mustache` extension
3. **Parameters**: Follow existing parameter naming conventions
4. **Validation**: Test template rendering with sample data

### Template Testing

```bash
# Validate template syntax
mustache --check template.mustache

# Render template with test data
mustache test-data.yaml template.mustache
```

## Best Practices

### 1. Parameter Usage
- Use descriptive parameter names
- Avoid hardcoded values where possible
- Provide default values when appropriate
- Document parameter requirements

### 2. Template Organization
- Group related configurations together
- Use consistent indentation and formatting
- Include comments for complex configurations
- Separate concerns into different templates

### 3. Version Management
- Pin specific versions for stability
- Document version upgrade procedures
- Test version changes in non-production
- Maintain compatibility matrices

### 4. Security Considerations
- Follow principle of least privilege
- Avoid embedding secrets in templates
- Use IAM roles and service accounts properly
- Enable encryption where supported

### 5. Maintenance
- Regularly update component versions
- Monitor for security advisories
- Test template changes thoroughly
- Document breaking changes