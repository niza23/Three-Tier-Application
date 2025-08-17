
# MERN Three-Tier Application on AWS EKS

This project demonstrates how to deploy a **MERN stack (MongoDB, Express/Node.js, React)** three-tier application on **Amazon EKS (Elastic Kubernetes Service)**.  
The deployment includes containerizing the application, pushing images to Amazon ECR, creating an EKS cluster, deploying manifests, and exposing the application with an AWS Application Load Balancer (ALB).


## 📌 Tech Stack
- **Frontend**: React.js (Dockerized, deployed on EKS)
- **Backend**: Node.js/Express (Dockerized, deployed on EKS)
- **Database**: MongoDB (Official Docker image with Persistent Volumes in EKS)
- **Infrastructure**: AWS EC2, EKS, ECR, IAM, ALB
- **Tools**: Docker, eksctl, kubectl, Helm

## 🚀 Steps to Deploy

### 1. Install AWS CLI 
- Install AWS CLI v2
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure
```
* Create an **IAM user** with programmatic access and necessary permissions.
* Configure credentials:

  ```bash
  aws configure
  ```
### 2. Setup EC2 Instance
- Launch an EC2 instance (Ubuntu).
- Install dependencies:
- 
  ```bash
  sudo apt-get update
  sudo apt install docker.io -y
  sudo chown $USER /var/run/docker.sock
  ```

* Verify Docker is installed and running.

---

### 3. Application Source Code & Frontend Setup

* Clone the GitHub repository containing the MERN To-Do List application.
* Navigate to the **frontend** folder and create a `Dockerfile`.
* Build and run the frontend container:

  ```bash
  docker build -t frontend:latest .
  docker run -d -p 3000:3000 frontend:latest
  docker ps
  ```
* Update EC2 security group → add inbound rule for port **3000**.
* Verify frontend in browser:
  `http://<EC2_PUBLIC_IP>:3000`
* Stop container after testing:

  ```bash
  docker kill <container_id>
  ```

---

### 4. Push Images to Amazon ECR

* Authenticate Docker to ECR:

  ```bash
  aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-west-2.amazonaws.com
  ```
* Build, tag, and push images:

  **Frontend**

  ```bash
  docker build -t frontend:latest .
  docker tag frontend:latest <account-id>.dkr.ecr.us-west-2.amazonaws.com/frontend:latest
  docker push <account-id>.dkr.ecr.us-west-2.amazonaws.com/frontend:latest
  ```

  **Backend**

  ```bash
  docker build -t backend:latest .
  docker tag backend:latest <account-id>.dkr.ecr.us-west-2.amazonaws.com/mern-backend:v1
  docker push <account-id>.dkr.ecr.us-west-2.amazonaws.com/mern-backend:v1
  ```

---

### 5. Backend Setup

* Backend code includes `index.js`.

* Create a `Dockerfile` for backend.

* Test locally before pushing:

  ```bash
  docker run -d -p 3500:3500 backend:latest
  ```

* Push backend image to ECR (as above).

---

### 6. Install EKS Tools

* **Install eksctl**:

  ```bash
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
  ```

* **Install kubectl**:

  ```bash
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin
  kubectl version --short --client
  ```

---

### 7. Create EKS Cluster

* Create cluster:

  ```bash
  eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.micro --nodes-min 2 --nodes-max 2
  ```
* Update kubeconfig:

  ```bash
  aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
  ```
* Verify nodes:

  ```bash
  kubectl get nodes
  ```

---

### 8. Kubernetes Manifests

Organize manifests by tier.

* **Database (MongoDB)**:

  * `deployment.yaml`, `secret.yaml`, `service.yaml`, `pv.yaml`, `pvc.yaml`

* **Backend**:

  * `deployment.yaml`, `service.yaml`

* **Frontend**:

  * `deployment.yaml`, `service.yaml`

* **Ingress**:

  * `ingress.yaml`

Create a namespace:

```bash
kubectl create namespace dev
```

Apply manifests:

```bash
kubectl apply -f <manifest>.yaml -n dev
kubectl get all -n dev
```

---

### 9. Setup AWS Load Balancer Controller

This allows ingress routing through AWS ALB.

* Download IAM policy:

  ```bash
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
  ```

* Create IAM policy:

  ```bash
  aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
  ```

* Associate OIDC provider:

  ```bash
  eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=three-tier-cluster --approve
  ```

* Create IAM service account:

  ```bash
  eksctl create iamserviceaccount \
    --cluster=three-tier-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve --region=us-west-2
  ```

---

### 10. Deploy ALB Controller with Helm

* Install Helm:

  ```bash
  sudo snap install helm --classic
  ```

* Add and update repo:

  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  helm repo update eks
  ```

* Install controller:

  ```bash
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=three-tier-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
  ```

* Verify deployment:

  ```bash
  kubectl get deployment -n kube-system aws-load-balancer-controller
  ```

---

### 11. Apply Ingress

* Deploy ingress:

  ```bash
  kubectl apply -f ingress.yaml -n dev
  ```

* Get ingress address:

  ```bash
  kubectl get ingress -n dev
  ```

* Update DNS to map domain/subdomain → ALB endpoint.

---

## ✅ Final Outcome

* MongoDB running inside EKS with persistent storage.
* Backend Node.js API deployed with connection to MongoDB.
* Frontend React app deployed and exposed via AWS ALB.
* Application accessible over the internet using ingress.

---

