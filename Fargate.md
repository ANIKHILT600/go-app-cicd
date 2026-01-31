# Continue with : EKS Fargate 

This document explains **how and why EKS Fargate is used in this project**, what changes compared to EC2-based clusters, and how to deploy applications, Ingress, and Argo CD correctly on Fargate.


## Key Differences: EC2 vs Fargate

| Area         | EC2 Node Group  | Fargate              |
| ------------ | --------------- | -------------------- |
| Worker nodes | Managed by user | Fully managed by AWS |
| NodePort     | Supported       | ❌ Not supported      |
| SSH access   | Possible        | ❌ Not possible       |
| Scaling      | Node-based      | Pod-based            |
| Maintenance  | Required        | Not required         |
| Ingress      | Optional        | Mandatory            |

---

**Important Concept: Fargate Profiles**

A **Fargate profile** tells EKS **which pods should run on Fargate**.


## 5. Create eks cluster using fargate

The EKS cluster is created using the eksctl utility:
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
```
eksctl get cluster --region us-east-1
```

**Required Namespaces and Fargate Profiles**

The following namespaces are used

| Namespace       | Purpose                                  |
| --------------- | ---------------------------------------- |
| `default`       | Application workloads (already exists)   |
| `ingress-nginx` | NGINX Ingress Controller                 |
| `argocd`        | Argo CD components                       |

**Create namespaces**:
```
kubectl create namespace argocd
```
```
kubectl create namespace ingress-nginx
```
---

**Fargate profiles must be created after namespaces exist**:

**Application Namespace**

```bash
eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name fp-default --namespace default
```

**Ingress Namespace**

```bash
eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name fp-ingress --namespace ingress-nginx
```

**Argo CD Namespace**


```bash
eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name fp-argocd --namespace argocd
```

---

## 6. Application Deployment on Fargate

Deploy YAML's to default namespace:

To apply: 
```
kubectl apply -f k8s/manifests/deployment.yaml

kubectl apply -f k8s/manifests/service.yaml

kubectl apply -f k8s/manifests/ingress.yaml

```
To verify pods: 
```
kubectl get pods

kubectl get svc

kubectl get ingress
```
---

## 7. Verifying Service Functionality

NodePort is not supported in EKS Fargate because there are no worker nodes. So we can not perform loacal test using NodePort.

SKIPPING this step

## 8. Ingress Controller on Fargate

The NGINX Ingress Controller:

* Runs as pods on Fargate
* Provisions an AWS Network Load Balancer automatically
* Routes external traffic to ClusterIP services

command for installation:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.1/deploy/static/provider/aws/deploy.yaml
```

**Verification**: After running the installation command, you can check if the Nginx Ingress Controller pods are running using 
```
kubectl get pods -n ingress-nginx
```

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