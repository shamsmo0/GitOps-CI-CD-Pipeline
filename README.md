# GitOps-CI-CD-Pipeline# 🐾 PetClinic Microservices — GitOps CI/CD Pipeline

> Automated deployment of a Spring Boot microservices application using Jenkins, Docker Hub, ArgoCD, and Kubernetes on AWS — provisioned with Terraform and Ansible.

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER WORKFLOW                           │
│                                                                     │
│   Push Code → GitHub Repos → Jenkins CI → Docker Hub               │
│                                    ↓                                │
│              Jenkins updates image tag in GitOps Repo               │
│                                    ↓                                │
│              ArgoCD detects change → deploys to K8s                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Full Pipeline

```
GitHub (3 Service Repos)
        │
        │  webhook / poll
        ▼
Jenkins CI Server  ──── Jenkins Shared Library
        │
        │  1. Checkout Code
        │  2. Maven Build
        │  3. Run Tests
        │  4. Package JAR
        │  5. Build Docker Image
        │  6. Push to Docker Hub
        │  7. Update GitOps Repo (image tag)
        ▼
petclinic-gitops (GitHub)
        │
        │  ArgoCD watches every 3 min
        ▼
ArgoCD Server (on K8s Master)
        │
        │  Auto Sync + Self Heal
        ▼
Kubernetes Cluster (AWS)
        │
        ├── service-a pods (2 replicas)
        ├── service-b pods (2 replicas)
        └── service-c pods (2 replicas)
```

---

## 🗂️ Repositories

| Repository | Description | Link |
|------------|-------------|------|
| `petclinic-service-a` | Spring Boot microservice A | [GitHub](https://github.com/shamsmo0/-petclinic-service-a.git) |
| `petclinic-service-b` | Spring Boot microservice B | [GitHub](https://github.com/shamsmo0/-petclinic-service-b.git) |
| `petclinic-service-c` | Spring Boot microservice C | [GitHub](https://github.com/shamsmo0/-petclinic-service-c.git) |
| `jenkins-shared-library` | Reusable Jenkins pipeline logic | [GitHub](https://github.com/shamsmo0/jenkins-shared-library.git) |
| `petclinic-gitops` | K8s manifests — ArgoCD watches this | [GitHub](https://github.com/shamsmo0/petclinic-gitops.git) |

---

## 🐳 Docker Hub Images

| Image | Link |
|-------|------|
| `shamsmo0h/service-a` | [Docker Hub](https://hub.docker.com/repository/docker/shamsmo0h/service-a) |
| `shamsmo0h/service-b` | [Docker Hub](https://hub.docker.com/repository/docker/shamsmo0h/service-b) |
| `shamsmo0h/service-c` | [Docker Hub](https://hub.docker.com/repository/docker/shamsmo0h/service-c) |

---

## ☁️ Infrastructure (AWS)

### Provisioning Stack

| Tool | Role |
|------|------|
| **Terraform** | Provisions AWS infrastructure (VPC, Subnets, EC2, IAM, Security Groups) |
| **Ansible** | Configures EC2 nodes — installs containerd, kubeadm, kubelet, kubectl |

### AWS Resources

```
AWS Account
└── VPC (10.0.0.0/16)
    └── Public Subnet (10.0.1.0/24)
        ├── EC2 t3.medium — k8s-master     (Control Plane)
        ├── EC2 t3.medium — k8s-worker-1   (Worker Node)
        └── EC2 t3.medium — k8s-worker-2   (Worker Node)
```

### Kubernetes Cluster

| Node | Role | Instance Type |
|------|------|---------------|
| `k8s-master` | Control Plane (API Server, etcd, Scheduler) | t3.medium |
| `k8s-worker-1` | Worker Node | t3.medium |
| `k8s-worker-2` | Worker Node | t3.medium |

### Security Group — Open Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH |
| 8080 | TCP | Jenkins UI |
| 6443 | TCP | Kubernetes API Server |
| 31080 | TCP | ArgoCD UI (NodePort) |
| 30000–32767 | TCP | Kubernetes NodePort Services |
| 2379–2380 | TCP | etcd (internal) |
| 10250–10252 | TCP | kubelet API (internal) |
| 8472 | UDP | Flannel VXLAN (internal) |

---

## 🔧 Tech Stack

| Category | Technology |
|----------|------------|
| Language | Java 17 |
| Framework | Spring Boot |
| Build Tool | Maven |
| Containerization | Docker |
| Container Registry | Docker Hub |
| CI Server | Jenkins |
| Pipeline Logic | Jenkins Shared Library |
| GitOps | ArgoCD |
| Orchestration | Kubernetes v1.29 (kubeadm) |
| CNI | Flannel / Calico |
| IaC | Terraform |
| Configuration Management | Ansible |
| Cloud Provider | AWS (EC2, VPC, IAM) |
| OS | Amazon Linux 2 |

---

## 🚀 CI/CD Flow — Step by Step

### Continuous Integration (Jenkins)

1. Developer pushes code to one of the service repos
2. Jenkins detects the change via webhook or polling
3. Jenkins Shared Library runs the pipeline stages:
   - `buildApp()` — Maven compile
   - `runTests()` — Maven test
   - `packageApp()` — Maven package (JAR)
   - `buildDockerImage()` — Docker build
   - `pushDockerImage()` — Docker push to Docker Hub
   - `updateManifest()` — updates image tag in `petclinic-gitops`

### Continuous Delivery (ArgoCD GitOps)

1. ArgoCD watches `petclinic-gitops` repo every 3 minutes
2. Detects new image tag in the deployment manifest
3. Automatically syncs and applies the updated manifest to K8s
4. Self-heals if any manual change is made to the cluster

---

## 📁 GitOps Repo Structure (`petclinic-gitops`)

```
petclinic-gitops/
├── service-a/
│   ├── deployment.yaml    ← image: shamsmo0h/service-a:<tag>
│   └── service.yaml
├── service-b/
│   ├── deployment.yaml
│   └── service.yaml
├── service-c/
│   ├── deployment.yaml
│   └── service.yaml
└── applications.yaml      ← ArgoCD Application manifests (all 3 services)
```

---

## 🛠️ How to Reproduce This Setup

### Prerequisites

- AWS account with IAM user (Access Key + Secret Key)
- AWS Key Pair (.pem file)
- Local machine with: `terraform`, `ansible`, `aws-cli`, `kubectl`

### Step 1 — Provision Infrastructure

```bash
git clone https://github.com/shamsmo0/petclinic-gitops.git
cd terraform/
terraform init
terraform apply
# exports IPs to ansible/inventory.ini automatically
```

### Step 2 — Configure the Cluster

```bash
cd ansible/
ansible-playbook -i inventory.ini site.yml
```

### Step 3 — Install Jenkins on Master

```bash
ssh -i ~/.ssh/k8s.pem ec2-user@<MASTER_IP>
sudo yum install -y fontconfig dejavu-sans-fonts java-17-openjdk jenkins
sudo systemctl start jenkins
```

### Step 4 — Deploy ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

### Step 5 — Apply ArgoCD Applications

```bash
kubectl apply -f applications.yaml
```

### Step 6 — Configure Jenkins

- Add Docker Hub credentials (`dockerhub-creds`)
- Add GitHub credentials (`github-creds`)
- Add `jenkins-shared-library` as a Global Shared Library
- Create Multibranch Pipeline jobs for each service

---

## 🌐 Access

| Service | URL |
|---------|-----|
| Jenkins | `http://<MASTER_IP>:8080` |
| ArgoCD | `http://<MASTER_IP>:31080` |
| Service A | `http://<MASTER_IP>:<NODEPORT>/service-a` |
| Service B | `http://<MASTER_IP>:<NODEPORT>/service-b` |
| Service C | `http://<MASTER_IP>:<NODEPORT>/service-c` |

---

## 👤 Author

**Shams** — AWS & DevOps Engineer
- GitHub: [@shamsmo0](https://github.com/shamsmo0)
- Docker Hub: [shamsmo0h](https://hub.docker.com/u/shamsmo0h)
