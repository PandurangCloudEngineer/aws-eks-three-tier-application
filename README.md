# 🎓 Student Management System — Kubernetes on AWS EKS

A full-stack student management application (React frontend + Node.js backend + MySQL database) containerized with Docker and deployed on **Amazon EKS** using Kubernetes manifests.

> Register students, list them, manage records — all running on a production-grade Kubernetes cluster on AWS.

---

## 📐 Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │              AWS EKS Cluster                 │
                        │                                              │
Internet ──► LoadBalancer SVC ──► Frontend Pod (React)                │
                        │              │                               │
                        │         ClusterIP SVC                        │
                        │              │                               │
                        │         Backend Pod (Node.js)                │
                        │              │                               │
                        │         ClusterIP SVC                        │
                        │              │                               │
                        │         MySQL Pod                            │
                        │                                              │
                        └─────────────────────────────────────────────┘
                                       │
                              ┌────────┴────────┐
                              │   Amazon ECR     │
                              │  frontend image  │
                              │  backend image   │
                              └─────────────────┘
```

---

## 🗂️ Repository Structure

```
├── frontend/
│   ├── Dockerfile
│   └── ...                        # React app source
├── backend/
│   ├── Dockerfile
│   └── ...                        # Node.js/Express source
├── k8s/
│   ├── eks-cluster.yaml           # EKS cluster definition (eksctl)
│   ├── mysql-pod.yaml             # MySQL Pod + ClusterIP Service
│   ├── backend-deployment.yaml    # Backend Deployment + ClusterIP Service
│   └── frontend-deployment.yaml  # Frontend Deployment + LoadBalancer Service
├── docs/
│   └── architecture.md
└── README.md
```

---

## ⚙️ Prerequisites

| Tool | Purpose |
|------|---------|
| [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) | Interact with AWS services |
| [eksctl](https://eksctl.io/) | Create & manage EKS clusters |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Deploy and manage Kubernetes resources |
| [Docker](https://docs.docker.com/get-docker/) | Build container images |

Make sure your AWS CLI is configured:
```bash
aws configure
```

---

## 🚀 Deployment Guide

### Step 1 — Create the EKS Cluster

```bash
eksctl create cluster -f k8s/eks-cluster.yaml
```

Update your local kubeconfig after the cluster is ready:
```bash
aws eks update-kubeconfig --region <your-region> --name <cluster-name>
```

---

### Step 2 — Build Docker Images

**Frontend:**
```bash
cd frontend
docker build -t student-frontend .
```

**Backend:**
```bash
cd backend
docker build -t student-backend .
```

---

### Step 3 — Push Images to Amazon ECR

Create ECR repositories:
```bash
aws ecr create-repository --repository-name student-frontend --region <your-region>
aws ecr create-repository --repository-name student-backend  --region <your-region>
```

Authenticate Docker with ECR:
```bash
aws ecr get-login-password --region <your-region> | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
```

Tag and push images:
```bash
# Frontend
docker tag student-frontend:latest <account-id>.dkr.ecr.<region>.amazonaws.com/student-frontend:latest
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/student-frontend:latest

# Backend
docker tag student-backend:latest <account-id>.dkr.ecr.<region>.amazonaws.com/student-backend:latest
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/student-backend:latest
```

---

### Step 4 — Attach ECR Access Policy to Node IAM Role

So that EKS worker nodes can pull images from ECR:

```bash
# Get the node instance role name
NODE_ROLE=$(aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query "nodegroup.nodeRole" \
  --output text | awk -F'/' '{print $NF}')

# Attach the ECR read-only policy
aws iam attach-role-policy \
  --role-name $NODE_ROLE \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

---

### Step 5 — Deploy MySQL

```bash
kubectl apply -f k8s/mysql-pod.yaml
```

Verify the pod and service are running:
```bash
kubectl get pods
kubectl get svc
```

---

### Step 6 — Deploy the Backend

Update the ECR image URI in `k8s/backend-deployment.yaml`, then:
```bash
kubectl apply -f k8s/backend-deployment.yaml
```

The backend connects to MySQL using the service name as the hostname (Kubernetes internal DNS).

---

### Step 7 — Deploy the Frontend

Get the backend service name (used as the API URL):
```bash
kubectl get svc
```

Update `REACT_APP_API_URL` in `k8s/frontend-deployment.yaml` with the backend service name, then:
```bash
kubectl apply -f k8s/frontend-deployment.yaml
```

---

### Step 8 — Access the Application

Get the external LoadBalancer URL:
```bash
kubectl get svc student-frontend-svc
```

Copy the `EXTERNAL-IP` and open it in your browser. It may take a minute for AWS to provision the load balancer.

---

## 📄 Kubernetes Manifests

### `k8s/eks-cluster.yaml`
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: "uat-dev"
  region: "ap-south-1"
  version: "1.34"

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 3
    volumeSize: 30
    privateNetworking: true
    ssh:
      publicKeyName: pandu_final
```

---

### `k8s/mysql-pod.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: datastore-db
  name: datastore-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: datastore-db
  strategy: {}
  template:
    metadata:
      labels:
        app: datastore-db
    spec:
      containers:
      - image: mysql
        name: mysql
        env:
          - name: MYSQL_ROOT_PASSWORD
            value: test123
```

---

### `k8s/backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: datastore-backend
  labels:
    type: backend

spec:
  template:
    metadata:
      name: datastore-backend
      labels:
        type: backend

    spec:
      containers:
        - name: datastore-backend
          image: 868917998696.dkr.ecr.ap-south-1.amazonaws.com/datastore-be:v2
          env:
            - name: MYSQL_HOST
              value: jdbc:mysql://datastore-db-svc:3306/datastore?createDatabaseIfNotExist=true

            - name: MYSQL_USERNAME
              value: root

            - name: MYSQL_PASSWORD
              value: test123

  replicas: 1

  selector:
    matchLabels:
      type: backend

---
apiVersion: v1
kind: Service

metadata:
  name: datastore-backend-svc

spec:
  ports:
    - port: 8081
      targetPort: 8084

  selector:
    type: backend
```

---

### `k8s/frontend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: datastore-frontend
  labels:
    type: frontend

spec:
  replicas: 3

  selector:
    matchLabels:
      type: frontend

  template:
    metadata:
      labels:
        type: frontend

    spec:
      containers:
        - name: datastore-frontend
          image: 868917998696.dkr.ecr.ap-south-1.amazonaws.com/datastore-fe:v2

          env:
            - name: API_URL
              value: "http://datastore-backend-svc:8081"

          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service

metadata:
  name: datastore-fe-svc

spec:
  type: LoadBalancer

  selector:
    type: frontend

  ports:
    - port: 8081
      targetPort: 8080
      protocol: TCP
```

---

## 🔍 Useful Commands

```bash
# Check all running resources
kubectl get all

# View logs for a pod
kubectl logs <pod-name>

# Describe a pod (for debugging)
kubectl describe pod <pod-name>

# Delete all resources
kubectl delete -f k8s/

# Get LoadBalancer external IP
kubectl get svc student-frontend-svc
```

---

## 🧹 Cleanup

To avoid AWS charges, delete all resources when done:

```bash
# Delete Kubernetes resources
kubectl delete -f k8s/

# Delete the EKS cluster
eksctl delete cluster -f k8s/eks-cluster.yaml

# Delete ECR repositories
aws ecr delete-repository --repository-name student-frontend --force --region <your-region>
aws ecr delete-repository --repository-name student-backend  --force --region <your-region>
```

---

## 💡 Key Concepts Used

| Concept | Where Used |
|--------|-----------|
| **EKS (Elastic Kubernetes Service)** | Managed Kubernetes control plane on AWS |
| **ECR (Elastic Container Registry)** | Private Docker image registry |
| **Pod** | MySQL runs as a single Pod |
| **Deployment** | Frontend and backend run as Deployments |
| **ClusterIP Service** | Internal communication: frontend→backend, backend→MySQL |
| **LoadBalancer Service** | Exposes frontend to the internet via AWS ELB |
| **IAM Role + Policy** | Node group role given ECR read access |
| **Kubernetes DNS** | Service names used as hostnames (`mysql-svc`, `backend-svc`) |

---

## 📸 Features

- ✅ Register a new student
- ✅ List all students
- ✅ View student details
- ✅ Delete a student record

---

## 👤 Author

Built as a hands-on AWS + Kubernetes project.  
Feel free to fork and extend!

---

## 📝 License

MIT
