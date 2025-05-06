# Full-Stack-DevOps-Project

# 🚀 Task Master Pro

Task Master Pro is a task management application originally developed by [Aditya Jaiswal](https://github.com/jaiswaladi246/Task-Master-Pro). I have used this application as a reference project to design and implement a **production-grade CI/CD pipeline** from scratch, integrating a full DevOps toolchain with Jenkins, SonarQube, Trivy, Docker, Kubernetes, and Terraform.

---

## 📌 Features

* Create, update, delete tasks
* Mark tasks as complete/incomplete
* User authentication and registration
* RESTful APIs
* Web-based interface (Spring Boot)

---

## 🧠 Tech Stack & Tools

| Layer            | Tool/Tech               |
| ---------------- | ----------------------- |
| Backend          | Java 17, Spring Boot    |
| Build Tool       | Apache Maven            |
| Version Control  | Git & GitHub            |
| CI/CD            | Jenkins, GitHub Actions |
| Containerization | Docker                  |
| Security Scan    | Trivy                   |
| Code Quality     | SonarQube               |
| Image Registry   | Docker Hub, Nexus       |
| Orchestration    | Kubernetes (EKS)        |
| IaC              | Terraform               |

---

## 🔧 ⚖️ Setup & Installation

### ✅ Prerequisites

* Java Development Kit (JDK 17+)
* Apache Maven (3.6+)
* Docker
* Kubernetes Cluster (Self-managed or cloud)
* GitHub account with access to repository
* Docker Hub account

### 💻 Run Locally

```bash
git clone https://github.com/jaiswaladi246/Task-Master-Pro.git
cd Task-Master-Pro
mvn clean install
mvn spring-boot:run
```

Visit: `http://localhost:8080`

---

🌐 AWS Overview

This project uses Amazon Web Services (AWS) to provision and manage infrastructure. Specifically, we leverage Amazon Elastic Kubernetes Service (EKS) to orchestrate containerized applications, integrated with supporting services such as:

Amazon VPC for networking

IAM for access control

ECR/Docker Hub for container registry

S3 (optional) for state storage (via remote Terraform backend)

Ensure your AWS credentials and CLI are properly configured before executing any Terraform commands.

![Screenshot (19)](https://github.com/user-attachments/assets/f9a93a9d-335a-465d-916f-053f0f019133)

**Security Groups:**
The following inbound rules are configured in the security group associated with the Task Master Pro application to allow necessary network traffic:

![Screenshot (21)](https://github.com/user-attachments/assets/c97721fc-f183-4f11-b61e-f24dc1cd604e)


## 💐 Kubernetes Setup with Terraform

This project provisions an EKS cluster using Terraform for hosting the Task Master Pro application.

### Key Terraform Components:

* `provider.tf`: AWS provider configuration
* `vpc.tf`: Custom VPC for EKS
* `eks.tf`: EKS cluster definition
* `nodegroup.tf`: Worker node group
* `outputs.tf`: Outputs cluster kubeconfig and endpoint

### Terraform Setup Steps:

```bash
cd infra/terraform
terraform init
terraform plan
terraform apply
```

After successful provisioning, update kubeconfig:

```bash
aws eks --region <region> update-kubeconfig --name devopsshack-cluster
```

The output after successful cluster creation will look like this:

![terraform apply](https://github.com/user-attachments/assets/16dc4aa5-2ef7-49d7-b6c6-cc9fa3fa22c8)

![Screenshot (20)](https://github.com/user-attachments/assets/9dea2096-8ddb-47ec-b446-7018c8c27208)

---

## 🔐 RBAC Setup for Jenkins

### Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

### Role (Namespace Scope)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups: [""]
    resources:
      - pods
      - configmaps
      - secrets
      - services
      - persistentvolumeclaims
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: webapps
```

### ClusterRole (Cluster Scope)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-cluster-role
rules:
  - apiGroups: [""]
    resources:
      - persistentvolumes
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["storage.k8s.io"]
    resources:
      - storageclasses
    verbs: ["get", "list", "watch"]
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-cluster-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-cluster-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: webapps
```

---

## 🔄 CI/CD Pipelines

### 🗖️ GitHub Actions (CD to Production)

1. Pull dev image & tag as production
2. Push to Docker Hub
3. Deploy to Kubernetes (EKS)

```yaml
name: CDPROD

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: self-hosted
    steps:
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Push Docker Image
      run: |
          docker pull adijaiswal/devtaskmaster:latest
          docker tag adijaiswal/devtaskmaster:latest adijaiswal/prodtaskmaster:latest
          docker push adijaiswal/prodtaskmaster:latest

    - name: Kubectl Action
      uses: tale/kubectl-action@v1
      with:
        base64-kube-config: ${{ secrets.KUBE_CONFIG_PROD }}

    - run: |
          kubectl apply -f deployment-service.yml
```

---

### 🔧 Jenkins CI Pipeline

1. Git Checkout
2. Compile (Maven)
3. Unit Test
4. Trivy FS Scan
5. SonarQube Analysis
6. Docker Build
7. Trivy Image Scan
8. Docker Push
9. Deploy to Kubernetes (EKS)
10. Verify Deployment

```groovy
pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/jaiswaladi246/Task-Master-Pro.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Unit-Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }

        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectName=taskmaster \
                      -Dsonar.projectKey=taskmaster \
                      -Dsonar.java.binaries=target'''
                }
            }
        }

        stage('Docker build & Tag') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                    sh 'docker build -t chandan712/taskmaster:latest .'
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o image.html chandan712/taskmaster:latest'
            }
        }

        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                    sh 'docker push chandan712/taskmaster:latest'
                }
            }
        }

        stage('K8 Deploy') {
            steps {
                withKubeConfig(credentialsId: 'k-token', namespace: 'webapps', serverUrl: 'https://<EKS-CLUSTER-URL>') {
                    sh 'kubectl apply -f deployment-service.yml'
                }
            }
        }

        stage('Verify K8 Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k-token', namespace: 'webapps', serverUrl: 'https://<EKS-CLUSTER-URL>') {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
                }
            }
        }
    }
}
```

After a successful Jenkins build, your pipeline stage view will look like this:

![Screenshot (15)](https://github.com/user-attachments/assets/6150bf10-f847-471e-afe6-c9f0c50b06cd)

After a successful Jenkins build, the SonarQube analysis provides insights into the code quality. Here's an example of a successful analysis for the Task Master Pro project:

![Screenshot (13)](https://github.com/user-attachments/assets/faa940ed-caf9-48c2-86c4-c5990f7da090)


As you can see in the screenshot, the Task Master Pro project has **passed** the Quality Gate with the following metrics:

* **Bugs:** 0
* **Vulnerabilities:** 0
* **Hotspot Reviews:** 0.0% (2 to review)
* **Code Smells:** 2
* **Coverage:** 0.0%
* **Duplications:** 1.6%
* **Lines of Code:** 1.6K

This indicates that the codebase, at the time of this analysis, has no reported bugs or vulnerabilities and a low level of duplication. Addressing the code smells and increasing test coverage would further improve the code quality.

---

## 🔐 GitHub Secrets

| Secret               | Purpose                    |
| -------------------- | -------------------------- |
| `DOCKERHUB_USERNAME` | Docker Hub login           |
| `DOCKERHUB_TOKEN`    | Docker Hub token           |
| `KUBE_CONFIG_PROD`   | Base64 Kubeconfig for prod |

---

## 📄 License

MIT License. Feel free to use, fork, and contribute.

---

## 🙌 Credits

Special thanks to:

* Aditya Jaiswal for the original [Task Master Pro](https://github.com/jaiswaladi246/Task-Master-Pro) codebase.
* Jenkins, Docker, SonarQube, Trivy, Kubernetes, Terraform, Nexus, GitHub Actions communities for the DevOps tooling.
