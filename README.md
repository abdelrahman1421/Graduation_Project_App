
# DevOps Projetc Using GCP

The project is to automate and a fulle cycle of App starting from writing Dockerfile , then build and push to Artifact Repo,
after that Deploy the App to the GKE , Jenkins is used to make full CI/CD automation cycle.
# Projetc Overview
![jenkins](https://user-images.githubusercontent.com/73159522/202114201-f73f964e-a9f7-4814-a074-7694c7ae6c3b.png)

# Project Architecture
![Team document](https://user-images.githubusercontent.com/73159522/202046368-35207f00-2e9a-43ad-a659-ec8315bdd9ac.jpeg)


## 🛠 Tools :
| Tool | Purpose |
| ------ | ------ |
| [ Terraform ](https://www.terraform.io) | Terraform is an open-source infrastructure as a code software. |
| [ Google Kubernetes Engine (GKE) ](https://cloud.google.com/kubernetes-engine) | Google Kubernetes Engine (GKE) is a managed, production-ready environment for running containerized applications. |
| [ Helm ](https://helm.sh) | Helm Charts help you define, install, and upgrade even the most complex Kubernetes applications. |
| [ Docker ](https://www.docker.com) | Docker is a software tool used to deliver software in containers.|
| [ Jenkins ](https://www.jenkins.io) | Jenkins – an open-source automation server is enabling developers to build, test, and deploy their software. |


## Configure Infrastructure Using Terraform :
Four modules were designed to provide a stable Infrastructure to the project
- ###  Network Module Contains The Following Resources :
  - VPC that project will be assigned to
  - Two subnets one for GKE and another for Bastion Host
  - NAT Gateway and Router
  - Firewall to allow SSH Connection

- ###  Service Account Module Contains The Following Resources :
  - Service Account For GKE
  - Service Account For VM

- ###  GKE Module Contains The Following Resources :
  - Private GKE
  - Container Node Pool


- ###  VM  Module Contains The Following Resources :
  - VM Pastion 

## First Step: Create Infrastructure :
### 1. Clone The Repo:
```
git clone https://github.com/abdelrahman1421/Graduation_Project_Infra.git

```

### 2. Change Directory TO Terraform :
```
cd Graduation_Project_Infra

```
### 3. Initialize provider Plugins :
```
terraform init

```

### 4. Check Terraform Plan :
```
terraform plan -var-file prod.tfvars

```
### 5. Apply Infrastructure :
```
terraform apply -var-file prod.tfvars

```
![Screenshot from 2022-11-16 08-35-37](https://user-images.githubusercontent.com/73159522/202112039-ce353106-25e0-4391-9507-c6f6d84599bc.png)

## Second Step: Connect To GKE Usnig Pastion VM :
> Now after the Infrastructure is built navigate to `Compute Engine` from the GCP console then `VM instances` and click the SSH to `private-vm` to configure to to work with GKE cluster by running the following commands:
### 1. Install Kubectl
```
sudo apt-get install kubectl
```
### 2. Install GKE gcloud auth Plugin
```
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```
### 3. Log in with your Credentials
> Copy the appeared link and paste it in your browser login with your account then copy auth key and paste it in bastion terminal:
```
gcloud auth login
```
![Screenshot from 2022-11-16 08-38-59](https://user-images.githubusercontent.com/73159522/202112156-a6163402-0c59-47c7-9a65-8617dceae3b1.png)

### 4. Connect to GKE Cluster
> Go to the `Kubernetes Engine` Page in your `Clusters` tab you will find the `private-cluster`
![Screenshot from 2022-11-16 08-40-03](https://user-images.githubusercontent.com/73159522/202113231-b9040a4b-8c33-4c60-9fa0-87474e00c922.png)
![Screenshot from 2022-11-16 08-41-05](https://user-images.githubusercontent.com/73159522/202113343-2d432903-d1ce-406e-a1db-ee53fa36a7c7.png)

### 5. Create Jenkins Namespace For Jenkins Deployment
```
kubectl create namespace jenkins

```
![Screenshot from 2022-11-16 08-41-20](https://user-images.githubusercontent.com/73159522/202112448-1f583035-ee6a-4111-94b0-2cf083816c2a.png)

### 6. Create Production Namespace For App Deployment
```
kubectl create namespace production

```
![Screenshot from 2022-11-16 08-41-38](https://user-images.githubusercontent.com/73159522/202112507-b4e2d649-4770-4610-aade-f3e5655d30df.png)

## Third Step: Install Jenkins Using Helm On GKE 
### 1. Create Helm Intall Script
 

```
touch helm.sh

```
### 2. Echo Lines to helm.sh

```
echo "
#!/bin/bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm " > helm.sh

```
### 3. Change Permessions And Run Script
```
sudo chmod +x helm.sh
sh helm.sh
```
### 4. Add Jenkins Repo
```
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

### 5. Pull Jenkins Using Helm
```
helm pull --untar jenkins/jenkins
```
### 6. Edit Jenkins Chart values.yaml File
> As the default service of this chart is `ClusterIp` so we need to Change it to `LoadBalancer`
Replace the `ServiceType` value from `ClusterIP` to `LoadBlancer` in Line 129:
> Save the file and go back to the home directory
```
cd jenkins
vim values.yaml
cd ..
```
### 7. Now Install Jenkins Chart
```
helm install jenkins-master ./jenkins -n jenkins
```
![Screenshot from 2022-11-16 08-56-19](https://user-images.githubusercontent.com/73159522/202112741-8eb45dff-039b-4728-9926-0b3eafa25e15.png)

### 8. Get `admin` user Password
> Jenkins login user is `admin`
```
  kubectl exec --namespace jenkins -it svc/jenkins-master -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```
### 9. Get the `Jenkins URL`
```
export SERVICE_IP=$(kubectl get svc --namespace jenkins jenkins-master --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
echo http://$SERVICE_IP:8080/login
```
## Final Step: Build Your CI/CD Pipeline
### 1. Add Credentials in Jenkins
- #### Dockerhub Credentials
  > Add your Dockerhub Credentials `(Username and Password)` wit ID  `dcokerhub`.

- #### Service Account Credentials
  > Go to GCP Console and navigate to  `Service accounts` from the `IAM & Admin` page
   Click on your `Service accounts` then click on the `KEYS` Tab then `Add Key` then `Create new key`, for `Key type` Select `JSON`
   > Now go to Jenkins and Make a New credential, select `Secret` for `credentials kind` then upload the Service Account you just downloaded
    NOTE: for `Secret ID` enter `Service-Account-Cred`
 
  ![Screenshot from 2022-11-16 08-59-30](https://user-images.githubusercontent.com/73159522/202113629-a6a16f44-6e76-4d0e-b37d-c07a496e0d33.png)
 
  ![Screenshot from 2022-11-16 09-00-38](https://user-images.githubusercontent.com/73159522/202113734-7d3e9480-67b1-4333-bda2-0f2421df04f8.png)



### 2. Create Jenkins/CI Pipeline:
- Pull Code from GitHub
- Build the Application image
- Push Image to DockerHub
- Trigger CD Pipeline to run
> When creating pipeline select from `Build Triggers` option `GitHub hook trigger for GITScm polling`
![Screenshot from 2022-11-16 07-49-27](https://user-images.githubusercontent.com/73159522/202095684-081f3870-0a5c-42ae-8d20-da5623200788.png)


### 3. Enable Webhook Trigger For Jenkins_CI Pipeline:
> Go to repo that holds the project then navigate to `settings` tab
   - Chose `webhooks` from the left bar
   - Select from top right `Add webhook`
   - Add your jenkins public IP then add this word as it's `/github-webhook/`
   - The final URL must be like that `http://JenkinsIP:port/github-webhook/`
   - Chose the action that you want to activate your pipeline according to it push , pull ...etc
     ![Screenshot from 2022-11-16 08-10-43](https://user-images.githubusercontent.com/73159522/202098268-38ff1d8a-3bab-4a3b-9e65-849add7568d4.png)


### 3. Create CD Pipeline:
- Enbale Build Trigger for `Jenkins_CD Pipeline` according to `Jenkins_CI pipeline build status`
  > While creating your `Jenkins_CD Pipeline` select from `Build Triggers` option `Build after other projects are built` then
    enter your CI pipeline name fro me `Jenkins_CI` then chose `Trigger only if build is stable`
- Deploy our Application in GKE
  ![Screenshot from 2022-11-16 08-18-31](https://user-images.githubusercontent.com/73159522/202099535-308a701f-9994-40ac-b56b-d6d8e7c0f810.png)
  
  
  ![Screenshot from 2022-11-16 09-15-03](https://user-images.githubusercontent.com/73159522/202114927-69e514bb-2abb-4455-9115-286fe8bcf468.png)
  
  ![Screenshot from 2022-11-16 09-14-18](https://user-images.githubusercontent.com/73159522/202115025-67d1499e-2736-4955-bfc5-0a38c4242b4a.png)


## Final Result
![Screenshot from 2022-11-16 09-12-25](https://user-images.githubusercontent.com/73159522/202110922-16701eb3-6311-4cf7-8450-25c2e903e3fa.png)

## Final Part: Clean up 💣
```
cd terraform
terraform destroy
```
