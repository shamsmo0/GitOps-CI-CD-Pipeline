# 🐾 PetClinic Microservices — GitOps CI/CD Pipeline

> Automated deployment of a Spring Boot microservices application using **Jenkins**, **Docker Hub**, **ArgoCD**, and **Kubernetes** on **AWS** — provisioned with **Terraform** and configured with **Ansible**.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repositories](#repositories)
- [Infrastructure](#infrastructure)
- [CI/CD Pipeline](#cicd-pipeline)
- [GitOps with ArgoCD](#gitops-with-argocd)
- [Docker Images](#docker-images)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)

---

## Overview

PetClinic is a classic Spring Boot application re-architected into **3 independent microservices** (A, B, C), each with its own GitHub repository, Dockerfile, and Kubernetes deployment. The entire lifecycle — from code commit to production deployment — is fully automated through a GitOps pipeline.

```
Developer pushes code
        ↓
Jenkins CI builds, tests, and pushes Docker image
        ↓
Jenkins updates image tag in GitOps repo
        ↓
ArgoCD detects change and syncs to Kubernetes
        ↓
Kubernetes rolls out new pods automatically
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CI Pipeline (Jenkins)                     │
│  Checkout → Maven Build → Test → Docker Build → Push → Sync │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              GitOps Engine (petclinic-gitops)                │
│         service-a/  │  service-b/  │  service-c/            │
│       deployment.yaml             deployment.yaml            │
└─────────────────────────────────────────────────────────────┘
                              ↓  (ArgoCD watches)
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster on AWS (kubeadm)             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  service-a   │  │  service-b   │  │  service-c   │      │
│  │  (2 replicas)│  │  (2 replicas)│  │  (2 replicas)│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│              Nginx Ingress Controller                        │
│     /service-a  │  /service-b  │  /service-c                │
└─────────────────────────────────────────────────────────────┘
```

---

## Repositories

| Repository | Description |
|------------|-------------|
| [Deploy-K8S-Cluster-in-AWS](https://github.com/shamsmo0/Deploy-K8S-Cluster-in-AWS.git) | Infrastructure as Code — Terraform + Ansible to provision K8s cluster on AWS |
| [petclinic-service-a](https://github.com/shamsmo0/-petclinic-service-a.git) | Spring Boot microservice A — source code, Dockerfile, Jenkinsfile |
| [petclinic-service-b](https://github.com/shamsmo0/-petclinic-service-b.git) | Spring Boot microservice B — source code, Dockerfile, Jenkinsfile |
| [petclinic-service-c](https://github.com/shamsmo0/-petclinic-service-c.git) | Spring Boot microservice C — source code, Dockerfile, Jenkinsfile |
| [jenkins-shared-library](https://github.com/shamsmo0/jenkins-shared-library.git) | Reusable Jenkins pipeline steps shared across all services |
| [petclinic-gitops](https://github.com/shamsmo0/petclinic-gitops.git) | GitOps engine — Kubernetes manifests watched by ArgoCD |

---

## Infrastructure

The Kubernetes cluster is provisioned on **AWS** using a two-step IaC approach:

### Terraform — Infrastructure as Code

Provisions all AWS resources automatically:

- **VPC** with public subnet (`10.0.0.0/16`)
- **Internet Gateway** and Route Tables
- **Security Groups** with Kubernetes-specific port rules
- **3 EC2 instances** (`t3.medium`): 1 master + 2 workers
- **IAM roles** for EC2 instances
- **30GB gp3 EBS volumes** per node

### Ansible — Configuration Management

Configures the EC2 instances into a production-ready Kubernetes cluster:

- Disables swap, loads kernel modules, sets sysctl params
- Installs `containerd`, `kubeadm`, `kubelet`, `kubectl`
- Runs `kubeadm init` on the master node
- Installs **Flannel CNI** for pod networking
- Joins worker nodes to the cluster automatically

### Cluster Topology

| Node | Instance Type | Role |
|------|--------------|------|
| k8s-master | t3.medium | Control Plane (API Server, etcd, Scheduler, Controller Manager) |
| k8s-worker-1 | t3.small | Worker Node |
| k8s-worker-2 | t3.small | Worker Node |

**AWS Region:** `us-east-1`

### Security Group Port Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH access |
| 6443 | TCP | Kubernetes API Server |
| 2379-2380 | TCP | etcd (internal) |
| 10250-10252 | TCP | kubelet API (internal) |
| 8472 | UDP | Flannel VXLAN (internal) |
| 30000-32767 | TCP | NodePort Services |
| 8080 | TCP | Jenkins Web UI |
| 31080 | TCP | ArgoCD Web UI |

---

## CI/CD Pipeline

Each service has a **Jenkinsfile** that uses the shared library and runs the following stages:

```
1. Checkout       — Clone source code from GitHub
2. Build          — mvn clean compile
3. Test           — mvn test
4. Package        — mvn package (produces JAR)
5. Docker Build   — Build image tagged with build number
6. Push           — Push to Docker Hub (shamsmo0h/<service>:latest)
7. Update GitOps  — Update image tag in petclinic-gitops repo
```

### Jenkins Shared Library

The `jenkins-shared-library` repo contains reusable Groovy steps used across all 3 service pipelines:

| Function | Description |
|----------|-------------|
| `buildApp()` | Maven compile |
| `runTests()` | Maven test |
| `packageApp()` | Maven package (JAR) |
| `buildDockerImage()` | Docker build + tag |
| `pushDockerImage()` | Push to Docker Hub |
| `deployToKubernetes()` | Update image tag in GitOps repo |

---

## GitOps with ArgoCD

ArgoCD runs inside the Kubernetes cluster and watches the `petclinic-gitops` repository for any changes to Kubernetes manifests.

### Applications

| App | Source Path | Namespace | Sync Policy |
|-----|-------------|-----------|-------------|
| service-a | `service-a/` | default | Automated |
| service-b | `service-b/` | default | Automated |
| service-c | `service-c/` | default | Automated |

### Sync Features

- ✅ **Auto-sync** — detects and applies changes automatically
- ✅ **Self-heal** — reverts any manual changes made directly to the cluster
- ✅ **Prune** — removes Kubernetes resources deleted from Git

### Access ArgoCD UI

```
http://<MASTER_IP>:31080
Username: admin
Password: kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

---

## Docker Images

All images are hosted on Docker Hub under `shamsmo0h`:

| Service | Docker Hub |
|---------|-----------|
| service-a | [shamsmo0h/service-a](https://hub.docker.com/repository/docker/shamsmo0h/service-a) |
| service-b | [shamsmo0h/service-b](https://hub.docker.com/repository/docker/shamsmo0h/service-b) |
| service-c | [shamsmo0h/service-c](https://hub.docker.com/repository/docker/shamsmo0h/service-c) |

Base image: `eclipse-temurin:17-jre-alpine`

---

## Tech Stack

| Tool | Version | Purpose |
|------|---------|---------|
| Java | 17 | Application runtime |
| Spring Boot | 3.x | Microservices framework |
| Maven | 3.x | Build and test tool |
| Docker | latest | Containerization |
| Kubernetes | v1.29 | Container orchestration (kubeadm) |
| Jenkins | latest | CI server |
| ArgoCD | latest | GitOps continuous delivery |
| Terraform | >= 1.5 | AWS infrastructure provisioning |
| Ansible | latest | K8s cluster configuration |
| AWS EC2 | t3.medium | Cloud compute nodes |
| Flannel | latest | Kubernetes CNI plugin |
| GitHub | — | Source code + GitOps engine |
| Docker Hub | — | Container image registry |

---

## Getting Started

### Prerequisites

- AWS account with CLI configured (`aws configure`)
- Terraform >= 1.5
- Ansible + `boto3` (`pip3 install ansible boto3`)
- `kubectl` installed locally
- SSH key pair created in AWS

### 1. Provision AWS Infrastructure

```bash
git clone https://github.com/shamsmo0/Deploy-K8S-Cluster-in-AWS.git
cd Deploy-K8S-Cluster-in-AWS/terraform/
terraform init
terraform plan
terraform apply
```

### 2. Configure Kubernetes Cluster

```bash
# Generate Ansible inventory from Terraform outputs
terraform output -raw ansible_inventory > ../ansible/inventory.ini

# Run Ansible playbook
cd ../ansible/
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini site.yml
```

### 3. Verify Cluster

```bash
kubectl get nodes
# NAME           STATUS   ROLES           AGE
# k8s-master     Ready    control-plane
# k8s-worker-1   Ready    <none>
# k8s-worker-2   Ready    <none>
```

### 4. Deploy ArgoCD Applications

```bash
kubectl apply -f applications.yaml
kubectl get applications -n argocd
```

### 5. Trigger the Pipeline

Push any change to a service repo — Jenkins will automatically:
1. Build and test the application
2. Build and push the Docker image
3. Update the image tag in `petclinic-gitops`
4. ArgoCD will detect the change and deploy

### 6. Destroy Infrastructure (when done)

```bash
git clone https://github.com/shamsmo0/Deploy-K8S-Cluster-in-AWS.git
cd Deploy-K8S-Cluster-in-AWS/terraform/
terraform destroy
```

---

## Full Flow Summary

```
1.  Developer pushes code to service repo on GitHub
2.  Jenkins Multibranch Pipeline triggers automatically
3.  Maven builds and tests the Spring Boot application
4.  Docker image built and tagged with build number
5.  Image pushed to Docker Hub (shamsmo0h/<service>:BUILD_NUMBER)
6.  Jenkins clones petclinic-gitops and updates image tag
7.  Jenkins pushes updated manifest back to GitHub
8.  ArgoCD detects change in petclinic-gitops (every 3 min)
9.  ArgoCD syncs updated manifests to Kubernetes cluster
10. Kubernetes performs rolling update with zero downtime
11. Service accessible via Nginx Ingress at /service-a|b|c
```
