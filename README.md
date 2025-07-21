# AWS EKS Cluster with Karpenter and Harness Cluster Orchestrator

This repository contains configuration files and documentation for setting up and managing AWS EKS clusters with Karpenter for auto-scaling and Harness Cluster Orchestrator for cost optimization.

## Table of Contents

1. [Utilities and Tools](#utilities-and-tools)
2. [AWS Concepts](#aws-concepts)
3. [Kubernetes Concepts](#kubernetes-concepts)
4. [Karpenter Concepts](#karpenter-concepts)
5. [Step-by-Step Setup Guide](#step-by-step-setup-guide)
6. [Key Configuration Files](#key-configuration-files)
7. [Command Reference](#command-reference)
8. [Instance Types](#instance-types)
9. [Troubleshooting](#troubleshooting)

## Utilities and Tools

### eksctl
- **Purpose**: Command-line utility for creating and managing EKS clusters
- **Usage**: Creating clusters, managing nodegroups, configuring OIDC providers
- **Why it's useful**: Simplifies the process of creating EKS clusters with sensible defaults

### kubectl
- **Purpose**: Command-line tool for interacting with Kubernetes clusters
- **Usage**: Managing Kubernetes resources, viewing logs, executing commands in containers
- **Why it's useful**: Primary interface for all Kubernetes operations

### AWS CLI
- **Purpose**: Command-line interface for AWS services
- **Usage**: Managing IAM roles, EC2 instances, and other AWS resources
- **Why it's useful**: Automates AWS resource management and integrates with scripts

### Helm
- **Purpose**: Package manager for Kubernetes
- **Usage**: Installing and managing Kubernetes applications
- **Why it's useful**: Simplifies the deployment of complex applications like Karpenter

### Karpenter
- **Purpose**: Kubernetes node autoscaler
- **Usage**: Automatically provisioning nodes based on workload demands
- **Why it's useful**: More efficient and responsive than Cluster Autoscaler, with just-in-time node provisioning

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

## Step-by-Step Setup Guide

### 1. Creating the EKS Cluster

```bash
# Create the EKS cluster with eksctl
exsctl create cluster \
  --name karpenter-cluster-v4 \
  --region eu-west-1 \
  --version 1.32 \
  --with-oidc \
  --vpc-private-subnets subnet-xxxx,subnet-yyyy \
  --vpc-public-subnets subnet-aaaa,subnet-bbbb \
  --nodegroup-name initial-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

This command creates:
- An EKS cluster named `karpenter-cluster-v4` in the eu-west-1 region
- A managed node group with t3.medium instances
- An OIDC provider for IAM roles for service accounts
- VPC with public and private subnets

The initial t3.medium nodes are necessary to:
- Host the Kubernetes control plane components
- Run system pods like CoreDNS and kube-proxy
- Provide a stable foundation for Karpenter installation

### 2. Setting up IAM Roles for Karpenter

```bash
# Create IAM roles for Karpenter
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com

# Create IAM policy for Karpenter controller
aws iam create-policy \
  --policy-name KarpenterControllerPolicy-karpenter-cluster-v4 \
  --policy-document file://karpenter-controller-policy.json

# Create IAM role for Karpenter controller
aws iam create-role \
  --role-name KarpenterControllerRole-karpenter-cluster-v4 \
  --assume-role-policy-document file://karpenter-controller-trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name KarpenterControllerRole-karpenter-cluster-v4 \
  --policy-arn arn:aws:iam::357919113896:policy/KarpenterControllerPolicy-karpenter-cluster-v4

# Create instance profile for Karpenter nodes
aws iam create-instance-profile \
  --instance-profile-name KarpenterNodeInstanceProfile-karpenter-cluster-v4

# Create IAM role for Karpenter nodes
aws iam create-role \
  --role-name KarpenterNodeRole-karpenter-cluster-v4 \
  --assume-role-policy-document file://karpenter-node-trust-policy.json

# Attach necessary policies to node role
aws iam attach-role-policy \
  --role-name KarpenterNodeRole-karpenter-cluster-v4 \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name KarpenterNodeRole-karpenter-cluster-v4 \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy \
  --role-name KarpenterNodeRole-karpenter-cluster-v4 \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Add node role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name KarpenterNodeInstanceProfile-karpenter-cluster-v4 \
  --role-name KarpenterNodeRole-karpenter-cluster-v4
```

These commands create the necessary IAM roles and policies for Karpenter to:
- Launch and terminate EC2 instances
- Manage instance profiles
- Access EKS cluster resources

### 3. Installing Karpenter

```bash
# Add Karpenter Helm repository
helm repo add karpenter https://charts.karpenter.sh
helm repo update

# Install Karpenter
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::357919113896:role/KarpenterControllerRole-karpenter-cluster-v4 \
  --set clusterName=karpenter-cluster-v4 \
  --set clusterEndpoint=$(aws eks describe-cluster --name karpenter-cluster-v4 --query "cluster.endpoint" --output text) \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-karpenter-cluster-v4
```

This installs Karpenter with:
- A dedicated namespace
- Service account linked to the IAM role
- Cluster-specific configuration

### 4. Creating EC2NodeClass

We created a single EC2NodeClass file that defines the AWS-specific configuration for our nodes:

```bash
# Create the EC2NodeClass
kubectl apply -f t3-large-class-final.yaml
```

### 5. Creating NodePools

We created two NodePool files to define different instance types for different workloads:

```bash
# Create the NodePools
kubectl apply -f current-t3-large-pool.yaml
kubectl apply -f t2-medium-pool-update.yaml
```

### 6. Verifying the Setup

```bash
# Check that Karpenter is running
kubectl get pods -n karpenter

# Check the NodePools
kubectl get nodepools

# Check the EC2NodeClasses
kubectl get ec2nodeclasses

# Check the nodes
kubectl get nodes
```

### Current Status

- 12 nodes in the cluster
- 1 node in "Ready" state, 11 nodes in "NotReady" state (managed by Karpenter)
- Active scaling based on workload demands

## Key Configuration Files

### EC2NodeClass Files

We created separate EC2NodeClass files for each instance type, though their configurations are very similar. This approach gives us flexibility to make instance-specific optimizations if needed in the future.

#### t3-large-class-final.yaml
```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: t3-large-class
spec:
  amiFamily: AL2  # Amazon Linux 2 AMI
  instanceProfile: KarpenterNodeInstanceProfile-karpenter-cluster-v4  # IAM instance profile
  metadataOptions:
    httpEndpoint: enabled  # Enable metadata service
    httpProtocolIPv6: disabled  # Disable IPv6 metadata
    httpPutResponseHopLimit: 2  # Security setting for metadata service
    httpTokens: required  # Require tokens for metadata service (security best practice)
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4  # Select security groups by tag
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4  # Select subnets by tag
```

#### t2-medium-class-final.yaml
```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: t2-medium-class
spec:
  amiFamily: AL2  # Amazon Linux 2 AMI
  instanceProfile: KarpenterNodeInstanceProfile-karpenter-cluster-v4  # IAM instance profile
  metadataOptions:
    httpEndpoint: enabled  # Enable metadata service
    httpProtocolIPv6: disabled  # Disable IPv6 metadata
    httpPutResponseHopLimit: 2  # Security setting for metadata service
    httpTokens: required  # Require tokens for metadata service (security best practice)
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4  # Select security groups by tag
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: karpenter-cluster-v4  # Select subnets by tag
```

### NodePool Files

We created two separate NodePool files to define different instance types for different workloads:

#### current-t3-large-pool.yaml
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: t3-large-pool
spec:
  disruption:
    budgets:
    - nodes: 10%  # Maximum 10% of nodes can be disrupted at once
    consolidationPolicy: WhenUnderutilized  # Consolidate nodes when underutilized
    expireAfter: 720h  # Replace nodes after 30 days for security/updates
  template:
    spec:
      nodeClassRef:
        name: t3-large-class  # Reference to the EC2NodeClass
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
        - t3.large  # Use t3.large instances
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand  # Use on-demand instances (not spot)
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
    - nodes: 10%  # Maximum 10% of nodes can be disrupted at once
    consolidationPolicy: WhenUnderutilized  # Consolidate nodes when underutilized
    expireAfter: 720h  # Replace nodes after 30 days for security/updates
  template:
    spec:
      nodeClassRef:
        name: t2-medium-class  # Reference to the EC2NodeClass
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
        - t2.medium  # Use t2.medium instances
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand  # Use on-demand instances (not spot)
```

### On-Demand vs Spot Instances

In our NodePool configurations, we specifically chose to use on-demand instances instead of spot instances:

```yaml
- key: karpenter.sh/capacity-type
  operator: In
  values:
  - on-demand  # Use on-demand instances (not spot)
```

#### Why On-Demand Instead of Spot?

1. **Reliability**: On-demand instances provide guaranteed availability without unexpected termination, which is critical for production workloads.

2. **Predictable Costs**: While more expensive than spot instances, on-demand instances have predictable pricing which helps with budgeting.

3. **No Interruptions**: Spot instances can be terminated with just 2 minutes notice if AWS needs the capacity, which can disrupt workloads.

4. **Consistent Performance**: On-demand instances provide consistent performance without the risk of sudden termination.

#### Is It Necessary?

The choice between on-demand and spot instances depends on your specific requirements:

- **For Production Workloads**: On-demand is often preferred for reliability and stability.
- **For Cost Optimization**: Spot instances can save 70-90% on costs but with the risk of interruption.
- **For Hybrid Approach**: You can create multiple NodePools, some using on-demand for critical workloads and others using spot for fault-tolerant workloads.

If you want to use spot instances instead, you would simply change the capacity-type requirement:

```yaml
- key: karpenter.sh/capacity-type
  operator: In
  values:
  - spot  # Use spot instances instead of on-demand
```

### Other Important Files

#### karpenter-controller-policy.json
This file defines the IAM policy for the Karpenter controller, allowing it to manage EC2 instances, IAM roles, and other AWS resources.

#### karpenter-node-trust-policy.json
This file defines the trust policy for the IAM role used by the EC2 instances provisioned by Karpenter.

## Command Reference

### kubectl Commands

- **kubectl get nodes**: List all nodes in the cluster
  ```bash
  kubectl get nodes
  ```

- **kubectl describe node [node-name]**: Get detailed information about a specific node
  ```bash
  kubectl describe node ip-192-168-136-173.eu-west-1.compute.internal
  ```
  This command is particularly useful for investigating nodes in NotReady state, as it shows:
  - Node conditions (Ready, DiskPressure, MemoryPressure, etc.)
  - Allocated resources and capacity
  - Events that might explain why a node is NotReady
  - Taints that might be preventing pods from scheduling

- **kubectl get nodes -o wide**: Get more detailed information about all nodes
  ```bash
  kubectl get nodes -o wide
  ```
  This shows additional columns including the internal and external IP addresses, OS image, kernel version, and container runtime.

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
