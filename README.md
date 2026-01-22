# GO-APP-CICD
Implement DevOps practices in the Go web application

# Overview
This project provides a comprehensive guide to implementing end-to-end DevOps practices on a Golang web application, transforming a project without existing DevOps practices into a fully operational and scalable system.

# The project covers the following key stages

**Containerization** : Learn how to containerize the Go web application using a multi-stage Docker build to create optimized and secure Docker images. The project details the process of building and running the Docker image, troubleshooting common issues like Go version mismatches.

**Kubernetes Manifests** : Kubernetes YAML manifests for deployment, service, and Ingress. It emphasizes understanding the structure of these files rather than memorizing them and showcases how to deploy and verify each component on a Kubernetes cluster.

**Kubernetes Cluster Setup** : The project demonstrates setting up an Amazon EKS (Elastic Kubernetes Service) cluster, including prerequisites like kubectl, eksctl, and AWS CLI installation and configuration. It also explains how to expose the application using NodePort mode for testing.

**Ingress Controller Configuration** : The role of an Ingress controller, specifically NGINX Ingress Controller, in exposing the application to the outside world. It clarifies the concepts of Ingress, Ingress controller, and load balancer, and see how to map a DNS name to the load balancer's IP address.

**Helm Chart Creation** : The project introduces Helm for managing Kubernetes applications across different environments (e.g., development, QA, production). It shows how to create a Helm chart and parameterize values to avoid hardcoding configurations.


# Detailed workflow for implementing end-to-end DevOps on a Golang web application.

The workflow steps are as follows:

**Understanding the Project Locally** : Before containerizing, the first step is to get the application's GitHub repository, build it, and run it locally to understand its functionality, how it's accessed (e.g., specific paths like /courses), and its dependencies.

**Containerization with Multi-Stage Dockerfile** : This involves writing a Dockerfile that uses a multi-stage build process to create a smaller and more secure Docker image. The project details setting up a base stage for building the application and a final stage using a distroless image, copying the binary and static content, exposing ports, and running the application. It also covers troubleshooting issues like Go version mismatches during the build.

**Pushing Docker Image to Registry** : Once the Docker image is successfully built and tested locally, it needs to be pushed to a Docker image registry (e.g., Docker Hub) so that Kubernetes can pull it for deployment.

**Creating Kubernetes Manifests** : This step involves writing YAML files for the core Kubernetes resources:
- Deployment.yaml: To manage the application's pods, including replicas and the Docker image to be used.
- Service.yaml: To expose the application within the Kubernetes cluster, providing a stable network endpoint for the pods.
- Ingress.yaml: To manage external access to the services within the cluster, defining host-based routing rules.

**Setting Up a Kubernetes Cluster** : This explains how to set up an Amazon EKS (Elastic Kubernetes Service) cluster, including the necessary prerequisites (kubectl, eksctl, AWS CLI) and authentication.

**Deploying Kubernetes Manifests** : Applying the created deployment, service, and Ingress YAML files to the Kubernetes cluster using kubectl apply.

**Verifying Service Functionality** : Temporarily changing the service type to NodePort to verify if the application is accessible on the cluster nodes before setting up the Ingress controller.

**Configuring Ingress Controller** : Installing and configuring an Ingress controller (specifically NGINX Ingress Controller) to manage the external access defined by the Ingress resource. The controller watches the Ingress resource and provisions a load balancer to expose the application.

**DNS Mapping for External Access** : Mapping the load balancer's external address to a custom domain name (e.g., by modifying /etc/hosts locally for demonstration purposes) to allow external access to the application via the Ingress rules.

**Creating Helm Chart for Multi-Environment Deployments** : Setting up a Helm chart for the application to enable easier deployment and management across different environments (development, QA, production) by parameterizing values like image tags.