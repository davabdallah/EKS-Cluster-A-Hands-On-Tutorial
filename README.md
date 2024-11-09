# EKS-Cluster-A-Hands-On-Tutorial

### Prerequisites

Before getting started with EKS, you need to have the following:

- AWS account
- IAM administrator user with access keys
- AWS CLI, Kubectl, and Eksclt are installed on your local machine:
  
  - AWS CLI Instructions can be found here.

  - Kubectl Instructions can be found here.

  - Eksctl Instructions can be found here.

Run the below command to configure AWS CLI using the admin access key

```sh
aws configure
```

## Create an EKS Cluster

- Launch an EKS Fargate cluster using eksctl by running the below command from your local machine

```sh
eksctl create cluster --name <cluster-name> --region <your-region> --fargate
```

  - This command will first create a VPC to host the EKS cluster.


### Create and  Associate IAM OIDC Provider for the EKS Cluster

To use AWS IAM roles for Kubernetes service accounts on our EKS cluster, we need to create and associate OIDC providers.

```sh
eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --region <your-region> --approve
```

- Now The Cluster has been created and we are ready to deploy the application

## Deploy the game Application

1. Create a Fargate profile 

```sh
eksctl create fargateprofile --cluster <cluster-name> \
    --region <region> \
    --name alb-sample-app \
    --namespace game-2048
```

2. Deploy the game deployment, service and Ingress

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

you can find all the details about the deployments through this link

This command creates:

- **A Namespace:** A dedicated namespace called game-2048 .
- **A Deployment:** Deploys 5 replicas of the game-2048 application.
- **A Service:** Exposes the application on a NodePort.
- **An Ingress:** Creates an Application Load Balancer to expose the application outside the cluster.

There is no Ingress address and We cannot access the application from the internet because we need to create an AWS Load Balancer Controller to expose it.


## Create an AWS Load balancer Controller
- The AWS Load Balancer Controller manages AWS Elastic Load Balancers for a Kubernetes cluster. 
- It directs traffic from the internet to the cluster's ingress resources, exposing your cluster applications.

- We need to create an IAM policy, an IAM role, and a service account within K8S for the AWS controller load balancer.
- **Download the IAM Policy**

```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

- **Create an IAM Policy**

```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

- **Create an IAM Role and an EKS service account**

```sh
eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller-sa \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

- **Deploy AWS load balancer controller using helm chart**

```sh
helm repo add eks https://aws.github.io/eks-charts
```

```sh
helm repo update eks
```

```sh
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller-sa \
  --set region=<region> \
  --set vpcId=<vpc-id>
```

- **Verify the controller deployment**

```sh
kubectl get deployment -n kube-system aws-load-balancer-controller
```

The ingress address become the elb endpoint Now, and we can access the Application through the load balancer endpoint 

## Conclusion

This tutorial provides a comprehensive guide on creating your first EKS cluster on AWS and deploying a simple application. 

It highlights the key benefits of using EKS, including simplified management, scalability, and security.
