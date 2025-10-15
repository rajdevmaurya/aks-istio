# AKS + Istio 1.27.1 Demo Deployment

Complete guide for setting up Azure Kubernetes Service (AKS) with Istio 1.27.1, deploying the Bookinfo sample application, and configuring Istio observability dashboards.

## 📋 Prerequisites

Before starting, ensure you have:

- **Azure CLI** installed and logged in
- **kubectl** installed and configured
- **Bash shell** (Linux, macOS, or Git Bash for Windows)
- **Internet access** for downloading Istio and sample application manifests
- **Azure subscription** with appropriate permissions

## 🔧 Configuration Variables

```bash
#!/bin/bash

LOGFILE="aks_istio_setup.log"
exec > >(tee -i $LOGFILE)
exec 2>&1

echo "===== START AKS + ISTIO SETUP ====="
echo "Date: $(date)"

# -----------------------------
# Variables
# -----------------------------
RESOURCE_GROUP="aks-istio-rg"
AKS_CLUSTER="aks-istio-cluster"
LOCATION="eastus"
NODE_COUNT=1
NODE_SIZE="Standard_B2s"
NAMESPACE="myapp"
MAX_PODS=2
K8S_VERSION="1.32.7"   # Supported version in eastus
ISTIO_VERSION="1.27.1"  # ✅ Verified stable version

# Derive branch name automatically (major.minor)
ISTIO_MAJOR_MINOR="${ISTIO_VERSION%%.*}.${ISTIO_VERSION#*.}"
ISTIO_RELEASE_BRANCH="release-${ISTIO_MAJOR_MINOR}"

echo "Variables:"
echo "Resource Group: $RESOURCE_GROUP"
echo "Cluster Name: $AKS_CLUSTER"
echo "Location: $LOCATION"
echo "Node Count: $NODE_COUNT"
echo "Node Size: $NODE_SIZE"
echo "Namespace: $NAMESPACE"
echo "Max Pods per deployment: $MAX_PODS"
echo "Kubernetes Version: $K8S_VERSION"
echo "Istio Version: $ISTIO_VERSION"
echo "Istio GitHub Branch: $ISTIO_RELEASE_BRANCH"

# -----------------------------
# Azure Login
# -----------------------------
echo "Logging in to Azure..."
az account show > /dev/null 2>&1
if [ $? -ne 0 ]; then
    az login --use-device-code
else
    echo "Already logged in to Azure."
fi

# -----------------------------
# Create Resource Group
# -----------------------------
az group create --name $RESOURCE_GROUP --location $LOCATION

# -----------------------------
# Create AKS Cluster
# -----------------------------
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER \
    --node-count $NODE_COUNT \
    --node-vm-size $NODE_SIZE \
    --kubernetes-version $K8S_VERSION \
    --generate-ssh-keys \
    --enable-managed-identity \
    --enable-addons monitoring

# -----------------------------
# Configure kubectl
# -----------------------------
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
kubectl get nodes || { echo "❌ kubectl cannot connect to AKS cluster"; exit 1; }

# -----------------------------
# Install Istio (Cross-platform Safe)
# -----------------------------
echo "Downloading Istio version $ISTIO_VERSION..."

# Detect OS type
OS_TYPE=$(uname | tr '[:upper:]' '[:lower:]')
if [[ "$OS_TYPE" == *"mingw"* || "$OS_TYPE" == *"msys"* || "$OS_TYPE" == *"cygwin"* ]]; then
    ISTIO_TGZ="istio-${ISTIO_VERSION}-win.zip"
    ISTIO_URL="https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/${ISTIO_TGZ}"
else
    ISTIO_TGZ="istio-${ISTIO_VERSION}-linux-amd64.tar.gz"
    ISTIO_URL="https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/${ISTIO_TGZ}"
fi

curl -L "$ISTIO_URL" -o "$ISTIO_TGZ"
if [ $? -ne 0 ]; then
    echo "❌ Failed to download Istio ${ISTIO_VERSION}."
    exit 1
fi

echo "Extracting Istio..."
if [[ "$ISTIO_TGZ" == *.zip ]]; then
    unzip -q "$ISTIO_TGZ"
else
    tar -xzf "$ISTIO_TGZ"
fi

cd "istio-${ISTIO_VERSION}" || { echo "❌ Failed to enter Istio directory"; exit 1; }

# Add istioctl to PATH (handles Windows or Linux)
if [[ "$OS_TYPE" == *"mingw"* || "$OS_TYPE" == *"msys"* || "$OS_TYPE" == *"cygwin"* ]]; then
    alias istioctl='./bin/istioctl.exe'
else
    export PATH=$PWD/bin:$PATH
fi

echo "Verifying Istio CLI..."
istioctl version --remote=false || { echo "❌ istioctl failed to run"; exit 1; }

echo "Installing Istio demo profile..."
istioctl install --set profile=demo -y

# -----------------------------
# Namespace + Sidecar Injection
# -----------------------------
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace $NAMESPACE istio-injection=enabled --overwrite
kubectl get namespace -L istio-injection

# -----------------------------
# Deploy Bookinfo Sample Apphttps://raw.githubusercontent.com/istio/istio/release-1.27/samples/bookinfo/platform/kube/bookinfo.yaml
# -----------------------------
echo "Deploying Bookinfo sample application..."
BOOKINFO_PLATFORM_YAML="https://raw.githubusercontent.com/istio/istio/release-1.27/samples/bookinfo/platform/kube/bookinfo.yaml"
kubectl apply -n $NAMESPACE -f "$BOOKINFO_PLATFORM_YAML" || { echo "❌ Failed to apply Bookinfo YAML"; exit 1; }

kubectl scale deployment productpage-v1 -n $NAMESPACE --replicas=$MAX_PODS || true
kubectl scale deployment reviews-v1 -n $NAMESPACE --replicas=$MAX_PODS || true
kubectl get pods -n $NAMESPACE

# -----------------------------
# Apply Istio Gateway
# -----------------------------
BOOKINFO_GATEWAY_YAML="https://raw.githubusercontent.com/istio/istio/release-1.27/samples/bookinfo/networking/bookinfo-gateway.yaml"
kubectl apply -n $NAMESPACE -f "$BOOKINFO_GATEWAY_YAML"

# -----------------------------
# Wait for Ingress Gateway
# -----------------------------
echo "Waiting for Istio Ingress Gateway EXTERNAL-IP..."
for i in {1..30}; do
    EXTERNAL_IP=$(kubectl get svc istio-ingressgateway -n istio-system \
        --template="{{range .status.loadBalancer.ingress}}{{.ip}}{{end}}" 2>/dev/null)
    if [ -n "$EXTERNAL_IP" ]; then
        echo "✅ Istio Ingress Gateway EXTERNAL-IP: $EXTERNAL_IP"
        echo "🌐 Access Bookinfo: http://$EXTERNAL_IP/productpage"
        break
    fi
    echo "Waiting for IP... ($i/30)"
    sleep 10
done

if [ -z "$EXTERNAL_IP" ]; then
    echo "⚠️  Istio Ingress Gateway EXTERNAL-IP not assigned yet."
fi

echo "===== SETUP COMPLETE ====="
echo "Logs saved to $LOGFILE"

```

## 🚀 Quick Start

### Option 1: Automated Setup (Recommended)

Use the provided automation script for a complete setup:

```bash
chmod +x setup_aks_istio.sh
./setup_aks_istio.sh
```

The script will automatically:
- Create AKS cluster
- Download and install Istio 1.27.1
- Deploy Bookinfo sample application
- Configure namespace injection
- Set up Istio gateway

### Option 2: Manual Step-by-Step Setup

Follow the detailed steps below for manual installation.

---

## 📖 Manual Installation Steps

### Step 1: Create Azure Resources

Create resource group:
```bash
az group create --name aks-istio-rg --location eastus
```

Create AKS cluster:
```bash
az aks create \
    --resource-group aks-istio-rg \
    --name aks-istio-cluster \
    --node-count 1 \
    --node-vm-size Standard_B2s \
    --kubernetes-version 1.32.7 \
    --generate-ssh-keys \
    --enable-managed-identity \
    --enable-addons monitoring
```

Configure kubectl:
```bash
az aks get-credentials --resource-group aks-istio-rg --name aks-istio-cluster --overwrite-existing
```

Verify cluster:
```bash
kubectl get nodes
kubectl get ns
```

### Step 2: Install Istio 1.27.1

Download Istio:
```bash
curl -L https://github.com/istio/istio/releases/download/1.27.1/istio-1.27.1-linux-amd64.tar.gz -o istio-1.27.1-linux-amd64.tar.gz
tar -xzf istio-1.27.1-linux-amd64.tar.gz
cd istio-1.27.1
```

Add istioctl to PATH:
```bash
export PATH=$PWD/bin:$PATH
```

Verify installation:
```bash
istioctl version
```

Install Istio with demo profile:
```bash
istioctl install --set profile=demo -y
```

### Step 3: Enable Istio Sidecar Injection

Create namespace:
```bash
kubectl create namespace myapp --dry-run=client -o yaml | kubectl apply -f -
```

Enable automatic sidecar injection:
```bash
kubectl label namespace myapp istio-injection=enabled --overwrite
```

Verify injection is enabled:
```bash
kubectl get namespace -L istio-injection
```

### Step 4: Deploy Bookinfo Application

Deploy the application:
```bash
kubectl apply -n myapp -f https://raw.githubusercontent.com/istio/istio/release-1.27/samples/bookinfo/platform/kube/bookinfo.yaml
```

Scale deployments:
```bash
kubectl scale deployment productpage-v1 -n myapp --replicas=2
kubectl scale deployment reviews-v1 -n myapp --replicas=2
```

Apply Istio gateway:
```bash
kubectl apply -n myapp -f https://raw.githubusercontent.com/istio/istio/release-1.27/samples/bookinfo/networking/bookinfo-gateway.yaml
```

Check deployment status:
```bash
kubectl get pods -n myapp
```

### Step 5: Get Ingress Gateway External IP

Wait for external IP assignment:
```bash
for i in {1..30}; do
  EXTERNAL_IP=$(kubectl get svc istio-ingressgateway -n istio-system --template="{{range .status.loadBalancer.ingress}}{{.ip}}{{end}}" 2>/dev/null)
  if [ -n "$EXTERNAL_IP" ]; then
    echo "✅ Istio Ingress Gateway EXTERNAL-IP: $EXTERNAL_IP"
    echo "🌐 Access Bookinfo: http://$EXTERNAL_IP/productpage"
    break
  fi
  echo "Waiting for IP... ($i/30)"
  sleep 10
done
```

### Step 6: Expose Istio Observability Dashboards

By default, dashboards use ClusterIP. Patch them to LoadBalancer for external access:

**Kiali Dashboard:**
```bash
kubectl -n istio-system patch svc kiali -p '{
  "spec": {
    "ports": [
      {
        "name": "http-kiali-80",
        "port": 80,
        "targetPort": 20001
      },
      {
        "name": "http-metrics",
        "port": 9090,
        "targetPort": 9090
      }
    ],
    "type": "LoadBalancer"
  }
}'
```

**Grafana Dashboard:**
```bash
kubectl -n istio-system patch svc grafana -p '{"spec":{"type":"LoadBalancer"}}'
```

**Prometheus:**
```bash
kubectl -n istio-system patch svc prometheus -p '{"spec":{"type":"LoadBalancer"}}'
```

**Jaeger Tracing:**
```bash
kubectl -n istio-system patch svc tracing -p '{
  "spec": {
    "ports": [
      {
        "name": "http-tracing",
        "port": 16685,
        "targetPort": 16685
      }
    ],
    "type": "LoadBalancer"
  }
}'
```

Get all external IPs:
```bash
kubectl get svc -n istio-system
```

---

## 🌐 Accessing Dashboards

Once external IPs are assigned, access the dashboards:

| Dashboard | Default Port | URL Format |
|-----------|-------------|------------|
| **Bookinfo App** | 80 | `http://<ISTIO_INGRESS_IP>/productpage` |
| **Kiali** | 80 | `http://<KIALI_IP>/kiali` |
| **Grafana** | 3000 | `http://<GRAFANA_IP>:3000` |
| **Prometheus** | 9090 | `http://<PROMETHEUS_IP>:9090` |
| **Jaeger Tracing** | 16685 | `http://<TRACING_IP>:16685` |

---

## 🔍 Troubleshooting

### Dashboard Not Accessible

Check if pods are running:
```bash
kubectl get pods -n istio-system
```

Watch pods starting:
```bash
kubectl get pods -n istio-system -w
```

Test dashboard locally from within the cluster:
```bash
# Kiali
kubectl exec -n istio-system -it <kiali-pod-name> -- curl -s localhost:20001

# Jaeger
kubectl exec -n istio-system -it <tracing-pod-name> -- curl -s localhost:16685
```

### Bookinfo Pods Not Running

Check pod logs:
```bash
kubectl logs <pod-name> -n myapp
```

Describe pod for events:
```bash
kubectl describe pod <pod-name> -n myapp
```

### External IP Pending

Verify LoadBalancer service status:
```bash
kubectl get svc -n istio-system -w
```

Check Azure load balancer creation in the portal or CLI.

---

## 🧹 Cleanup

Delete the AKS cluster:
```bash
az aks delete --resource-group aks-istio-rg --name aks-istio-cluster --yes --no-wait
```

Delete the resource group:
```bash
az group delete --name aks-istio-rg --yes --no-wait
```

