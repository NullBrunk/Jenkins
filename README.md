<div align="center">

# Jenkins
_CI/CD pipelines for a Java + Node.js application, delivered on AWS EC2 or Kubernetes_

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazonaws&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?logo=sonarqubeserver&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA)

</div>

Jenkins DevSecOps pipelines that take an application from source code to a running service: **build and test**, **quality and security** checks, Docker images **packaging**, servers **delivery** . Everything runs automatically, from a single Jenkinsfile.

Originally written during my junior DevOps internship, then made generic so it can be reused on any project.

## Overview

<img src="docs/overview.svg" alt="Your code goes through a Jenkins pipeline: build and test, check code quality, package in Docker images, scan for security issues, then deliver to EC2 servers or a Kubernetes cluster. The team is notified in a chat." width="100%">

| Pipeline | Delivers the backend to | Delivers the frontend to |
| :-- | :-- | :-- |
| [EC2 pipeline](pipelines/ec2_pipeline/Jenkinsfile) | An EC2 server | An EC2 server |
| [K8s pipeline](pipelines/k8s_pipeline/Jenkinsfile) | A Kubernetes cluster | An EC2 server |

---

## EC2 pipeline

The complete pipeline: it builds, tests, checks, packages and delivers the application to EC2 servers.

<img src="docs/ec2-pipeline.svg" alt="The eight steps of the EC2 pipeline: announce, get the code, build and test the backend, build and test the frontend, check code quality, package the apps, security scan, deliver to the backend and frontend servers. Test results, security report, final message and cleanup follow every run." width="100%">

| # | Step | What it does | Stage name in the Jenkinsfile |
| :-- | :-- | :-- | :-- |
| 1 | Get the code | Clones the application repository over SSH | `Backend: Checkout` |
| 2 | Build & test backend | `mvn clean package`, then `mvn test` with the `ci` Spring profile | `Backend: Build`, `Backend: Test` |
| 3 | Build & test frontend | `npm ci`, `npm run build`, `npm run test` | `Frontend: Install`, `Build`, `Test` |
| 4 | Check code quality | SonarQube analysis of the backend | `Backend: SonarQube Analysis` |
| 5 | Package the apps | Builds both Docker images and pushes them to Docker Hub, tagged with the build number and `latest` | `Docker Build & push backend`, `Docker: Build & push frontend` |
| 6 | Security scan | Trivy scan of the backend image, written to an HTML report | `Trivy Scan` |
| 7 | Deliver | Updates the backend and frontend servers | `Deploy on EC2` |

### Delivery to EC2

<img src="docs/deploy-ec2.svg" alt="Jenkins publishes the new version to Docker Hub, logs in to each server over SSH, and each server downloads the new version and restarts the app with docker compose." width="100%">

Jenkins connects to each server over SSH and runs `docker compose pull && docker compose up -d`: the server downloads the new images from Docker Hub and restarts the application with them. The SSH key is stored in Jenkins' credentials, and the two servers are set with the `BACKEND_HOST` and `FRONTEND_HOST` variables.

---

## Kubernetes pipeline

The same pipeline, except that the **backend runs on a Kubernetes cluster** instead of an EC2 server. Steps 1 to 7 are identical; only the delivery changes.

<img src="docs/k8s-pipeline.svg" alt="The Kubernetes pipeline: same first seven steps, then the backend is delivered to a Kubernetes cluster and the frontend to an EC2 server." width="100%">

| | EC2 pipeline | K8s pipeline |
| :-- | :-- | :-- |
| Backend delivery | SSH + `docker compose` on an EC2 server | `kubectl set image` on the cluster |
| Frontend delivery | SSH + `docker compose` on an EC2 server | Unchanged: SSH + `docker compose` on an EC2 server |
| Image tag | Build number | Unique generated tag |
| Stages | `Deploy on EC2` | `Set new image k8s`, `Deploy frontend on EC2` |

### Delivery to Kubernetes

<img src="docs/deploy-k8s.svg" alt="Jenkins publishes the new image to Docker Hub with a unique tag, then runs kubectl set image. Kubernetes pulls the image and replaces the running copies of the backend one by one." width="100%">

Each run gives the images a **unique tag**, so Kubernetes can tell that a new version exists. Jenkins then runs a single command, `kubectl set image deployment/backend backend=<image>:<tag>`, and Kubernetes takes over: it pulls the new image and replaces the running copies of the backend step by step (a rolling update).

---

## Quality and security

<img src="docs/quality-security.svg" alt="SonarQube analyses the backend source code and sends a quality report to its server. Trivy scans the backend Docker image and produces a security report in Jenkins." width="100%">

- **SonarQube** analyses the backend code (`mvn sonar:sonar`) and sends the results to the SonarQube server configured in Jenkins.
- **Trivy** scans the backend Docker image for known vulnerabilities and writes an HTML report using a custom template. It runs with `--exit-code 0`: it reports its findings without failing the build.

---

## Setup

### Jenkins configuration

| Item | Used for |
| :-- | :-- |
| Tools `M3` and `N20` | Maven 3 and Node.js 20 (Manage Jenkins → Tools) |
| SonarQube server named `sonarqube` | `withSonarQubeEnv` (Manage Jenkins → System) |
| Plugins | Pipeline, Git, Credentials Binding, Docker Pipeline, SSH Agent, NodeJS, SonarQube Scanner, JUnit, HTML Publisher |

On the Jenkins agent: Docker, the [Trivy](https://github.com/aquasecurity/trivy) binary and its HTML template (`/var/jenkins_home/trivy` and `/var/jenkins_home/html.tpl`), and, for the K8s pipeline, `kubectl` configured to reach your cluster.

### Credentials

| ID | Type | Used for |
| :-- | :-- | :-- |
| `jenkins-ssh-key` | SSH private key | Cloning the application repository from GitHub |
| `docker-hub-creds` | Username and password | Pushing the Docker images |
| `aws-key` | SSH private key | Logging in to the servers to deliver the new version |
| `discord-webhook` | Secret text | Posting the chat messages |

### Variables

Set these in the `environment` block of the Jenkinsfile.

| Variable | Purpose |
| :-- | :-- |
| `GITHUB_REPO_SSH`, `GITHUB_BRANCH` | Application repository and branch to build |
| `DOCKER_REGISTRY` | Docker registry URL |
| `DOCKER_BACKEND_IMAGE`, `DOCKER_FRONTEND_IMAGE` | Names of the Docker images (`user/backend`, `user/frontend`) |
| `SPRING_DATASOURCE_URL`, `SPRING_PROFILES_ACTIVE` | Database and Spring profile used by the backend tests |
| `BACKEND_HOST`, `FRONTEND_HOST` | Servers to deliver to (EC2 pipeline). The K8s pipeline only uses `FRONTEND_HOST` |

Then create a **Pipeline** job in Jenkins and point it to the Jenkinsfile of the pipeline you want to use.

## Works well with

The servers these pipelines deliver to can be prepared with my [Ansible playbooks](https://github.com/NullBrunk/Ansible): common admin user, SSH access, stable internal host names and Docker.

## Skills demonstrated

- **CI/CD**: declarative Jenkins pipelines, multi-stage builds, credentials management
- **Containers**: Docker image build and publishing, Docker Compose delivery
- **Kubernetes**: rolling updates with `kubectl`
- **DevSecOps**: static analysis with SonarQube, image scanning with Trivy
- **Observability**: chat notifications, test and security reports
- **AWS**: EC2 delivery over SSH
