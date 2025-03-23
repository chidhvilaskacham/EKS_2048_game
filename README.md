# Amazon EKS Fargate Deployment with 2048 Game Example

This repository provides a step-by-step guide to deploy a 2048 game application on Amazon EKS using Fargate and an Application Load Balancer (ALB). It includes instructions for setting up the EKS cluster, configuring IAM roles, deploying the 2048 game, and installing the AWS Load Balancer Controller.

---

## Prerequisites

Before you begin, ensure you have the following tools installed and configured:

1. **kubectl** - Kubernetes command-line tool.  
   [Installation Guide](https://kubernetes.io/docs/tasks/tools/)  
   This tool is used to interact with your Kubernetes cluster.

2. **eksctl** - CLI tool for Amazon EKS.  
   [Installation Guide](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)  
   This tool simplifies the creation and management of EKS clusters.

3. **AWS CLI** - AWS command-line interface.  
   [Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)  
   This tool is used to interact with AWS services. After installation, configure it using:
   ```bash
   aws configure

   1. Create an EKS Cluster with Fargate
Create the Cluster
Run the following command to create an EKS cluster using Fargate:

```sh
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
This command:
Creates an EKS cluster named demo-cluster in the us-east-1 region.

Uses Fargate as the compute provider, which means you don’t need to manage worker nodes. Fargate automatically provisions the required compute resources.

Delete the Cluster
To clean up and delete the cluster, run:

```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```

2. Deploy the 2048 Game Application
Create a Fargate Profile
Fargate profiles determine which Kubernetes namespaces should run on Fargate. Create a Fargate profile for the game-2048 namespace:

```sh
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```
This ensures that any workloads in the game-2048 namespace will run on Fargate.

Deploy the 2048 Game
Deploy the 2048 game using the provided Kubernetes manifest:

```sh
Copy
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
This manifest creates:

A Deployment to run the 2048 game pods.
A Service to expose the game internally.
An Ingress resource to configure the ALB for external access.

3. Configure IAM OIDC Provider
What is OIDC?
OIDC (OpenID Connect) is an identity layer built on top of OAuth 2.0. It allows Kubernetes to authenticate with AWS using IAM roles. This is required for the AWS Load Balancer Controller to manage ALBs on your behalf.

Steps to Configure OIDC
Export the Cluster Name:

```sh
export cluster_name=demo-cluster
```
Extract the OIDC ID:
The OIDC ID is a unique identifier for your cluster's OIDC provider. Extract it using:

```sh
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
Check for an Existing OIDC Provider:
Verify if an OIDC provider is already configured for your cluster:

```sh
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```
Create the OIDC Provider (if missing):
If no OIDC provider is found, create one:

```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

This step ensures that your EKS cluster can assume IAM roles using OIDC.

4. Install the AWS Load Balancer Controller
The AWS Load Balancer Controller manages ALBs for your Kubernetes Ingress resources.

Download the IAM Policy
Download the IAM policy required for the AWS Load Balancer Controller:

```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
Create the IAM Policy
Create the IAM policy using the downloaded file:

```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
This policy grants the controller permissions to manage ALBs.

Create an IAM Role for the Controller
Replace <ACCOUNT_ID> with your AWS account ID and run:

```sh
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
This creates an IAM role that the AWS Load Balancer Controller can assume.

Install the AWS Load Balancer Controller via Helm
Add the EKS Helm repository and install the controller. Replace <VPC_ID> with your VPC ID:

```sh
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC_ID>
```
Verify the Installation
Check that the AWS Load Balancer Controller is running:

```sh
kubectl get deployment -n kube-system aws-load-balancer-controller
```
5. Access the 2048 Game
Once the ALB is provisioned, retrieve the URL for the 2048 game:

```sh
kubectl get ingress -n game-2048
```
Open the URL in your browser to play the game.

Cleanup
To avoid unnecessary charges, clean up the resources after testing.

Delete the 2048 Game Application
```sh
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Delete the EKS Cluster
```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```
Delete the IAM Policy
Replace <ACCOUNT_ID> with your AWS account ID:

```sh
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```
Notes
VPC ID: To retrieve your VPC ID, run:

```sh
aws ec2 describe-vpcs --filters "Name=tag:alpha.eksctl.io/cluster-name,Values=demo-cluster" --query "Vpcs[].VpcId" --output text
```
ALB Provisioning: It may take 5-10 minutes for the ALB to be provisioned and become available.

IAM Role: Ensure you replace <ACCOUNT_ID> with your AWS account ID in all commands.

References
AWS EKS Documentation
AWS Load Balancer Controller Documentation
Kubernetes Documentation
