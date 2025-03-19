# Kubernetes Development Environment with WSL2

## Understanding WSL2 for Kubernetes Development

### Why WSL2?

WSL2 (Windows Subsystem for Linux 2) represents a fundamental shift in how Windows integrates with Linux. Here's a deep dive into why it's the superior choice for Kubernetes development:

1. **Architecture Benefits**

   - WSL2 uses a real Linux kernel, unlike WSL1's translation layer
   - Runs in a lightweight VM with direct memory access
   - Provides true Linux system calls, essential for container operations
   - Minimal overhead compared to traditional VMs or Docker Desktop

2. **Networking Architecture**

   - WSL2 implements a virtual network adapter
   - Direct network interface access from Linux subsystem
   - No NAT or port forwarding required for most operations
   - Native support for Linux networking stack

3. **Resource Management**
   - Dynamic memory allocation (only uses what it needs)
   - Better CPU scheduling with Windows host
   - Faster I/O operations due to native Linux filesystem
   - Lower memory footprint compared to Docker Desktop

### How Does It Work?

1. **Network Integration**

   ```plaintext
   Windows Host <-> WSL2 VM <-> Container Network
        |               |              |
   Host Network    VM Network    Container Network
   ```

   - WSL2 creates a virtual network adapter
   - Containers run natively in Linux namespace
   - Direct network stack access from Linux

2. **Container Runtime Flow**

   ```plaintext
   Docker Engine (in WSL2) -> containerd -> runc -> Linux Kernel
   ```

   - Native Linux container runtime
   - No Windows translation layer
   - Direct kernel system calls

3. **NodePort Access Mechanism**
   ```plaintext
   Windows Host (localhost:30000)
         ↓
   WSL2 Network Interface
         ↓
   Kubernetes NodePort Service
         ↓
   Container Pod
   ```
   - Direct access without port mapping
   - Native Linux networking stack
   - No additional configuration needed

### What Makes It Better?

1. **Development Workflow Improvements**

   - Faster container build times
   - Native Linux tooling support
   - Better IDE integration
   - Seamless debugging experience

2. **Kubernetes-Specific Benefits**

   - Native support for all Kubernetes features
   - Better performance for:
     - Volume mounts
     - Network operations
     - Container startup times
   - Full compatibility with Linux-based tools

3. **Resource Utilization**
   | Feature | WSL2 | Docker Desktop |
   |---------|------|----------------|
   | Memory Overhead | Lower (~150MB base) | Higher (~2GB minimum) |
   | CPU Usage | Dynamic | Reserved |
   | Disk Space | Minimal | 2GB+ |
   | Startup Time | Seconds | Minutes |

4. **NodePort Accessibility Example**

   Windows Docker Desktop:

   ```bash
   # Requires explicit port forwarding
   kubectl port-forward service/my-service 8080:30000
   curl localhost:8080  # Indirect access
   ```

   WSL2:

   ```bash
   # Direct access
   curl localhost:30000  # Works immediately
   ```

## Setup Instructions

### 1. Install WSL2

```bash
# Enable WSL feature
wsl --install

# Set WSL2 as default
wsl --set-default-version 2

# Install Ubuntu (recommended distro)
wsl --install -d Ubuntu
```

### 2. Install Required Tools

```bash
# Update package list
sudo apt update
sudo apt upgrade

# Install Docker
sudo apt install docker.io

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### 3. Start Minikube

```bash
# Start Minikube with Docker driver
minikube start --driver=docker

# Verify installation
kubectl get nodes
minikube status
```

Then create a dir for K8s files:

```bash
mkdir -p ~/kubernetes/assignment1
cd ~/kubernetes/assignment1
```

Next, create YAML files:

```bash# Create nginx-deployment.yaml
cat > nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create nginx-service.yaml
cat > nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30007
  type: NodePort
EOF
```

### 4. Working with NodePorts

In WSL2, you can directly access NodePort services without additional configuration:

```bash
# Deploy a sample service
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml

# Verify everything is running
kubectl get pods
kubectl get services

# Get the NodePort
kubectl get svc nginx-service

# Access directly via localhost:NODEPORT
# Example: If NodePort is 30007
minikube ip
curl $(minikube ip):30007
```

## Advanced Configuration

### WSL2 Memory Management

```ini
# In %UserProfile%\.wslconfig
[wsl2]
memory=6GB
processors=4
swap=2GB
```

### Network Optimization

```bash
# In /etc/sysctl.conf
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
```

### Performance Tuning

1. **Storage Performance**

   - Keep projects in Linux filesystem
   - Use `/home/<user>` directory
   - Avoid Windows filesystem mounts

2. **Container Image Management**
   - Use local registry in WSL2
   - Implement proper image cleanup
   - Utilize multi-stage builds

## Best Practices

1. Keep WSL2 and Ubuntu updated
2. Use Ubuntu as your primary distribution
3. Configure resource limits in `.wslconfig` if needed
4. Maintain regular backups of your WSL environment
5. Regular system maintenance:
   ```bash
   # WSL system cleanup
   wsl --shutdown
   diskpart
   compact vhdx
   ```
6. Development workflow optimization:
   - Use Linux-native tools
   - Implement proper volume mounts
   - Configure IDE for remote WSL development

## Monitoring and Maintenance

1. **Resource Monitoring**

   ```bash
   # Check WSL2 resource usage
   top
   htop
   docker stats
   ```

2. **Log Management**
   ```bash
   # View Kubernetes logs
   kubectl logs -f <pod-name>
   # View system logs
   journalctl -xe
   ```

## Additional Resources

- [Official WSL Documentation](https://docs.microsoft.com/en-us/windows/wsl/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
