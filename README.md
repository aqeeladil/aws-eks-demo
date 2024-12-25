# Introduction to EKS:

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service by AWS that simplifies the deployment, management, and scaling of containerized applications.

1. **Manual Kubernetes Challenges:**
    - Setting up Kubernetes manually involves managing master nodes (control plane) and worker nodes (data plane), renewing certificates, managing scheduler crashes, etc.

2. **EKS Solution:**
    - Managed Control Plane: AWS manages the control plane (master nodes) (API Server, etcd, and scheduler) for you. AWS handles upgrades, patches, and ensures high availability and easier maintenanceof the control plane.
    - Flexible worker node setup: Use EC2 instances or AWS Fargate (serverless compute).
    - Automated Updates: EKS automatically updates the Kubernetes version, eliminating the need for manual intervention and ensuring that the cluster stays up-to-date with the latest features and security patches.
    - Scalability: EKS can automatically scale the Kubernetes control plane based on demand, ensuring the cluster remains responsive as the workload increases.
    - AWS Integration: EKS seamlessly integrates with various AWS services, such as AWS IAM for authentication and authorization, Amazon VPC for networking, and AWS Load Balancers for service exposure.
    - Security and Compliance: EKS is designed to meet various security standards and compliance requirements, providing a secure and compliant environment for running containerized workloads.
    - Monitoring and Logging: EKS integrates with AWS CloudWatch for monitoring cluster health and performance metrics, making it easier to track and troubleshoot issues.
    - Ecosystem and Community: Being a managed service, EKS benefits from continuous improvement, support, and contributions from the broader Kubernetes community.
    - Reduces DevOps complexity compared to setting up and managing a Kubernetes cluster using tools like kubeadm or kops.

3. **Control Plane (Managed by AWS):**
    - API Server: Handles requests from users and services.
    - etcd: Stores cluster state and configurations.
    - Controller Manager: Ensures desired state matches actual state.
    - Scheduler: Allocates workloads to appropriate nodes.

4. **Data Plane (Managed by User via EC2 or Fargate):**
    - EC2 Instances: Custom-configured nodes for more control (customizable but requires manual scaling and updates).
    - AWS Fargate: Serverless compute for Kubernetes pods, no infrastructure management required (serverless containers with automatic scaling).

5. **Cons of using Eks:**
    - Cost: EKS is a managed service, and this convenience comes at a cost. Running an EKS cluster may be more expensive compared to self-managed Kubernetes, especially for large-scale deployments.
    - Less Control: While EKS provides a great deal of automation, it also means that you have less control over the underlying infrastructure and some Kubernetes configurations.

6. **Pros of Manual (Self-Managed) Kubernetes:**
    - Cost-Effective: Self-managed Kubernetes allows you to take advantage of EC2 spot instances and reserved instances, potentially reducing the overall cost of running Kubernetes clusters.
    - Flexibility: With self-managed Kubernetes, you have full control over the cluster's configuration and infrastructure, enabling customization and optimization for specific use cases.
    - EKS-Compatible: Self-managed Kubernetes on AWS can still leverage various AWS services and features, enabling integration with existing AWS resources.
    - Experimental Features: Self-managed Kubernetes allows you to experiment with the latest Kubernetes features and versions before they are officially supported by EKS.

7. **Cons of using Self-Managed Kubernetes:**
    - Complexity: Setting up and managing a self-managed Kubernetes cluster can be complex and time-consuming, especially for those new to Kubernetes or AWS.
    - Maintenance Overhead: Self-managed clusters require manual management of Kubernetes control plane updates, patches, and high availability.
    - Scaling Challenges: Scaling the control plane of a self-managed cluster can be challenging, and it requires careful planning to ensure high availability during scaling events.
    - Security and Compliance: Self-managed clusters may require additional effort to implement best practices for security and compliance compared to EKS, which comes with some built-in security features.
    - Lack of Automation: Self-managed Kubernetes requires more manual intervention and scripting for certain operations, which can increase the risk of human error.

## Deploying a sample game application (2048) to the EKS cluster.

### 1. Launch and connect to the EC2 Instance:
- Select the latest Ubuntu AMI.
- Choose a t2.medium or larger instance type (for sufficient resources).
- Assign a security group allowing: 
    - SSH (Port `22`)
    - HTTP (Port `80`)
- Connect to the instance.
    - `ssh -i "your-key.pem" ubuntu@<instance-public-ip>`

### 2. Update the System & Install Pre-requisites
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar unzip -y

# Install Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

# Install Aws CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
rm awscliv2.zip

# Configure awscli (Obtain ```Access-Key-ID``` and ```Secret-Access-Key``` from the AWS Management Console).
aws configure

# Install eksctl
curl -LO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
tar -xzf eksctl_$(uname -s)_amd64.tar.gz
sudo mv eksctl /usr/local/bin
eksctl version
rm eksctl_$(uname -s)_amd64.tar.gz

# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 3. Cluster Setup with eksctl using Aws Fargate
```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate

# This will:
# Create a VPC with public and private subnets.
# Set up Fargate profiles for namespaces (default and kube-system).
# Deploy a Kubernetes control plane managed by AWS.
# Configures networking for the cluster.
# --fargate: Sets up Fargate profiles to manage serverless pods (worker nodes).
# This step takes 10-15 minutes. 

# Verfiy Cluster
eksctl get cluster --name demo-cluster --region us-east-1

# Configuring kubectl for EKS:
aws eks update-kubeconfig --region us-east-1 --name demo-cluster

# Verify cluster access:
kubectl get nodes

# Create a namespace for the application:
kubectl create namespace game-2048
```

### 4. Create Custom Fargate profile dedicated to game-2048 namespace
```bash
# # If you want to deploy resources in a custom namespace (e.g., game-2048), create a Fargate profile
# create a custom Fargate profile for the demo-cluster. 
# The profile allows Kubernetes pods in the game-2048 namespace to run on AWS Fargate.
# this Fargate profile applies only to the game-2048 namespace.
# Pods in other namespaces (besides default and kube-system) won't run on Fargate unless explicitly configured.
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name fargate-profile-1 \
    --namespace game-2048

# Verify the Fargate profile creation:
eksctl get fargateprofile --cluster demo-cluster --region us-east-1

# This setup ensures that:
# System-level pods (e.g., CoreDNS) run in the kube-system namespace on Fargate.
# Application pods (e.g., the 2048 game app) run in the game-2048 namespace on Fargate.
```

### 5. Deploy the Application on EKS
```bash
# deploy.yaml specifies the application pod replicas and container image.
kubectl apply -f mainfests/2048/deploy.yaml -n game-2048
# service.yaml exposes pods internally using ClusterIP and externally using LoadBalancer or NodePort.
kubectl apply -f mainfests/2048/service.yaml -n game-2048

# Verify that pods and services are running:
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get all -n game-2048
```

### 6. Setting Up Application Load Balancer (ALB) Ingress Controller which manages application load balancers for external traffic.
```bash
# Associate an IAM OIDC identity provider to integrate and authenticate Kubernetes service accounts with AWS IAM roles.

export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Check if there is an IAM OIDC provider configured already
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n
# If not, run the below command to enable IAM OIDC Provider for the EKS cluster:
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster demo-cluster \
  --approve

# Attach IAM Role to Service Account: Grant specific permissions.

# Download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create IAM Policy for the ALB controller:
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create a Kubernetes service account with an IAM role
# Attach the policy to the Kubernetes service account:
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  --override-existing-serviceaccounts

# The ALB controller watches Kubernetes Ingress resources and automatically creates and configures an ALB for traffic routing.
# Public traffic routed via `ALB` → `Ingress Controller` → `Service` → `Pod`. 

# Deploy the ALB Controller using Helm:
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install the controller:
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set us-east-1 \
  --set vpcId=<your-vpc-id>

# Verify Installation:
kubectl get pods -n kube-system
```

### 7. Apply the Ingress resource to expose the application via ALB
```bash
kubectl apply -f mainfests/2048/ingress.yaml -n game-2048

# Ingress Resource (ingress.yaml) exposes the app externally using hostname-based rules.

# Verify Ingress:
kubectl get ingress -n game-2048
```

### 8. Access the Application
- Access Application using the <EXTERNAL-IP> from the Ingress output.
    - `kubectl get ingress -n game-2048`
    - `http://<load-balancer-address>`

### 9. Monotir and Verify
```bash
# Check Resources
kubectl get pods --all-namespaces
kubectl get all -n game-2048

# Logs of a Pod
kubectl logs <pod-name> -n game-2048

# Check the Ingress Resource
kubectl get deploy -n kube-system
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl logs deployment/aws-load-balancer-controller -n kube-system

# View ALB in AWS Console
# Go to EC2 Console > Load Balancers
# Verify ALB setup and listeners.
```

### 10. Cleanup Resources
```bash
eksctl delete cluster --name demo-cluster --region us-east-1

# Verify deletion:
eksctl get cluster --name demo-cluster --region us-east-1
```
