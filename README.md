# Amazon EKS Fargate Deployment with 2048 Game

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
This command creates an EKS cluster named `demo-cluster` in the `us-east-1` region and uses Fargate as the compute provider. This means that Kubernetes pods will run in a fully managed environment without needing EC2 instances.
```sh
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

### **Delete the Cluster**
To clean up and delete the cluster, run the following command. This removes all resources associated with the cluster, preventing unwanted costs.
```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```

## 2. Deploy the 2048 Game Application
The 2048 game will be deployed as a Kubernetes application in a dedicated namespace called `game-2048`. AWS Fargate allows you to run this deployment without provisioning and managing EC2 instances.

### **Create a Fargate Profile**
A Fargate profile tells AWS which Kubernetes namespace should run on Fargate. This ensures that all workloads in `game-2048` will use AWS-managed resources.
```sh
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### **Deploy the 2048 Game**
This command deploys the 2048 game using a Kubernetes manifest. It defines the required resources such as a deployment, a service, and an ingress rule.
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

## 3. Configure IAM OIDC Provider
AWS IAM OIDC (OpenID Connect) allows Kubernetes to securely authenticate with AWS services. The AWS Load Balancer Controller requires this setup to create and manage ALBs.

### **Export the Cluster Name:**
Stores the cluster name as an environment variable for easier reference.
```sh
export cluster_name=demo-cluster
```

### **Extract the OIDC ID:**
Retrieves the unique identifier for the cluster’s OIDC provider.
```sh
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

### **Check for an Existing OIDC Provider:**
Verifies if an IAM OIDC provider is already set up for the cluster.
```sh
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

### **Create the OIDC Provider (if missing):**
Associates the EKS cluster with an IAM OIDC provider to enable authentication.
```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## 4. Install the AWS Load Balancer Controller
The AWS Load Balancer Controller enables Kubernetes to provision and manage ALBs in AWS. This is required for exposing services to the internet.

### **Download the IAM Policy**
This policy grants the controller the necessary permissions to manage ALBs.
```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### **Create the IAM Policy**
```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### **Create an IAM Role for the Controller**
This role allows the AWS Load Balancer Controller to interact with AWS services.
```sh
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### **Install the ALB Controller using Helm**
This installs the ALB Controller in the Kubernetes cluster, enabling external access to services.
```sh
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
```sh
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

## 5. Access the 2048 Game
To check the status of the ingress and retrieve the ALB address:
```sh
kubectl get ingress -n game-2048
```

![image](https://github.com/user-attachments/assets/6b5c5b70-cdb0-494d-8cbd-149fd29d199b)


## 6. Cleanup
To remove AWS resources and avoid unnecessary costs, refer to the cleanup steps above.

## 7. References
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [AWS Load Balancer Controller Documentation](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

