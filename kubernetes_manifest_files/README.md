Created **Kubernetes manifests** for each tier of the MERN app.

#### 🗄️ Database (MongoDB)

- **Deployment** → runs MongoDB pod using the official image (`mongo:4.4.6`).
- **PersistentVolume & PVC** → stores MongoDB data so it persists across restarts.
- **Secret** → stores MongoDB credentials (username & password).
- **Service** (`mongodb-svc`) → exposes MongoDB internally so the backend can connect.

#### ⚙️ Backend (Node.js API)

- **Deployment** → runs 2 replicas of backend pods using the **custom ECR image** (`mern-backend:v1`).
- **Environment variables** → connect to MongoDB using the service `mongodb-svc` and credentials from Kubernetes secrets.
- **Service** (`api`) → exposes the backend inside the cluster.
- **Probes** → health checks with `/ok` endpoint to ensure pod readiness.

#### 🎨 Frontend (React)

- **Deployment** → runs the frontend pod using the **custom ECR image** (`frontend:latest`).
- **Environment variable** → tells React app the backend API URL (`REACT_APP_BACKEND_URL`).
- **Service** (`frontend`) → exposes the React app inside the cluster.

#### 🌐 Ingress (Application Load Balancer)

- **Ingress** → provisions an AWS ALB and routes traffic:
  - `http://YOUR_URL.study/api/*` → routes to **backend service**.
  - `http://YOUR_URL.study/` → routes to **frontend service**.
- DNS record updated to point domain → ALB endpoint.

### 🔑 Important Point — Where Your Custom Images Are Used

- **MongoDB** uses the **official public image** (`mongo:4.4.6`).
- **Backend (Node.js API)** → uses **your custom ECR image** (`mern-backend:v1`).
- **Frontend (React)** → uses **your custom ECR image** (`frontend:latest`).

So you have **two container images you built and pushed**, and they are used in the backend and frontend deployments. MongoDB runs directly from the official Docker Hub image.

---
