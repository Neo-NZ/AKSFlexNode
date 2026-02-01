# AKS Cluster Setup for E2E Testing

This guide provides step-by-step instructions for creating an Azure Kubernetes Service (AKS) cluster for AKSFlexNode E2E testing.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Detailed Setup](#detailed-setup)
- [Configuration Options](#configuration-options)
- [Validation](#validation)
- [Cost Optimization](#cost-optimization)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

---

## Overview

AKSFlexNode requires a specific AKS cluster configuration for E2E testing:

**Key Requirements:**
- ✅ **Azure RBAC enabled** - Required for Arc MSI authentication
- ✅ **Managed Identity** - For cluster authentication
- ✅ **Network connectivity** - Test VMs must reach API server (port 443)
- ✅ **Minimum size** - 1 node is sufficient for testing
- ✅ **Cost-optimized** - Use B-series VMs for testing

**Estimated Cost:** $30-50/month for a minimal test cluster

---

## Prerequisites

### Required Tools

```bash
# Azure CLI (version 2.0+)
az --version

# If not installed:
# Linux/WSL
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew install azure-cli

# Windows
# Download from: https://aka.ms/installazurecliwindows
```

### Azure Permissions

Your Azure account needs:
- **Subscription Contributor** role OR
- **Custom role** with these permissions:
  - `Microsoft.ContainerService/managedClusters/write`
  - `Microsoft.Resources/deployments/*`
  - `Microsoft.Network/virtualNetworks/*`
  - `Microsoft.Authorization/roleAssignments/write`

### Check Permissions

```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "Your Subscription Name"

# Verify you have permissions
az aks list --query "[].{Name:name, Location:location}" -o table
```

---

## Quick Start

### Option 1: Automated Script (Recommended)

Create and run `scripts/setup/create-aks-cluster.sh`:

```bash
#!/bin/bash
set -euo pipefail

# ============================================================================
# AKS Cluster Creation Script for AKSFlexNode E2E Testing
# ============================================================================

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Configuration - Modify these values
RESOURCE_GROUP="${RESOURCE_GROUP:-rg-aksflexnode-e2e-tests}"
LOCATION="${LOCATION:-westus}"
CLUSTER_NAME="${CLUSTER_NAME:-aks-flexnode-e2e-cluster}"
NODE_COUNT="${NODE_COUNT:-1}"
NODE_VM_SIZE="${NODE_VM_SIZE:-Standard_B2s}"
K8S_VERSION="${K8S_VERSION:-}"  # Empty = latest stable

# ============================================================================
# Functions
# ============================================================================

print_header() {
    echo -e "\n${BLUE}============================================================================${NC}"
    echo -e "${BLUE}$1${NC}"
    echo -e "${BLUE}============================================================================${NC}\n"
}

print_success() {
    echo -e "${GREEN}✅ $1${NC}"
}

print_error() {
    echo -e "${RED}❌ $1${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠️  $1${NC}"
}

print_info() {
    echo -e "${BLUE}ℹ️  $1${NC}"
}

# ============================================================================
# Main Script
# ============================================================================

print_header "AKS Cluster Setup for AKSFlexNode E2E Testing"

# Verify Azure CLI is logged in
print_info "Verifying Azure CLI authentication..."
if ! az account show &>/dev/null; then
    print_error "Not logged in to Azure CLI"
    echo "Please run: az login"
    exit 1
fi

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
SUBSCRIPTION_NAME=$(az account show --query name -o tsv)
print_success "Logged in to Azure"
print_info "Subscription: $SUBSCRIPTION_NAME ($SUBSCRIPTION_ID)"

# Display configuration
print_header "Configuration"
echo "Resource Group:  $RESOURCE_GROUP"
echo "Location:        $LOCATION"
echo "Cluster Name:    $CLUSTER_NAME"
echo "Node Count:      $NODE_COUNT"
echo "Node VM Size:    $NODE_VM_SIZE"
echo "Kubernetes:      ${K8S_VERSION:-Latest Stable}"
echo ""

read -p "Continue with this configuration? (y/N): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    print_warning "Aborted by user"
    exit 0
fi

# ============================================================================
# Step 1: Create Resource Group
# ============================================================================

print_header "Step 1: Creating Resource Group"

if az group show --name "$RESOURCE_GROUP" &>/dev/null; then
    print_warning "Resource group '$RESOURCE_GROUP' already exists"
else
    print_info "Creating resource group '$RESOURCE_GROUP' in '$LOCATION'..."
    az group create \
        --name "$RESOURCE_GROUP" \
        --location "$LOCATION" \
        --output none

    print_success "Resource group created"
fi

# ============================================================================
# Step 2: Get Latest Kubernetes Version (if not specified)
# ============================================================================

if [ -z "$K8S_VERSION" ]; then
    print_header "Step 2: Getting Latest Kubernetes Version"

    K8S_VERSION=$(az aks get-versions \
        --location "$LOCATION" \
        --query "values[?isPreview==null].version | sort(@) | [-1]" \
        --output tsv)

    print_info "Latest stable Kubernetes version: $K8S_VERSION"
else
    print_header "Step 2: Using Specified Kubernetes Version"
    print_info "Kubernetes version: $K8S_VERSION"
fi

# ============================================================================
# Step 3: Create AKS Cluster
# ============================================================================

print_header "Step 3: Creating AKS Cluster"
print_warning "This may take 5-10 minutes..."

# Check if cluster already exists
if az aks show --resource-group "$RESOURCE_GROUP" --name "$CLUSTER_NAME" &>/dev/null; then
    print_error "Cluster '$CLUSTER_NAME' already exists in '$RESOURCE_GROUP'"
    print_info "To recreate, first delete the existing cluster:"
    echo "  az aks delete --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --yes --no-wait"
    exit 1
fi

# Create AKS cluster
az aks create \
    --resource-group "$RESOURCE_GROUP" \
    --name "$CLUSTER_NAME" \
    --location "$LOCATION" \
    --kubernetes-version "$K8S_VERSION" \
    --node-count "$NODE_COUNT" \
    --node-vm-size "$NODE_VM_SIZE" \
    --enable-managed-identity \
    --enable-azure-rbac \
    --network-plugin azure \
    --generate-ssh-keys \
    --load-balancer-sku standard \
    --vm-set-type VirtualMachineScaleSets \
    --no-wait

print_info "AKS cluster creation initiated (running in background)"

# Wait for cluster to be ready
print_info "Waiting for cluster to be ready..."
az aks wait \
    --resource-group "$RESOURCE_GROUP" \
    --name "$CLUSTER_NAME" \
    --created \
    --interval 30 \
    --timeout 600

print_success "AKS cluster created successfully"

# ============================================================================
# Step 4: Get Cluster Credentials
# ============================================================================

print_header "Step 4: Configuring kubectl Access"

az aks get-credentials \
    --resource-group "$RESOURCE_GROUP" \
    --name "$CLUSTER_NAME" \
    --overwrite-existing \
    --output none

print_success "kubectl configured"

# ============================================================================
# Step 5: Verify Cluster
# ============================================================================

print_header "Step 5: Verifying Cluster"

print_info "Cluster nodes:"
kubectl get nodes

print_info "Cluster info:"
kubectl cluster-info

print_info "Kubernetes version:"
kubectl version --short 2>/dev/null || kubectl version

# ============================================================================
# Step 6: Get Cluster Information
# ============================================================================

print_header "Step 6: Cluster Information"

CLUSTER_RESOURCE_ID=$(az aks show \
    --resource-group "$RESOURCE_GROUP" \
    --name "$CLUSTER_NAME" \
    --query id -o tsv)

CLUSTER_FQDN=$(az aks show \
    --resource-group "$RESOURCE_GROUP" \
    --name "$CLUSTER_NAME" \
    --query fqdn -o tsv)

CLUSTER_IDENTITY=$(az aks show \
    --resource-group "$RESOURCE_GROUP" \
    --name "$CLUSTER_NAME" \
    --query identity.principalId -o tsv)

print_success "AKS Cluster Setup Complete!"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 CLUSTER DETAILS"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Cluster Name:        $CLUSTER_NAME"
echo "Resource Group:      $RESOURCE_GROUP"
echo "Location:            $LOCATION"
echo "Kubernetes Version:  $K8S_VERSION"
echo "Node Count:          $NODE_COUNT"
echo "Node VM Size:        $NODE_VM_SIZE"
echo ""
echo "Cluster Resource ID: $CLUSTER_RESOURCE_ID"
echo "Cluster FQDN:        $CLUSTER_FQDN"
echo "Managed Identity:    $CLUSTER_IDENTITY"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📝 SAVE THESE VALUES FOR GITHUB SECRETS"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "E2E_RESOURCE_GROUP:   $RESOURCE_GROUP"
echo "E2E_AKS_CLUSTER_NAME: $CLUSTER_NAME"
echo "E2E_AKS_RESOURCE_ID:  $CLUSTER_RESOURCE_ID"
echo "E2E_LOCATION:         $LOCATION"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🔗 QUICK LINKS"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Azure Portal:"
echo "  https://portal.azure.com/#resource$CLUSTER_RESOURCE_ID"
echo ""
echo "Add GitHub Secrets:"
echo "  https://github.com/YOUR_ORG/AKSFlexNode/settings/secrets/actions"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ NEXT STEPS"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "1. Add the secrets above to your GitHub repository"
echo "2. Create service principal for CI/CD authentication"
echo "3. Set up E2E testing workflows"
echo ""
echo "For detailed instructions, see:"
echo "  - docs/GITHUB_AZURE_AUTH_GUIDE.md"
echo "  - docs/E2E_CICD_GUIDE.md"
echo ""
```

**Make it executable and run:**

```bash
chmod +x scripts/setup/create-aks-cluster.sh
./scripts/setup/create-aks-cluster.sh
```

---

### Option 2: Manual Step-by-Step

If you prefer manual commands:

```bash
# Set variables
RESOURCE_GROUP="rg-aksflexnode-e2e-tests"
LOCATION="westus"
CLUSTER_NAME="aks-flexnode-e2e-cluster"
NODE_COUNT=1
NODE_VM_SIZE="Standard_B2s"

# 1. Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# 2. Create AKS cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --node-count $NODE_COUNT \
  --node-vm-size $NODE_VM_SIZE \
  --enable-managed-identity \
  --enable-azure-rbac \
  --network-plugin azure \
  --generate-ssh-keys

# 3. Get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing

# 4. Verify
kubectl get nodes
kubectl cluster-info
```

---

## Detailed Setup

### Step 1: Choose Configuration

#### Minimum Configuration (Cost-Optimized)

**Best for:** E2E testing, development

```bash
NODE_COUNT=1
NODE_VM_SIZE="Standard_B2s"        # 2 vCPU, 4GB RAM
# Cost: ~$30/month
```

#### Standard Configuration

**Best for:** Multi-node testing, production-like

```bash
NODE_COUNT=2
NODE_VM_SIZE="Standard_D2s_v3"     # 2 vCPU, 8GB RAM
# Cost: ~$140/month
```

#### Location Selection

Choose location closest to your testing infrastructure:

```bash
# List available locations
az account list-locations --query "[].{Name:name, DisplayName:displayName}" -o table

# Popular options:
# - eastus
# - westus
# - westeurope
# - southeastasia
```

### Step 2: Verify Kubernetes Version

```bash
# List available Kubernetes versions in your location
az aks get-versions --location westus -o table

# Get latest stable version
K8S_VERSION=$(az aks get-versions \
  --location westus \
  --query "values[?isPreview==null].version | sort(@) | [-1]" \
  --output tsv)

echo "Latest stable: $K8S_VERSION"
```

### Step 3: Advanced Configuration Options

#### Option A: Basic (Recommended for E2E)

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --kubernetes-version $K8S_VERSION \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --enable-azure-rbac \
  --network-plugin azure \
  --generate-ssh-keys
```

**Key Flags:**
- `--enable-managed-identity` - Use Azure Managed Identity (required)
- `--enable-azure-rbac` - Enable Azure RBAC for Kubernetes auth (required for Arc MSI)
- `--network-plugin azure` - Use Azure CNI networking
- `--generate-ssh-keys` - Auto-generate SSH keys for node access

#### Option B: Production-Like

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --kubernetes-version $K8S_VERSION \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --enable-azure-rbac \
  --network-plugin azure \
  --load-balancer-sku standard \
  --vm-set-type VirtualMachineScaleSets \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys
```

**Additional Features:**
- Cluster autoscaler (scale 1-3 nodes)
- Azure Monitor integration
- Standard Load Balancer
- VM Scale Sets

#### Option C: Cost-Optimized with Auto-Shutdown

```bash
# Create basic cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --location $LOCATION \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --enable-azure-rbac \
  --network-plugin azure \
  --generate-ssh-keys

# Create auto-shutdown automation (see Maintenance section)
```

---

## Configuration Options

### Network Configuration

#### Azure CNI (Recommended)

```bash
--network-plugin azure
--network-policy azure       # Optional: enable network policies
--pod-cidr 10.244.0.0/16     # Optional: custom pod CIDR
--service-cidr 10.0.0.0/16   # Optional: custom service CIDR
--dns-service-ip 10.0.0.10   # Optional: custom DNS IP
```

**Pros:**
- Pods get IPs from VNet
- Better integration with Azure
- Required for some features

**Cons:**
- Uses more IP addresses

#### Kubenet (Alternative)

```bash
--network-plugin kubenet
```

**Pros:**
- More IP-efficient
- Lower cost

**Cons:**
- Less Azure integration

### Authentication & RBAC

**Azure RBAC (Required for AKSFlexNode):**

```bash
--enable-azure-rbac          # Enable Azure RBAC for K8s
--enable-managed-identity    # Use Managed Identity
```

### Node Pool Configuration

#### System Node Pool (Created Automatically)

```bash
--node-count 1               # Number of nodes
--node-vm-size Standard_B2s  # VM size
--node-osdisk-size 30        # OS disk size in GB
--max-pods 110               # Max pods per node
```

#### Add User Node Pool (Optional)

```bash
# Add a second node pool for workloads
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name userpool \
  --node-count 1 \
  --node-vm-size Standard_B2s
```

### Tags and Labels

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --tags \
    environment=e2e-testing \
    project=aksflexnode \
    cost-center=engineering \
    auto-shutdown=true \
  --nodepool-labels \
    nodepool=system \
    workload=test
```

---

## Validation

### Step 1: Verify Cluster Creation

```bash
# Check cluster status
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "powerState.code" -o tsv

# Should output: Running
```

### Step 2: Test kubectl Access

```bash
# Get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing

# Check connection
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces
```

### Step 3: Verify Azure RBAC

```bash
# Check if Azure RBAC is enabled
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "aadProfile.enableAzureRbac" -o tsv

# Should output: true
```

### Step 4: Test Cluster Functionality

```bash
# Deploy test pod
kubectl run nginx-test --image=nginx --port=80

# Check pod status
kubectl get pods

# Expose pod
kubectl expose pod nginx-test --type=LoadBalancer --port=80

# Get external IP (may take 2-3 minutes)
kubectl get svc nginx-test --watch

# Test connection
EXTERNAL_IP=$(kubectl get svc nginx-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP

# Cleanup
kubectl delete pod nginx-test
kubectl delete svc nginx-test
```

### Step 5: Verify Managed Identity

```bash
# Get cluster managed identity
CLUSTER_IDENTITY=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query identity.principalId -o tsv)

echo "Cluster Managed Identity: $CLUSTER_IDENTITY"

# List role assignments
az role assignment list \
  --assignee $CLUSTER_IDENTITY \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  -o table
```

---

## Cost Optimization

### 1. Use B-Series Burstable VMs

```bash
# Cost-effective for testing
--node-vm-size Standard_B2s   # ~$30/month per node
```

**B-Series Comparison:**

| Size | vCPU | RAM | Cost/Month | Best For |
|------|------|-----|------------|----------|
| B1s | 1 | 1GB | ~$8 | Minimal testing |
| B2s | 2 | 4GB | ~$30 | E2E testing ⭐ |
| B2ms | 2 | 8GB | ~$60 | Heavy workloads |
| B4ms | 4 | 16GB | ~$120 | Performance testing |

### 2. Enable Cluster Auto-Stop

Create auto-shutdown script `scripts/setup/auto-shutdown-aks.sh`:

```bash
#!/bin/bash
set -euo pipefail

RESOURCE_GROUP="rg-aksflexnode-e2e-tests"
CLUSTER_NAME="aks-flexnode-e2e-cluster"

# Stop cluster (deallocate nodes)
echo "Stopping AKS cluster..."
az aks stop \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --no-wait

echo "✅ Cluster stop initiated"
echo "Nodes will be deallocated (you won't be charged for compute)"
```

Create auto-start script `scripts/setup/auto-start-aks.sh`:

```bash
#!/bin/bash
set -euo pipefail

RESOURCE_GROUP="rg-aksflexnode-e2e-tests"
CLUSTER_NAME="aks-flexnode-e2e-cluster"

# Start cluster
echo "Starting AKS cluster..."
az aks start \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --no-wait

echo "✅ Cluster start initiated"
echo "Waiting for cluster to be ready..."

az aks wait \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --custom "powerState.code=='Running'" \
  --interval 30 \
  --timeout 600

echo "✅ Cluster is running"
```

**Usage:**

```bash
# Stop cluster when not in use (e.g., evenings/weekends)
./scripts/setup/auto-shutdown-aks.sh

# Start cluster when needed
./scripts/setup/auto-start-aks.sh
```

**Savings:** ~60-70% reduction in costs if stopped during non-working hours

### 3. Use Azure Dev/Test Subscription

If you have access to Azure Dev/Test subscription pricing:
- Up to 40% discount on VMs
- Check eligibility: Visual Studio subscribers, MSDN, etc.

### 4. Schedule Automatic Start/Stop

**Option A: Azure Automation**

```bash
# Create automation account
az automation account create \
  --resource-group $RESOURCE_GROUP \
  --name "automation-aksflexnode" \
  --location $LOCATION

# Create runbooks for start/stop
# (Detailed instructions in Azure Portal)
```

**Option B: GitHub Actions Schedule**

```yaml
name: AKS Cluster Management

on:
  schedule:
    - cron: '0 0 * * 1-5'   # Stop at midnight Mon-Fri
    - cron: '0 8 * * 1-5'   # Start at 8 AM Mon-Fri

jobs:
  manage-cluster:
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Stop or Start Cluster
        run: |
          HOUR=$(date +%H)
          if [ $HOUR -eq 0 ]; then
            az aks stop -g ${{ secrets.E2E_RESOURCE_GROUP }} -n ${{ secrets.E2E_AKS_CLUSTER_NAME }}
          else
            az aks start -g ${{ secrets.E2E_RESOURCE_GROUP }} -n ${{ secrets.E2E_AKS_CLUSTER_NAME }}
          fi
```

### 5. Monitor Costs

```bash
# Enable cost analysis
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --enable-cost-analysis

# View costs in Azure Portal:
# Cost Management + Billing → Cost analysis → Group by: Resource
```

**Set up budget alert:**

```bash
# Create budget for cluster resource group
az consumption budget create \
  --amount 100 \
  --budget-name "aksflexnode-e2e-budget" \
  --resource-group $RESOURCE_GROUP \
  --time-grain Monthly \
  --start-date $(date +%Y-%m-01) \
  --end-date $(date -d "+1 year" +%Y-%m-01)
```

---

## Maintenance

### Regular Operations

#### Update Cluster

```bash
# Check available upgrades
az aks get-upgrades \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  -o table

# Upgrade cluster
az aks upgrade \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --kubernetes-version 1.28.5
```

#### Scale Nodes

```bash
# Scale up
az aks scale \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 2

# Scale down
az aks scale \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 1
```

#### View Cluster Logs

```bash
# Enable diagnostic settings (one-time)
az monitor diagnostic-settings create \
  --resource $CLUSTER_RESOURCE_ID \
  --name "aks-diagnostics" \
  --logs '[{"category":"kube-apiserver","enabled":true}]' \
  --workspace $LOG_ANALYTICS_WORKSPACE_ID

# View logs in Azure Portal:
# AKS Cluster → Logs → Run queries
```

### Backup and Restore

```bash
# Backup cluster configuration
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  > aks-cluster-backup.json

# Export kubectl resources
kubectl get all --all-namespaces -o yaml > k8s-resources-backup.yaml
```

---

## Troubleshooting

### Issue 1: Cluster Creation Fails

**Symptoms:**
```
ERROR: Operation failed with status: 'Bad Request'
```

**Solutions:**

```bash
# Check quota limits
az vm list-usage --location $LOCATION -o table

# Check if name is already taken
az aks list --query "[?name=='$CLUSTER_NAME']" -o table

# Verify service principal permissions
az role assignment list --assignee $(az account show --query user.name -o tsv)
```

### Issue 2: Cannot Connect with kubectl

**Symptoms:**
```
Unable to connect to the server: dial tcp: lookup xxx on xxx:53: no such host
```

**Solutions:**

```bash
# Re-get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --overwrite-existing \
  --admin  # Use admin credentials

# Check cluster is running
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query powerState

# Verify network connectivity
FQDN=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query fqdn -o tsv)
ping $FQDN
```

### Issue 3: Nodes Not Ready

**Symptoms:**
```
NAME                                STATUS     ROLES   AGE   VERSION
aks-nodepool1-12345678-vmss000000  NotReady   agent   5m    v1.28.5
```

**Solutions:**

```bash
# Check node status
kubectl describe node <node-name>

# Check system pods
kubectl get pods -n kube-system

# Check node logs
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  -o table

# Restart node (last resort)
az vmss restart \
  --resource-group MC_${RESOURCE_GROUP}_${CLUSTER_NAME}_${LOCATION} \
  --name <vmss-name> \
  --instance-ids 0
```

### Issue 4: Azure RBAC Not Working

**Symptoms:**
```
Error from server (Forbidden): nodes is forbidden: User cannot list resource "nodes"
```

**Solutions:**

```bash
# Verify Azure RBAC is enabled
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "aadProfile.enableAzureRbac"

# Grant yourself cluster admin role
CLUSTER_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query id -o tsv)
USER_ID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --assignee $USER_ID \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --scope $CLUSTER_ID

# Wait 5-10 minutes for propagation
```

### Issue 5: High Costs

**Symptoms:**
Unexpected Azure bills

**Solutions:**

```bash
# Check cluster state
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query "powerState.code"

# Stop cluster if not needed
az aks stop \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME

# Check for orphaned resources
az resource list \
  --resource-group $RESOURCE_GROUP \
  --query "[].{Name:name, Type:type}" \
  -o table

# Review load balancer costs (often forgotten)
az network lb list \
  --resource-group MC_${RESOURCE_GROUP}_${CLUSTER_NAME}_${LOCATION} \
  -o table
```

---

## Delete Cluster

### When to Delete

- ✅ Cluster no longer needed
- ✅ Switching to new configuration
- ✅ Testing completed

### Soft Delete (Stop Cluster)

```bash
# Stop cluster (keeps configuration, low cost)
az aks stop \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME

# Cost: Only storage (~$5/month)
# Restart anytime with: az aks start
```

### Hard Delete (Remove Everything)

```bash
# Delete cluster only
az aks delete \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --yes \
  --no-wait

# Delete entire resource group (cluster + all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait
```

**Warning:** This is permanent and cannot be undone!

---

## Summary

### Quick Reference Card

```bash
# Create cluster
az aks create -g rg-aksflexnode-e2e-tests -n aks-flexnode-e2e-cluster \
  --node-count 1 --node-vm-size Standard_B2s \
  --enable-managed-identity --enable-azure-rbac \
  --network-plugin azure --generate-ssh-keys

# Get credentials
az aks get-credentials -g rg-aksflexnode-e2e-tests -n aks-flexnode-e2e-cluster

# Verify
kubectl get nodes

# Stop cluster (save costs)
az aks stop -g rg-aksflexnode-e2e-tests -n aks-flexnode-e2e-cluster

# Start cluster
az aks start -g rg-aksflexnode-e2e-tests -n aks-flexnode-e2e-cluster

# Delete cluster
az aks delete -g rg-aksflexnode-e2e-tests -n aks-flexnode-e2e-cluster --yes
```

### Recommended Configuration for AKSFlexNode E2E

```bash
Resource Group:    rg-aksflexnode-e2e-tests
Cluster Name:      aks-flexnode-e2e-cluster
Location:          westus (or closest to you)
Node Count:        1
Node VM Size:      Standard_B2s
Kubernetes:        Latest stable
Network Plugin:    Azure CNI
RBAC:              Azure RBAC enabled
Managed Identity:  Enabled

Estimated Cost:    ~$30-40/month (running 24/7)
                   ~$10-15/month (with auto-stop during off-hours)
```

---

## Next Steps

After creating your AKS cluster:

1. **Save cluster information** for GitHub Secrets:
   ```bash
   E2E_RESOURCE_GROUP=rg-aksflexnode-e2e-tests
   E2E_AKS_CLUSTER_NAME=aks-flexnode-e2e-cluster
   E2E_AKS_RESOURCE_ID=/subscriptions/.../aks-flexnode-e2e-cluster
   E2E_LOCATION=westus
   ```

2. **Set up authentication** (see `docs/GITHUB_AZURE_AUTH_GUIDE.md`):
   - Create service principal with OIDC
   - Assign proper RBAC roles
   - Configure GitHub secrets

3. **Implement E2E pipeline** (see `docs/E2E_CICD_GUIDE.md`):
   - Create workflow files
   - Test VM provisioning
   - Validate end-to-end flow

4. **Configure cost optimization**:
   - Set up auto-stop/start schedule
   - Enable cost alerts
   - Monitor usage

---

## Additional Resources

- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
