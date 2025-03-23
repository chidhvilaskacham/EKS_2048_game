# Amazon EKS Fargate Deployment with 2048 Game Example

This repository provides a step-by-step guide to deploy a 2048 game application on Amazon EKS using Fargate and an Application Load Balancer (ALB). It includes instructions for setting up the EKS cluster, configuring IAM roles, deploying the 2048 game, and installing the AWS Load Balancer Controller.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Create an EKS Cluster with Fargate](#1-create-an-eks-cluster-with-fargate)
3. [Deploy the 2048 Game Application](#2-deploy-the-2048-game-application)
4. [Configure IAM OIDC Provider](#3-configure-iam-oidc-provider)
5. [Install the AWS Load Balancer Controller](#4-install-the-aws-load-balancer-controller)
6. [Access the 2048 Game](#5-access-the-2048-game)
7. [Cleanup](#cleanup)
8. [References](#references)

## Prerequisites
Before you begin, ensure you have the following tools installed and configured:

- **kubectl** - Kubernetes command-line tool. [Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- **eksctl** - CLI tool for Amazon EKS. [Installation Guide](https://eksctl.io/introduction/installation/)
- **AWS CLI** - AWS command-line interface. [Installation Guide](https://aws.amazon.com/cli/)

After installing AWS CLI, configure it using:
```sh
aws configure
```

## 1. Create an EKS Cluster with Fargate
Amazon EKS (Elastic Kubernetes Service) allows you to run Kubernetes applications on AWS without managing the underlying infrastructure. Fargate is a serverless compute engine that automatically provisions resources, eliminating the need for managing worker nodes.

### **Create the Cluster**
Run the following command to create an EKS cluster using Fargate:
```sh
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
This command:
- Creates an EKS cluster named `demo-cluster` in the `us-east-1` region.
- Uses Fargate as the compute provider, so you don’t need to manage worker nodes.

### **Delete the Cluster**
To clean up and delete the cluster, run:
```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```

## 2. Deploy the 2048 Game Application
The 2048 game is deployed as a Kubernetes application. It requires a **namespace** (game-2048) and a deployment manifest that defines the necessary resources.

### **Create a Fargate Profile**
Fargate profiles determine which Kubernetes namespaces should run on Fargate. Create a Fargate profile for the `game-2048` namespace:
```sh
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### **Deploy the 2048 Game**
Apply the Kubernetes manifest:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
This manifest creates:
- A **Deployment** to run the 2048 game pods.
- A **Service** to expose the game internally.
- An **Ingress resource** to configure the ALB for external access.

## 3. Configure IAM OIDC Provider
AWS IAM OIDC (OpenID Connect) allows Kubernetes to authenticate with AWS services securely. The AWS Load Balancer Controller needs this setup to manage ALBs.

### **Export the Cluster Name:**
```sh
export cluster_name=demo-cluster
```
- Stores the cluster name as an environment variable for easier reference.

### **Extract the OIDC ID:**
```sh
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
- Retrieves the unique identifier for the cluster’s OIDC provider.

### **Check for an Existing OIDC Provider:**
```sh
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```
- Verifies if an IAM OIDC provider is already set up for the cluster.

### **Create the OIDC Provider (if missing):**
```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```
- Associates the EKS cluster with an IAM OIDC provider to enable authentication.

## 4. Install the AWS Load Balancer Controller
The AWS Load Balancer Controller enables Kubernetes to provision and manage Application Load Balancers (ALBs) in AWS. This is required for exposing services to the internet.

### **Download the IAM Policy**
```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
- Downloads the necessary IAM policy that grants permissions for ALB management.

### **Create the IAM Policy**
```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
- Creates a policy that defines the permissions required by the AWS Load Balancer Controller.

### **Create an IAM Role for the Controller**
Replace `<ACCOUNT_ID>` with your AWS account ID:
```sh
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## 5. Access the 2048 Game
```sh
kubectl get ingress -n game-2048
```

## Cleanup
Refer to the cleanup steps above to remove AWS resources and avoid unnecessary costs.

## References
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS Load Balancer Controller Documentation](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

