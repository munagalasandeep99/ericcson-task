
# Ericsson Task

## Kubernetes architecture

I have set up a Kubernetes cluster using Amazon EKS with two nodes.
![image](https://github.com/user-attachments/assets/367c0183-f773-402f-b22a-45e4b052913a)


## Architecture
I have created an Amazon EKS (Elastic Kubernetes Service) cluster in the us-east-1 region, distributed across two Availability Zones to ensure high availability and data reliability.

To support this architecture, I configured:

- Two public subnets (one in each Availability Zone)

- Two private subnets (one in each Availability Zone)

Networking and Placement:
The EKS control plane is deployed in the public subnets. This allows easy access for cluster management and interaction using kubectl, the AWS CLI, and other tools.

The worker nodes (node groups) are deployed in the private subnets to enhance security by preventing direct exposure to the internet. These nodes interact with the outside world only through NAT gateways or load balancers as needed.

Add-ons Installed:
To extend the functionality of the EKS cluster and ensure smooth operations, I have installed the following EKS-managed add-ons:

Amazon EBS CSI Driver – for dynamic provisioning and management of EBS volumes used by Kubernetes pods.

CoreDNS – to provide internal DNS resolution for service discovery within the Kubernetes cluster.

AWS Load Balancer Controller – to provision and manage AWS Application Load Balancers (ALB) and Network Load Balancers (NLB) for Kubernetes services.

Amazon VPC CNI Plugin – for efficient pod networking, allowing each pod to get an IP address from the VPC CIDR range.

# kubernetes control plane and worker node.

![image](https://github.com/user-attachments/assets/5eb6a0b4-e06e-4304-a3c3-f8181abec54e) 

<B>Kubernetes Control Plane</B>

The control plane is the brain of a Kubernetes cluster. It manages the overall cluster, makes global decisions (like scheduling), and detects/responds to cluster events. Key components include:

<B>API Server</B>: Exposes the Kubernetes API; it's the main entry point for users and tools.

<B>Scheduler</B>: Decides which node a pod should run on based on resource availability and other constraints.

<B>Controller Manager</B>: Runs background controllers that handle tasks like node management and replication.

<B>etcd</B>: A key-value store used to persist cluster state and configuration.

<B>Kubernetes Worker Nodes</B>
Worker nodes are the machines (VMs or physical servers) that run the actual application workloads (containers). Each node runs:

<B>kubelet</B>: Ensures containers are running as defined in the pod specs.

<B>kube-proxy</B>: Handles network routing for services and pod communication.

<B>Container runtime</B> (e.g., containerd, Docker): Runs the containers.

<B>What Happens When You Run a Kubernetes Command </B>

example command:
```shell
kubectl run myapp --image=nginx
```

Step-by-step process:

- kubectl sends the request to the Kubernetes API Server.

- The Scheduler selects an appropriate Worker Node to run the Pod.

- The Controller Manager creates and monitors the Pod.

- The selected Worker Node receives the Pod specification.

- The Kubelet on that Node pulls the nginx image and starts the container inside a Pod.




### Task 1: Build a Kubernetes Cluster.
- Create a Ec2 instance with instance policy which has administrative access.
- now ssh into it and run this
```shell
#!/bin/bash
# For Ubuntu 22.04
# Intsalling Java
sudo apt update -y
sudo apt install openjdk-17-jre -y
sudo apt install openjdk-17-jdk -y
java --version

# Installing Jenkins
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y

# Installing Docker 
#!/bin/bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
sudo chmod 777 /var/run/docker.sock


# Installing AWS CLI
#!/bin/bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# Installing Kubectl
#!/bin/bash
sudo apt update
sudo apt install curl -y
sudo curl -LO "https://dl.k8s.io/release/v1.28.4/bin/linux/amd64/kubectl"
sudo chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client


# Installing eksctl
#! /bin/bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Installing Terraform
#!/bin/bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt install terraform -y

# Intalling Helm
#! /bin/bash
sudo snap install helm --classic

```
- you can create either using eksctl or the terraform code in the git repo
# using eksctl

- configure to aws
![image](https://github.com/user-attachments/assets/87df9c28-bfc6-4088-9e80-769d8b305d5e)
- run this command
```shell
eksctl create cluster --name Three-Tier-K8s-EKS-Cluster --region us-east-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-east-1 --name Three-Tier-K8s-EKS-Cluster
```
![image](https://github.com/user-attachments/assets/65d574ef-c3d4-4a55-a57d-047dc4c24600)
- Now, we will configure the Load Balancer on our EKS because our application will have an ingress controller.
```shell
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
- Create the IAM policy using the below command
```shell
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
- Create OIDC Provider
```shell
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=Three-Tier-K8s-EKS-Cluster --approve
```
- Create a Service Account by using below command and replace your account ID with your one
```shell
eksctl create iamserviceaccount --cluster=Three-Tier-K8s-EKS-Cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-east-1
```
![image](https://github.com/user-attachments/assets/5dcd8202-259c-45ae-9557-6937a816825f)

- Run the below command to deploy the AWS Load Balancer Controller
```shell
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=Three-Tier-K8s-EKS-Cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

```
- now  the eks cluster is ready of deployment
# using terraform
```shell
git clone https://github.com/munagalasandeep99/ericcson-task.git
cd ericcson-task/eks-terraform/main
terraform init
terraform plan 
terraform apply --auto-approve
```
- now after running this you still need to run this
```shell
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=Three-Tier-K8s-EKS-Cluster --approve

eksctl create iamserviceaccount --cluster=Three-Tier-K8s-EKS-Cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-east-1

sudo snap install helm --classic

helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller
```

- now eks setup is completed
### Task 2: Deploy a Sample Application
- The sample application is a todo application developed using React for front end, Node.js for Backend ad mongodb as database
- Now from the work station,ec2 instance clone the repository using the command
```Shell
kubectl create namespace three-tier
https://github.com/munagalasandeep99/ericcson-task.git
cd ericcson/manifest-files
kubectl create -f . 
```
```Shell
cd Frontend
kubectl create -f . 
```
```Shell
cd ../Database
kubectl create -f . 
```
```Shell
cd ../Backend
kubectl create -f . 
```

## deliverables
- Application code and kubernetes manifests:
  the applcation code is present in Application_code file and manifest-files repectively in above repository

- how to access the front end
I have used domain to access the frontend which is mentioned in the ingress.yaml file, i hosted a domain in route 53, you can access just by using the domain name, if you donot want to use domain name then open the /manifest-files/ingress.yml file and remove the host part then you can access it via load balancer generated by ingress 

```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mainlb
  namespace: three-tier
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    # alb.ingress.kubernetes.io/subnets: "subnet-0aa439b4ddafcca10,subnet-0b95df318219145ca,subnet-0ccca36474902829a"
spec:
  ingressClassName: alb
  rules:
    - http: 
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000
```
```shell
kubectl get ingress mainlb -n three-tier
```
- from this you get loadbalancer
```shell
NAME      CLASS   HOSTS           ADDRESS                                                               PORTS   AGE
mainlb    alb                  k8s-mainlb-threetie-abcdefgh123456-1234567890.us-west-2.elb.amazonaws.com   80      5m

```
- use the address and paste it in browser you can access frontend and perform crud operations
## video href="https://drive.google.com/file/d/130-2luUJOdhsTmaNA7UzeeI31t-fUfyh/view?usp=sharing"




### Task 3:Monitoring with Prometheus &amp; Grafana.
- Step 1: Add Helm Repositories

```bash
# Add Prometheus Community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts

# Update Helm repos
helm repo update
```
- Step 2: Install Prometheus and Grafana
```bash
# Install Prometheus
helm install prometheus prometheus-community/prometheus

# Install Grafana
helm install grafana grafana/grafana
```
- Step 3: Verify Services
```bash
kubectl get svc
```
look for prometheus-server and graphana
- Step 4: Expose Prometheus and Grafana (Change Service Type)
```bash
kubectl edit svc prometheus-server
```
modify the yaml
```yaml
spec:
  type: LoadBalancer
```
edit Graphana service
```bash
kubectl edit svc grafana
```
modify
```yaml
spec:
  type: LoadBalancer
```
- step 5:Get External Access URLs
```bash
kubectl get svc
```
note down the loadbalncer dns for Prometheus and graphana

now you can access prometheus and graphana ui using loadbancer dns you get.

## deliverables
- The congig files are present in prometheus and grafana folder of the repository
  ![prometheus](https://github.com/user-attachments/assets/6477bc1f-b69e-4cd1-8d14-674ebbf1ff7b)
  ![grafana](https://github.com/user-attachments/assets/8e563102-1132-4466-b168-2d944dd7165a)
  ![grafana2](https://github.com/user-attachments/assets/6d38f664-ef9b-4cc4-a65c-51ae793c2adf)

description of metrics being monitored

 Pod Status Metrics
- **kube_pod_status_phase**: Tracks pod lifecycle phase (e.g., Running, Failed). Shows pod health and state distribution.
- **kube_pod_container_status_restarts_total**: Counts container restarts in pods. Indicates pod stability issues.
- **kubelet_running_pod_count**: Counts running pods on nodes with role `.*node.*`. Monitors node workload.

 Pod CPU Usage Metrics
- **container_cpu_usage_seconds_total**: Measures CPU time used by containers (in seconds or millicores). Tracks actual CPU usage.
- **kube_pod_container_resource_requests{resource="cpu"}**: Minimum CPU requested by containers for scheduling.
- **kube_pod_container_resource_limits{resource="cpu"}**: Maximum CPU allowed for containers, enforced by Kubernetes.

 Pod Memory Usage Metrics
- **container_memory_working_set_bytes**: Measures active memory usage by containers. Tracks actual memory consumption.
- **kube_pod_container_resource_requests{resource="memory"}**: Minimum memory requested by containers for scheduling.
- **kube_pod_container_resource_limits{resource="memory"}**: Maximum memory allowed for containers, enforced by Kubernetes.




# Task 4
- file: monitor.sh and prometheus-query.py are present in scripts folder above.
- Monitor.sh - checks the memory and cpu usage of the pod and alerts us by printing messages ,before running the script change the pod name and name space , if wanted change the thresholds.
  ```shell
  chmod +x monitor.sh
  ./monitor.sh
  ```
  - file prometheus-query.py - displays all the pods their cpu usage and memory usage and stores in a json file .
  - file specific-pod.py - displays the current cpu and memory usage and stores in a json file, before running you need to mention the pod name and namespace name
 ```shell
  pip install requests
  pip prometheus-query.py
  pip specific-pod.py
  ```
<b>deliverables</b>
- it are placed in scriptin results folder
- monitor.sh results
```bash
Pod: mongodb-59797b688c-kz5xz
CPU Usage: 0.004 cores
Memory Usage: 0.147461 GB
⚠️ ALERT: CPU usage (0.004 cores) exceeds threshold (0.0005 cores)
⚠️ ALERT: Memory usage (0.147461 GB) exceeds threshold (0.005 GB)
```
- prometheus-query.py results
```bash
    "namespace": "argocd",
    "pod": "argocd-dex-server-868bf8bc97-5xkzz",
    "cpu_usage_cores": 0.00011863982447377208,
    "memory_usage_bytes": 17244160.0
  },
  {
    "namespace": "kube-system",
    "pod": "aws-load-balancer-controller-7dcc74f48b-q268p",
    "cpu_usage_cores": 0.0012479037558203326,
    "memory_usage_bytes": 36925440.0
  },
  {
    "namespace": "kube-system",
    "pod": "coredns-54d6f577c6-5bmth",
    "cpu_usage_cores": 0.0010439434835204096,
    "memory_usage_bytes": 26140672.0
  },
  {
    "namespace": "kube-system",
    "pod": "kube-proxy-44wpn",
    "cpu_usage_cores": 0.00024203313184795584,
    "memory_usage_bytes": 26836992.0
  }
  
```
-specific-pod.py results
```bash
{
  "pod": "grafana-7c96b5b9cc-rcpkb",
  "namespace": "default",
  "cpu_usage_cores": 0.0026442196210575974,
  "memory_usage_bytes": 127459328.0
}
```

# Task 5

For CICD  i used jenkins and Argocd
# configure jenkins
- copy the public IP of your Server and paste it on your favorite browser with an 8080 port
![image](https://github.com/user-attachments/assets/56cdf836-08b8-4b7c-ae8f-f06305811568)
- click on install suggested plugins
![image](https://github.com/user-attachments/assets/e9851a84-cca9-4098-92cc-a42fc5853e75)
- after installing the plugins, continue as admin
![image](https://github.com/user-attachments/assets/0e924ca2-c63d-43cf-a768-c7382a8809fd)
- start jenkins
![image](https://github.com/user-attachments/assets/e3f0d36b-3286-4246-a7ef-da0a176cd9ab)
- now go back to you server and configure aws
  ![image](https://github.com/user-attachments/assets/2846e699-a594-44fc-b4f3-06821a153f1a)
- Go to Manage Jenkins,Click on Plugins
![image](https://github.com/user-attachments/assets/d16bf6a1-71e3-4ad9-8074-d9627ed73742)
- now go to avalabil plugins and install this plugins
```
Docker
Docker Commons
Docker Pipeline
Docker API
docker-build-step
nodejs
aws-pipeline
```
- now add creditionals  in Dashboard -> Manage Jenkins -> System -> global credentials, like github creentials,aws credentials, which are needed
- now build pipelines named frontend and backend and paste the jenkin files present in Jenkins-Pipeline-Code
 # Install & Configure ArgoCD
- We will be deploying our application on a three-tier namespace. To do that, we will create a three-tier namespace on EKS
```shell
kubectl create namespace argocd
```
- Now install argocd using this command
```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```
- now expose argocd via load balancer
```shell
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
- get the load balencer dns
```shell
kubectl get svc argocd-server -n argocd
```
![image](https://github.com/user-attachments/assets/27e91fcb-ebb8-4196-87b8-0bc1c1fce1ed)

- To access the argoCD, copy the LoadBalancer DNS and hit on your favorite browser.

![image](https://github.com/user-attachments/assets/1c08b6bb-7ce5-4860-b147-27b7474bb4c5)

- collect the argocd password by performing this command
```shell
sudo apt install jq -y
export ARGOCD_SERVER='kubectl get svc argocd-server -n argocd -o json | jq - raw-output '.status.loadBalancer.ingress[0].hostname''
export ARGO_PWD='kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d'
echo $ARGO_PWD
```
- Enter the username as admin and password in argoCD and click on SIGN IN.
![image](https://github.com/user-attachments/assets/71fdda62-b615-4154-967d-e41db8d8b21a)


- now add repositoy details to argocd repositories using CONNECT REPO USING HTTPS
![image](https://github.com/user-attachments/assets/727c39ee-0a92-44fe-9cf6-0bc803b70bc2)

- now , create applications for frontend, backend, database and ingress, follow the snippets
![image](https://github.com/user-attachments/assets/f5050178-a432-4200-890b-d92124a1456b)
![image](https://github.com/user-attachments/assets/0afbaa49-20bd-4195-ae65-1a1c8aafa9d6)
![image](https://github.com/user-attachments/assets/78317c04-defb-4f1d-a67b-815525ffe895)

- now the CI/CD part is completed.

<b>deliverables</b>

- the cicd pipe line configuration is in Jenkins-Pipeline-Code folder, it has two jenkins files, one for frontend and one for backend.
- Docker files for front end is present in Application-Code/frontend and backend in Application-Code/Backend folders
- the kubernetes manifest files are present in the folder Kubernetes-Manifests-file
- screen shots of successful deployment
![image](https://github.com/user-attachments/assets/2d363586-0abb-4673-99b6-e248ae757327)
![image](https://github.com/user-attachments/assets/ba55df78-7d28-41fd-b9b4-dc160dfa15ff)
![image](https://github.com/user-attachments/assets/78317c04-defb-4f1d-a67b-815525ffe895)

# Descriptions:
<h> How pipeline is triggered </h>
- The pipeline runs automatically whenever a commit is pushed to the GitHub repository (https://github.com/munagalasandeep99/ericcson-task.git). This is set up using a GitHub webhook, which sends a notification to Jenkins whenever code is pushed to the master branch (or the configured branch). This ensures that any new code changes trigger the pipeline immediately. Additionally, you can manually start the pipeline through the Jenkins web interface, which is useful for rerunning builds or testing specific changes.

<h> how it talks to kubernetes cluster </h>
CI Process
The Continuous Integration (CI) phase is handled by the jenkins, the pipeline is triggered when a commit is made, in the pipeline it

Builds a Docker image from the application code.
Pushes the image to AWS Elastic Container Registry (ECR) using secure AWS credentials.
Updates the Kubernetes manifest file in the Git repository with the new Docker image tag.

CD Process (ArgoCD)

The Continuous Deployment (CD) phase is handled by ArgoCD. ArgoCD continuously monitors the GitHub repository for changes to the manifest files. When Jenkins updates the deployment.yaml with a new image tag, ArgoCD detects this change and applies the updated configuration to the Kubernetes cluster

<h> how monitoring is verified </h>

images of prometheus and grafana working
![image](https://github.com/user-attachments/assets/b0bdfa90-c77e-40f9-9830-951470861ae4)
![grafana2](https://github.com/user-attachments/assets/4a080cea-98e2-445e-b6c2-5688f11d4520)






