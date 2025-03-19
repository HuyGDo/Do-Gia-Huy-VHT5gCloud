# Nginx Proxy with Multiple Static Websites in Kubernetes

This guide demonstrates how to deploy multiple static websites with an Nginx proxy in Kubernetes, building upon Assignment 2.

## Project Overview

We'll create:

- Two static website deployments (web1 and web2)
- An Nginx proxy to route traffic based on paths
- Three NodePort services (web1, web2, and nginx-proxy)

## Project Structure

First, install tree to view the structure:

```bash
sudo apt install tree
```

The project structure will look like this:

```plaintext
assignment3/
├── web1/
│   ├── website/
│   ├── nginx-config/
│   │   └── nginx.conf
│   └── Dockerfile
├── web2/
│   ├── website/
│   ├── nginx-config/
│   │   └── nginx.conf
│   └── Dockerfile
├── nginx-proxy/
│   ├── nginx-config/
│   │   └── nginx.conf
│   └── Dockerfile
└── k8s/
    ├── web1-deployment.yaml
    ├── web1-service.yaml
    ├── web2-deployment.yaml
    ├── web2-service.yaml
    ├── nginx-proxy-deployment.yaml
    └── nginx-proxy-service.yaml
```

## Step-by-Step Setup

### 1. Create Project Structure

```bash
# Create project directories
mkdir -p ~/kubernetes/assignment3/{web1,web2,nginx-proxy,k8s}
mkdir -p ~/kubernetes/assignment3/web1/{website,nginx-config}
mkdir -p ~/kubernetes/assignment3/web2/{website,nginx-config}
mkdir -p ~/kubernetes/assignment3/nginx-proxy/nginx-config
cd ~/kubernetes/assignment3
```

### 2. Set Up Web1 (From Assignment 2)

```bash
# Copy files from Assignment 2
cp -r ~/kubernetes/assignment2/website/* web1/website/
cp ~/kubernetes/assignment2/nginx-config/nginx.conf web1/nginx-config/
cp ~/kubernetes/assignment2/Dockerfile web1/
```

### 3. Set Up Web2 (New Website)

```bash
# Download different template for web2
cd web2
wget https://www.free-css.com/assets/files/free-css-templates/download/page296/neogym.zip
unzip neogym.zip -d website/
```

Create Nginx config for web2 with proper permissions and directory listing:

```bash
cat > nginx-config/nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

    # Remove any restrictions and enable directory listing
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
        autoindex on;  # Enable directory listing
    }
}
EOF
```

Similarly, update web1's nginx configuration:

```bash
cat > web1/nginx-config/nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

    # Remove any restrictions and enable directory listing
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
        autoindex on;  # Enable directory listing
    }
}
EOF
```

Create Dockerfile for web2:

```bash
cat > Dockerfile << 'EOF'
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY website/ /usr/share/nginx/html/
COPY nginx-config/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
EOF
```

### 4. Set Up Nginx Proxy

Create Nginx proxy configuration with proper path rewriting:

```bash
cat > nginx-proxy/nginx-config/nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

    # Web1 Configuration
    location /web1/ {
        # Rewrite the URL to remove /web1/ before proxying
        rewrite ^/web1/(.*) /$1 break;
        proxy_pass http://web1-service;

        # Enhanced proxy headers for better forwarding
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Web2 Configuration
    location /web2/ {
        # Rewrite the URL to remove /web2/ before proxying
        rewrite ^/web2/(.*) /$1 break;
        proxy_pass http://web2-service;

        # Enhanced proxy headers for better forwarding
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

Key Changes in Nginx Configurations:

1. **Web1 and Web2 Configurations**:

   - Added `autoindex on` to enable directory listing
   - Simplified location block to remove restrictions
   - Explicit root and index directives
   - Added `try_files` directive for better file serving

2. **Nginx Proxy Configuration**:
   - Added URL rewriting to strip /web1/ and /web2/ prefixes
   - Enhanced proxy headers for better request forwarding
   - Removed trailing slash from proxy_pass to prevent double slashes
   - Added X-Forwarded-\* headers for proper protocol handling

### 5. Create Kubernetes Manifests

Create web1 deployment and service:

```bash
cat > k8s/web1-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web1
  template:
    metadata:
      labels:
        app: web1
    spec:
      containers:
      - name: web1
        image: web1:v1
        ports:
        - containerPort: 80
EOF

cat > k8s/web1-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web1-service
spec:
  selector:
    app: web1
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

Create web2 deployment and service:

```bash
cat > k8s/web2-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web2
  template:
    metadata:
      labels:
        app: web2
    spec:
      containers:
      - name: web2
        image: web2:v1
        ports:
        - containerPort: 80
EOF

cat > k8s/web2-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web2-service
spec:
  selector:
    app: web2
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

Create nginx-proxy deployment and service:

```bash
cat > k8s/nginx-proxy-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-proxy
  template:
    metadata:
      labels:
        app: nginx-proxy
    spec:
      containers:
      - name: nginx-proxy
        image: nginx-proxy:v1
        ports:
        - containerPort: 80
EOF

cat > k8s/nginx-proxy-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-proxy
spec:
  type: NodePort
  selector:
    app: nginx-proxy
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30008
EOF
```

### 6. Deploy to Kubernetes

```bash
# Ensure services are running
sudo service docker start
minikube start --driver=docker

# Build and load images
cd ~/kubernetes/assignment3

# Build web1
cd web1
docker build -t web1:v1 .
minikube image load web1:v1

# Build web2
cd ../web2
docker build -t web2:v1 .
minikube image load web2:v1

# Build nginx-proxy
cd ../nginx-proxy
docker build -t nginx-proxy:v1 .
minikube image load nginx-proxy:v1

# Deploy everything
cd ..
kubectl apply -f k8s/

# Verify deployments
kubectl get pods
kubectl get services
```

### 7. Test the Setup

```bash
# Get Minikube IP
export MINIKUBE_IP=$(minikube ip)

# Test web1
curl http://$MINIKUBE_IP:30008/web1/

# Test web2
curl http://$MINIKUBE_IP:30008/web2/
```

## Troubleshooting

### Common Issues

1. **Service Discovery Issues**

   ```bash
   # Check DNS resolution
   kubectl exec -it nginx-proxy-<pod-id> -- nslookup web1-service
   kubectl exec -it nginx-proxy-<pod-id> -- nslookup web2-service
   ```

2. **Proxy Configuration Issues**

   ```bash
   # Check nginx-proxy logs
   kubectl logs -l app=nginx-proxy

   # Check nginx configuration
   kubectl exec -it nginx-proxy-<pod-id> -- nginx -t
   ```

3. **Network Issues**

   ```bash
   # Test connectivity from proxy to backends
   kubectl exec -it nginx-proxy-<pod-id> -- wget -qO- http://web1-service
   kubectl exec -it nginx-proxy-<pod-id> -- wget -qO- http://web2-service
   ```

4. **403 Forbidden Errors**

   ```bash
   # Check file permissions in containers
   kubectl exec -it $(kubectl get pod -l app=web1 -o jsonpath='{.items[0].metadata.name}') -- ls -la /usr/share/nginx/html/
   kubectl exec -it $(kubectl get pod -l app=web2 -o jsonpath='{.items[0].metadata.name}') -- ls -la /usr/share/nginx/html/

   # Verify nginx configurations
   kubectl exec -it $(kubectl get pod -l app=nginx-proxy -o jsonpath='{.items[0].metadata.name}') -- nginx -T
   ```

5. **Path Resolution Issues**
   ```bash
   # Test internal service access
   kubectl exec -it $(kubectl get pod -l app=nginx-proxy -o jsonpath='{.items[0].metadata.name}') -- curl http://web1-service/
   kubectl exec -it $(kubectl get pod -l app=nginx-proxy -o jsonpath='{.items[0].metadata.name}') -- curl http://web2-service/
   ```

## Understanding the 403 Forbidden Issue and Solution

### The Problem

Initially, when accessing the websites through the nginx-proxy using:

```bash
curl http://$MINIKUBE_IP:30008/web1/
curl http://$MINIKUBE_IP:30008/web2/
```

We encountered 403 Forbidden errors. This happened due to several nginx configuration issues:

1. **Path Rewriting Issue**:

   - The original proxy configuration used `proxy_pass http://web1-service/;`
   - The trailing slash caused nginx to append the full request path after the URL
   - This resulted in requests like `http://web1-service//web1/index.html`
   - The double slash and incorrect path caused nginx to fail finding the files

2. **Directory Access Permissions**:

   - Default nginx configuration is restrictive about directory listings
   - When proxying with path prefixes (/web1/, /web2/), the original paths weren't properly mapped
   - Nginx couldn't find the exact files due to path mismatches

3. **Request Header Problems**:
   - Insufficient proxy headers being passed to backend services
   - Host header conflicts between proxy and backend services

### The Solution

We made several key changes to fix these issues:

1. **In Web1 and Web2 Configurations**:

```nginx
location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;
    autoindex on;  # Enable directory listing
}
```

- Added `autoindex on` to allow directory listing
- Used `try_files` to properly handle path resolution
- Simplified location block to remove restrictions
- Explicit root and index directives for clear path handling

2. **In Nginx Proxy Configuration**:

```nginx
location /web1/ {
    rewrite ^/web1/(.*) /$1 break;  # Remove prefix before proxying
    proxy_pass http://web1-service;  # No trailing slash

    # Enhanced proxy headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

- Added URL rewriting to properly handle path prefixes
- Removed trailing slash in proxy_pass to prevent path issues
- Added comprehensive proxy headers for proper request forwarding

### Why It Works Now

1. **Path Handling**:

   - The `rewrite` rule strips /web1/ or /web2/ from the URL
   - Example: `/web1/index.html` → `/index.html`
   - Backend services receive clean paths without prefixes

2. **Directory Access**:

   - `autoindex on` allows directory listing when needed
   - `try_files` directive provides fallback options
   - Proper root directory configuration ensures files are found

3. **Request Forwarding**:

   - Complete set of proxy headers ensures proper request context
   - Correct Host header handling prevents routing issues
   - X-Forwarded headers maintain client information

4. **Clean URLs**:
   - No double slashes in paths
   - Proper path rewriting at the proxy level
   - Consistent URL structure throughout the proxy chain

This configuration ensures that requests are properly rewritten and forwarded, maintaining clean URLs while providing necessary access permissions and proper proxy headers.

## Successfully deployed!

```bash
huy@DESKTOP-2S1B3SS:~/kubernetes/assignment3$ export MINIKUBE_IP=$(minikube ip)
curl -v http://$MINIKUBE_IP:30008/web1/
curl -v http://$MINIKUBE_IP:30008/web2/
*   Trying 192.168.49.2:30008...
* Connected to 192.168.49.2 (192.168.49.2) port 30008
> GET /web1/ HTTP/1.1
> Host: 192.168.49.2:30008
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
<head><title>Index of /</title></head>
<body>
<h1>Index of /</h1><hr><pre><a href="../">../</a>
<a href="html/">html/</a>                                              19-Mar-2025 17:59  <head><title>Index of /</title></head>
<body>
<h1>Index of /</h1><hr><pre><a href="../">../</a>
<a href="html/">html/</a>                                              19-Mar-2025 17:59                   -
</pre><hr></body>
</html>
* Connection #0 to host 192.168.49.2 left intact
*   Trying 192.168.49.2:30008...
* Connected to 192.168.49.2 (192.168.49.2) port 30008
> GET /web2/ HTTP/1.1
> Host: 192.168.49.2:30008
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.27.4
< Date: Wed, 19 Mar 2025 18:14:42 GMT
< Content-Type: text/html
< Transfer-Encoding: chunked
< Connection: keep-alive
<
<html>
<head><title>Index of /</title></head>
<body>
<h1>Index of /</h1><hr><pre><a href="../">../</a>
<a href="neogym-html/">neogym-html/</a>                                       15-Sep-2020 10:23                   -
</pre><hr></body>
</html>
```

## Additional Resources

- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Kubernetes Service Discovery](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Nginx Location Directive](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)
