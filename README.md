# Employee Management System - Docker & Kubernetes Setup Guide

## Prerequisites
- Docker installed
- Minikube installed
- kubectl installed

## Project Structure
Create the following directory structure:
```
employee-app/
├── database/
│   ├── Dockerfile          # Place this in database folder
│   └── init.sql
├── frontend/
│   ├── Dockerfile          # Place this in frontend folder
│   ├── app.py
│   ├── requirements.txt
│   └── templates/
│       └── index.html
└── kubernetes/
    ├── mysql-deployment.yaml
    ├── mysql-service.yaml
    ├── frontend-deployment.yaml
    └── frontend-service.yaml
```

**Important**: Each Dockerfile must be in its respective folder (database/Dockerfile and frontend/Dockerfile) so the COPY commands can find the files.

## Step 1: Build Docker Images

### 1.1 Build MySQL Docker Image
```bash
cd employee-app/database
docker build -t employee-mysql:latest .
```

### 1.2 Build Frontend Docker Image
```bash
cd ../frontend
docker build -t employee-frontend:latest .
```

### 1.3 Verify Images
```bash
docker images | grep employee
```

## Step 2: Start Minikube

```bash
# Start minikube
minikube start

# Verify minikube is running
minikube status
```

## Step 3: Build and Load Images into Minikube

### For Linux/Mac:
```bash
# Use minikube's Docker daemon (important!)
eval $(minikube docker-env)

# Rebuild MySQL image
cd employee-app/database
docker build -t employee-mysql:latest .

# Rebuild Frontend image
cd ../frontend
docker build -t employee-frontend:latest .

# Verify images are in minikube
docker images | grep employee
```

### For Windows (PowerShell):
```powershell
# Method 1: Use Minikube's Docker daemon
minikube docker-env --shell powershell | Invoke-Expression

# Rebuild MySQL image
cd employee-app\database
docker build -t employee-mysql:latest .

# Rebuild Frontend image
cd ..\frontend
docker build -t employee-frontend:latest .

# Verify images are in minikube
docker images | findstr employee
```

### For Windows (Alternative - Recommended):
```powershell
# Build images locally first
cd employee-app\database
docker build -t employee-mysql:latest .

cd ..\frontend
docker build -t employee-frontend:latest .

# Load images into Minikube
minikube image load employee-mysql:latest
minikube image load employee-frontend:latest

# Verify images are in Minikube
minikube image ls | findstr employee
```

### For Windows (Command Prompt):
```cmd
# Use Minikube's Docker daemon
@FOR /f "tokens=*" %i IN ('minikube docker-env --shell cmd') DO @%i

# Rebuild MySQL image
cd employee-app\database
docker build -t employee-mysql:latest .

# Rebuild Frontend image
cd ..\frontend
docker build -t employee-frontend:latest .

# Verify images
docker images | findstr employee
```

## Step 4: Deploy to Kubernetes

### 4.1 Deploy MySQL
```bash
cd ../kubernetes

# Create MySQL deployment and service
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml

# Verify MySQL pod is running (wait until STATUS is Running)
kubectl get pods -l app=mysql
kubectl get services -l app=mysql
```

### 4.2 Wait for MySQL to be Ready
```bash
# Watch the pod until it's running
kubectl get pods -w

# Check logs if needed
kubectl logs -l app=mysql
```

### 4.3 Deploy Frontend
```bash
# Create frontend deployment and service
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

# Verify frontend pods are running
kubectl get pods -l app=frontend
kubectl get services -l app=frontend
```

## Step 5: Access the Application

### 5.1 Get Minikube IP
```bash
minikube ip
```

### 5.2 Access the Application
Open your browser and navigate to:
```
http://<minikube-ip>:30080
```

For example, if minikube IP is `192.168.49.2`:
```
http://192.168.49.2:30080
```

### 5.3 Alternative: Use Port Forwarding
If NodePort doesn't work, use port forwarding:
```bash
kubectl port-forward service/frontend-service 5000:5000
```
Then access: `http://localhost:5000`

## Step 6: Monitoring and Troubleshooting

### View All Resources
```bash
kubectl get all
```

### Check Pod Status
```bash
kubectl get pods
kubectl describe pod <pod-name>
```

### View Logs
```bash
# Frontend logs
kubectl logs -l app=frontend --tail=50 -f

# MySQL logs
kubectl logs -l app=mysql --tail=50 -f
```

### Check Services
```bash
kubectl get services
kubectl describe service frontend-service
kubectl describe service mysql-service
```

### Test Database Connection
```bash
# Get MySQL pod name
kubectl get pods -l app=mysql

# Connect to MySQL pod
kubectl exec -it <mysql-pod-name> -- mysql -u appuser -papppassword employee_db

# Inside MySQL, run:
SHOW TABLES;
SELECT * FROM employees;
EXIT;
```

## Step 7: Testing the Application

1. **Add Employee**: Fill in the form and click "Add Employee"
2. **View Employees**: All employees are displayed in the table
3. **Delete Employee**: Click the "Delete" button next to any employee

## Step 8: Scale the Application

### Scale Frontend Pods
```bash
# Scale to 3 replicas
kubectl scale deployment frontend-deployment --replicas=3

# Verify scaling
kubectl get pods -l app=frontend
```

### View Pod Distribution
```bash
kubectl get pods -o wide
```

## Step 9: Update Application

If you make changes to the code:

```bash
# Rebuild image in minikube context
eval $(minikube docker-env)
cd frontend
docker build -t employee-frontend:latest .

# Restart deployment
kubectl rollout restart deployment/frontend-deployment

# Check rollout status
kubectl rollout status deployment/frontend-deployment
```

## Step 10: Cleanup

### Delete All Resources
```bash
cd kubernetes
kubectl delete -f frontend-service.yaml
kubectl delete -f frontend-deployment.yaml
kubectl delete -f mysql-service.yaml
kubectl delete -f mysql-deployment.yaml
```

### Or Delete by Label
```bash
kubectl delete all -l app=frontend
kubectl delete all -l app=mysql
```

### Stop Minikube
```bash
minikube stop
```

### Delete Minikube Cluster
```bash
minikube delete
```

## Common Issues and Solutions

### Issue 1: ImagePullBackOff
**Solution**: Make sure you're using minikube's Docker daemon:
```bash
eval $(minikube docker-env)
# Then rebuild images
```

### Issue 2: Frontend Can't Connect to Database
**Solution**: 
- Check MySQL pod is running: `kubectl get pods -l app=mysql`
- Check service exists: `kubectl get svc mysql-service`
- Check frontend logs: `kubectl logs -l app=frontend`

### Issue 3: Can't Access Application
**Solution**:
```bash
# Get minikube IP
minikube ip

# Check service
kubectl get svc frontend-service

# Try port forwarding instead
kubectl port-forward service/frontend-service 5000:5000
```

### Issue 4: MySQL Pod Keeps Restarting
**Solution**: Check logs for errors:
```bash
kubectl logs -l app=mysql --previous
```

## Additional Commands

### Open Minikube Dashboard
```bash
minikube dashboard
```

### SSH into Minikube
```bash
minikube ssh
```

### View Cluster Info
```bash
kubectl cluster-info
```

### Get Node Information
```bash
kubectl get nodes
kubectl describe node minikube
```

## Architecture Overview

```
User Browser
    ↓
Frontend Service (NodePort :30080)
    ↓
Frontend Deployment (2 replicas)
    ↓
MySQL Service (ClusterIP :3306)
    ↓
MySQL Deployment (1 replica)
    ↓
MySQL Database
```

## Notes

- **Pods vs Nodes**: In this setup, we're creating 2 pods (one for MySQL, two for frontend). In Minikube, there's typically one node, but multiple pods can run on it.
- **Services**: Services provide stable network endpoints for pods
- **imagePullPolicy: Never**: Tells Kubernetes to use locally built images
- **NodePort**: Exposes the frontend service on a static port (30080)
- **ClusterIP**: MySQL service is only accessible within the cluster

## Next Steps

1. **Persistent Storage**: Add PersistentVolume for MySQL data
2. **ConfigMaps**: Move configuration to ConfigMaps
3. **Secrets**: Store passwords in Kubernetes Secrets
4. **Ingress**: Set up Ingress for better routing
5. **Monitoring**: Add Prometheus and Grafana
6. **CI/CD**: Integrate with Jenkins or GitHub Actions

Happy Learning! 🚀
