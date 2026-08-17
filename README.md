### Project1 – Notes App CI/CD

A full-stack Notes Application built with React, Django, and MySQL, containerized using Docker and deployed to Kubernetes using a Jenkins + Docker Hub + GitOps + Argo CD CI/CD pipeline.

⸻

1. Project Overview

This project demonstrates an end-to-end CI/CD and GitOps workflow.

Application Stack

Component	Technology
Frontend	React
Backend	Django + Gunicorn
Database	MySQL
WSGI Server	Gunicorn
Containerization	Docker
Container Registry	Docker Hub
CI/CD	Jenkins
Orchestration	Kubernetes
GitOps	Argo CD
Source Control	GitHub

⸻

2. Repository Structure

Two separate GitHub repositories are used.

Application Repository

Project1-notesapp-application

This repository contains:

* Django application
* React application
* Dockerfiles
* Application source code

DevOps / GitOps Repository

Project1-notesapp-devops

This repository contains the Kubernetes manifests and Argo CD configuration.

Project1-notesapp-devops/
│
├── k8s/
│   ├── 1.config.yml
│   ├── 2.django_service.yml
│   ├── 3.django_deployment.yml
│   ├── 4.react_service.yml
│   ├── 5.react_deployment.yml
│   ├── 6.mysql_service.yml
│   ├── 7.mysql_deployment.yml
│   ├── 8.ingress.yml
│   ├── 9.hpa.yml
│   └── 10.vpa.yml
│
├── k8s_secrets/
│   ├── namespace.yml
│   └── secrets.yml
│
├── argocd/
│   ├── project.yml
│   └── notesapp-application.yml
│
├── .gitignore
└── Jenkinsfile

Note: k8s_secrets/ is added to .gitignore and is kept locally. It is shown above only to document the project structure.

⸻

3. Application Architecture

                         Notes Application
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
        React Frontend                    Django Backend
           Port 3000                         Port 8000
                                                │
                                                ▼
                                           MySQL Database

The React frontend and Django backend are built as separate Docker images.

MySQL is deployed separately in Kubernetes and is not included inside the Django or React Docker images.

Django runs using Gunicorn as the production WSGI server instead of Django’s development runserver.

⸻

4. Docker Configuration

4.1 Django Dockerfile

The Django backend uses Gunicorn to serve the Django application through WSGI.

FROM python:latest
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "<django_project>.wsgi:application", "--bind", "0.0.0.0:8000"]

Important

Replace:

<django_project>

with the Django project package that contains:

wsgi.py

For example, if your structure is:

project/
├── manage.py
├── notes/
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── __init__.py

then the command becomes:

CMD ["gunicorn", "notes.wsgi:application", "--bind", "0.0.0.0:8000"]

Django container

* Base image: Python
* Working directory: /app
* Application port: 8000
* Dependencies are installed from requirements.txt
* Gunicorn is used as the WSGI server
* Django is served through wsgi.py
* Django runserver is not used

⸻

4.2 Django Requirements

Make sure requirements.txt contains Gunicorn.

Example:

Django
gunicorn
mysqlclient

If your project already contains other dependencies, keep them as well.

The important requirement is:

gunicorn

Jenkins does not need a separate Gunicorn installation because Gunicorn is installed inside the Docker image through requirements.txt.

⸻

4.3 React Dockerfile

The React frontend uses:

FROM node:22
WORKDIR /app/
COPY . /app/
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]

React container

* Base image: Node.js 22
* Working directory: /app
* Application port: 3000
* Dependencies are installed using npm install
* React starts using npm start

For a production deployment, React should ideally be built into static assets and served using Nginx or another production web server.

⸻

5. Build Docker Images

The Django and React applications are built as separate Docker images.

5.1 Build Django Image

Run from the directory containing the Django Dockerfile:

docker build --no-cache -t nithingowda46/notes-app:backend .

5.2 Build React Image

Run from the main application directory:

docker build --no-cache -t nithingowda46/notes-app:frontend ./mynotes

⸻

6. Push Docker Images to Docker Hub

Login to Docker Hub:

docker login

Push the Django image:

docker push nithingowda46/notes-app:backend

Push the React image:

docker push nithingowda46/notes-app:frontend

These images are later used by Kubernetes.

⸻

7. Kubernetes Repository

Create a separate repository:

Project1-notesapp-devops

The purpose of this repository is to maintain the Kubernetes manifests separately from the application source code.

This follows a GitOps-style repository structure.

⸻

8. Kubernetes Manifests

The k8s directory contains the Kubernetes configuration:

k8s/
├── 1.config.yml
├── 2.django_service.yml
├── 3.django_deployment.yml
├── 4.react_service.yml
├── 5.react_deployment.yml
├── 6.mysql_service.yml
├── 7.mysql_deployment.yml
├── 8.ingress.yml
├── 9.hpa.yml
└── 10.vpa.yml

Purpose of the Files

File	Purpose
1.config.yml	Application configuration
2.django_service.yml	Exposes Django inside Kubernetes
3.django_deployment.yml	Deploys Django pods
4.react_service.yml	Exposes React inside Kubernetes
5.react_deployment.yml	Deploys React pods
6.mysql_service.yml	Provides network access to MySQL
7.mysql_deployment.yml	Deploys MySQL
8.ingress.yml	Provides external HTTP/HTTPS routing
9.hpa.yml	Configures Horizontal Pod Autoscaling
10.vpa.yml	Configures Vertical Pod Autoscaling

The Django Kubernetes Deployment continues to expose port 8000.

Gunicorn listens on:

0.0.0.0:8000

Therefore, the existing Django Service and Ingress configuration can continue to use port 8000, provided the current YAML is already configured for that port.

⸻

9. Namespace and Secrets

Sensitive Kubernetes configuration is kept separately:

k8s_secrets/
├── namespace.yml
└── secrets.yml

The directory is added to .gitignore:

k8s_secrets/

Therefore, these files are not committed to the GitOps repository.

Apply Namespace

kubectl apply -f k8s_secrets/namespace.yml

Apply Secrets

kubectl apply -f k8s_secrets/secrets.yml

These are applied separately before the normal Kubernetes resources.

Never commit real passwords, tokens, API keys, or other sensitive values to a public Git repository.

⸻

10. Kubernetes Deployment Flow

The basic Kubernetes setup is:

Namespace
   ↓
Secrets
   ↓
ConfigMap
   ↓
Django Deployment + Service
   ↓
React Deployment + Service
   ↓
MySQL Deployment + Service
   ↓
Ingress
   ↓
HPA
   ↓
VPA

Django pods run the application using:

Gunicorn
   ↓
Django WSGI

instead of:

Django runserver

⸻

11. Argo CD

Argo CD is used to implement the GitOps deployment process.

Argo CD continuously monitors the Kubernetes manifests stored in:

Project1-notesapp-devops

When Jenkins updates the Docker image version in the Kubernetes deployment files and pushes the changes to GitHub, Argo CD detects the Git change and synchronizes the Kubernetes cluster.

⸻

12. Argo CD AppProject

Create an Argo CD AppProject.

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: project-2
  namespace: argocd
spec:
  description: Production project for Notes Application
  sourceRepos:
    - https://github.com/NithinGowda46/Project1-notesapp-devops.git
  destinations:
    - namespace: project-2
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
    - group: ""
      kind: PersistentVolume
    - group: ""
      kind: PersistentVolumeClaim
    - group: storage.k8s.io
      kind: StorageClass
    - group: "*"
      kind: "*"
  namespaceResourceWhitelist:
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: ""
      kind: Service
    - group: ""
      kind: ServiceAccount
    - group: ""
      kind: PersistentVolumeClaim
    - group: apps
      kind: Deployment
    - group: apps
      kind: StatefulSet
    - group: apps
      kind: ReplicaSet
    - group: apps
      kind: DaemonSet
    - group: networking.k8s.io
      kind: Ingress
    - group: autoscaling
      kind: HorizontalPodAutoscaler
    - group: autoscaling.k8s.io
      kind: VerticalPodAutoscaler
    - group: policy
      kind: PodDisruptionBudget
    - group: batch
      kind: Job
    - group: batch
      kind: CronJob
    - group: "*"
      kind: "*"
  orphanedResources:
    warn: true

Apply the AppProject

kubectl apply -f argocd/project.yml

project-2 under spec.project refers to the Argo CD AppProject.

The Kubernetes namespace project-2 is a separate Kubernetes resource, even though they use the same name in this project.

⸻

13. Argo CD Application

Create an Argo CD Application:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: notesapp
  namespace: argocd
spec:
  project: project-2
  source:
    repoURL: https://github.com/NithinGowda46/Project1-notesapp-devops.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: project-2
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - PruneLast=true
      - PrunePropagationPolicy=foreground

Apply the Argo CD Application

kubectl apply -f argocd/notesapp-application.yml

Argo CD will now monitor:

GitHub
   ↓
Project1-notesapp-devops
   ↓
k8s/

and synchronize the Kubernetes resources.

⸻

14. Jenkins CI/CD Pipeline

Jenkins is responsible for:

1. Cloning the application repository.
2. Logging into Docker Hub.
3. Building the Django Docker image.
4. Pushing the Django image.
5. Building the React Docker image.
6. Pushing the React image.
7. Cloning the GitOps repository.
8. Updating the Django image tag.
9. Updating the React image tag.
10. Committing the changes.
11. Pushing the changes to GitHub.
12. Argo CD detects the Git change and deploys the new version.

Gunicorn does not require any additional Jenkins stage because it is installed and executed inside the Django Docker image.

⸻

15. Jenkinsfile

The Jenkins pipeline is:

pipeline {
    agent any
    stages {
        stage('Git Config') {
            steps {
                sh '''
                    git config --global user.name "Nithin Gowda"
                    git config --global user.email "nithingowdai46@gmail.com"
                '''
            }
        }
        stage('Clone Application') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/NithinGowda46/Project1-notesapp-application.git'
            }
        }
        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerid',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                        -u "$DOCKER_USER" \
                        --password-stdin
                    '''
                }
            }
        }
        stage('Build Django Image') {
            steps {
                sh '''
                    docker build --no-cache \
                    -t nithingowda46/notes-app:${BUILD_NUMBER}-back .
                '''
            }
        }
        stage('Push Django Image') {
            steps {
                sh '''
                    docker push \
                    nithingowda46/notes-app:${BUILD_NUMBER}-back
                '''
            }
        }
        stage('Build React Image') {
            steps {
                sh '''
                    docker build --no-cache \
                    -t nithingowda46/notes-app:${BUILD_NUMBER}-front ./mynotes
                '''
            }
        }
        stage('Push React Image') {
            steps {
                sh '''
                    docker push \
                    nithingowda46/notes-app:${BUILD_NUMBER}-front
                '''
            }
        }
        stage('Clone GitOps Repo') {
            steps {
                dir('gitops') {
                    git branch: 'main',
                        url: 'https://github.com/NithinGowda46/Project1-notesapp-devops.git'
                }
            }
        }
        stage('Update Django Image') {
            steps {
                dir('gitops') {
                    sh '''
                        sed -i \
                        "s|image: nithingowda46/notes-app:.*-back|image: nithingowda46/notes-app:${BUILD_NUMBER}-back|" \
                        "k8s/3.django_deployment.yml"
                    '''
                }
            }
        }
        stage('Update React Image') {
            steps {
                dir('gitops') {
                    sh '''
                        sed -i \
                        "s|image: nithingowda46/notes-app:.*-front|image: nithingowda46/notes-app:${BUILD_NUMBER}-front|" \
                        "k8s/5.react_deployment.yml"
                    '''
                }
            }
        }
        stage('Git Commit') {
            steps {
                dir('gitops') {
                    sh '''
                        git add \
                        "k8s/3.django_deployment.yml" \
                        "k8s/5.react_deployment.yml"
                        git commit \
                        -m "Update backend and frontend images to build ${BUILD_NUMBER}" \
                        || echo "No changes to commit"
                    '''
                }
            }
        }
        stage('Git Push') {
            steps {
                dir('gitops') {
                    sh '''
                        git push origin main
                    '''
                }
            }
        }
    }
}

Important: The sed -i commands use Linux syntax. This Jenkinsfile is therefore intended for a Linux Jenkins agent/server. On macOS, the sed syntax would use sed -i "".

No Jenkinsfile change is required specifically for Gunicorn.

⸻

16. Docker Image Versioning

Jenkins uses the Jenkins BUILD_NUMBER to create unique Docker image tags.

For example, if Jenkins runs build number 10:

nithingowda46/notes-app:10-back
nithingowda46/notes-app:10-front

The Kubernetes deployment files are automatically updated with these image versions.

Example:

image: nithingowda46/notes-app:10-back

and:

image: nithingowda46/notes-app:10-front

This avoids relying on the latest tag and allows each Jenkins build to have a traceable image version.

⸻

17. Complete CI/CD Flow

                    Developer
                        │
                        │ git push
                        ▼
        ┌──────────────────────────────┐
        │ GitHub Application Repository│
        │ Project1-notesapp-application│
        └──────────────┬───────────────┘
                       │
                       ▼
                    Jenkins
                       │
                       ▼
                Clone Application
                       │
                       ▼
                 Docker Login
                       │
              ┌────────┴────────┐
              ▼                 ▼
        Build Django       Build React
           Image              Image
              │                 │
              ▼                 ▼
        Push Docker Hub   Push Docker Hub
              │                 │
              └────────┬────────┘
                       ▼
              Clone GitOps Repo
                       │
                       ▼
        Project1-notesapp-devops
                       │
                       ▼
             Update Image Tags
                       │
                       ▼
                  Git Commit
                       │
                       ▼
                   Git Push
                       │
                       ▼
                    Argo CD
                       │
                       ▼
             Kubernetes Cluster
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       React         Django        MySQL
                     │
                     ▼
                  Gunicorn
                     │
                     ▼
                  Django WSGI

⸻

18. GitOps Deployment Flow

Argo CD follows the GitOps model.

Jenkins
   │
   │ Updates image tag
   ▼
GitOps Repository
   │
   │ Git commit + push
   ▼
Argo CD
   │
   │ Detects repository change
   ▼
Kubernetes
   │
   │ Pulls new Docker image
   ▼
New Application Version

With:

syncPolicy:
  automated:
    prune: true
    selfHeal: true

Argo CD can automatically synchronize the desired state stored in Git with the Kubernetes cluster.

⸻

19. Verify Kubernetes Deployment

Check the Namespace

kubectl get namespaces

Check Pods

kubectl get pods -n project-2

Check Services

kubectl get svc -n project-2

Check Deployments

kubectl get deployments -n project-2

Check Ingress

kubectl get ingress -n project-2

Check HPA

kubectl get hpa -n project-2

Check VPA

kubectl get vpa -n project-2

⸻

20. Verify Argo CD

Check the Argo CD Application:

kubectl get applications -n argocd

Check the AppProject:

kubectl get appproject -n argocd

Describe the Application:

kubectl describe application notesapp -n argocd

The Argo CD Application should eventually show the desired Git revision as synchronized with the Kubernetes cluster.

⸻

21. Important Configuration Requirements

Before running the Jenkins pipeline, make sure the Jenkins agent has:

* Git installed
* Docker installed
* Permission to execute Docker commands
* Jenkins Docker Hub credentials configured
* Permission to push to the GitOps GitHub repository

The GitOps repository must contain the expected directory:

k8s/

The deployment files must exist:

k8s/3.django_deployment.yml
k8s/5.react_deployment.yml

The Argo CD Application must point to:

Repository:
Project1-notesapp-devops
Branch:
main
Path:
k8s

The Django Docker image must also have:

gunicorn

available through requirements.txt.

⸻

22. Security Notes

Do not commit sensitive information such as:

* Database passwords
* Docker Hub passwords
* GitHub access tokens
* AWS credentials
* API keys
* Kubernetes secret values

The Kubernetes secret files are kept outside Git using:

k8s_secrets/

Jenkins credentials should be stored using Jenkins Credentials Manager.

⸻

23. Production Considerations

This project is designed as a DevOps learning/portfolio project.

For a production deployment, consider:

* Use a fixed Python base image instead of python:latest.
* Use Gunicorn instead of Django runserver.
* Configure Gunicorn workers appropriately for the workload.
* Add a proper health check/readiness probe for the Django application.
* Build React as production static assets and serve them through Nginx or another production web server.
* Use managed MySQL such as Amazon RDS instead of running the database directly inside Kubernetes when appropriate.
* Use Kubernetes Secrets integrated with a proper secret-management solution.
* Avoid wildcard permissions in Argo CD AppProjects.
* Use GitHub credentials/tokens securely for Jenkins Git operations.
* Use Docker build caching to reduce build time.
* Add automated tests before Docker image creation.
* Add vulnerability scanning for Docker images.
* Configure monitoring and logging.
* Use TLS/HTTPS for the application.
* Use a production-grade Kubernetes cluster instead of a local development cluster.
* Use resource requests and limits for Kubernetes workloads.
* Configure persistent storage appropriately for MySQL.
* Configure database backups and disaster recovery.

⸻

24. Final Architecture

                   ┌───────────────────────┐
                   │       Developer       │
                   └───────────┬───────────┘
                               │
                            git push
                               │
                               ▼
              ┌──────────────────────────────┐
              │ Application GitHub Repository│
              │ Project1-notesapp-application│
              └──────────────┬───────────────┘
                             │
                             ▼
                     ┌─────────────┐
                     │   Jenkins   │
                     └──────┬──────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
          Django Image             React Image
                │                       │
                └───────────┬───────────┘
                            ▼
                       Docker Hub
                            │
                            ▼
              ┌──────────────────────────┐
              │      GitOps Repository   │
              │ Project1-notesapp-devops │
              └─────────────┬────────────┘
                            │
                     Image tag update
                            │
                            ▼
                       GitHub Push
                            │
                            ▼
                       ┌─────────┐
                       │ Argo CD │
                       └────┬────┘
                            │
                           Sync
                            │
                            ▼
                  ┌──────────────────┐
                  │    Kubernetes    │
                  │                  │
                  │ React            │
                  │ Django           │
                  │ Gunicorn         │
                  │ MySQL            │
                  │ Ingress          │
                  │ HPA              │
                  │ VPA              │
                  └──────────────────┘

⸻

25. Summary

This project implements a complete CI/CD and GitOps workflow:

GitHub
  ↓
Jenkins
  ↓
Docker Build
  ↓
Docker Hub
  ↓
GitOps Repository
  ↓
Argo CD
  ↓
Kubernetes
  ↓
React + Django + Gunicorn + MySQL

The application source code and Kubernetes deployment configuration are maintained in separate repositories.

Docker images are versioned using Jenkins build numbers.

Jenkins builds and pushes the Django and React images, then updates the corresponding Kubernetes image tags in the GitOps repository.

Argo CD detects the Git change and automatically synchronizes Kubernetes with the desired state stored in Git.

The Django application is served using Gunicorn and WSGI instead of Django’s development runserver.

⸻

👨‍💻 Author

Nithin Gowda

This project demonstrates an end-to-end DevOps workflow using Docker, Jenkins, Kubernetes, GitOps, Argo CD, Gunicorn, and WSGI.
