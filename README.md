# aws-eks-infra

<div align="center">

*aws cdk application written in java that provisions an amazon eks (elastic kubernetes
service) cluster with managed addons, custom helm charts, observability integration, and node groups.*

[![license: mit](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![java](https://img.shields.io/badge/Java-21%2B-blue.svg)](https://www.oracle.com/java/)
[![aws cdk](https://img.shields.io/badge/AWS%20CDK-latest-orange.svg)](https://aws.amazon.com/cdk/)
[![vpc](https://img.shields.io/badge/Amazon-VPC-ff9900.svg)](https://aws.amazon.com/vpc/)
[![eks](https://img.shields.io/badge/Amazon-EKS-ff9900.svg)](https://aws.amazon.com/eks/)
[![opentelemetry](https://img.shields.io/badge/OpenTelemetry-Enabled-blueviolet.svg)](https://opentelemetry.io/)
[![grafana](https://img.shields.io/badge/Grafana-Observability-F46800.svg)](https://grafana.com/)

</div>

## overview

+ eks cluster with rbac configuration
+ aws managed eks addons (vpc cni, ebs csi driver, coredns, kube proxy, pod identity agent, cloudwatch container
  insights)
+ helm chart-based addons (cert-manager, aws load balancer controller, karpenter, csi secrets store)
+ grafana cloud observability integration
+ managed node groups with bottlerocket ami's
+ sqs queue for node interruption handling

## prerequisites

+ [java 21+](https://sdkman.io/)
+ [maven](https://maven.apache.org/download.cgi)
+ [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
+ [aws cdk cli](https://docs.aws.amazon.com/cdk/v2/guide/getting-started.html)
+ [github cli](https://cli.github.com/)
+ [common cdk repo](https://github.com/stxkxs/cdk-common) `gh repo clone stxkxs/cdk-common`
+ prepare aws environment by running `cdk bootstrap` with the appropriate aws account and region:

  ```bash
  cdk bootstrap aws://<account-id>/<region>
  ```

    + replace `<account-id>` with your aws account id and `<region>` with your desired aws region (e.g., `us-west-2`).
    + this command sets up the necessary resources for deploying cdk applications, such as an S3 bucket for storing
      assets and a CloudFormation execution role
    + for more information, see the aws cdk documentation:
        + https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html
        + https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-bootstrap.html

## deployment

1. build projects:
   ```bash
   mvn -f cdk-common/pom.xml clean install
   mvn -f aws-eks-infra/pom.xml clean install
   ```

2. update configuration files:
    + create `aws-eks-infra/cdk.context.json` from `aws-eks-infra/cdk.context.template.json` with your
      aws account details
        + `:account` - aws account id
        + `:region` - aws region (e.g., `us-west-2`
        + `:domain` - registered domain name for ses (e.g., `stxkxs.io`)
        + `:environment` - this should not be changed unless you add a new set of resources to configure that
          environment
        + `:version` - version of the resources to deploy, this is used to differentiate between different versions of
          the resources
            + currently set to prototype/v1 for the resources at
              `aws-eks-infra/src/main/resources/prototype/v1`

    + cluster access configuration
        ```json
        {
          "hosted:eks:administrators": [
            {
              "username": "administrator",
              "role": "arn:aws:iam::000000000000:role/AWSReservedSSO_AdministratorAccess_abc",
              "email": "admin@aol.com"
            }
          ],
          "hosted:eks:users": [
            {
              "username": "user",
              "role": "arn:aws:iam::000000000000:role/AWSReservedSSO_DeveloperAccess_abc",
              "email": "user@aol.com"
            }  
          ]
        }
        ```

        + **administrators**: iam roles that will have full admin access to the cluster
        + **users**: iam roles that will have read-only access to the cluster
            + `username`: used for identifying the user in kubernetes rbac
            + `role`: aws iam role arn (typically from aws sso) that will be mapped to admin permissions through
              aws-auth configmap
            + `email`: for identification and traceability purposes

3. deploy eks infrastructure:
    ```bash
    cd aws-eks-infra
    
    cdk synth
    cdk deploy
    ```

4. use it:
    ```bash
    aws eks update-kubeconfig --name fff-eks --region us-west-2
   
    kubectl get nodes
    kubectl get pods -A
    ```

## license

[mit license](LICENSE)

for your convenience, you can find the full mit license text at

+ [https://opensource.org/license/mit/](https://opensource.org/license/mit/) (official osi website)
+ [https://choosealicense.com/licenses/mit/](https://choosealicense.com/licenses/mit/) (choose a license website)
