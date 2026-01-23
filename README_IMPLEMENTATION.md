# Implementation steps

----------------------------------------------------

# 1. Understanding & testing the Project Locally:

**a.** Get the app code from developer team and clone/copy locally.

![Developer_provided_code_screenshot](static/images/Developer_provided_code_screenshot.png)

**b.** Generally you will get above code from developer, Where:

- static folder contains web app code.

- The main.go file is the entry point of your Go application. It contains the main() function where your program execution begins.

- The go.mod file declares the module path and tracks the project's dependencies. If you don't get it, then initialize your Go module in your local terminal:
```
go mod init github.com/<git-username>/<git-repo>
```
example:
```
go mod init github.com/nikhil600/go-app-cicd
```

- "main" is the compiled binary. The below command is used to build the Go application and compile it into a binary executable file. Below is how you can build it:
```
go build -o main main.go
```
- Once the main binary is created, it can be executed using ./main or go run main.go to run the application in local to test it.
```
go run main.go
```
If the process is running and want to terminate use below:
```
Get-Process | Where-Object {$_.Name -eq "main" -or $_.ProcessName -eq "main"} | Stop-Process -Force -ErrorAction SilentlyContinue; "Process terminated"
```

Note: The server will start on port 8080. You can not access web-app on localhost:8080. As per developer instructions you can access it by navigating to http://localhost:8080/courses in your web browser.


# 2. Containerization with Multi-Stage Dockerfile:

You use a Dockerfile to containerize your project. We are creating a multi-stage Dockerfile to implement images with reduced size and enhanced security.

Here's how a multi-stage Dockerfile achieves this:

**Stage 1 (Builder Stage)**: You can build your Docker image, download all dependencies, and use any base image. This stage compiles your application and creates the executable binary.

**Stage 2 (Final Stage)**: A distroless image is used as the base image, which contributes to security and a reduced image size. The binary built in Stage 1 is then copied to this distroless image.

**Static Content**: The static files (e.g., HTML, CSS) that are not bundled within the binary are also copied to the distroless image.


**1. Build the docker image**:
```
docker build -t anikhilt600/go-web-app:v1 .
```
Note: 
*The Go image version issue may occurs during the Docker build process when you run the docker build command.*

*The error message "go.mod requires go >= 1.25.6 (running go 1.21.13; GOTOOLCHAIN=local)" indicates that the Go application's go.mod file specifies a minimum Go version of 1.25.6. However, in your Dockerfile, you initially used an older Go base image, specifically Go 1.21.*

*To resolve this, change the Go version in the Dockerfile to 1.25 or 1.25.6 to match the version required by the project and rebuild the docker image.*

Verify image
```
docker images
```

**2. Test the docker image locally**: Run the docker run command, and verify in browser.
```
docker run -p 8080:8080 -it anikhilt600/go-web-app:v1
```

# 3. Pushing Docker Image to Registry

Push the tested docker image to dockerhub, so we can use image for kubernetes deployment.
```
docker push anikhilt600/go-web-app:v1
```


# 4. Creating Kubernetes Manifest

Create a deployment.yaml, service.yaml, and ingress.yaml files in /k8s/manifests/.

deployment.yaml file is used to deploy the application on a Kubernetes cluster. [refer how to create deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

service.yaml file defines a Kubernetes service, which provides a stable network endpoint for your application. [refer how to create service](https://kubernetes.io/docs/concepts/services-networking/service/)

ingress.yaml file configures Kubernetes Ingress, which manages external access to services within the cluster, typically HTTP and HTTPS. [refer how to create ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

**significance of labels and selectors in Kubernetes**

Labels are key-value pairs attached to Kubernetes objects (like pods, deployments, and services). They are used to organize and select subsets of objects.

Selectors are used by Kubernetes controllers (like Deployment controllers and Services) to find and manage objects that match specific labels.

In our yaml files label app: go-web-app is assigned to the pod, and the service's selector is configured to match this label. This ensures that the service routes traffic only to pods with this specific label, enabling proper service discovery within the cluster.
Similarly, ingressClassName acts as a selector for Ingress controllers, telling them which Ingress resources to watch.


# 5. Setting Up a Kubernetes Cluster

Before starting the EKS cluster creation, several command-line tools must be installed and configured on your local machine

**Installing the AWS CLI**:
Download and install the AWS CLI on your local machine. You can find installation instructions for various operating systems [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).

*Configuring AWS CLI Credentials:*
Access Keys (for Programmatic Access):
- If you selected "Programmatic access" during user creation, you will receive access keys (Access Key ID and Secret Access Key).
- Store these access keys securely, as they will be used to authenticate API requests made to AWS services.

Open a terminal or command prompt and run the following command:
```
aws configure
```
Enter the access key ID and secret access key of the IAM user you created earlier.
Choose a default region and output format for AWS CLI commands.

**Installing kubectl**:
Install kubectl on your local machine. Instructions can be found [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

**Installing eksctl**:
eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see [Installing or updating](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).


**Create eks cluster**

The EKS cluster is created using the eksctl utility:
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
```
eksctl get cluster --region us-east-1
```


# 6. Deploying Kubernetes Manifests

Once eks cluster created, the cluster is ready for deployments.

Applying three core Kubernetes YAML files:

**deployment.yaml**: This file defines the desired state of your application, specifying the Docker image to use (go-web-app:V1) and the number of replicas (pods) to run.

To apply: 
```
kubectl apply -f k8s/manifest/deployment.yaml
```
To verify pods: 
```
kubectl get pods
```

**service.yaml**: This file creates a stable network endpoint for your application's pods, allowing consistent access even if pods are replaced or moved.

To apply: 
```
kubectl apply -f k8s/manifest/service.yaml
```
To verify service: 
```
kubectl get svc
```

**ingress.yaml**: This configures external access to your service, defining rules for routing HTTP/S traffic based on hostnames, for this demo project it's set to route traffic for go-web-app.local to the go-web-app service.

To apply: 
```
kubectl apply -f k8s/manifest/ingress.yaml
```
To verify ingress: 
```
kubectl get ingress
```

# 7. Verifying Service Functionality

Applying deployment.yaml creates pods. Applying service.yaml with ClusterIP type creates a stable internal IP for these pods. However, ClusterIP services are only reachable within the Kubernetes cluster, not from the public internet. And applying ingress.yaml defines rules for external access, but these rules are useless without an Ingress Controller.

This is a crucial intermediate step to confirm your application is running correctly within the cluster and is reachable via the Service. By changing the service type to NodePort, Kubernetes exposes the application on a specific port on every node's IP address. This allows you to access the application from the internet using any node's external IP and the assigned NodePort.

**Editing Service Type to NodePort**
We initially defines the service with type ClusterIP, which means it's only accessible from within the cluster and not from the public internet. To test external access before setting up the Ingress controller, the service type is changed to NodePort:

How to edit: 
```
kubectl edit svc go-web-app 
```
and change type: ClusterIP to type: NodePort

*Why NodePort*: A NodePort service exposes your application on a static port on each node's IP address. This allows you to access the application from outside the cluster using any of the node's external IP addresses and the assigned NodePort.

*How to Get service PORT*: To get the service port, you can use the command:
```
kubectl get svc
```
*How to Get Nodes External IP*: To get the external IP addresses of your EKS cluster's nodes, you can use the command:
```
kubectl get nodes -o wide
```
This command displays detailed information about your cluster nodes, including the EXTERNAL-IP column, which provides the public IP addresses you can use to access the NodePort service. (Can pick any node IP)

**To verify the service functionality** Nagivate to browser tab and enter below in address:
```
Nodes_External_IP:service_PORT
```
Example: 54.161.25.151:32099/courses

The NodePort verification is a testing method, while the Ingress Controller is the production-ready solution for exposing applications via HTTP/S.



# 8. Configuring Ingress Controller

The Ingress Controller is essential because the Kubernetes cluster itself cannot directly create external load balancers or manage sophisticated routing rules for external traffic.The Ingress Controller watches your ingress.yaml rules, provisions a load balancer, and then routes external traffic based on your configured hostnames and paths.

**Installation of Nginx Ingress Controller**:

We are using the Nginx Ingress Controller for AWS Network Load Balancer (NLB). You can refer official Nginx Ingress Controller documentation to find installation steps already provided in the associated GitHub repository for easy copy-pasting [here](https://kubernetes.github.io/ingress-nginx/deploy/#aws).

Below is the command for installation:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/aws/deploy.yaml
```

**Verification**: After running the installation command, you can check if the Nginx Ingress Controller pods are running using 
```
kubectl get pods -n ingress-nginx
```

- The ingress.yaml file contains an ingressClassName field, set to nginx in this case. This class name is crucial because in environments with multiple Ingress controllers (e.g., Nginx, ALB, Kong), it tells a specific Ingress controller which Ingress resources it should "watch" and manage.

- The Ingress Controller itself starts with a configuration that tells it which ingressClassName it is responsible for.

- Once the Nginx Ingress Controller is running, it watches your ingress.yaml and provisions a Network Load Balancer (NLB) in AWS. You can verify this by running *kubectl get ingress* , which will show the DNS ADDRESS of the newly provisioned load balancer.
```
kubectl get ingress
```

If you try to access the application directly using the NLB's domain name address, you will gets a "404 Not Found" error. This is because the Ingress.yaml resource is configured to only accept requests on a specific hostname, go-web-app.local, not directly on the load balancer's default domain.


# 9. DNS Mapping for External Access

To resolve the "404 Not Found" error and successfully access the application, we need to do local DNS mapping. Mapping the load balancer's external address to a custom domain name (e.g., by modifying /etc/hosts locally for demonstration purposes) to allow external access to the application via the Ingress rules.

**DNS Mapping using /etc/hosts**:

- First, get the NLB_DNS_ADDRESS of the newly created load balancer
```
kubectl get ingress

```

- Then, the IP address of the newly created load balancer is obtained using nslookup on the load balancer's domain name.
```
nslookup NLB_DNS_ADDRESS
```

- Then, this IP address is manually mapped to the configured hostname (go-web-app.local) in the local machine's /etc/hosts file.


- After this mapping, when you access go-web-app.local in your browser, your local machine resolves it to the load balancer's IP, and the load balancer, now receiving the request with the correct hostname, routes it to your application.

Note: In a real-world corporate environment, this DNS mapping would be done in a proper DNS system (like AWS Route 53 or an on-premise DNS server), not just in /etc/hosts.