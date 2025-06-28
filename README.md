# Multi-Cloud Disaster Recovery Project

Multi-Cloud Disaster Recovery Project: Flask + MySQL Application
Table of Contents
Project Overview

Architecture Diagram

Technology Stack

Prerequisites

Local Development Setup (Docker Compose)

Files

Running Locally

Cloud Deployment (Kubernetes via Terraform & GitOps)

Application Containerization

Kubernetes Manifests

Infrastructure Provisioning (Terraform)

Azure Kubernetes Service (AKS)

Amazon Elastic Kubernetes Service (EKS)

Automated Deployment (Argo CD GitOps)

Disaster Recovery Strategy (DNS Failover)

Testing Disaster Recovery

Cleanup

Contributing

License

1. Project Overview
This project implements a robust multi-cloud disaster recovery (DR) strategy for a simple web application. The core application consists of a Python Flask frontend and a MySQL database backend. The entire stack is containerized using Docker and deployed on Kubernetes clusters across two major cloud providers: Azure Kubernetes Service (AKS) and Amazon Elastic Kubernetes Service (EKS).

The solution leverages Terraform for infrastructure as code (IaC) to provision and manage the Kubernetes clusters, ensuring repeatability and consistency. Argo CD is used for GitOps, enabling automated, declarative deployments to both clusters directly from a Git repository. Finally, DNS failover is configured to automatically redirect user traffic to the healthy cluster in the event of a regional outage or primary cluster failure, ensuring high availability and business continuity.

2. Architecture Diagram
text
+------------------+     +------------------------+     +------------------+
|                  |     | Global DNS (e.g., R53) |     |                  |
|   User Request   |---->| (Health Checks &       |---->|   Healthy Cloud  |
|                  |     | Failover Routing)      |     |     Cluster      |
+------------------+     +------------------------+     | (Primary/Backup) |
                                                         |                  |
                                                         |  +------------+  |
                                                         |  | Kubernetes |  |
                                                         |  | Cluster    |  |
                                                         |  | (AKS/EKS)  |  |
                                                         |  | +--------+ |  |
                                                         |  | | Flask  | |  |
                                                         |  | | App    | |  |
                                                         |  | +---+----+ |  |
                                                         |  |     |      |  |
                                                         |  |   +---+----+ |  |
                                                         |  |   | MySQL  | |  |
                                                         |  |   | DB     | |  |
                                                         |  |   +--------+ |  |
                                                         |  +------------+  |
                                                         |                  |
                                                         +------------------+
                                                                ^  |
                                                                |  | GitOps Sync
                                                                |  v
                                                              +------------+
                                                              |            |
                                                              | Git Repo   |
                                                              | (Manifests)|
                                                              |            |
                                                              +------------+
3. Technology Stack
Frontend: Flask (Python)

Backend: MySQL

Containerization: Docker

Container Orchestration: Kubernetes (K8s)

Cloud Providers: Azure (AKS) & Amazon Web Services (AWS) (EKS)

Infrastructure as Code (IaC): Terraform

GitOps: Argo CD

DNS Management: AWS Route 53 (or Azure Traffic Manager)

Version Control: Git / GitHub

4. Prerequisites
Python 3.x

Docker Desktop (with Kubernetes enabled, if testing locally)

kubectl (Kubernetes command-line tool)

Terraform CLI

Azure CLI

AWS CLI

An Azure account with sufficient permissions

An AWS account with sufficient permissions

A Docker Hub account (or other container registry)

A GitHub account

5. Local Development Setup (Docker Compose)
This section details how to run the application locally using Docker Compose for development and testing.

Files
app.py: The Flask application code. (See attached file 1)

requirements.txt: Python dependencies for the Flask app. (See attached file 3)

Dockerfile: Defines how to build the Flask application's Docker image.

docker-compose.yml: Orchestrates the Flask app and MySQL database for local development. (See attached file 2)

Running Locally
Clone the repository:

bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
Build and run the containers using Docker Compose:

bash
docker compose up --build
This command will build the Flask app image, pull the MySQL image, create a network for them, and start both services.

The Flask app will be accessible on http://localhost:5000.

Verify the application:
Open your web browser and navigate to http://localhost:5000. You should see the message: "Hello from the MySQL database!"

Stop and clean up:
To stop the running containers, press Ctrl+C in your terminal. To remove the containers, networks, and volumes:

bash
docker compose down
6. Cloud Deployment (Kubernetes via Terraform & GitOps)
This section covers the deployment of the application to production-like environments in Azure and AWS using Infrastructure as Code and GitOps principles.

Application Containerization
The Dockerfile defines the build process for the Flask application. After building, the image is pushed to Docker Hub to be accessible by Kubernetes clusters in different clouds.

Build the Docker image:

bash
docker build -t flask-mysql-demo:latest .
Tag the image for Docker Hub:

bash
docker tag flask-mysql-demo:latest <your-dockerhub-username>/flask-mysql-demo:latest
Push the image to Docker Hub:

bash
docker push <your-dockerhub-username>/flask-mysql-demo:latest
Kubernetes Manifests
The application and database are defined using standard Kubernetes YAML manifests, enabling declarative deployments. These files will be stored in a Git repository for GitOps.

mysql-deployment.yaml: Defines the MySQL Deployment and Service. Includes an emptyDir volume for simplicity; in production, this should be replaced with a PersistentVolumeClaim.

flask-deployment.yaml: Defines the Flask application Deployment and Service (exposed as a NodePort for testing, could be LoadBalancer or Ingress in production).

Infrastructure Provisioning (Terraform)
Terraform is used to provision the Kubernetes clusters (AKS and EKS) in a consistent and repeatable manner.

Azure Kubernetes Service (AKS)
Terraform Files:

aks/main.tf: Defines the Azure Resource Group and AKS cluster.

Deployment:

bash
cd aks
terraform init
terraform apply
After deployment, configure kubectl:

bash
az aks get-credentials --resource-group <resource-group-name> --name <aks-cluster-name>
Amazon Elastic Kubernetes Service (EKS)
Terraform Files:

eks/main.tf: Defines the AWS EKS cluster, node groups, and necessary VPC/Subnet configurations.

Deployment:

bash
cd eks
terraform init
terraform apply
After deployment, configure kubectl:

bash
aws eks --region <aws-region> update-kubeconfig --name <eks-cluster-name>
Note: Ensure your AWS CLI is configured with appropriate credentials and region.

Automated Deployment (Argo CD GitOps)
Argo CD is deployed on both AKS and EKS clusters to automate the synchronization of Kubernetes manifests from this Git repository.

Install Argo CD on each cluster:

bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Create an Argo CD Application:
A YAML file (e.g., argocd-app.yaml) defines the Argo CD application, pointing to this Git repository and the target cluster.

text
# Example argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flask-mysql-dr
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<your-username>/<your-repo>.git' # Point to YOUR repo
    targetRevision: HEAD
    path: . # Path to your Kubernetes manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
Apply the Argo CD application manifest to each cluster:

bash
kubectl apply -f argocd-app.yaml -n argocd
This sets up continuous synchronization, ensuring any changes pushed to the Git repository are automatically applied to the respective Kubernetes cluster.

7. Disaster Recovery Strategy (DNS Failover)
A DNS-based failover mechanism (e.g., using AWS Route 53 or Azure Traffic Manager) is implemented to route user traffic to the active and healthy cluster.

Get Public IPs: Obtain the external IP address of the flask-service (LoadBalancer IP or Ingress IP) from both AKS and EKS clusters.

Configure DNS Records:

Create a hosted zone for your domain (e.g., myapp.example.com).

Create two A records (or alias records) for your application, pointing to the external IPs of the AKS and EKS services.

Configure health checks for each endpoint to monitor the application's availability.

Set the routing policy to "Failover", with one cluster designated as primary and the other as secondary.

MySQL Database Replication: For a robust DR, MySQL database replication (e.g., asynchronous master-slave replication, or using cloud-managed replicated databases) would be configured between the regions. This project focuses on application-level DR for simplicity.

8. Testing Disaster Recovery
To validate the DR strategy:

Simulate an outage: Manually scale down the flask-app deployment in the primary cluster (e.g., AKS) to 0 replicas, or delete the entire namespace.

Observe DNS failover: Monitor the DNS records to confirm that traffic is redirected to the secondary cluster (EKS) within the configured DNS TTL and health check intervals.

Verify application availability: Confirm that the application remains accessible and functional through the DNS endpoint.

Restore primary: Scale up the primary cluster's deployment or restore the namespace.

Observe failback: Monitor the DNS to confirm traffic returns to the primary (if configured for active-passive failback).

9. Cleanup
To avoid incurring cloud costs, remember to tear down the provisioned infrastructure.

Delete Kubernetes deployments:

bash
# For AKS
kubectl config use-context <aks-context>
kubectl delete -f mysql-deployment.yaml
kubectl delete -f flask-deployment.yaml
kubectl delete namespace argocd # Also deletes Argo CD
# For EKS
kubectl config use-context <eks-context>
kubectl delete -f mysql-deployment.yaml
kubectl delete -f flask-deployment.yaml
kubectl delete namespace argocd # Also deletes Argo CD
Destroy Terraform-provisioned infrastructure:

bash
cd aks
terraform destroy
cd ../eks
terraform destroy
Remove Docker images locally:

bash
docker rmi <your-dockerhub-username>/flask-mysql-demo:latest
docker rmi flask-mysql-demo:latest
Remove DNS records and health checks from your DNS provider.

10. Contributing
Feel free to fork this repository, open issues, or submit pull requests.

11. License
This project is open-source and available under the MIT License.
