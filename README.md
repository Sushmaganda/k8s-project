# k8s-project

🔹 Project Components


1. Backend (Python API)

**Dockerfile (backend/Dockerfile):**


FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]


**requirements.txt:**

flask==2.2.5
pytest==7.4.4


**app.py:**

from flask import Flask
app = Flask(__name__)

@app.route("/api")
def hello():
    return {"message": "Hello from Backend!"}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

**2. Frontend (Nginx)**

Dockerfile (frontend/Dockerfile):
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html


**index.html:**

<!DOCTYPE html>
<html>
<head><title>Frontend</title></head>
<body>
  <h1>Hello from Frontend!</h1>
</body>
</html>



🔹 Kubernetes Manifests

**Backend Deployment (k8s/backend-deploy.yaml)**

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: backend:latest
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000


**Frontend Deployment (k8s/frontend-deploy.yaml)**

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: frontend:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort



🔹 Scaling & Rolling Updates
**- Scale replicas:**
  
kubectl scale deployment backend-deployment --replicas=5


**- Rolling update: Update image version in YAML:**

containers:
- name: backend
  image: backend:v2

**Apply changes:**

kubectl apply -f k8s/backend-deploy.yaml
kubectl rollout status deployment backend-deployment


🔹 GitHub Actions Pipeline
**.github/workflows/deploy.yaml**

name: CI/CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Build Backend Image
      run: docker build -t backend:latest ./backend

    - name: Build Frontend Image
      run: docker build -t frontend:latest ./frontend

    - name: Save Images
      run: |
        docker save backend:latest -o backend.tar
        docker save frontend:latest -o frontend.tar
    - name: Upload Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: docker-images
        path: "*.tar"

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
    - name: Download Artifacts
      uses: actions/download-artifact@v3
      with:
        name: docker-images

    - name: Load Images
      run: |
        docker load -i backend.tar
        docker load -i frontend.tar

    - name: Run Backend Tests
      run: docker run --rm backend:latest pytest -v

    - name: Run Frontend Smoke Test
      run: docker run --rm frontend:latest curl -f http://localhost || exit 1

  deploy:
    runs-on: ubuntu-latest
    needs: test
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Set up Kind Cluster
      uses: helm/kind-action@v1.8.0

    - name: Load Images into Kind
      run: |
        kind load docker-image backend:latest
        kind load docker-image frontend:latest

    - name: Apply Kubernetes Manifests
      run: |
        kubectl apply -f k8s/backend-deploy.yaml
        kubectl apply -f k8s/frontend-deploy.yaml

    - name: Verify Deployment
      run: kubectl rollout status deployment backend-deployment

  rollback:
    runs-on: ubuntu-latest
    needs: deploy
    if: failure()
    steps:
    - name: Rollback Backend
      run: kubectl rollout undo deployment backend-deployment
    - name: Rollback Frontend
      run: kubectl rollout undo deployment frontend-deployment
      

🔹 Pipeline Flow
- Build → Docker images created and stored as artifacts.
- Test → Unit tests (backend) + smoke tests (frontend).
- Deploy → Apply manifests to Kind cluster.
- Verify → Ensure rollout success.
- Rollback → Automatic undo if deployment fails.


🔹 Interview Explanation Structure
1. Project Overview
“I built a full-stack project using Docker and Kubernetes (Kind cluster). The system has two components:

- A frontend served by Nginx.
- A backend Python API built with Flask.
Both are containerized with Docker and deployed on Kubernetes.”

2. Dockerization
“I wrote Dockerfiles for both services.

- The backend Dockerfile installs dependencies from requirements.txt and runs app.py.
- The frontend Dockerfile uses Nginx and serves a static HTML page.
This ensures reproducible builds and portable containers.”

3. Kubernetes Deployment
“I created Kubernetes manifests for each service:

- Deployment objects define replicas and container specs.
- Service objects expose the pods internally and extern
- For example, the backend runs on port 5000, and the frontend is exposed via NodePort on port 80.”

4. Scaling & Rolling Updates
“To demonstrate Kubernetes features:

- I scaled the backend deployment from 2 to 5 replicas using kubectl scale.
- I performed a rolling update by changing the image version in the deployment YAML and applying it.
Kubernetes replaced pods gradually, ensuring zero downtime. If something failed, I used kubectl rollout undo to rollback.”

5. CI/CD with GitHub Actions
“I automated the workflow with GitHub Actions:

- Build stage: Builds Docker images for frontend and backend.
- Test stage: Runs backend unit tests with pytest and a frontend smoke test with curl.
- Push stage: Pushes tested images to GitHub Container Registry.
- Deploy stage: Applies Kubernetes manifests to a - Kind cluster, verifies rollout status.
- Rollback stage: If deployment fails, it automatically reverts to the last stable version.”

6. Multi-Environment Setup
“To mimic enterprise workflows, I extended the pipeline:

- Deploy first to staging.
- Run integration tests.
- Require manual approval before promoting to production.
This ensures stability and controlled releases.”

7. Key Takeaways
“This project demonstrates:

- Containerization with Docker.
- Orchestration with Kubernetes (scaling, rolling updates, rollback).
- End-to-end CI/CD automation with GitHub Actions.
- Multi-environment deployment with approvals.
It shows how I can build reliable, reproducible, and production-ready pipelines.”

🔹 Interview Tip
When explaining, emphasize why you did each step:
- Docker → portability.
- Kubernetes → scalability & resilience.
- GitHub Actions → automation & consistency.
- Multi-env → enterprise-grade safety.

