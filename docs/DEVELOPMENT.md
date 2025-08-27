# Development & Contribution Guide

## Overview

This guide provides comprehensive information for developers who want to contribute to, modify, or extend the EKS infrastructure project. It covers code organization, development workflows, testing strategies, and contribution guidelines.

## Project Structure

### Repository Organization

```
aws-eks-infra/
├── src/
│   └── main/
│       ├── java/                     # CDK Java application code
│       │   └── io/stxkxs/eks/
│       │       ├── Launch.java       # Main application entry point
│       │       └── stack/
│       │           ├── DeploymentConf.java    # Configuration model
│       │           └── DeploymentStack.java   # Main CDK stack
│       └── resources/
│           └── prototype/v1/         # Template configurations
│               ├── conf.mustache     # Main configuration template
│               ├── eks/              # EKS-specific templates
│               ├── helm/             # Helm chart configurations
│               └── policy/           # IAM policy templates
├── docs/                             # Documentation
├── cdk.json                          # CDK configuration
├── cdk.context.template.json         # Context template
├── pom.xml                          # Maven build configuration
└── README.md                        # Project overview
```

### Code Organization

#### Java Source Structure
- **Launch.java**: Application entry point and CDK app initialization
- **DeploymentConf.java**: Configuration data model with validation
- **DeploymentStack.java**: Main CDK stack orchestrating nested stacks

#### Resource Templates
- **Configuration Templates**: Mustache templates for dynamic configuration
- **Policy Templates**: IAM policy definitions with parameterization
- **Helm Values**: Custom values for Helm chart deployments

## Development Environment Setup

### Prerequisites

#### Required Tools
```bash
# Java Development Kit 21+
sdk install java 21.0.1-oracle

# Maven for dependency management
brew install maven

# AWS CLI for AWS operations
brew install awscli

# AWS CDK CLI
npm install -g aws-cdk

# kubectl for Kubernetes operations
brew install kubectl

# Optional: GitHub CLI
brew install gh
```

#### IDE Setup
**Recommended: IntelliJ IDEA**
```bash
# Install IntelliJ IDEA
brew install --cask intellij-idea

# Required plugins:
# - AWS Toolkit
# - Kubernetes
# - Mustache
```

**VS Code Alternative**
```bash
# Install VS Code
brew install --cask visual-studio-code

# Required extensions:
# - Extension Pack for Java
# - AWS Toolkit
# - Kubernetes
# - Mustache
```

### Local Development Setup

#### 1. Clone Dependencies
```bash
# Clone common CDK library (if needed as dependency)
gh repo clone stxkxs/cdk-common
cd cdk-common
mvn clean install
cd ..

# Clone EKS infrastructure
gh repo clone stxkxs/aws-eks-infra
cd aws-eks-infra
```

#### 2. Configure Environment
```bash
# Copy context template
cp cdk.context.template.json cdk.context.json

# Edit context with your AWS account details
# Required fields:
# - :account (AWS account ID)
# - :region (AWS region)
# - :domain (your domain for tagging)
# - hosted:eks:administrators (admin users)
# - hosted:eks:users (regular users)
```

#### 3. Build Project
```bash
# Build with Maven
mvn clean compile

# Run tests
mvn test

# Package application
mvn package
```

#### 4. CDK Operations
```bash
# Bootstrap CDK (first time only)
cdk bootstrap

# Synthesize CloudFormation
cdk synth

# View differences
cdk diff

# Deploy to AWS
cdk deploy
```

## Development Workflow

### Feature Development Process

#### 1. Planning Phase
- [ ] Create GitHub issue describing the feature
- [ ] Design the feature architecture
- [ ] Identify affected components
- [ ] Plan testing strategy
- [ ] Review security implications

#### 2. Implementation Phase
```bash
# Create feature branch
git checkout -b feature/new-addon-support

# Make changes
# - Update Java code if needed
# - Add/modify templates
# - Update documentation

# Test changes locally
mvn test
cdk synth
```

#### 3. Testing Phase
```bash
# Unit tests
mvn test

# Integration tests
cdk deploy --profile dev

# Verify deployment
kubectl get pods -A
kubectl get nodes
```

#### 4. Review and Merge
- [ ] Create pull request
- [ ] Code review by team
- [ ] Security review if applicable
- [ ] Merge to main branch

### Code Style Guidelines

#### Java Code Standards
```java
// Use descriptive variable names
private final NetworkNestedStack networkStack;
private final EksNestedStack eksStack;

// Proper error handling
try {
    // CDK operations
} catch (Exception e) {
    log.error("Failed to create stack: {}", e.getMessage(), e);
    throw new RuntimeException("Stack creation failed", e);
}

// Use builder patterns for complex objects
NestedStackProps.builder()
    .description(describe(conf.common(), "eks::network"))
    .build()
```

#### Template Standards
```yaml
# Use consistent indentation (2 spaces)
managed:
  awsVpcCni:
    name: vpc-cni
    version: v1.19.6-eksbuild.1
    serviceAccount:
      metadata:
        name: aws-node
        namespace: kube-system

# Use meaningful parameter names
{{hosted:id}}-eks
{{hosted:account}}
{{hosted:region}}

# Include comments for complex configurations
# This configuration enables pod identity for service accounts
podIdentityAgent:
  name: eks-pod-identity-agent
  version: v1.3.7-eksbuild.2
```

#### Documentation Standards
- Use clear, concise language
- Include code examples
- Provide troubleshooting steps
- Keep documentation up-to-date with code changes

## Testing Strategy

### Unit Testing

#### Java Unit Tests
```java
@Test
public void testDeploymentStackCreation() {
    // Given
    DeploymentConf conf = createTestConfiguration();
    
    // When
    DeploymentStack stack = new DeploymentStack(app, conf, stackProps);
    
    // Then
    assertThat(stack.getNetwork()).isNotNull();
    assertThat(stack.getEks()).isNotNull();
}

@Test
public void testConfigurationValidation() {
    // Test configuration validation logic
    DeploymentConf conf = DeploymentConf.builder()
        .account("123456789012")
        .region("us-west-2")
        .build();
    
    assertThat(conf.isValid()).isTrue();
}
```

#### Template Testing
```bash
# Validate Mustache templates
mustache --check src/main/resources/prototype/v1/conf.mustache

# Test template rendering
cat test-data.json | mustache - conf.mustache > rendered-config.yaml

# Validate YAML syntax
yamllint rendered-config.yaml
```

### Integration Testing

#### CDK Synthesis Testing
```bash
# Test CDK synthesis without deployment
cdk synth > /dev/null
echo "Synthesis test: $?"

# Test with different configurations
CDK_CONTEXT='{"hosted:environment":"test"}' cdk synth
```

#### Deployment Testing
```bash
# Deploy to test environment
cdk deploy --profile test-account

# Run validation tests
./scripts/validate-deployment.sh

# Cleanup test deployment
cdk destroy --profile test-account
```

### End-to-End Testing

#### Automated E2E Tests
```bash
#!/bin/bash
# scripts/e2e-test.sh

set -e

echo "Starting E2E tests..."

# Deploy infrastructure
cdk deploy --require-approval never

# Wait for cluster to be ready
kubectl wait --for=condition=Ready nodes --all --timeout=600s

# Test addon functionality
kubectl apply -f tests/e2e/test-workloads.yaml
kubectl wait --for=condition=Ready pods --all -n test --timeout=300s

# Test storage
kubectl apply -f tests/e2e/storage-test.yaml

# Test networking
kubectl apply -f tests/e2e/network-test.yaml

# Test security
kubectl apply -f tests/e2e/security-test.yaml

# Cleanup
kubectl delete -f tests/e2e/
cdk destroy --force

echo "E2E tests completed successfully!"
```

### Performance Testing

#### Load Testing
```yaml
# tests/performance/load-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-test
spec:
  containers:
  - name: load-generator
    image: appropriate/curl
    command:
    - /bin/sh
    - -c
    - |
      while true; do
        curl -s http://test-service/health
        sleep 1
      done
```

#### Resource Testing
```bash
# Test resource limits
kubectl apply -f tests/performance/resource-stress.yaml

# Monitor resource usage
kubectl top nodes
kubectl top pods -A
```

## Adding New Components

### Adding New AWS Managed Addons

#### 1. Update Addon Template
```yaml
# src/main/resources/prototype/v1/eks/addons.mustache
newAddon:
  name: new-addon-name
  version: v1.0.0-eksbuild.1
  preserveOnDelete: false
  resolveConflicts: preserve
  serviceAccount:
    metadata:
      name: new-addon-sa
      namespace: kube-system
    role:
      name: {{hosted:id}}-new-addon-sa
      managedPolicyNames:
        - NewAddonPolicy
      customPolicies: []
```

#### 2. Add IAM Policy (if needed)
```json
// src/main/resources/prototype/v1/policy/new-addon.mustache
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeNetworkInterfaces"
      ],
      "Resource": "*"
    }
  ]
}
```

#### 3. Update Documentation
- Add to ADDONS.md
- Update TEMPLATE_REFERENCE.md
- Include in troubleshooting guides

### Adding New Helm Charts

#### 1. Chart Configuration
```yaml
# Add to addons.mustache
newHelmChart:
  chart:
    name: new-chart
    repository: https://charts.example.com
    release: new-chart-release
    version: 1.0.0
    namespace: new-chart-namespace
    values: helm/new-chart.mustache
```

#### 2. Values Template
```yaml
# src/main/resources/prototype/v1/helm/new-chart.mustache
global:
  clusterName: {{hosted:id}}-eks
  region: {{hosted:region}}

serviceAccount:
  create: false
  name: {{hosted:id}}-new-chart-sa

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

#### 3. Service Account (if needed)
```yaml
# Add to addons.mustache
newHelmChart:
  serviceAccount:
    metadata:
      name: {{hosted:id}}-new-chart-sa
      namespace: new-chart-namespace
    role:
      name: {{hosted:id}}-new-chart-sa
      customPolicies:
        - name: {{hosted:id}}-new-chart
          policy: policy/new-chart.mustache
```

### Adding New Configuration Options

#### 1. Update Configuration Model
```java
// src/main/java/io/stxkxs/eks/stack/DeploymentConf.java
@Data
@Builder
public class DeploymentConf {
    private Common common;
    private VpcConf vpc;
    private EksConf eks;
    private NewFeatureConf newFeature; // Add new configuration section
}

@Data
@Builder
public class NewFeatureConf {
    private boolean enabled;
    private String version;
    private Map<String, String> settings;
}
```

#### 2. Update Configuration Template
```yaml
# src/main/resources/prototype/v1/conf.mustache
hosted:
  newFeature:
    enabled: true
    version: "1.0.0"
    settings:
      key1: value1
      key2: value2
```

#### 3. Update Stack Implementation
```java
// Use new configuration in stack creation
if (conf.getNewFeature().isEnabled()) {
    // Create new feature resources
    createNewFeatureResources(conf.getNewFeature());
}
```

## Security Considerations

### Code Security

#### 1. Secrets Management
```java
// Never hardcode secrets
// ❌ BAD
String apiKey = "secret-key-12345";

// ✅ GOOD  
String apiKey = System.getenv("API_KEY");
```

#### 2. IAM Permissions
```yaml
# Follow least privilege principle
customPolicies:
  - name: minimal-policy
    policy: |
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": [
              "s3:GetObject"  # Only specific actions needed
            ],
            "Resource": "arn:aws:s3:::specific-bucket/*"  # Specific resources only
          }
        ]
      }
```

#### 3. Template Security
```yaml
# Encrypt sensitive data
kms:
  alias: {{hosted:id}}-encryption-key
  enabled: true
  enableKeyRotation: true

# Use secure defaults
defaultStorageClass: 
  encrypted: "true"
  type: gp3
```

### Security Testing

#### 1. Static Analysis
```bash
# Run security scanning
mvn org.owasp:dependency-check-maven:check

# Check for known vulnerabilities
mvn versions:display-dependency-updates
```

#### 2. Template Validation
```bash
# Validate IAM policies
aws iam validate-policy-definition --policy-definition file://policy.json

# Check for overly permissive policies
checkov -f policy.json
```

## Performance Optimization

### Build Performance

#### 1. Maven Optimization
```xml
<!-- pom.xml -->
<properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <maven.compiler.release>21</maven.compiler.release>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <fork>true</fork>
                <meminitial>128m</meminitial>
                <maxmem>512m</maxmem>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### 2. CDK Performance
```bash
# Use CDK cache
export CDK_CACHE_ENABLED=true

# Parallel synthesis
cdk synth --all --concurrency 4

# Skip unchanged stacks
cdk deploy --hotswap
```

### Runtime Performance

#### 1. Resource Optimization
```yaml
# Right-size resources
resources:
  requests:
    cpu: 100m      # Start small
    memory: 128Mi
  limits:
    cpu: 500m      # Allow bursting
    memory: 512Mi
```

#### 2. Startup Optimization
```yaml
# Use init containers for setup
initContainers:
- name: setup
  image: busybox
  command: ['sh', '-c', 'setup tasks here']

# Optimize readiness probes
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

## Troubleshooting Development Issues

### Common Build Issues

#### 1. Maven Dependency Conflicts
```bash
# Analyze dependency tree
mvn dependency:tree

# Resolve conflicts
mvn dependency:resolve-sources
```

#### 2. CDK Synthesis Errors
```bash
# Verbose output
cdk synth --verbose

# Debug mode
CDK_DEBUG=true cdk synth
```

#### 3. Template Rendering Issues
```bash
# Test template with sample data
echo '{"hosted":{"id":"test"}}' | mustache - template.mustache

# Validate YAML output
mustache data.json template.mustache | yamllint -
```

### Debugging Deployment Issues

#### 1. CloudFormation Errors
```bash
# Check stack events
aws cloudformation describe-stack-events --stack-name eks-platform

# Get detailed error information
aws cloudformation describe-stack-resources --stack-name eks-platform
```

#### 2. EKS Issues
```bash
# Check cluster status (replace with your actual cluster name)
aws eks describe-cluster --name {{hosted:id}}-eks

# Check addon status
aws eks list-addons --cluster-name {{hosted:id}}-eks
aws eks describe-addon --cluster-name {{hosted:id}}-eks --addon-name vpc-cni
```

## Contribution Guidelines

### Pull Request Process

#### 1. Preparation
- [ ] Create issue describing the change
- [ ] Fork repository and create feature branch
- [ ] Make changes following code standards
- [ ] Add/update tests
- [ ] Update documentation

#### 2. Testing
- [ ] Run unit tests: `mvn test`
- [ ] Run integration tests: `cdk synth`
- [ ] Test deployment in dev environment
- [ ] Verify all checks pass

#### 3. Documentation
- [ ] Update relevant documentation files
- [ ] Add changelog entry
- [ ] Update version numbers if needed
- [ ] Include migration notes if breaking changes

#### 4. Review Process
- [ ] Submit pull request with clear description
- [ ] Address review feedback
- [ ] Ensure CI/CD checks pass
- [ ] Obtain required approvals

### Code Review Checklist

#### Reviewer Checklist
- [ ] **Functionality**: Does the code work as intended?
- [ ] **Security**: Are there any security vulnerabilities?
- [ ] **Performance**: Will this impact performance?
- [ ] **Maintainability**: Is the code readable and maintainable?
- [ ] **Testing**: Are there adequate tests?
- [ ] **Documentation**: Is documentation updated?

#### Security Review
- [ ] No hardcoded secrets or credentials
- [ ] Proper IAM permissions (least privilege)
- [ ] Input validation and sanitization
- [ ] Secure communication protocols
- [ ] Encryption of sensitive data

## Release Process

### Version Management

#### Semantic Versioning
- **Major** (X.0.0): Breaking changes
- **Minor** (0.X.0): New features, backward compatible
- **Patch** (0.0.X): Bug fixes, backward compatible

#### Release Preparation
```bash
# Update version
mvn versions:set -DnewVersion=1.2.0

# Update changelog
echo "## [1.2.0] - $(date +%Y-%m-%d)" >> CHANGELOG.md

# Create release commit
git add .
git commit -m "Release v1.2.0"
git tag v1.2.0
```

### Deployment Pipeline

#### CI/CD Integration
```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 21
      uses: actions/setup-java@v2
      with:
        java-version: '21'
    - name: Run tests
      run: mvn test
    - name: CDK synth
      run: cdk synth
```

### Maintenance

#### Regular Maintenance Tasks
- [ ] **Weekly**: Dependency updates
- [ ] **Monthly**: Security patch review
- [ ] **Quarterly**: Major version updates
- [ ] **Annually**: Architecture review

#### Monitoring and Metrics
- Track build success rates
- Monitor deployment times
- Measure test coverage
- Review performance metrics