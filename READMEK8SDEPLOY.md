
# **EasyShop – Kubernetes Deployment Guide (Kind + Docker + MongoDB + Ingress + HPA)**

EasyShop is a full-stack e-commerce application deployed using Docker, Kubernetes, MongoDB, migration jobs, autoscaling (HPA), and NGINX Ingress.  
This guide documents the complete DevOps workflow to build, containerize, deploy, and validate the app in a **local Kind Kubernetes cluster on Ubuntu Linux**.

---
## 📋 Prerequisites Installation

---
### Linux
# 🐳 **1. Docker Setup**

### Install Docker
>    ```bash
>    curl -fsSL https://get.docker.com -o get-docker.sh
>    sudo sh get-docker.sh
>    rm get-docker.sh
###
### Allow your user to run Docker commands without needing sudo and add current user in docker group
>   ```bash
>    sduo usermod -aG docker $USER && newgrp docker
### Build & push application image
```bash
docker build -t <dockerhub-username>/easyshop:latest .
docker push <dockerhub-username>/easyshop:latest
```

### Build & push migration image
```bash
docker build -t <dockerhub-username>/easyshop-migration:latest -f scripts/Dockerfile.migration .
docker push <dockerhub-username>/easyshop-migration:latest
```

---

# ☸️ **2. Kind Kubernetes Cluster Setup**

### Install Kind
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Create cluster
```bash
kind create cluster --name easyshop --config k8s/00-kind-config.yaml
```

---

# 📦 **3. Apply Kubernetes Resources**

### Namespace
```bash
kubectl apply -f k8s/01-namespace.yaml
```

### Storage (PV & PVC)
```bash
kubectl apply -f k8s/02-mongodb-pv.yaml
kubectl apply -f k8s/03-mongodb-pvc.yaml
```
### Environment Setup
Create `k8s/04-configmap.yaml` with the following content:
>   ```yaml
>   apiVersion: v1
>   kind: ConfigMap
>   metadata:
>     name: easyshop-config
>     namespace: easyshop
>   data:
>     MONGODB_URI: "mongodb://mongodb-service:27017/easyshop"
>     NODE_ENV: "production"
>     NEXT_PUBLIC_API_URL: "http://YOUR_EC2_PUBLIC_IP/api"  # Replace with your YOUR_EC2_PUBLIC_IP
>     NEXTAUTH_URL: "http://YOUR_EC2_PUBLIC_IP"             # Replace with your YOUR_EC2_PUBLIC_IP
>     NEXTAUTH_SECRET: "HmaFjYZ2jbUK7Ef+wZrBiJei4ZNGBAJ5IdiOGAyQegw="
>     JWT_SECRET: "e5e425764a34a2117ec2028bd53d6f1388e7b90aeae9fa7735f2469ea3a6cc8c"

 [!IMPORTANT]
> When deploying to EC2, make sure to replace `your-ec2-ip` with your actual EC2 instance's public IP address.

To generate secure secret keys, use these commands in your terminal:
```bash
# For NEXTAUTH_SECRET
openssl rand -base64 32

# For JWT_SECRET
openssl rand -hex 32
```
Create `k8s/05-secrets.yaml` Update the NEXTAUTH_SECRET and JWT_SECRET  :

### ConfigMap & Secrets
```bash
kubectl apply -f k8s/04-configmap.yaml
kubectl apply -f k8s/05-secrets.yaml
```

### Deploy MongoDB
```bash
kubectl apply -f k8s/06-mongodb-service.yaml
kubectl apply -f k8s/07-mongodb-statefulset.yaml
```

### Deploy EasyShop App
```bash
kubectl apply -f k8s/08-easyshop-deployment.yaml
kubectl apply -f k8s/09-easyshop-service.yaml
```

---

# 🌐 **4. NGINX Ingress Setup**

### Install NGINX Ingress Controller  Install NGINX 
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### Wait for controller to be ready:
```bash
kubectl wait --namespace ingress-nginx   --for=condition=ready pod   --selector=app.kubernetes.io/component=controller   --timeout=90s
```

### Apply Ingress
```bash
kubectl apply -f k8s/10-ingress.yaml
```

---

# 📈 **5. Enable Autoscaling (HPA)**

- Install Metrics Server in Kind Cluster:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
- Edit the Metrics Server Deployment
```bash
kubectl -n kube-system edit deployment metrics-server
```
- Add the security bypass to deployment under `container.args`
```bash
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
```
- Restart the deployment
```bash
kubectl -n kube-system rollout restart deployment metrics-server
```
- Verify if the metrics server is running
```bash
kubectl get pods -n kube-system
kubectl top nodes
```

## Apply HPA:
```bash
kubectl apply -f k8s/11-hpa.yaml
```

---

# 🗄️ **6. Run Database Migration Job**

```bash
kubectl apply -f k8s/12-migration-job.yaml
```

Check job:
```bash
kubectl get jobs -n easyshop
kubectl logs job/db-migration -n easyshop
```

---

# 📡 **7. Accessing the Application**

The application will be available at:  
```
http://<your-ip>.nip.io
```

Example:  
```
http://51.20.251.235.nip.io
```
> [!WARNING]
> - http://`<public-ip>`.nip.io
>
>  
> `nip.io` is a free wildcard DNS service that automatically maps any subdomain to the corresponding IP address.

> [!NOTE] 
> - If you access `http://51.20.251.235.nip.io`, it resolves to `51.20.251.235`.
> - If you use `app.51.20.251.235.nip.io`, it still resolves to `51.20.251.235`.
>
> **Why Use `nip.io`?**  
> - **Simplifies local and remote testing:** No need to set up custom DNS records.  
> - **Useful for Kubernetes Ingress:** You can access services using public IP-based domains.  
> - **Great for temporary or dynamic environments:** Works with CI/CD pipelines, cloud VMs, and local testing.  
>
> **Who Provides This Service?**  
> - `nip.io` is an **open-source project maintained by Vincent Bernat**.  
> - It is provided **for free**, with no registration required.  
> - The service works by dynamically resolving any subdomain containing an IP address.
> 
> **More details:** [https://nip.io](https://nip.io)






## Troubleshooting
1. If pods are not starting, check logs:
   
   ```bash
   kubectl logs -n easyshop <pod-name>
    ```
2. For MongoDB connection issues:
   
   ```bash
   kubectl exec -it -n easyshop mongodb-0 -- mongosh
    ```
3. To restart deployments:
   
   ```bash
   kubectl rollout restart deployment/easyshop -n easyshop
    ```
---

# 🔍 **8. Verification Commands**

```bash
kubectl get all -n easyshop
kubectl exec -it -n easyshop mongodb-0 -- mongosh --eval "db.serverStatus()"
kubectl get ingress -n easyshop
watch kubectl get all -n easyshop
```

---

# 🧹 **9. Cleanup**

```bash
kind delete cluster --name easyshop
```

---

# 🏁 **10. Summary**

- Multi-stage Docker builds  
- Migration Docker image  
- Kubernetes (NS, PV, PVC)  
- MongoDB StatefulSet  
- ConfigMap + Secrets  
- EasyShop Deployment + Service  
- NGINX Ingress  
- HPA Autoscaling  
- Migration Job  
- Fully working on Kind  


