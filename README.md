AKS + Istio 1.27.1 Demo Deployment

This guide describes the complete setup of AKS with Istio 1.27.1, deployment of the Bookinfo sample application, and configuration of Istio dashboards including Kiali, Grafana, Prometheus, Jaeger/Tracing.

Prerequisites

Azure CLI installed and logged in

kubectl installed

Bash shell (Linux, macOS, or Git Bash for Windows)

Internet access for Istio and sample app YAMLs

Step 1: Variables & Script Setup

Key parameters:

RESOURCE_GROUP="aks-istio-rg"
AKS_CLUSTER="aks-istio-cluster"
LOCATION="eastus"
NODE_COUNT=1
NODE_SIZE="Standard_B2s"
NAMESPACE="myapp"
MAX_PODS=2
K8S_VERSION="1.32.7"
ISTIO_VERSION="1.27.1"

