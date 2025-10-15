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
RESOURCE_GROUP="aks-istio-rg"
AKS_CLUSTER="aks-istio-cluster"
LOCATION="eastus"
NODE_COUNT=1
NODE_SIZE="Standard_B2s"
NAMESPACE="myapp"
MAX_PODS=2
K8S_VERSION="1.32.7"
ISTIO_VERSION="1.27.1"
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

---

## 📝 Notes & Best Practices

- **LoadBalancer IPs**: Ensure Azure assigns external IPs before accessing dashboards. This may take 2-5 minutes.
- **Port Conflicts**: Always patch Kiali ports explicitly to avoid conflicts between UI and metrics endpoints.
- **Resource Monitoring**: Use `kubectl get pods -n istio-system -w` to monitor dashboard pod initialization.
- **Security**: For production, use Ingress controllers with TLS instead of direct LoadBalancer exposure.
- **Cost Optimization**: Standard_B2s nodes are suitable for demos but not production workloads.

---

## 📚 Additional Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [Bookinfo Sample Application](https://istio.io/latest/docs/examples/bookinfo/)
- [Azure AKS Documentation](https://docs.microsoft.com/azure/aks/)
- [Kiali Documentation](https://kiali.io/docs/)

---

## 🤝 Contributing

Feel free to open issues or submit pull requests for improvements to this setup guide.

## 📄 License

This guide is provided as-is for educational and demonstration purposes.
