# Deploying a Scalable Three-Tier Application on AWS EKS

This project demonstrates the deployment of a **scalable Three-Tier Application** architecture (Web, Application, and Database layers) on **Amazon Elastic Kubernetes Service (EKS)**. The guide includes setting up infrastructure, managing permissions, containerizing applications, and deploying workloads in a robust, scalable environment.

---

## Project Overview

In a three-tier architecture, the application is logically split into three components:

1. **Web Tier**: The user-facing layer, responsible for serving frontend components.
2. **Application Tier**: Manages the business logic and backend services.
3. **Database Tier**: Manages the data storage and retrieval operations.

Using **AWS EKS**, this architecture is containerized for scalability, high availability, and automated management.

---

## Steps Involved

### 1. Create and Connect to an EC2 Instance
To interact with AWS services, set up and Launch an EC2 instance that acts as your control plane.

---

### 2. Install Required Tools in EC2 Instance
The following tools are installed to facilitate interactions with AWS services and Kubernetes clusters, Install all this on an EC2 Instance
- **AWS CLI**: For interacting with AWS resources.
- **eksctl**: Simplifies EKS cluster creation and management.
- **Docker**: For containerizing application components.
- **kubectl**: For managing Kubernetes workloads.

```bash
  # Install AWS CLI
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  sudo apt install unzip
  unzip awscliv2.zip
  sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
  aws configure
  
  # Install eksctl
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
  
  # Install Docker
  sudo apt-get update
  sudo apt install docker.io
  docker ps
  sudo chown $USER /var/run/docker.sock
  
  # Install kubectl
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin
  kubectl version --short --client
  ```
---

### 3. Build and Push the Docker Image into Elastic Container Registry (ECR)
Application components are containerized into Docker images. These images are pushed to **AWS Elastic Container Registry (ECR)**, a managed Docker image repository service.Build and Push your Docker Image for both Frontend and Backend

  ```bash
  # Authenticate Docker with ECR
  aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
      
  # Build your Docker Image
  docker push -t <your App Name?
      
  # After the build completes, tag your Image
  docker tag <image-name>:latest <account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:latest
      
  #Push your Image into ECR
  docker push <account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:latest
  ```
---

### 4. Create an EKS Cluster and Deploy Containers
An **EKS cluster** is created to host the Kubernetes workloads. Application containers are deployed within the cluster using **Kubernetes manifests** to manage Pods, Services, and Deployments.Before deploying the containers into EKS, create a namespace and give the public ECR URI Id for both backend and frontend that we created previously and after that use kubectl apply command to create deplyment and service.

  ```bash
    # Create the EKS Cluster
    eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
    
    #Configure `kubectl` for the Cluster
    aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
    kubectl get nodes
    
    # Deploy the Frontend, backend and database Application
    kubectl create namespace <name-space name>
    kubectl apply -f <yaml-file name>
    kubectl get deployment -n <name-space name>
    kubectl get svc -n <name-space name>
    kubectl get pods -n <name-space name>
    
    # For Database create secrets before entering the above commands
    kubectl create secrets <secret-name>
  ```
![image](https://github.com/user-attachments/assets/fbf7e79f-1660-4c5d-8db1-8d01d4f4d171)

---
### 5. Configure IAM Role and Add IAM OIDC Provider
For secure communication and access, an **IAM role** is created and associated with the EKS cluster. An **IAM OIDC provider** is also added to allow fine-grained permissions for Kubernetes workloads.

  ```bash
    # Enable IAM OIDC Provider
    eksctl utils associate-iam-oidc-provider --region <region> --cluster <cluster-name> --approve
    
    # Download the IAM Policy
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
    
    # Create IAM Policy
    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json
    
    # Create and associate role
    eksctl create iamserviceaccount \
      --cluster=<your-cluster-name> \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --role-name AmazonEKSLoadBalancerControllerRole \
      --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
      --region=<your-region> \
      --approve
```
---

### 6. Install and Deploy the AWS Load Balancer Controller
The **AWS Load Balancer Controller** is installed to manage and provision AWS load balancers (e.g., ALB/NLB) for the application. This ensures smooth and scalable routing of external traffic to the application.

  ```bash
    # Deploy AWS Load Balancer Controller
    
    sudo snap install helm --classic
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update eks
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
    kubectl get deployment -n kube-system aws-load-balancer-controller
    
    # Use this command for setting Routing for ALB Controller
    kubectl apply -f full_stack_lb.yaml
    kubectl get ing -n <name-space>

 ```
![image](https://github.com/user-attachments/assets/c67638ab-8437-477f-9e9f-3ca6bca6ae0c)

---
