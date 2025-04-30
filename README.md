
# 🛠️ Deploy Django App to Azure Kubernetes Service (AKS) using Terraform and Docker

## 🌍 Real-World Scenario

You're a DevOps Engineer tasked with deploying a Django-based real estate web app on Azure. You will:

- Provision infrastructure using Terraform (Linux VM and AKS cluster)
- Set up a development environment with Java, Maven, Docker, Python, and Terraform using a Bash script
- Containerize the Django app
- Deploy the app to AKS

---

## 📁 Project Structure

```
django-aks-deployment/
├── terraform/
│   └── main.tf                      # Terraform file for VM and AKS provisioning
├── setup/
│   └── install-devtools.sh          # Script to install Java, Docker, Python, etc.
├── docker/
│   └── Dockerfile                   # Dockerfile for Django app
├── app/
│   └── themillionestate-django/     # Cloned app directory
├── k8s/
│   └── deployment.yaml              # Kubernetes manifest
└── README.md
```
---

## create_project.py
```python
import os

# Base directory
base_dir = "/home/lilia/VIDEOS/django-aks-deployment"

# File structure with content
files = {
    "terraform/main.tf": '''
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "rg-devops"
  location = "East US"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-cluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aks-devops"

  default_node_pool {
    name       = "default"
    node_count = 1
    vm_size    = "Standard_DS2_v2"
  }

  identity {
    type = "SystemAssigned"
  }

  kubernetes_version = "1.28.3"
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}
''',

    "setup/install-devtools.sh": '''
#!/bin/bash

sudo apt-get update -y
sudo apt-get upgrade -y

# Java
sudo apt install openjdk-17-jdk openjdk-17-jre -y
java --version

# Maven
sudo apt install maven -y
mvn -version

# Docker
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
docker --version

# Python venv and pip
sudo apt install python3.10-venv -y
sudo apt install python3-pip -y

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
sudo chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client

# Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y

# doctl
wget https://github.com/digitalocean/doctl/releases/download/v1.94.0/doctl-1.94.0-linux-amd64.tar.gz
tar xf doctl-1.94.0-linux-amd64.tar.gz
sudo mv doctl /usr/local/bin
''',

    "docker/Dockerfile": '''
FROM python:3.9
ENV PYTHONUNBUFFERED=1
WORKDIR /code
COPY requirements.txt /code/
RUN pip install -r requirements.txt
COPY . /code/
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
''',

    "k8s/deployment.yaml": '''
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      containers:
      - name: django-app
        image: yourdockerhubusername/django-app:v1
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  type: LoadBalancer
  selector:
    app: django-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
'''
}

# Create files with content
for path, content in files.items():
    full_path = os.path.join(base_dir, path)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    with open(full_path, "w") as f:
        f.write(content.strip())

import ace_tools as tools; tools.display_dataframe_to_user(name="Created Project Files", dataframe=None)



```


---

## ⚙️ Step-by-Step Setup Guide

### ✅ 1. Provision a Linux Server with Terraform

- Navigate to `terraform/` and initialize:

```bash
terraform init
terraform apply
```

- This creates a Linux VM and optionally an AKS cluster.

---

### ✅ 2. Create a Linux User

SSH into the VM and run:

```bash
sudo adduser devuser
sudo usermod -aG sudo devuser
su - devuser
```

---

### ✅ 3. Install Dev Tools (Java, Maven, Docker, Python, Terraform, etc.)

Make the script executable and run:

```bash
chmod +x setup/install-devtools.sh
./setup/install-devtools.sh
```

---

### ✅ 4. Run the Django Application

```bash
git clone https://github.com/ashwindibu/themillionestate-django.git
cd themillionestate-django
python3 -m venv .env
source .env/bin/activate
pip install -r requirements.txt
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py createsuperuser
python3 manage.py runserver 0.0.0.0:8000
```

> Access the app at: `http://<your-vm-ip>:8000`

---

## 🐳 Step 5: Dockerize the Application

### Dockerfile

```Dockerfile
FROM python:3.9
ENV PYTHONUNBUFFERED=1
WORKDIR /code
COPY requirements.txt /code/
RUN pip install -r requirements.txt
COPY . /code/
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

### Docker Commands

```bash
docker build -t yourdockerhubusername/django-app:v1 .
docker login
docker push yourdockerhubusername/django-app:v1
```

---

## ☸️ Step 6: Deploy to Azure Kubernetes (AKS)

### Terraform: AKS Resource

Add AKS configuration to `terraform/main.tf` and run:

```bash
terraform apply
```

### Connect to AKS

```bash
az aks get-credentials --resource-group rg-devops --name aks-cluster
```

### Kubernetes Manifest

`k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      containers:
      - name: django-app
        image: yourdockerhubusername/django-app:v1
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  type: LoadBalancer
  selector:
    app: django-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

### Deploy

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get svc
```

> Use the external IP to access your app in the browser.


---

## ✅ Summary

This project automates the deployment of a Django app using:

- Terraform for infrastructure
- Docker for containerization
- Bash script for dev setup
- AKS for container orchestration



---

