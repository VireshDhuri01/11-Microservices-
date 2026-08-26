# 11-Microservices E-Commerce Application

A containerized **11-microservice e-commerce application** deployed on
**AWS EKS** with a complete **CI/CD and GitOps workflow** using **Jenkins**,
**Amazon ECR**, **Kubernetes**, and **Argo CD**.

## 🚀 Project Overview

This project demonstrates how a microservices application can be built,
containerized, continuously integrated, and automatically deployed to
Kubernetes.

The application consists of multiple independent services that
communicate with each other through Kubernetes Services.

## 🏗️ Architecture

![Project Architecture](./docs/e-commercedevops.png)

## 🔄 Application Service Flow

The main request flow is:

![Service Flow](./docs/serviceflow2.png)

## ☁️ Infrastructure

The application runs on:

-   **AWS EKS** --- Kubernetes cluster
-   **Kubernetes** --- container orchestration
-   **Amazon ECR** --- Docker image registry
-   **AWS Load Balancer** --- external application access
-   **Jenkins** --- CI/CD Pipeline for each service
-   **Argo CD** --- GitOps continuous delivery

## 🔧 CI/CD Pipeline

The project uses Jenkins for continuous integration and Argo CD for
continuous deployment.

``` text
Developer
    │
    ▼
  GitHub
    │
    │ Webhook
    ▼
 Jenkins
    │
    ├── Detect changed service
    │
    ├── Build Docker image
    │
    ├── Push image to Amazon ECR
    │
    └── Update Kubernetes deployment.yml
             │
             ▼
          GitHub
             │
             │ GitOps
             ▼
          Argo CD
             │
             ▼
        AWS EKS Cluster
             │
             ▼
       Kubernetes Pods
```

### Intelligent Change Detection

- Each service has its own `Jenkinsfile`.
- Jenkins checks the Git commit changes before building.
- Does application code change ?? If Yes then New Image is Built further pushed to ECR. If No, Skip Build.
- **This prevents every microservice pipeline from rebuilding when an unrelated service changes.**
- **It also prevents an infinite webhook loop when Jenkins updates only the Kubernetes manifest:**

**Jenkins Pipelines**
![Project Architecture](./docs/11-microservices-pipelines.png)

**Payment Service Pipeline Example**

![Project Architecture](./docs/Paymentservice-pipeline.png)

**Further Managed Argo CD**
![Project Architecture](./docs/argocdapps.png)


**Actual Pipeline**
``` bash
	pipeline {
        agent any
        tools {
            jdk 'jdk'
            }
        environment {
                AWS_REGION = 'us-east-1'
                ECR_REPO   = 'paymentservice'
                IMAGE_TAG  = "${BUILD_NUMBER}"
                    }

    stages {
        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')])
                {
                git branch: 'main', url: 'https://github.com/VireshDhuri01/11-Microservices-.git'
                sh "git pull --rebase https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/VireshDhuri01/11-Microservices- main"
                }
            }
        }
		stage('Detect Changes') {
                    steps {
                        script {
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                         ).trim()
                        echo "Changed files:"
                        echo changedFiles
                    def serviceFiles = changedFiles
                        .split('\n')
                        .findAll { it.startsWith("${env.ECR_REPO}/") }
                    def applicationChanged = serviceFiles.any { file ->
                        !file.startsWith("${env.ECR_REPO}/manifest/")
                        }
                     env.SERVICE_CHANGED = applicationChanged.toString()
                        echo "${ECR_REPO} application changed: ${env.SERVICE_CHANGED}"
                    }
                }
            }
        stage('Build Image') {
            when {
                expression {
                    env.SERVICE_CHANGED == 'true'
                }
            }
            steps {
                sh '''
                    docker build -t $ECR_REPO:$IMAGE_TAG ./$ECR_REPO
                '''
            }
        }
        stage('Create Respective Repo in ECR') {
           when {
            expression {
                env.SERVICE_CHANGED == 'true'
                    }
                }
            steps {
                withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                    string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')])
                {
                    sh """
                    aws configure set aws_access_key_id $AWS_ACCESS_KEY
                    aws configure set aws_secret_access_key $AWS_SECRET_KEY
                    """
                    sh """
                    aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} || \
                    aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}
                    """
                    }
                }
            }
        stage('Image Tag, Push & Cleanup') {
            when {
                expression {
                    env.SERVICE_CHANGED == 'true'
                }
            }
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${params.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${params.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}               
                    docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                    """
                sh '''
                    docker rmi -f $(docker images -q) || true
                    '''
                }
            }
        stage('Update Kubernetes Manifest') {
            when {
                expression {
                    env.SERVICE_CHANGED == 'true'
                }
            }
            steps {
                script {
                def IMAGE = "${params.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
                sh """ sed -i 's|image: .*|image: ${IMAGE}|' ${ECR_REPO}/manifest/deployment.yml"""
                    }
                }
            }
        stage('Push Manifest to GitHub') {
            when {
                    expression {
                    env.SERVICE_CHANGED == 'true'
                    }
                }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) 
                    {sh """
                git config user.name "Viresh Dhuri"
                git config user.email "vireshdhuri28@gmail.com"
                git add ${ECR_REPO}/manifest/deployment.yml
                git commit -m "Update image to build ${IMAGE_TAG}" || true
                git push https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/VireshDhuri01/11-Microservices-.git HEAD:main
                """
                }
            }
        }
    }
}    
```

**Each microservice contains its application code, Dockerfile, Jenkins
pipeline, and Kubernetes manifests.**

## 📌 Project Outcome

**The complete application is deployed on AWS EKS and accessible through
an AWS Load Balancer.**

Some Images.
![Project Architecture](./docs/app.png)
![Project Architecture](./docs/app2.png)
![Project Architecture](./docs/app3.png)

The workflow from **code change → Docker image → ECR → GitHub manifest
update → Argo CD → Kubernetes deployment** is automated.
