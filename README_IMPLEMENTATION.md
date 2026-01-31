# High level flow

Local Test
  ↓
Docker Build & Test
  ↓
Docker Registry
  ↓
EKS (Fargate)
  ↓
Ingress + LoadBalancer
  ↓
DNS Mapping
  ↓
Helm
  ↓
GitHub Actions (CI)
  ↓
Argo CD (CD)
  ↓
Live Application


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
eksctl create cluster --name demo-cluster --region us-east-1
```
```
eksctl get cluster --region us-east-1
```

NOTE: If you want to use aws fargate instead of ec2 refer : [EKS Fargate Guide](./Fargate.md)

# 6. Deploying Kubernetes Manifests

Once eks cluster created, the cluster is ready for deployments.

Applying three core Kubernetes YAML files:

**deployment.yaml**: This file defines the desired state of your application, specifying the Docker image to use (go-web-app:V1) and the number of replicas (pods) to run.

To apply: 
```
kubectl apply -f k8s/manifests/deployment.yaml
```
To verify pods: 
```
kubectl get pods
```

**service.yaml**: This file creates a stable network endpoint for your application's pods, allowing consistent access even if pods are replaced or moved.

To apply: 
```
kubectl apply -f k8s/manifests/service.yaml
```
To verify service: 
```
kubectl get svc
```

**ingress.yaml**: This configures external access to your service, defining rules for routing HTTP/S traffic based on hostnames, for this demo project it's set to route traffic for go-web-app.local to the go-web-app service.

To apply: 
```
kubectl apply -f k8s/manifests/ingress.yaml
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
nslookup <NLB_DNS_ADDRESS>
```

- Then, this IP address is manually mapped to the configured hostname (go-web-app.local) in the local machine's /etc/hosts file.

For Linux edit:
```
vim /etc/hosts
```
For windows edit:
```
C:\Windows\System32\drivers\etc\hosts
```
and add *IP hostname* at last. For example: *3.232.186.240 go-web-app*

- After this mapping, when you access go-web-app.local in your browser, your local machine resolves it to the load balancer's IP, and the load balancer, now receiving the request with the correct hostname, routes it to your application.

On browser:
```
go-web-app.local/courses
```

Note: In a real-world corporate environment, this DNS mapping would be done in a proper DNS system (like AWS Route 53 or an on-premise DNS server), not just in /etc/hosts.


# 10. Creating Helm Chart for Multi-Environment Deployments

Helm is used in this project to manage the deployment of the application to different environments.

**Why Helm is being used**:

*Environment Management*: Instead of creating separate Kubernetes manifest files (deployment, service, Ingress) for each environment (like Dev, QA, or Production), Helm allows for variable substitution. This means you can define placeholders for things like image names (e.g., go-web-app:dev, go-web-app:qa, go-web-app:prod) in a single Helm chart.

*Reduced Redundancy*: It avoids the need to maintain multiple copies of similar Kubernetes YAML files for different environments.

*Simplified Deployment*: Developers or testing teams can simply pass the required tag name (e.g., dev, qa, prod) as a variable when deploying the application using the Helm chart.

**Installing Helm**:
Install Helm on your local machine. Instructions can be found [here](https://helm.sh/docs/intro/install/).

Verify the helm installation:
```
helm version
```

**Creating Helm chart**:
Create helm folder in project root, and go inside the helm folder:
```
cd helm
```
Create new helm charts: This command automatically generates a folder structure with pre-configured Helm chart files 
```
helm create go-web-app-chart
```
The created folder go-web-app-chart contains default configuration files. You can delete the charts subfolder, as it's not needed for this project.
```
rm -rf charts
```

**Looking into helm folder**
The charts.yaml, templates/, and values.yaml files are automatically generated when you create/initialize a Helm chart.

Here's a breakdown of what these files are generally used for in a Helm chart:

*Chart.yaml*: This file contains metadata about the Helm chart, such as the chart's name, version, and a brief description. It's essentially the identity card for your Helm chart.

*templates/*: This directory holds the actual Kubernetes manifest files (like deployment.yaml, service.yaml, ingress.yaml) that define your application's resources. These files are written using Go template language, allowing you to insert dynamic values.

*values.yaml*: This file stores the default configuration values for your Helm chart. These values are then injected into the templates within the templates/ directory. For example, you might define the Docker image name or the number of replicas here. This allows you to customize deployments for different environments without modifying the core template files themselves.


**Move the Kubernetes manifests to helm templates**
When the helm create command is executed, it automatically generates a templates directory along with Chart.yaml and values.yaml files. Delete the helm/go-web-app-chart/templates/* :
```
rm -rf templates/*
```
and Move the content of your previously created Kubernetes manifests (from the k8s/manifests folder) into this templates directory:
```
cd templates
```
```
cp ../../../ k8s/manifests/* .
```
By doing this, you're essentially replacing the direct application of individual, hardcoded *.yaml files with a parameterized Helm chart:

The deployment.yaml, service.yaml, and ingress.yaml that you manually created earlier are effectively converted into templates within the helm/go-web-app-chart/templates/.
These templates will then use variables (defined in values.yaml) to allow for dynamic configuration based on the target environment, rather than needing separate, distinct YAML files for each one.

**Update deployment.yaml and Value.yaml**

1. Modify deployment.yaml (inside templates/) to use template variables:

Open the deployment.yaml file (located in helm/go-web-app-chart/templates/) and replace static values with Helm's templating syntax.

Example *image: anikhilt600/go-web-app:V1* change this to *image: {{ .Values.image.repository }}:{{ .Values.image.tag }}**.

This tells Helm: "Look into the values.yaml file, find the image section, and then use the values for repository and tag."

2. Define values in values.yaml:

Open the values.yaml file (located in helm/go-web-app-chart/) and define the default values for the variables used in your templates.

Example for the above template:
yaml
image:
repository: anikhilt600/go-web-app
tag: V1

For different environments, you could either create override values.yaml files (e.g., values-dev.yaml, values-prod.yaml) or pass specific values via the helm install/upgrade command.

By separating the configuration from the Kubernetes manifest structure, Helm allows for flexible and consistent deployments across different environments, which is the primary reason it's introduced in this project.

**Helm Chart Installtion and verification**

You need to delete the manually created Kubernetes deployments, services, and Ingress resources before installing them via a new Helm chart.

Delete the Deployment, Service and Ingress:
```
kubectl delete -f deployment.yaml

kubectl delete -f service.yaml

kubectl delete -f ingress.yaml
```
```
kubectl get all
```

Once the old resources are deleted, you can install your new Helm chart.

Navigate to the Helm chart's parent directory helm/go-web-app-chart:
```
cd helm
```

Install the Helm chart:
```
helm install go-web-app ./go-web-app-chart
```

(Optional) If you have specific values to override (e.g., for a "prod" environment), you can use the --set flag or a separate values.yaml file:
```
helm install go-web-app ./go-web-app-chart --set image.tag=prod
```
Or using a custom values file:
```
helm install go-web-app ./go-web-app-chart -f values-prod.yaml
```

Finally, verify if helm install deployments, services, and Ingress resources:
```
kubectl get deployment

kubectl get svc

kubectl get ing
```

As we verified helm, uninstall helm as :
```
helm uninstall
```

# 11. Continuous Integration (CI)

The GitHub Actions CI ensures that every time a developer pushes code, an updated, tested, and containerized version of the application is automatically built and stored in a registry, ready for deployment by Argo CD.

- Create a folder .github/workflows in root project. Then create cicd.yaml file. [Refer here to prepare yaml file for CI](https://github.com/marketplace?type=actions&category=container-ci)

**Setup secrets**
You will need to store your dockerhub DOCKERHUB_USERNAME, DOCKERHUB_TOKEN and github TOKEN in secrets:

To do so, please go to Github repository (go-app-cicd) --> setting --> secret & variables -->Repository secret --> new repository secret


**Test & Verify the CI**
To test and virify CI part: 

1. Push your code to remote github repository
```
git status

git add .

git commit -m "new update"

git push
```

2. Got to remote github repository "go-app-cicd" --> Actions, you will able to see the CI happening.
- Once CI completed, it will create and push docker image to your DOCKERHUB with name github commit Run_id
- You can check and verify in helm/go-web-app/values.yaml that image tag is updated to github commit Run_id


# 12. Continuous Delivery (CD)



**Install Argo CD**
Install Argo CD using manifests:
```
kubectl create namespace argocd
```
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Access the Argo CD UI (Loadbalancer service)**
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

*Access the Argo CD UI (Loadbalancer service) - For Windows*
```
kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```
- Once the service/argocd-server patched, run the below command to get the argocd-server external IP/DNS to access the UI on browser:
```
kubectl get svc -n argocd
```
Example argocd-server external IP/DNS : *abc84g52849gwjkwryr8462dfy-hrqo6iksn.us-est-1.elb.amazonaws.com*

- Argocd-server external IP/DNS will take time to provision the LB. If you don't want to wait for it, you can access it via node external-IP and argocd-server port:

Get the node external IP using:
```
kubectl get nodes -o wide
```
And on browser paste node-external-IP:rgocd-server port. For example : *54.161.25.151:32308*

- Agcocd UI will open on browser. Username will be * admin*, for password do below:

1. Copy the Name of secret argocd-initial-admin-secret from below command:
```
kubectl get secrets -n argocd
```
2. Edit the secret which has the initial password:
```
kubectl edit secrets argocd-initial-admin-secret -n argocd
```
Copy the password which is base64 encoded

3. Decode the password to get the decoded password to use in UI password.
```
echo password | base64 --decode
```
Note: do not copy the last % from password.


**Configure the ArgoCD**
Once login to argocd UI, configure it as below:

1. As argocd is on same kubernetes cluster, create New App

- Application name: go-web-app
- Project name : default
- sync policy : automatic (check - self heal option)

- Repository url : https://github.com/ANIKHILT600/go-app-cicd.git
- path : (it will auto identify helm path)

- cluster url : (select default one) https://kubernetes.default.svc
- Namespace : default

- Helm --> values file : values.yaml

Note: Before creating argocd app, check and verify that you don't have any resource running. Also check on browser if you able to see app on *go-web-app.local*


**Verify CD**
```
kubectl get deploy

kubectl get svc

kubectl get ingress
```

On browser try *go-web-app.local*


# 13. Verifying complete CICD

1. Go to github repo and navigate to static/home
2. Do any changes in code and commit the changes
3. Observe the browser app updated changes.