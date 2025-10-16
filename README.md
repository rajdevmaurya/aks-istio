# AKS + Istio 1.27.1 Demo Deployment


##  Quick Start

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


```

Verify cluster:
```bash
#!/bin/bash
#
# AKS + Istio setup with Istio native monitoring (Kiali, Grafana, Prometheus, Jaeger)
# Includes patching Kiali and Jaeger for external LoadBalancer access.
#

LOGFILE="aks_istio_setup.log"
exec > >(tee -i "$LOGFILE")
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
K8S_VERSION="1.32.7"
ISTIO_VERSION="1.27.1"
ISTIO_RELEASE_BRANCH="release-1.27"

echo "Variables:"
echo "  Resource Group: $RESOURCE_GROUP"
echo "  Cluster Name: $AKS_CLUSTER"
echo "  Location: $LOCATION"
echo "  Node Count: $NODE_COUNT"
echo "  Node Size: $NODE_SIZE"
echo "  Namespace: $NAMESPACE"
echo "  Kubernetes Version: $K8S_VERSION"
echo "  Istio Version: $ISTIO_VERSION"
echo "  Istio Release Branch: $ISTIO_RELEASE_BRANCH"
echo "------------------------------------"

# -----------------------------
# Azure Login
# -----------------------------
echo "Checking Azure login..."
az account show > /dev/null 2>&1
if [ $? -ne 0 ]; then
    az login --use-device-code || { echo "Azure login failed"; exit 1; }
else
    echo "Already logged in."
fi

# -----------------------------
# Resource Group
# -----------------------------
echo "Ensuring resource group '$RESOURCE_GROUP' exists..."
az group create --name "$RESOURCE_GROUP" --location "$LOCATION" -o none || {
    echo "Failed to create or access resource group"; exit 1;
}

# -----------------------------
# Create AKS Cluster (without Azure Monitor)
# -----------------------------
echo "Creating AKS cluster (without Azure Monitor)..."
az aks create \
    --resource-group "$RESOURCE_GROUP" \
    --name "$AKS_CLUSTER" \
    --node-count "$NODE_COUNT" \
    --node-vm-size "$NODE_SIZE" \
    --kubernetes-version "$K8S_VERSION" \
    --generate-ssh-keys \
    --enable-managed-identity \
    -o none || { echo "AKS creation failed"; exit 1; }

# -----------------------------
# Configure kubectl
# -----------------------------
echo "Configuring kubectl..."
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER" --overwrite-existing
kubectl get nodes || { echo "kubectl cannot connect to AKS cluster"; exit 1; }

# -----------------------------
# Install Istio
# -----------------------------
echo "Downloading Istio version $ISTIO_VERSION..."
ISTIO_TGZ="istio-${ISTIO_VERSION}-linux-amd64.tar.gz"
ISTIO_URL="https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/${ISTIO_TGZ}"
curl -L "$ISTIO_URL" -o "$ISTIO_TGZ" || { echo "Failed to download Istio"; exit 1; }

tar -xzf "$ISTIO_TGZ"
cd "istio-${ISTIO_VERSION}" || { echo "Failed to enter Istio directory"; exit 1; }

export PATH=$PWD/bin:$PATH

echo "Verifying istioctl..."
istioctl version --remote=false || { echo "istioctl failed"; exit 1; }

echo "Installing Istio demo profile..."
istioctl install --set profile=demo -y || { echo "Istio install failed"; exit 1; }

# -----------------------------
# Namespace + Sidecar Injection
# -----------------------------
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace "$NAMESPACE" istio-injection=enabled --overwrite
kubectl get namespace -L istio-injection

# -----------------------------
# Deploy Bookinfo
# -----------------------------
echo "Deploying Bookinfo sample app..."
BOOKINFO_YAML="https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/bookinfo/platform/kube/bookinfo.yaml"
kubectl apply -n "$NAMESPACE" -f "$BOOKINFO_YAML" || { echo "Failed to deploy Bookinfo"; exit 1; }

kubectl scale deployment productpage-v1 -n "$NAMESPACE" --replicas="$MAX_PODS" || true
kubectl scale deployment reviews-v1 -n "$NAMESPACE" --replicas="$MAX_PODS" || true
kubectl get pods -n "$NAMESPACE"

# -----------------------------
# Apply Istio Gateway
# -----------------------------
BOOKINFO_GATEWAY_YAML="https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/bookinfo/networking/bookinfo-gateway.yaml"
kubectl apply -n "$NAMESPACE" -f "$BOOKINFO_GATEWAY_YAML"

# -----------------------------
# Install Istio Addons (Monitoring Tools)
# -----------------------------
echo "Installing Istio monitoring addons..."
kubectl apply -f "https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/kiali.yaml"
kubectl apply -f "https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/grafana.yaml"
kubectl apply -f "https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/prometheus.yaml"
kubectl apply -f "https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/jaeger.yaml"

# -----------------------------
# Patch Kiali and Jaeger for external access
# -----------------------------
echo "Patching Kiali and Jaeger to expose dashboards externally..."

kubectl -n istio-system patch svc kiali -p '{
  "spec": {
    "ports": [
      { "name": "http-kiali-80", "port": 80, "targetPort": 20001 },
      { "name": "http-metrics", "port": 9090, "targetPort": 9090 }
    ],
    "type": "LoadBalancer"
  }
}'

kubectl -n istio-system patch svc tracing -p '{
  "spec": {
    "ports": [
      { "name": "http-tracing", "port": 16685, "targetPort": 16685 }
    ],
    "type": "LoadBalancer"
  }
}'

# Grafana and Prometheus (ensure LoadBalancer)
kubectl -n istio-system patch svc grafana -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n istio-system patch svc prometheus -p '{"spec":{"type":"LoadBalancer"}}'

echo "Waiting for monitoring pods and services to be ready..."
kubectl wait --for=condition=available deployment/kiali -n istio-system --timeout=300s || true
kubectl wait --for=condition=available deployment/grafana -n istio-system --timeout=300s || true
kubectl wait --for=condition=available deployment/prometheus -n istio-system --timeout=300s || true
kubectl wait --for=condition=available deployment/jaeger -n istio-system --timeout=300s || true

# -----------------------------
# Wait for Istio Ingress Gateway
# -----------------------------
echo "Waiting for Istio Ingress Gateway EXTERNAL-IP..."
for i in {1..30}; do
    EXTERNAL_IP=$(kubectl get svc istio-ingressgateway -n istio-system \
        --template="{{range .status.loadBalancer.ingress}}{{.ip}}{{end}}" 2>/dev/null)
    if [ -n "$EXTERNAL_IP" ]; then
        echo "Istio Ingress Gateway EXTERNAL-IP: $EXTERNAL_IP"
        break
    fi
    echo "Waiting for IP... ($i/30)"
    sleep 10
done

# -----------------------------
# Print URLs for dashboards
# -----------------------------
echo "===== DASHBOARD URLS ====="
kubectl get svc -n istio-system -o custom-columns=NAME:.metadata.name,TYPE:.spec.type,EXTERNAL-IP:.status.loadBalancer.ingress[0].ip,PORTS:.spec.ports[*].port
echo
echo "Bookinfo app: http://$EXTERNAL_IP/productpage"
echo "Kiali: http://$(kubectl get svc kiali -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/"
echo "Grafana: http://$(kubectl get svc grafana -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/"
echo "Prometheus: http://$(kubectl get svc prometheus -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/"
echo "Jaeger: http://$(kubectl get svc tracing -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):16685/"

echo "===== SETUP COMPLETE ====="
echo "Logs saved to $LOGFILE"

```

```bash

ISTIO_RELEASE_BRANCH="release-1.27"

kubectl apply -f https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/${ISTIO_RELEASE_BRANCH}/samples/addons/jaeger.yaml

kubectl -n istio-system patch svc kiali       -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n istio-system patch svc grafana     -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n istio-system patch svc prometheus  -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n istio-system patch svc tracing     -p '{"spec":{"type":"LoadBalancer"}}'


Install Istio with demo profile:
```bash
istioctl install --set profile=demo -y
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
    echo "Istio Ingress Gateway EXTERNAL-IP: $EXTERNAL_IP"
    echo "Access Bookinfo: http://$EXTERNAL_IP/productpage"
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

## Accessing Dashboards

Once external IPs are assigned, access the dashboards:

| Dashboard | Default Port | URL Format |
|-----------|-------------|------------|
| **Bookinfo App** | 80 | `http://<ISTIO_INGRESS_IP>/productpage` |
| **Kiali** | 80 | `http://<KIALI_IP>/kiali` |
| **Grafana** | 3000 | `http://<GRAFANA_IP>:3000` |
| **Prometheus** | 9090 | `http://<PROMETHEUS_IP>:9090` |
| **Jaeger Tracing** | 16685 | `http://<TRACING_IP>:16685` |

---

## Troubleshooting

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

Verify LoadBalancer service status:
```bash
kubectl get svc -n istio-system -w
```

Check Azure load balancer creation in the portal or CLI.

---

##Cleanup

Delete the AKS cluster:
```bash
az aks delete --resource-group aks-istio-rg --name aks-istio-cluster --yes --no-wait
```

Delete the resource group:
```bash
az group delete --name aks-istio-rg --yes --no-wait
```

