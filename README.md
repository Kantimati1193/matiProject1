#  Brain Tasks App – End-to-End DevOps Deployment

##  Project Overview

This project demonstrates a complete **CI/CD pipeline** for deploying a React application using:

* Docker (Containerization)
* AWS ECR (Image Registry)
* AWS EKS (Kubernetes Deployment)
* AWS CodeBuild (Build automation)
* AWS CodePipeline (CI/CD pipeline)
* Amazon CloudWatch (Monitoring)

##  Architecture



## Repository Structure


task1/Brain-Tasks-App/
├── src/
├── public/
├── Dockerfile
├── deployment.yaml
├── service.yaml
├── buildspec.yml
├── buildspec-deploy.yml
└── README.md

# Setup Instructions

## Phase 1: Run Application Locally

git clone https://github.com/<your-username>/<repo-name>.git
cd Brain-Tasks-App
npm install
npm start

App runs at:
http://localhost:3000


## Phase 2: Dockerize the Application

Dockerfile
# Step 1: Build React app
FROM node:18 AS build
WORKDIR /app

COPY package*.json ./
RUN npm install --legacy-peer-deps

COPY . .

RUN chmod -R 755 /app/node_modules/.bin

RUN npm run build

# Step 2: Serve using nginx
FROM nginx:alpine

COPY --from=build /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

### Build & Run Docker

docker build -t brain-app .
docker run -p 3000:80 brain-app

## Phase 3: Push Image to AWS ECR

aws ecr create-repository --repository-name brain-app --region ap-south-1

aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin 331738868643.dkr.ecr.ap-south-1.amazonaws.com

docker tag brain-app:latest 331738868643.dkr.ecr.ap-south-1.amazonaws.com/brain-app:latest
docker push 331738868643.dkr.ecr.ap-south-1.amazonaws.com/brain-app:latest

## Phase 4: Create EKS Cluster

eksctl create cluster \
--name brain-cluster \
--region ap-south-1 \
--node-type t3.small \
--nodes 2 \
--managed

Verify:

kubectl get nodes

---

## Phase 5: Kubernetes Deployment

### deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: brain-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: brain-app
  template:
    metadata:
      labels:
        app: brain-app
    spec:
      containers:
      - name: brain-app
        image: 331738868643.dkr.ecr.ap-south-1.amazonaws.com/brain-app:latest
        ports:
        - containerPort: 80
### service.yaml

apiVersion: v1
kind: Service
metadata:
  name: brain-app-service
spec:
  type: LoadBalancer
  selector:
    app: brain-app
  ports:
    - port: 80
      targetPort: 80

### Deploy to EKS

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml


## Phase 6: CodeBuild Setup

### buildspec.yml
version: 0.2

env:
  variables:
    AWS_REGION: ap-south-1
    CLUSTER_NAME: brain-cluster

phases:
  install:
    commands:
      - echo Installing kubectl...
      - curl -LO "https://dl.k8s.io/release/v1.27.0/bin/linux/amd64/kubectl"
      - chmod +x kubectl
      - mv kubectl /usr/local/bin/
      - kubectl version --client

  pre_build:
    commands:
      - echo Updating kubeconfig...
      - aws eks --region ap-south-1 update-kubeconfig --name $CLUSTER_NAME

  build:
    commands:
      - echo Deploying to EKS...
      - kubectl apply -f deployment.yaml
      - kubectl apply -f service.yaml

  post_build:
    commands:
      - echo Deployment completed
      
## Phase 7: CodePipeline Setup

### Pipeline Stages:

1. **Source**

   * GitHub repository

2. **Build**

   * CodeBuild project ('brain-app-build')

3. **Deploy**

   * CodeBuild project ('brain-app-deploy')

### buildspec-deploy.yml

version: 0.2

env:
  variables:
    AWS_REGION: ap-south-1
    CLUSTER_NAME: brain-cluster

phases:
  install:
    commands:
      - curl -LO "https://dl.k8s.io/release/v1.27.0/bin/linux/amd64/kubectl"
      - chmod +x kubectl
      - mv kubectl /usr/local/bin/

  pre_build:
    commands:
      - aws eks --region ap-south-1 update-kubeconfig --name $CLUSTER_NAME

  build:
    commands:
      - kubectl apply -f deployment.yaml
      - kubectl apply -f service.yaml

## IAM Configuration

### Add CodeBuild Role to EKS

kubectl edit configmap aws-auth -n kube-system

Add:

- rolearn: arn:aws:iam::331738868643:role/codebuild-brain-app-deploy-service-role
  username: codebuild
  groups:
    - system:masters

## Monitoring

### Services Used:

* AWS CodePipeline → Pipeline execution
* AWS CodeBuild → Build logs
* Amazon CloudWatch → Log monitoring


## Application Access

kubectl get svc

Open brower and enter the external-ip

http://<EXTERNAL-IP>

## Screenshots (Add these)

* ECR repository
<img width="950" height="415" alt="Screenshot 2026-05-02 165636" src="https://github.com/user-attachments/assets/5b9d9d8f-0be0-46a2-aa82-559be8071d60" />

* EKS cluster nodes
<img width="681" height="397" alt="Screenshot 2026-05-09 144247" src="https://github.com/user-attachments/assets/22b38153-3f66-484d-ac03-0d8b366cdf44" />

* CodePipeline success
  <img width="953" height="494" alt="Screenshot 2026-05-09 223343" src="https://github.com/user-attachments/assets/58d559b4-650e-4396-b941-63648ba46a87" />

* CodeBuild logs
  <img width="950" height="493" alt="Screenshot 2026-05-09 164352" src="https://github.com/user-attachments/assets/963e733c-1583-442a-bf7b-8d660101f1bb" />

* Application running in browser
<img width="956" height="473" alt="Screenshot 2026-05-09 174616" src="https://github.com/user-attachments/assets/090c8af3-21e0-4b22-98fe-e5b7f6601a34" />


## Conclusion

This project demonstrates a **production-ready deployment pipeline** using AWS services, enabling automated build, containerization, and deployment to Kubernetes.

---
