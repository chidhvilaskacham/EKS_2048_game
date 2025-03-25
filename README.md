# Amazon EKS Fargate Deployment with 2048 Game

This repository provides a step-by-step guide to deploy a 2048 game application on Amazon EKS using Fargate and an Application Load Balancer (ALB). It includes instructions for setting up the EKS cluster, configuring IAM roles, deploying the 2048 game, and installing the AWS Load Balancer Controller.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Create an EKS Cluster with Fargate](#create-an-eks-cluster-with-fargate)
- [Deploy the 2048 Game Application](#deploy-the-2048-game-application)
- [Configure IAM OIDC Provider](#configure-iam-oidc-provider)
- [Install the AWS Load Balancer Controller](#install-the-aws-load-balancer-controller)
- [Access the 2048 Game](#access-the-2048-game)
- [Cleanup](#cleanup)
- [References](#references)

## Prerequisites
Before you begin, ensure you have the following tools installed and configured:

- **kubectl** - Kubernetes command-line tool. [Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- **eksctl** - CLI tool for Amazon EKS. [Installation Guide](https://eksctl.io/introduction/)
- **AWS CLI** - AWS command-line interface. [Installation Guide](https://aws.amazon.com/cli/)

After installing AWS CLI, configure it using:
```sh
aws configure
```

## 1. Create an EKS Cluster with Fargate
Amazon EKS (Elastic Kubernetes Service) allows you to run Kubernetes applications on AWS without managing the underlying infrastructure. Fargate is a serverless compute engine that automatically provisions resources, eliminating the need for managing worker nodes.

### Create the Cluster
This command creates an EKS cluster named `demo-cluster` in the `us-east-1` region and uses Fargate as the compute provider. This ensures that Kubernetes pods will run in a fully managed environment without needing EC2 instances.
```sh
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

### Delete the Cluster
To clean up and delete the cluster, preventing unwanted costs:
```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```

## 2. Deploy the 2048 Game Application
The 2048 game will be deployed as a Kubernetes application in a dedicated namespace called `game-2048`. AWS Fargate allows you to run this deployment without provisioning and managing EC2 instances.

### Create a Fargate Profile
A Fargate profile tells AWS which Kubernetes namespace should run on Fargate, ensuring that all workloads in `game-2048` will use AWS-managed resources.
```sh
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### Deploy the 2048 Game
This command deploys the 2048 game using a Kubernetes manifest that defines the necessary resources, including a deployment, a service, and an ingress rule.
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

## 3. Configure IAM OIDC Provider
AWS IAM OIDC (OpenID Connect) allows Kubernetes to securely authenticate with AWS services. The AWS Load Balancer Controller requires this setup to create and manage ALBs.

### Export the Cluster Name
Stores the cluster name as an environment variable for easier reference.
```sh
export cluster_name=demo-cluster
```

### Extract the OIDC ID
Retrieves the unique identifier for the cluster’s OIDC provider.
```sh
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

### Check for an Existing OIDC Provider
Verifies if an IAM OIDC provider is already set up for the cluster.
```sh
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

### Create the OIDC Provider (if missing)
Associates the EKS cluster with an IAM OIDC provider to enable authentication.
```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## 4. Install the AWS Load Balancer Controller
The AWS Load Balancer Controller enables Kubernetes to provision and manage ALBs in AWS. This is required for exposing services to the internet.

### Download the IAM Policy
This policy grants the controller the necessary permissions to manage ALBs.
```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### Create the IAM Policy
Defines and creates a policy to provide necessary permissions.
```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### Create an IAM Role for the Controller
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

### Install the ALB Controller using Helm
Installs the AWS Load Balancer Controller using Helm, allowing automatic ALB provisioning.
```sh
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

## 5. Access the 2048 Game
Retrieve the ALB address to access the game.
```sh
kubectl get ingress -n game-2048
```
![image](https://github.com/user-attachments/assets/d24788d6-9997-4291-949e-efbb2abd1ec5)


## 6. Cleanup
To remove AWS resources and avoid unnecessary costs:

### Delete the 2048 Game Deployment
```sh
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

### Delete the Fargate Profile
```sh
eksctl delete fargateprofile --cluster demo-cluster --name alb-sample-app
```

### Delete the IAM Policy
```sh
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

### Delete the IAM Role
```sh
aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole
```

### Uninstall the AWS Load Balancer Controller
```sh
helm uninstall aws-load-balancer-controller -n kube-system
```

### Delete the EKS Cluster
```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```

## 7. References
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)

