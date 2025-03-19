# Static Website Deployment in Kubernetes using WSL2

This guide demonstrates how to deploy a static website using Nginx in Kubernetes, leveraging the WSL2 advantages for seamless container and Kubernetes development.

## Project Overview

We'll create a Kubernetes deployment that serves a static website using Nginx. The project includes:

- Custom Nginx configuration
- Containerized static website
- Kubernetes deployment with multiple replicas
- NodePort service for access

## Prerequisites

- WSL2 with Ubuntu installed
- Docker running in WSL2
- Minikube installed and configured
- kubectl installed

If you haven't set these up, please refer to [Assignment 1](../Assignment%201/README.md) for the initial setup.

## Project Structure

```plaintext
assignment2/
├── nginx-config/
│   └── nginx.conf
├── website/
│   └── [static website files]
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── Dockerfile
```

## Step-by-Step Setup

### 1. Create Project Structure

```bash
# Create project directories
mkdir -p ~/kubernetes/assignment2/{nginx-config,website,k8s}
cd ~/kubernetes/assignment2
```

### 2. Download Website Template

```bash
# Install required tools
sudo apt update
sudo apt install -y wget unzip

# Download and extract template
wget https://www.free-css.com/assets/files/free-css-templates/download/page288/fiu.zip
unzip fiu.zip -d website/
```

### 3. Create Nginx Configuration

```bash
# Create nginx.conf
cat > nginx-config/nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
EOF
```

### 4. Create Dockerfile

```bash
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:alpine

# Remove default nginx static assets
RUN rm -rf /usr/share/nginx/html/*

# Copy the static website
COPY website/ /usr/share/nginx/html/

# Copy nginx configuration
COPY nginx-config/nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80
EOF
```

### 5. Create Kubernetes Manifests

```bash
# Create deployment.yaml
cat > k8s/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-website
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-website
  template:
    metadata:
      labels:
        app: static-website
    spec:
      containers:
      - name: static-website
        image: static-website:v1
        ports:
        - containerPort: 80
EOF

# Create service.yaml
cat > k8s/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: static-website-service
spec:
  type: NodePort
  selector:
    app: static-website
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
EOF
```

### 6. Deploy to Kubernetes

```bash
# Ensure services are running
sudo service docker start
minikube start --driver=docker

# Build and load the image
docker build -t static-website:v1 .
minikube image load static-website:v1

# Deploy to Kubernetes
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Verify deployment
kubectl get pods
kubectl get services
```

### 7. Access the Website

Thanks to WSL2's superior network integration, you can access the website directly:

```bash
# Get Minikube IP
minikube ip

# Access using curl (works directly in WSL)
curl $(minikube ip):30007

# Alternative access method
minikube service static-website-service
```

## WSL2 Advantages in This Project

1. **Direct Container Building**

   - Native Docker support without Windows overhead
   - Faster build times due to native filesystem
   - No need for Windows-specific Docker configuration

2. **Seamless NodePort Access**

   - Direct access to NodePort 30007 without port forwarding
   - No need for additional port mapping like in Windows
   - Native network stack integration

3. **Performance Benefits**
   - Faster file operations for static website files
   - Better container startup times
   - Efficient image building and loading

## Troubleshooting

### Common Issues and Solutions

1. **Docker Service Issues**

   ```bash
   # Restart Docker service
   sudo service docker start
   ```

2. **Permission Problems**

   ```bash
   # Fix Docker permissions
   sudo chmod 666 /var/run/docker.sock
   ```

3. **Pod Status Verification**

   ```bash
   # Check pod status
   kubectl describe pod static-website
   kubectl logs -f <pod-name>
   ```

4. **Website Content Verification**
   ```bash
   # Check if website is accessible
   curl -I $(minikube ip):30007
   ```

### Verifying Deployment

```bash
# Check all resources
kubectl get all -l app=static-website

# Check pod logs
kubectl logs -l app=static-website

# Check service endpoints
kubectl get endpoints static-website-service
```

## The result should look like this!

```bash
huy@DESKTOP-2S1B3SS:~/kubernetes/assignment2$ curl $(minikube ip):30007
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Best Practices

1. Always use WSL's native filesystem for project files
2. Keep Docker images minimal using Alpine-based images
3. Implement proper health checks in Kubernetes deployments
4. Use meaningful labels and selectors
5. Implement proper resource limits in deployment manifests

## Additional Resources

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Kubernetes NodePort Services](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)
- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
