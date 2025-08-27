# Operational Runbooks

## Overview

This document provides comprehensive operational procedures, troubleshooting guides, and maintenance tasks for the EKS infrastructure. These runbooks are designed to help operators maintain, monitor, and troubleshoot the platform effectively.

## Deployment Operations

### Initial Deployment

#### Prerequisites Checklist
- [ ] AWS CLI configured with appropriate permissions
- [ ] CDK CLI installed and configured
- [ ] Java 21+ and Maven installed
- [ ] GitHub CLI configured (optional)
- [ ] CDK Bootstrap completed for target account/region

#### Deployment Procedure
```bash
# 1. Clone and build common dependencies
gh repo clone stxkxs/cdk-common
mvn -f cdk-common/pom.xml clean install

# 2. Build EKS infrastructure
mvn -f aws-eks-infra/pom.xml clean install

# 3. Configure context
cp cdk.context.template.json cdk.context.json
# Edit cdk.context.json with your AWS account details

# 4. Deploy infrastructure
cd aws-eks-infra
cdk synth
cdk deploy

# 5. Configure kubectl
aws eks update-kubeconfig --name {{hosted:id}}-eks --region <region>
kubectl get nodes
```

#### Deployment Validation
```bash
# Verify cluster status
kubectl get nodes
kubectl get pods -A

# Check managed addons
kubectl get pods -n kube-system
kubectl get pods -n aws-load-balancer
kubectl get pods -n cert-manager
kubectl get pods -n monitoring

# Verify storage class
kubectl get storageclass

# Test connectivity
kubectl run test-pod --image=nginx --rm -it -- /bin/bash
```

### Update Procedures

#### EKS Version Upgrade
```bash
# 1. Check current version
aws eks describe-cluster --name {{hosted:id}}-eks --query cluster.version

# 2. Update configuration
# Edit conf.mustache to update EKS version
eks:
  version: "1.34"  # Update to new version

# 3. Deploy changes
cdk deploy

# 4. Update node groups (if needed)
# Node groups will be updated automatically or require manual intervention
```

#### Addon Updates
```bash
# 1. Check current addon versions
aws eks describe-addon --cluster-name {{hosted:id}}-eks --addon-name vpc-cni

# 2. Update addon configuration
# Edit addons.mustache with new versions

# 3. Deploy updates
cdk deploy

# 4. Verify addon status
kubectl get pods -n kube-system
```

### Rollback Procedures

#### EKS Cluster Rollback
```bash
# 1. Identify last known good configuration
git log --oneline

# 2. Revert configuration changes
git checkout <last-good-commit>

# 3. Redeploy
cdk deploy

# 4. Verify cluster state
kubectl get nodes
kubectl get pods -A
```

## Monitoring and Alerting

### Health Monitoring

#### Cluster Health Checks
```bash
# Cluster status
kubectl get cs
kubectl cluster-info

# Node status
kubectl get nodes
kubectl describe nodes

# Pod status across namespaces
kubectl get pods -A --field-selector=status.phase!=Running

# Resource utilization
kubectl top nodes
kubectl top pods -A
```

#### Application Health Checks
```bash
# Check addon health
kubectl get pods -n kube-system
kubectl get pods -n aws-load-balancer
kubectl get pods -n cert-manager
kubectl get pods -n monitoring
kubectl get pods -n aws-secrets-store

# Check service endpoints
kubectl get endpoints -A

# Check ingress controllers
kubectl get ingress -A
```

### Performance Monitoring

#### Resource Utilization
```bash
# Node resource usage
kubectl describe nodes | grep -A 5 "Allocated resources"

# Pod resource usage
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Storage usage
kubectl get pv
kubectl get pvc -A
```

#### Network Monitoring
```bash
# Service connectivity
kubectl get svc -A

# Network policies (if implemented)
kubectl get networkpolicies -A

# DNS resolution
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default
```

### Log Management

#### Accessing Logs
```bash
# Pod logs
kubectl logs -f <pod-name> -n <namespace>
kubectl logs --previous <pod-name> -n <namespace>

# Multiple containers
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Aggregated logs
kubectl logs -l app=<app-label> -n <namespace>
```

#### CloudWatch Logs
```bash
# Access EKS control plane logs
aws logs describe-log-groups --log-group-name-prefix "/aws/eks"

# Stream logs (replace with your cluster name)
aws logs tail /aws/eks/{{hosted:id}}-eks/cluster --follow
```

### Alerting Configuration

#### CloudWatch Alarms
- **Node CPU Utilization > 80%**
- **Node Memory Utilization > 85%**
- **Pod Restart Count > 5**
- **Failed Pod Count > 0**
- **EKS API Server Errors > 10/min**

#### Grafana Alerts
- **Karpenter Node Provisioning Failures**
- **Certificate Expiration Warnings**
- **Storage Usage > 85%**
- **Network Policy Violations**

## Troubleshooting Guides

### Common Issues

#### 1. Pods Not Starting

**Symptoms:**
- Pods stuck in Pending state
- ImagePullBackOff errors
- CrashLoopBackOff

**Diagnostic Steps:**
```bash
# Check pod status
kubectl describe pod <pod-name> -n <namespace>

# Check node resources
kubectl describe nodes

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Check image pull secrets
kubectl get secrets -n <namespace>
```

**Common Solutions:**
- Resource constraints: Increase node capacity or adjust requests/limits
- Image issues: Verify image name, tag, and registry access
- Configuration errors: Check ConfigMaps and Secrets

#### 2. Network Connectivity Issues

**Symptoms:**
- Services not reachable
- DNS resolution failures
- Intermittent connectivity

**Diagnostic Steps:**
```bash
# Test DNS resolution
kubectl run test-dns --image=busybox --rm -it -- nslookup kubernetes.default

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Test connectivity between pods
kubectl exec -it <pod-name> -n <namespace> -- nc -zv <service-name> <port>

# Check network policies
kubectl get networkpolicies -A
```

**Common Solutions:**
- DNS issues: Restart CoreDNS pods
- Service discovery: Verify service labels and selectors
- Network policies: Review and adjust network policy rules

#### 3. Storage Issues

**Symptoms:**
- PVCs stuck in Pending
- Mount failures
- Storage full errors

**Diagnostic Steps:**
```bash
# Check PVC status
kubectl get pvc -A
kubectl describe pvc <pvc-name> -n <namespace>

# Check storage class
kubectl get storageclass
kubectl describe storageclass gp3-encrypted

# Check CSI driver
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

**Common Solutions:**
- Provisioning issues: Check EBS CSI driver and IAM permissions
- Capacity issues: Monitor volume usage and expand if needed
- Mount issues: Verify node permissions and SELinux contexts

#### 4. Karpenter Issues

**Symptoms:**
- Nodes not provisioning
- Spot instance interruptions
- Scheduling failures

**Diagnostic Steps:**
```bash
# Check Karpenter logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter

# Check node pools
kubectl get nodepools

# Check pending pods
kubectl get pods -A --field-selector=status.phase=Pending

# Check interruption queue
aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All
```

**Common Solutions:**
- Provisioning failures: Check IAM permissions and instance limits
- Cost optimization: Review instance type selection
- Interruption handling: Verify SQS queue configuration

### Performance Troubleshooting

#### High CPU Usage
```bash
# Identify high CPU pods
kubectl top pods -A --sort-by=cpu

# Check node CPU pressure
kubectl describe nodes | grep -A 10 "Conditions"

# Analyze specific pod
kubectl exec -it <pod-name> -n <namespace> -- top
```

#### Memory Issues
```bash
# Identify memory-intensive pods
kubectl top pods -A --sort-by=memory

# Check for OOMKilled pods
kubectl get pods -A -o wide | grep OOMKilled

# Analyze memory usage
kubectl describe pod <pod-name> -n <namespace>
```

#### Storage Performance
```bash
# Check I/O wait
kubectl exec -it <pod-name> -n <namespace> -- iostat

# Monitor disk usage
kubectl exec -it <pod-name> -n <namespace> -- df -h

# Check for storage bottlenecks
kubectl get events --field-selector reason=VolumeMount
```

## Backup and Recovery

### Data Backup Procedures

#### EBS Volume Snapshots
```bash
# Create manual snapshot
aws ec2 create-snapshot --volume-id <volume-id> --description "Manual backup"

# List snapshots
aws ec2 describe-snapshots --owner-ids self --filters "Name=tag:cluster,Values={{hosted:id}}-eks"

# Automated snapshots via DLM (Data Lifecycle Manager)
aws dlm create-lifecycle-policy --execution-role-arn <role-arn> --description "EKS backup policy"
```

#### Application Data Backup
```bash
# Database backups (example for PostgreSQL)
kubectl exec -it postgres-pod -n database -- pg_dump -U user dbname > backup.sql

# File system backups
kubectl exec -it <pod-name> -n <namespace> -- tar -czf /backup/data.tar.gz /data
```

#### Configuration Backup
```bash
# Export all Kubernetes resources
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml

# Export specific resources
kubectl get configmaps,secrets -A -o yaml > configs-backup.yaml

# CDK configuration
git add . && git commit -m "Configuration backup $(date)"
```

### Disaster Recovery Procedures

#### Cluster Recovery
```bash
# 1. Assess damage
kubectl get nodes
kubectl get pods -A

# 2. Restore from CDK
cdk destroy  # If necessary
cdk deploy

# 3. Restore data
# Restore from EBS snapshots
aws ec2 create-volume --snapshot-id <snapshot-id> --availability-zone <az>

# 4. Verify recovery
kubectl get nodes
kubectl get pods -A
```

#### Data Recovery
```bash
# Restore from PV snapshots
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: restored-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: <restored-volume-id>
    fsType: ext4
EOF
```

### Recovery Testing

#### Regular Recovery Drills
1. **Monthly**: Test EBS snapshot restoration
2. **Quarterly**: Full cluster rebuild from CDK
3. **Annually**: Complete disaster recovery simulation

#### Validation Procedures
```bash
# Data integrity checks
kubectl exec -it <pod-name> -n <namespace> -- checksum verify

# Application functionality tests
kubectl run test-app --image=<test-image> --rm -it

# Performance validation
kubectl run performance-test --image=<perf-image> --rm -it
```

## Scaling Operations

### Manual Scaling

#### Node Group Scaling
```bash
# Scale managed node group
aws eks update-nodegroup-config --cluster-name {{hosted:id}}-eks --nodegroup-name <nodegroup-name> --scaling-config minSize=1,maxSize=10,desiredSize=5

# Verify scaling
kubectl get nodes
```

#### Pod Scaling
```bash
# Scale deployment
kubectl scale deployment <deployment-name> -n <namespace> --replicas=5

# Scale statefulset
kubectl scale statefulset <statefulset-name> -n <namespace> --replicas=3
```

### Auto-scaling Configuration

#### Horizontal Pod Autoscaler (HPA)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### Vertical Pod Autoscaler (VPA)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  updatePolicy:
    updateMode: "Auto"
```

### Capacity Planning

#### Resource Analysis
```bash
# Historical resource usage
kubectl top nodes --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Resource requests vs limits
kubectl describe quota -A

# Capacity recommendations
# Use metrics from CloudWatch and Grafana for trend analysis
```

#### Growth Planning
1. **Monitor**: Continuous resource utilization monitoring
2. **Predict**: Use historical data for growth predictions
3. **Plan**: Prepare scaling strategies for anticipated growth
4. **Test**: Regular load testing to validate scaling

## Maintenance Procedures

### Regular Maintenance Tasks

#### Daily Tasks
- [ ] Check cluster health status
- [ ] Review pod restart counts
- [ ] Monitor resource utilization
- [ ] Check backup completion

#### Weekly Tasks
- [ ] Review security alerts
- [ ] Update documentation
- [ ] Performance analysis
- [ ] Cost optimization review

#### Monthly Tasks
- [ ] Security patching
- [ ] Addon version updates
- [ ] Disaster recovery testing
- [ ] Capacity planning review

#### Quarterly Tasks
- [ ] EKS version planning
- [ ] Architecture review
- [ ] Security audit
- [ ] Performance optimization

### Patching and Updates

#### Security Patching
```bash
# Check for available patches
aws eks describe-addon-versions --addon-name vpc-cni

# Apply patches
# Update configuration files and redeploy
cdk deploy

# Verify patch application
kubectl get pods -n kube-system
```

#### Node Updates
```bash
# Bottlerocket nodes update automatically
# Monitor update progress
kubectl get nodes -o wide

# Force node replacement if needed
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
```

### Performance Optimization

#### Resource Optimization
```bash
# Identify over-provisioned resources
kubectl describe nodes | grep -A 10 "Allocated resources"

# Right-size workloads
kubectl set resources deployment <deployment-name> --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi
```

#### Cost Optimization
- **Spot Instances**: Use Karpenter for spot instance utilization
- **Right-sizing**: Regular resource requirement analysis
- **Scheduling**: Optimize pod scheduling and affinity rules
- **Storage**: Use appropriate storage classes and cleanup unused volumes

## Emergency Procedures

### Critical Issue Response

#### Severity 1 - Cluster Down
1. **Immediate Assessment**: Check AWS service health
2. **Escalation**: Alert on-call engineer
3. **Diagnosis**: Check CloudWatch logs and events
4. **Mitigation**: Implement immediate workarounds
5. **Resolution**: Execute recovery procedures
6. **Communication**: Update stakeholders

#### Severity 2 - Degraded Performance
1. **Identification**: Isolate affected components
2. **Analysis**: Performance bottleneck analysis
3. **Mitigation**: Apply temporary fixes
4. **Resolution**: Implement permanent solution
5. **Validation**: Verify performance restoration

### Contact Information

#### Escalation Matrix
- **Level 1**: Platform Team (Response: 15 minutes)
- **Level 2**: Senior Engineers (Response: 30 minutes)
- **Level 3**: Architect/Management (Response: 1 hour)

#### Communication Channels
- **Slack**: #platform-alerts
- **Email**: platform-team@company.com
- **PagerDuty**: EKS platform service

### Post-Incident Procedures

#### Incident Documentation
1. **Timeline**: Detailed incident timeline
2. **Root Cause**: Root cause analysis
3. **Impact**: Business impact assessment
4. **Lessons Learned**: Key takeaways
5. **Action Items**: Preventive measures

#### Improvement Process
1. **Review**: Post-incident review meeting
2. **Documentation**: Update runbooks and procedures
3. **Training**: Update team training materials
4. **Monitoring**: Enhance monitoring and alerting