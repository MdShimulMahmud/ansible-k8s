# Kubernetes Cluster with Bastion Proxy Setup

## Architecture Overview

This setup enables external access to a Kubernetes cluster through a bastion server acting as a reverse proxy.

**Traffic Flow:**

1. Users access the application via a Domain Name (e.g., `app.example.com`)
2. Route53 resolves the domain to the Bastion Public IP (`103.204.80.245`)
3. Bastion Nginx receives the request and proxies it to the MetalLB IP (e.g., `172.17.100.200`)
4. MetalLB forwards the traffic to the Ingress Controller (NGINX)
5. Ingress Controller routes traffic to the appropriate Kubernetes Pods

---

## Phase 1: Kubernetes Cluster Setup (via Bastion)

**Assumptions:**

- You are logged into the Bastion server
- You can SSH into the Master and Worker nodes from Bastion
- OS is Ubuntu/Debian

### 1. Install Container Runtime & Kubeadm (On All 3 Nodes)

Run these commands on Master and Worker nodes via the Bastion:

```bash
# Allow IP forwarding
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Install Containerd (Skip if already installed)
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# Install Kubeadm, Kubelet, Kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2. Initialize Control Plane (On Master Node)

Run this only on the Master. Replace `<MASTER_PRIVATE_IP>` with the actual IP (e.g., `172.17.100.x`).

```bash
sudo kubeadm init --control-plane-endpoint=<MASTER_PRIVATE_IP> --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<MASTER_PRIVATE_IP>
```

> **Note:** We use `192.168.0.0/16` as the Pod CIDR because we will install Calico, which is the recommended CNI for MetalLB.

After initialization, configure kubectl on the Master:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3. Install CNI (Calico) & Join Workers (On Master)

```bash
# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# Get join command
kubeadm token create --print-join-command
```

Run the output join command on both Worker nodes.

---

## Phase 2: Install MetalLB

Since you want to use Private IPs for internal communication but accessible via the Bastion, we will configure MetalLB with an IP range from your local subnet.

**IP Range Planning:**

- Identify a free IP range in your `172.17.100.x` network
- Assuming your nodes occupy the lower IPs (e.g., `.10` to `.20`), we can use `.200` to `.210` for LoadBalancers

### 1. Install MetalLB Manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```

### 2. Configure IPAddressPool

Create a file `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.17.100.200-172.17.100.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
    - first-pool
```

Apply the configuration:

```bash
kubectl apply -f metallb-config.yaml
```

---

## Phase 3: Install Ingress Controller (NGINX)

Install the standard NGINX Ingress Controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

### Verify the IP

Wait a few minutes and run:

```bash
kubectl get svc -n ingress-nginx
```

You should see the `EXTERNAL-IP` field populated with an IP from your MetalLB pool (e.g., `172.17.100.200`).

---

## Phase 4: Configure Nginx on Bastion Server

Configure Nginx on your Bastion (`172.17.100.18`) to act as a Reverse Proxy.

### Install Nginx on Bastion

```bash
sudo apt update
sudo apt install nginx -y
```

### Create Configuration

Create a new config file (e.g., `/etc/nginx/sites-available/k8s-proxy`).

Replace `172.17.100.200` with the actual EXTERNAL-IP obtained in Phase 3:

```nginx
server {
    listen 80;
    server_name your-domain.com;  # The domain you will use in Route53

    location / {
        # Pass traffic to the MetalLB IP assigned to Ingress Controller
        proxy_pass http://172.17.100.200;

        # Standard Proxy Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/k8s-proxy /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## Phase 5: Route53 Configuration

1. Go to your AWS Route53 Console
2. Select your Hosted Zone
3. **Create an A Record:**
   - **Name:** `app` (or whatever subdomain you prefer)
   - **Type:** A - Routes traffic to an IPv4 address and some AWS resources
   - **Value:** `103.204.80.245` (The Public IP of your Bastion server)
4. Save

---

## Summary of Traffic Flow

1. User types `http://app.example.com`
2. DNS resolves to `103.204.80.245` (Bastion Public IP)
3. Request hits Bastion Nginx (Port 80)
4. Bastion Nginx proxies the request to `172.17.100.200` (MetalLB IP) via the private network
5. MetalLB routes traffic to the Ingress Controller Service inside the K8s cluster
6. Ingress Controller sends traffic to your backend Pods

---

## Important Note on SSL/HTTPS

This setup currently handles HTTP (Port 80). To enable HTTPS:

### Option A (Recommended): Cert-Manager in Kubernetes

Set up Cert-Manager inside Kubernetes to terminate SSL at the Ingress Controller. Configure Bastion Nginx to use `proxy_pass https://<METALLB_IP>` and ignore SSL verification (or use a self-signed cert between Bastion and Cluster).

### Option B: SSL Termination at Bastion

Terminate SSL at the Bastion Nginx. You would install the SSL certificate on the Bastion and talk HTTP to the cluster. This is simpler to set up but requires managing certs on the Bastion.
