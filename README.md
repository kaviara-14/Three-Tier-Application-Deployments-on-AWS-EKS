# Deployed a Three-Tier Application on AWS EKS

- Designed and deployed a scalable three-tier architecture (Web, Application, Database) using AWS Elastic Kubernetes Service (EKS).
- Containerized application components with Docker and managed images in AWS Elastic Container Registry (ECR).
- Automated cluster setup with eksctl, configured IAM roles, and integrated the OIDC provider for secure access control.
- Deployed Kubernetes workloads using kubectl, managing namespaces, deployments, and services.
- Configured AWS Load Balancer Controller for efficient traffic management via ALB/NLB, ensuring high availability and fault tolerance.
- Automated infrastructure setup with AWS CLI and helm, implementing best practices for scalability and security.
  
![image](https://github.com/user-attachments/assets/38ec4ed7-35e4-4d00-8a28-14a40b248966)

---

## Project Overview

In a three-tier architecture, the application is logically split into three components:

1. **Web Tier**: The user-facing layer, responsible for serving frontend components.
2. **Application Tier**: Manages the business logic and backend services.
3. **Database Tier**: Manages the data storage and retrieval operations.

Using **AWS EKS**, this architecture is containerized for scalability, high availability, and automated management.

---

## Steps for deployment


### 1. Create and Connect to an EC2 Instance
Start by setting up an EC2 instance that will serve as your control plane for managing the EKS cluster.

---

### 2. Install Required Tools in EC2 Instance
The following tools are installed to facilitate interactions with AWS services and Kubernetes clusters, Install all this on your EC2 Instance
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
Authenticate Docker with your AWS ECR and push the both Frontend and Backend image.
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
![image](https://github.com/user-attachments/assets/38cca81c-7451-4708-a6d7-628b90ce29a2)

---

### 4. Install and Configure Kubernetes(EKS Cluster)
Set up the EKS cluster using eksctl and configure kubectl.

  ```bash
    # Run the following command to setup EKS cluster
    eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
    
    #Configure `kubectl` for the Cluster
    aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
    kubectl get nodes

    #Create namespace
    kubectl create namespace <name-space name>

    # Creating Deployment and Services for frontend and backend
    kubectl apply -f <yaml-file name>
    kubectl get deployment -n <name-space name>
    kubectl get svc -n <name-space name>
    kubectl get pods -n <name-space name>
    
    # For Database Goto Database directory in K8s manifest path and deployment the apps
    cd Database
    kubectl apply -f .
    kubectl get all
  ```

![image](https://github.com/user-attachments/assets/2dc431b8-2a2c-4d94-9017-ce0aa318c90b)

---
### 5. Create a Service Account 
Create an IAM policy and associate it with a service account for the AWS Load Balancer Controller.

  ```bash
      
    # Below command fetch the iam policy for your ALB
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
    
    # This command creates the iam policy in your AWS account from the iam_policy.json file 
    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json
    
    # This command apply the load balancer policy to your eks cluster so that your eks cluster is working with your load balancer according to the policy
    eksctl utils associate-iam-oidc-provider --region <region> --cluster <cluster-name> --approve
  
    # This command create and attach an service account to your cluster so that your cluster is allowed to work with load balancer service.
    eksctl create iamserviceaccount \
      --cluster=<your-cluster-name> \
      --namespace=<name-space>\
      --name=aws-load-balancer-controller \
      --role-name AmazonEKSLoadBalancerControllerRole \
      --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
      --region=<your-region> \
      --approve
```
---

### 6. Setting up Application Load Balancer (ALB) and Ingress controller
Use Helm to deploy the AWS Load Balancer Controller.To access the application we need to configure below,
- We need to configure ALB (Application Load Balancer) to allow external traffic into AWS EKS Cluster.
- ***Ingress*** to enable internal routing between all three tiers within the cluster.
  
``` bash
    # Deploy the Load Balancer Controller, using Helm. Helm is a package manager for Kubernetes that simplifies deploying and managing applications on a Kubernetes cluster
    sudo snap install helm --classic
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update eks

    # Install the load balancer controller on your eks cluster
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n <name-space> --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
    kubectl get deployment -n <name-space> aws-load-balancer-controller
    
    # Now setup ingress for internal routing
    kubectl apply -f ingress.yaml
    kubectl get ing -n <name-space>
```
 

![image](https://github.com/user-attachments/assets/67c77f1b-f658-49d7-abec-83ce5e84f21d)
![image](https://github.com/user-attachments/assets/a74cfdc8-5f75-420d-8caf-471390215ee5)

---
