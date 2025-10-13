# Flask Hostname Display App - Kubernetes Deployment Guide

## Overview

This project demonstrates how to deploy a simple Flask web application in a Kubernetes environment. The application displays the hostname of the container it's running in, showcasing container orchestration concepts and Jinja2 templating.

**Learning Objectives:**
- Build and containerize a Python Flask application
- Deploy applications to Kubernetes using manifests
- Understand services and external access via NodePort
- Practice templating with Jinja2

**Components:**
- `Dockerfile` - Container image definition
- `app.py` - Flask application code
- `templates/index.html` - Jinja2 HTML template
- `kubernetes/flask-hostname.yaml` - Kubernetes deployment and service manifests

## Prerequisites

Before starting, ensure you have:

- **Docker** installed and running
- **Kubernetes cluster** available (Minikube, kind, or cloud cluster)
- **kubectl** configured to communicate with your cluster
- **Container registry access** (Docker Hub, GitHub Container Registry, etc.)
- **Basic terminal/command-line knowledge**

### Verify Prerequisites

```bash
# Check Docker
docker --version

# Check kubectl
kubectl version --client

# Check cluster connection
kubectl get nodes

# For Minikube users
minikube status
```

## Project Structure

Create the following directory structure:

```
flask-hostname-app/
├── Dockerfile
├── app.py
├── templates/
│   └── index.html
└── kubernetes/
    └── flask-hostname.yaml
```

## Application Files

### Dockerfile

Create a `Dockerfile` in the project root:

```dockerfile
# Use a lightweight official Python image
FROM python:3.11-slim

# Set working directory inside the container
WORKDIR /app

# Install Flask (no requirements.txt needed for this simple example)
RUN pip install --no-cache-dir flask

# Copy application code and templates
COPY app.py /app/
COPY templates /app/templates

# Expose the port Flask will run on
EXPOSE 5000

# Run the Flask application
CMD ["python", "app.py"]
```

### Flask Application (app.py)

Create `app.py` in the project root:

```python
from flask import Flask, render_template
import socket

# Create Flask application instance
app = Flask(__name__)

@app.route('/')
def welcome():
    # Get the hostname of the container/machine
    hostname = socket.gethostname()
    # Render the template with the hostname variable
    return render_template('index.html', hostname=hostname)

if __name__ == '__main__':
    # Run the app on all interfaces (0.0.0.0) so it's accessible from outside the container
    app.run(host='0.0.0.0', port=5000)
```

### HTML Template (templates/index.html)

Create the `templates` directory and `index.html` file:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Flask Hostname Display</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin-top: 50px;
            background-color: #f0f0f0;
        }
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            display: inline-block;
        }
        .hostname {
            color: #2196F3;
            font-weight: bold;
            font-size: 1.2em;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Welcome to the Flask App!</h1>
        <p>This application is running in a Kubernetes container.</p>
        <p>Container hostname: <span class="hostname">{{ hostname }}</span></p>
        <p><em>Each time you refresh, you might see the same or different hostname depending on your Kubernetes setup!</em></p>
    </div>
</body>
</html>
```

## Kubernetes Manifests

Create the `kubernetes` directory and `flask-hostname.yaml` file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-hostname-app
  labels:
    app: flask-hostname-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-hostname-app
  template:
    metadata:
      labels:
        app: flask-hostname-app
    spec:
      containers:
      - name: flask-hostname-app
        image: your-docker-namespace/flask-hostname-app:latest
        ports:
        - containerPort: 5000
        env:
        - name: FLASK_ENV
          value: "production"
---
apiVersion: v1
kind: Service
metadata:
  name: flask-hostname-service
  labels:
    app: flask-hostname-app
spec:
  type: NodePort
  selector:
    app: flask-hostname-app
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
    nodePort: 30036
```

## Step-by-Step Deployment Instructions

### Step 1: Build the Docker Image

Navigate to your project directory and build the Docker image:

```bash
# Navigate to project directory
cd flask-hostname-app

# Build the Docker image (replace 'your-docker-namespace' with your actual registry/username)
docker build -t your-docker-namespace/flask-hostname-app:latest .

# Verify the image was created
docker images | grep flask-hostname-app
```

### Step 2: Test Locally (Optional)

Before deploying to Kubernetes, test the container locally:

```bash
# Run the container locally
docker run -p 5000:5000 your-docker-namespace/flask-hostname-app:latest

# Open another terminal and test
curl http://localhost:5000
# Or open http://localhost:5000 in your browser

# Stop the container with Ctrl+C
```

### Step 3: Push to Container Registry

Push your image to a container registry so Kubernetes can access it:

```bash
# Log in to your container registry (e.g., Docker Hub)
docker login

# Push the image
docker push your-docker-namespace/flask-hostname-app:latest
```

### Step 4: Update Kubernetes Manifests

Edit `kubernetes/flask-hostname.yaml` and replace `your-docker-namespace` with your actual registry path:

```yaml
# Change this line:
image: your-docker-namespace/flask-hostname-app:latest
# To something like:
image: dockerhub-username/flask-hostname-app:latest
```

### Step 5: Deploy to Kubernetes

Apply the Kubernetes manifests:

```bash
# Apply the deployment and service
kubectl apply -f kubernetes/flask-hostname.yaml

# Verify deployment
kubectl get deployments
kubectl get services
kubectl get pods
```

### Step 6: Access the Application

Get the service details to find how to access your app:

```bash
# Get service information
kubectl get svc flask-hostname-service

# For Minikube users - get the Minikube IP
minikube ip
```

**Accessing the Application:**

- **Minikube:** `http://<minikube-ip>:30036`
- **Other Kubernetes clusters:** `http://<any-node-ip>:30036`

## Validation Steps

1. **Check Pod Status:**
   ```bash
   kubectl get pods
   # Should show STATUS: Running
   ```

2. **View Application Logs:**
   ```bash
   kubectl logs deployment/flask-hostname-app
   # Should show Flask development server running
   ```

3. **Test the Application:**
   - Open the URL in your browser
   - You should see a welcome page with the container hostname displayed
   - The hostname should match one of your pod names (check with `kubectl get pods`)

## Troubleshooting

### Common Issues and Solutions

**Pod not starting:**
```bash
# Check pod details
kubectl describe pod <pod-name>

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

**Image pull errors:**
- Verify your image name is correct in the YAML
- Ensure the image exists in your registry
- Check if your cluster can access the registry

**Cannot access the application:**
```bash
# Check if the service is running
kubectl get svc

# For Minikube, ensure it's running
minikube status

# Alternative: Use port-forwarding for testing
kubectl port-forward service/flask-hostname-service 8080:5000
# Then access http://localhost:8080
```

**Application errors:**
```bash
# Check application logs
kubectl logs deployment/flask-hostname-app

# Execute commands inside the pod
kubectl exec -it <pod-name> -- /bin/bash
```

## Understanding the Setup

**Why NodePort?**
NodePort is used in this educational example because it's simple and doesn't require additional infrastructure like load balancers. It exposes the service on a static port (30036) on each node's IP address.

**Why this hostname approach?**
Using `socket.gethostname()` demonstrates how containerized applications can access system information, and how this changes in different environments (local Docker vs. Kubernetes pods).

**Jinja2 Templating:**
Flask uses Jinja2 by default. The `{{ hostname }}` syntax in the HTML template gets replaced with the actual hostname value passed from the Flask route.

## Cleanup

To remove all resources created by this project:

```bash
# Delete the deployment and service
kubectl delete -f kubernetes/flask-hostname.yaml

# Remove local Docker image (optional)
docker rmi your-docker-namespace/flask-hostname-app:latest
```

## Extensions and Learning Opportunities

**For advanced students:**
1. **Scale the deployment:** Change `replicas: 1` to `replicas: 3` and observe multiple hostnames
2. **Add health checks:** Implement liveness and readiness probes
3. **Use ConfigMaps:** Store configuration separately from the image
4. **Add persistent storage:** Mount volumes for data persistence
5. **Implement Ingress:** Replace NodePort with Ingress for more realistic routing

**Additional commands to explore:**
```bash
# Scale the deployment
kubectl scale deployment flask-hostname-app --replicas=3

# Watch pods in real-time
kubectl get pods -w

# Port forward for local testing
kubectl port-forward deployment/flask-hostname-app 8080:5000
```

## Appendix

### Quick Reference Commands

```bash
# Build and push
docker build -t your-namespace/flask-hostname-app:latest .
docker push your-namespace/flask-hostname-app:latest

# Deploy
kubectl apply -f kubernetes/flask-hostname.yaml

# Check status
kubectl get all

# Access (Minikube)
minikube ip  # Use this IP with port 30036

# Cleanup
kubectl delete -f kubernetes/flask-hostname.yaml
```

### Alternative: ConfigMap Approach

If you prefer not to build a custom image, you can use a ConfigMap to inject the code:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-app-code
data:
  app.py: |
    from flask import Flask, render_template
    import socket
    app = Flask(__name__)
    @app.route('/')
    def welcome():
        hostname = socket.gethostname()
        return render_template('index.html', hostname=hostname)
    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=5000)
  index.html: |
    <!DOCTYPE html>
    <html><head><title>Flask App</title></head>
    <body><h1>Welcome!</h1><p>Hostname: <strong>{{ hostname }}</strong></p></body></html>
```

Then modify the deployment to use `python:3.11-slim` image and mount the ConfigMap.

---

**Happy Learning! 🚀**