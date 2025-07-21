# AWS EKS Cluster with Karpenter and Harness Cluster Orchestrator

This repository contains configuration files and documentation for setting up and managing AWS EKS clusters with Karpenter for auto-scaling and Harness Cluster Orchestrator for cost optimization.

## Table of Contents

1. [AWS Concepts](#aws-concepts)
2. [Kubernetes Concepts](#kubernetes-concepts)
3. [Karpenter Concepts](#karpenter-concepts)
4. [Cluster Setup](#cluster-setup)
5. [File Descriptions](#file-descriptions)
6. [Command Reference](#command-reference)
7. [Instance Types](#instance-types)
8. [Troubleshooting](#troubleshooting)

## AWS Concepts

### VPC (Virtual Private Cloud)
A VPC is a virtual network dedicated to your AWS account. It's logically isolated from other virtual networks in the AWS Cloud, providing a private, secure environment for your AWS resources like EC2 instances, databases, and Kubernetes clusters.

#### VPC Private Subnets
- **Private subnets** are subnets that don't have a direct route to the internet.
- Resources in private subnets can't be directly accessed from the internet.
- Typically used for backend services, databases, and application servers that don't need to be publicly accessible.
- In EKS clusters, worker nodes are often placed in private subnets for security.

#### VPC Public Subnets
- **Public subnets** have a direct route to the internet via an Internet Gateway.
- Resources in public subnets can be configured to be accessible from the internet.
- Typically used for load balancers, bastion hosts, and NAT gateways.
- In EKS clusters, public subnets are used for resources that need to be publicly accessible, like load balancers.

### IAM (Identity and Access Management)
IAM is AWS's system for securely controlling access to AWS services and resources. It allows you to create and manage AWS users and groups, and use permissions to allow or deny their access to AWS resources.

#### IAM Policies
- **Policies** are JSON documents that define permissions - what actions are allowed or denied on which AWS resources.
- They can be attached to IAM users, groups, or roles.
- Example: A policy might allow a user to list and read objects in an S3 bucket but not delete them.

#### ARN (Amazon Resource Name)
- **ARN** (e.g., `arn:aws:iam::357919113896:role/KarpenterNodeRole-karpenter-cluster-v4`) is a standardized way to uniquely identify AWS resources across all of AWS.
- Format: `arn:partition:service:region:account-id:resource-type/resource-id`
- In the example, it's an IAM role in account 357919113896 named KarpenterNodeRole-karpenter-cluster-v4.

## Kubernetes Concepts

### Cluster
A Kubernetes cluster is a set of machines, called nodes, that run containerized applications managed by Kubernetes. A cluster has at least one worker node and at least one master node (control plane).

### Nodes
Nodes are the worker machines in a Kubernetes cluster. They can be virtual or physical machines, depending on the cluster. Each node is managed by the control plane and contains the services necessary to run pods.

#### Worker Nodes
Worker nodes are the machines where your applications run. They communicate with the control plane and execute the workloads assigned to them. In AWS EKS, worker nodes are typically EC2 instances.

### Pods
Pods are the smallest deployable units in Kubernetes. A pod represents a single instance of a running process in your cluster and can contain one or more containers. Containers in a pod share storage, network resources, and are always scheduled together.

## Karpenter Concepts

### EC2NodeClass
EC2NodeClass is a Karpenter-specific custom resource that defines the configuration for EC2 instances that Karpenter can provision.

- It specifies the instance profile, security groups, subnets, and other EC2-specific configurations.
- It's referenced by NodePools to determine how to provision EC2 instances.
- Example properties include:
  - `amiFamily`: The type of Amazon Machine Image to use (e.g., AL2 for Amazon Linux 2)
  - `instanceProfile`: The IAM instance profile to attach to the instances
  - `securityGroupSelectorTerms`: Which security groups to use
  - `subnetSelectorTerms`: Which subnets to launch instances in

### NodePool
NodePool is a Karpenter-specific custom resource that defines when and how to provision nodes based on pod requirements.

- It references an EC2NodeClass to determine the infrastructure-specific configuration.
- It defines scaling behavior, disruption policies, and instance types.
- Example properties include:
  - `disruption.consolidationPolicy`: When to consolidate nodes (e.g., WhenUnderutilized)
  - `disruption.expireAfter`: When to expire and replace nodes (e.g., 720h for 30 days)
  - `requirements`: Constraints on the types of nodes to provision (e.g., instance type, capacity type)

### API Version v1beta1
v1beta1 indicates that the API is in beta stage, version 1.

- In Kubernetes, API versions follow this pattern: `[api-group]/[version]`
- Beta versions (like v1beta1) indicate that the feature is well-tested but the API might change in future versions.
- We use this version because it's the current version supported by Karpenter for the resources we're defining.
- Beta APIs are enabled by default in Kubernetes but might change in future releases.

## Cluster Setup

### karpenter-cluster-v4

Our main cluster, `karpenter-cluster-v4`, was created in the eu-west-1 region with the following configuration:

1. **Initial Setup**:
   - Created using eksctl with Kubernetes version 1.32
   - Set up with a managed node group using t3.medium instances
   - Configured with OIDC provider for IAM roles for service accounts

2. **Karpenter Installation**:
   - Installed using Helm
   - Configured with IAM roles and instance profile
   - Set up to use the cluster's VPC and subnets

3. **NodePool Configuration**:
   - Created two NodePools: t2-medium-pool and t3-large-pool
   - Each NodePool references its corresponding EC2NodeClass
   - Configured with disruption budgets and consolidation policies

4. **Current Status**:
   - 12 nodes in the cluster
   - 1 node in "Ready" state, 11 nodes in "NotReady" state (managed by Karpenter)
   - Active scaling based on workload demands

## File Descriptions

### EC2NodeClass Files

#### t3-large-class-final.yaml
```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: t3-large-class
spec:
  amiFamily: AL2
  instanceProfile: KarpenterNodeInstanceProfile-karpenter-cluster-v4
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 2
    httpTokens: required
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4
```

#### t2-medium-class-final.yaml
```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: t2-medium-class
spec:
  amiFamily: AL2
  instanceProfile: KarpenterNodeInstanceProfile-karpenter-cluster-v4
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 2
    httpTokens: required
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4
```

### NodePool Files

#### current-t3-large-pool.yaml
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: t3-large-pool
spec:
  disruption:
    budgets:
    - nodes: 10%
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
  template:
    spec:
      nodeClassRef:
        name: t3-large-class
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
        - t3.large
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
```

#### t2-medium-pool-update.yaml
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: t2-medium-pool
spec:
  disruption:
    budgets:
    - nodes: 10%
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
  template:
    spec:
      nodeClassRef:
        name: t2-medium-class
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
        - t2.medium
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
```

### IAM Policy Files

#### karpenter-controller-policy.json
This file defines the IAM policy for the Karpenter controller, allowing it to manage EC2 instances, IAM roles, and other AWS resources.

#### karpenter-node-trust-policy.json
This file defines the trust policy for the IAM role used by the EC2 instances provisioned by Karpenter.

### Setup Scripts

#### setup-karpenter-iam.sh
This script sets up the IAM roles and policies required by Karpenter.

#### install-karpenter.sh
This script installs Karpenter using Helm.

## Command Reference

### kubectl Commands

- **kubectl get nodes**: List all nodes in the cluster
  ```bash
  kubectl get nodes
  ```

- **kubectl get nodepools**: List all NodePools in the cluster
  ```bash
  kubectl get nodepools
  ```

- **kubectl get ec2nodeclasses**: List all EC2NodeClasses in the cluster
  ```bash
  kubectl get ec2nodeclasses
  ```

- **kubectl describe nodepool [name]**: View detailed information about a NodePool
  ```bash
  kubectl describe nodepool t3-large-pool
  ```

- **kubectl logs -n [namespace] [pod] -c [container]**: View logs from a container in a pod
  ```bash
  kubectl logs -n karpenter deployment/karpenter -c controller
  ```

- **kubectl apply -f [filename]**: Apply a configuration file to the cluster
  ```bash
  kubectl apply -f t3-large-class-final.yaml
  ```
  The `-f` flag specifies the file containing the resource definition. It can also be used with a directory or URL.

### AWS CLI Commands

- **aws eks update-kubeconfig**: Update kubeconfig to connect to an EKS cluster
  ```bash
  aws eks update-kubeconfig --name karpenter-cluster-v4 --region eu-west-1
  ```

- **aws iam create-role**: Create an IAM role
  ```bash
  aws iam create-role --role-name KarpenterNodeRole-karpenter-cluster-v4 --assume-role-policy-document file://karpenter-node-trust-policy.json
  ```

- **aws iam attach-role-policy**: Attach a policy to an IAM role
  ```bash
  aws iam attach-role-policy --role-name KarpenterNodeRole-karpenter-cluster-v4 --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
  ```

### eksctl Commands

- **eksctl create cluster**: Create an EKS cluster
  ```bash
  eksctl create cluster --name karpenter-cluster-v4 --region eu-west-1 --version 1.32 --with-oidc
  ```

### Helm Commands

- **helm repo add**: Add a Helm repository
  ```bash
  helm repo add karpenter https://charts.karpenter.sh
  ```

- **helm install**: Install a Helm chart
  ```bash
  helm install karpenter karpenter/karpenter --namespace karpenter --create-namespace
  ```

## Instance Types

### t2.medium
- **vCPUs**: 2
- **Memory**: 4 GiB
- **Network Performance**: Low to Moderate
- **EBS-Optimized**: Available
- **Cost**: Lower cost, good for development and testing
- **Burstable Performance**: Yes, can burst above baseline CPU performance using CPU credits

### t3.large
- **vCPUs**: 2
- **Memory**: 8 GiB
- **Network Performance**: Up to 5 Gbps
- **EBS-Optimized**: Yes
- **Cost**: Higher than t2.medium but more powerful
- **Burstable Performance**: Yes, with improved burst performance over t2

## Troubleshooting

### Common Issues

1. **Nodes in NotReady State**:
   - This is normal behavior for Karpenter, which scales down underutilized nodes.
   - Verify that new pods can be scheduled and that Karpenter provisions new nodes when needed.

2. **IAM Role Errors**:
   - Check that the IAM roles and policies are correctly configured.
   - Ensure that the instance profile exists and has the correct roles attached.

3. **EC2NodeClass Errors**:
   - Verify that the security groups and subnets specified in the EC2NodeClass exist.
   - Check that the AMI family is supported in your region.

4. **NodePool Scaling Issues**:
   - Check that the requirements in the NodePool match available instance types.
   - Verify that there are no AWS service quotas limiting the number of instances you can launch.

### Karpenter Logs

To view Karpenter logs for debugging:
```bash
kubectl logs -n karpenter deployment/karpenter -c controller
```

### AWS Resource Verification

To verify that AWS resources are correctly configured:
```bash
aws iam get-role --role-name KarpenterNodeRole-karpenter-cluster-v4
aws iam list-attached-role-policies --role-name KarpenterNodeRole-karpenter-cluster-v4
aws ec2 describe-security-groups --filters "Name=tag:karpenter.sh/discovery,Values=karpenter-cluster-v4"
```

